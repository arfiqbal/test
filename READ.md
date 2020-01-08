

Hacks by Ruddra
Writing Jenkins Pipeline For Openshift Deployment
Aug 11, 2018 · 9 Min Read · 0 Comment
Writing Jenkins Pipeline For Openshift Deployment

Pipeline is a set of instructions, which will be executed as per given sequence and produce a output. Jenkins Pipeline is simply writing these instructions in Jenkins. These pipelines can be written in Groovy.

In previous post we have defined deployment structure and made necessary preparations for deployment of a python+gunicorn+nginx+jenkins based project. In this post, we will discuss about the most important part of our CI/CD project, writing jenkins pipeline. This pipeline will provide continuous delivery(CD) for our deployment.
Plan for Pipeline

Our plan is to execute the following stages in serial order:

    Get Latest Code
    Install Dependencies
    Run Tests(and store them)
    Store Artifacts
    Create Build Configuration in Openshift(DEV PROJECT)
    Build in Openshift(DEV PROJECT)
    Create Deploy Configuration in Openshift(DEV PROJECT)
    Promote the image(to STAGE PROJECT)
    Deploy to Openshift(STAGE PROJECT)
    Scale Up(in STAGE PROJECT)

Stage 1-4 will be executed in Jenkins.Stage 5-10 will be executed in Openshift. If you are feeling confused about these stages, don't worry. We will be explaining them one by one. Please keep in mind, with every stage there is a description of possible outcome of the execution of that stage. But in this article, our task is only to create a pipeline, not execute it.
Steps
Step 0: Declaring Agent and ENV Variables

Stage Zero? Yep, this step is very important for writing pipeline. We will declare agent and environment variables in this step. In Jenkins, we need to specify a agent, in which the pipeline will be executing. We will be using python jenkins slave to execute our pipeline. We will discuss more about jenkins python slave in next article. The python agent should look like this:

pipeline {
    agent {
      node {label 'python'}
    }
}

We can write down our constant values in environment variables. For example:

environment {
    APPLICATION_NAME = 'python-nginx'
    GIT_REPO="http://github.com/ruddra/openshift-python-nginx.git"
    GIT_BRANCH="master"
    STAGE_TAG = "promoteToQA"
    DEV_PROJECT = "dev"
    STAGE_PROJECT = "stage"
    TEMPLATE_NAME = "python-nginx"
    ARTIFACT_FOLDER = "target"
    PORT = 8081;
}

Some variable declaration may seem confusing, but we will use them in next steps.
Step 1: Get Latest Code

Jenkins Pipeline execution is done in stages. All the stages of pipeline will be inside in one stages dictionary. Like:

stages {
    stage("One"){
    // Do Something
    }
    stage("Two"){
    // Do Something
    }
}

In first stage, we will be using Git Plugin which comes as default with Openshift Jenkins. We will be using following code for pulling the latest code:

stage('Get Latest Code') {
  steps {
    git branch: "${GIT_BRANCH}", url: "${GIT_REPO}" // declared in environment
  }
}

Step 2: Install Dependencies

In this stage, we will install python dependencies inside in a virtual environment. For that, we will first install virtualenv using pip install virtualenv. After activating the virtualenv, we will install the dependencies using pip install -r requirements.pip from dependencies defined in app>requirements.pip.

stage("Install Dependencies") {
    steps {
        sh """
        pip install virtualenv
        virtualenv --no-site-packages .
        source bin/activate
        pip install -r app/requirements.pip
        deactivate
        """
    }
}

Here we are using steps to execute the commands for installing dependencies.
Step 3: Run Tests

In this stage, we will run tests, so that we can make sure if any test fails, pipeline does not execute any further. We will also store the test results using JUnit Plugin.

First, we will activate our virtualenv(again!!) then run tests in app directory using nosetests. It will export test results in xml format. Then we will store the result using JUnit.

stage('Run Tests') {
    steps {
        sh '''
        source bin/activate
        nosetests app --with-xunit
        deactivate
        '''
        junit "nosetests.xml"
    }
}

Step 4: Storing Artifacts

Artifact may sound weird to you if you are familliar with Java. Because we are working with Python, not Java builds. Python does not require any builds, still we are storing a compressed file consists of Python codes as well as our Dockerfile and NGINX configurations(app,config,Dockerfile), and we are going to use this compressed file to deploy our application to openshift. You might think that storing that file is not necessary, but I feel that storing that might be necessary, so that in later you can find out what is being pushed to openshift or if there is any discrepancy between what you want to deploy and what you are deploying. Anyways, for this stage, we will make safe name for naming our compressed file. BUILD_NUMBER is an environment variable available in pipeline, which provides current build number which should be unique per build. We will be using APPLICATION_NAME + BUILD_NUMBER to make a safe build name. We will store the Artifacts in a special directory, for now lets use target folder inside workspace(Workspace is basically the place/path where the whole pipeline execution is happening).

stage('Store Artifact'){
  steps{
      script{
        def safeBuildName  = "${APPLICATION_NAME}_${BUILD_NUMBER}",
            artifactFolder = "${ARTIFACT_FOLDER}",
            fullFileName   = "${safeBuildName}.tar.gz",
            applicationZip = "${artifactFolder}/${fullFileName}"
            applicationDir = ["app",
                              "config",
                              "Dockerfile",
                              ].join(" ");
        def needTargetPath = !fileExists("${artifactFolder}")
        if (needTargetPath) {
            sh "mkdir ${artifactFolder}"
        }
        sh "tar -czvf ${applicationZip} ${applicationDir}"
        archiveArtifacts artifacts: "${applicationZip}", excludes: null,               onlyIfSuccessful: true
      }
  }
}

We will be using script to execute our instructions. Script console is basically for running arbitrary commands inside it.
Step 5: Create Build Configuration in Openshift

After Step 4, we will have a nice compressed file containing what we need for building our app in openshift. Before start building the app, we need to have a Build Configuration inside Openshift App. Basically Build Configuration is sort of a skeleton which contains instructions on how your source will be build. There are many strategies for build config. We will be using Docker strategy for building our code. That is why we have put the Dockerfile inside our compressed file. In build config, you need to provide from which source you want to build your code from. We will be using binary. Meaning we will provide a compressed file to openshift so that it can start building the image from it.

stage('Create Image Builder') {
    when {
        expression {
            openshift.withCluster() {
            openshift.withProject(DEV_PROJECT) {
                return !openshift.selector("bc", "${TEMPLATE_NAME}").exists();
                }
            }
        }
    }
    steps {
        script {
            openshift.withCluster() {
                openshift.withProject(DEV_PROJECT) {
                    openshift.newBuild("--name=${TEMPLATE_NAME}", "--docker-image=docker.io/nginx:mainline-alpine", "--binary=true")
                }
            }
        }
    }
}

This stage of pipeline will only execute once when the pipeline is executed for first time. Later when pipeline is executed for second time, third time… and so on, this stage will be skipped. Because build configuration is made only once. You can check if a build configuration is being created using oc get bc in terminal (in project dev) from your local machine. You can also check it using web interface of openshift in CI/CD projects > Builds. openshift.withCluster and openshift.withProject is basically openshift APIs open to Jenkins plugin. Using these APIs, you can execute commands inside openshift cluster and projects. Our new-build command is being executed inside DEV_PROJECT. Value of DEV_PROJECT is defined in environment variable, and the value is dev.

NB: Build Config, Deployment Config, Service, Image, Storage etc are Kubernetes basics. These terms will come more frequently in next stages. If you have some idea about them, it might help you to understand this tutorial even more. :)
Step 6: Build Image

Outcome of a successful build is an Image. So after a successful build, you will see an image inside Openshift in web interface(at Openshift_Url > console > DEV Project > browse > images). The most recent build outcomes to image tagged by latest. This is an important information, which will come handy in our next steps. In our build configuration(defined in last step), we configured that the build will executed from binary. Meaning if will expect a archive file to start the build process. As we have a archive file from step 4, we will use that file for this purpose.

stage('Build Image') {
  steps {
      script {
          openshift.withCluster() {
          openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("bc", "$TEMPLATE_NAME").startBuild("--from-archive=${ARTIFACT_FOLDER}/${APPLICATION_NAME}_${BUILD_NUMBER}.tar.gz", "--wait=true")
              }
          }
      }
  }
}

openshift.selector selects the build config named python-nginx (this value is set in ${TEMPLATE_NAME}). Using that build config, it initiates a build from the tar.gz file. After the build is complete, you will see an image named python-nginx and it should be tagged as latest.
Step 7: Deploy to Openshift in DEV

In this stage, we will deploy the image to DEV_PROJECT. Basically a Deployment Configuration is created in this step. Deployment Configuration is kind of like Build Config, but this is related to deployment. We will be using new-app to create our project from image built in last stage.

stage('Deploy to DEV') {
  when {
      expression {
        openshift.withCluster() {
          openshift.withProject(env.DEV_PROJECT) {
            return !openshift.selector('dc', "${TEMPLATE_NAME}").exists()
            }
        }
    }
  }
  steps {
    script {
        openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
                def app = openshift.newApp("${TEMPLATE_NAME}:latest")
                app.narrow("svc").expose("--port=${PORT}");
                def dc = openshift.selector("dc", "${TEMPLATE_NAME}")
                while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                    sleep 10
                  }
              }
          }
      }
  }
}

This step is executed only in first time execution of pipeline. In next pipeline builds, this stage is skipped. But still, a deployment will be automatically done after each build is completed successfully. Because of Image Change Trigger. Meaning, after each new image, an automated deployment will initiate. New deployment means, it will generate new pods and will destroy the old pods. There are two strategies for this change, rolling strategy and recreate strategy. app.narrow("svc").expose("--port=${PORT}"); this command will use the service built by new-app; then expose its route to port 8081, this value is set in Environment variable as well. You can see the new service and ports as well using oc get svc and oc get routes in dev project from your local machine.
Step 8: Promote Image to Stage

In this stage, jenkins will prompt you to promote or abort regarding if you want to continue deployment to stage project or not. If you click in promote(in Jenkins), then openshift promote the image created in step 6 to stage and tag it as promoteToQA(as per defined in environment).

stage('Promote to STAGE?') {
  steps {
      timeout(time:15, unit:'MINUTES') {
           input message: "Promote to STAGE?", ok: "Promote"
      }
      script {
          openshift.withCluster() {
          openshift.tag("${DEV_PROJECT}/${TEMPLATE_NAME}:latest", "${STAGE_PROJECT}/${TEMPLATE_NAME}:${STAGE_TAG}")
            }
        }
   }
}

Step 9: Deploy to Openshift in STAGE PROJECT

In this step, we will be looking if any previous deployment is done in STAGE PROJECT. If it exists, then we will delete service, route, deployment config of that deployment, and create a new app using the new image promoted in last step. Then will expose the route for this application in 8081 port.

stage('Rollout to STAGE') {
  steps {
      script {
            openshift.withCluster() {
              openshift.withProject(STAGE_PROJECT) {
                  if (openshift.selector('dc', '${TEMPLATE_NAME}').exists()) {
                      openshift.selector('dc', '${TEMPLATE_NAME}').delete()
                      openshift.selector('svc', '${TEMPLATE_NAME}').delete()
                       openshift.selector('route', '${TEMPLATE_NAME}').delete()
                }
            openshift.newApp("${TEMPLATE_NAME}:${STAGE_TAG}").narrow("svc").expose("--port=${PORT}")
            }
         }
      }
  }
}

Step 10: Scale in STAGE PROJECT

We will be using openshiftScale API provided by Jenkins Openshift Plugin. What it does is that, it creates replication of pods according the number of replication information provided to it via parameter replicaCount.

stage('Scale in STAGE') {
  steps {
      script {
          openshiftScale(namespace: "${STAGE_PROJECT}", deploymentConfig: "${TEMPLATE_NAME}", replicaCount: '3')
        }
    }
}

FYI: This API will be depricated from oc version 3.11 This is the last step of our pipeline. Lets save it in a file named Jenkinsfile. Please commit and push your changes to your Git repository.

The Jenkins file should look like this.
Conclusion

Lets discuss more about how to deploy our application using this pipeline in next article.

Feel free to comment if you find anything unclear. Thanks for reading. Cheers!!

Topics: Openshift , Jenkins , Pipeline
Get Better With Openshift
Subscribe to get monthly articles about Openshift and more.
We won't spam you. Unsubscribe at any time.

Older Post
Newer Post
Share Your Thoughts
M ↓   Markdown
