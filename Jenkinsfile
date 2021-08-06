// Define paramerter
def scmVars
// Uses Declarative syntax to run commands inside a container.

pipeline {
    agent {

        // user kubernetes as dynamic slave jenkins
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:20.10.3-dind
      command:
      - dockerd
      tty: true
      securityContext:
        privileged: true
    - name: helm
      image: lachlanevenson/k8s-helm:v3.6.0
      command:
      - cat
      tty: true
    - name: skan
      image: alcide/skan:v0.9.0-debug
      command:
      - cat
      tty: true
    - name: gradle
      image: gradle:6.0.1-jdk11
      command:
      - cat
      tty: true   
    - name: java-node
      image: timbru31/java-node:11-alpine-jre-14
      command:
      - cat
      tty: true
      volumeMounts:
      - mountPath: /home/jenkins/dependency-check-data
        name: dependency-check-data
  volumes:
  - name: dependency-check-data
    hostPath:
      path: /tmp/dependency-check-data


"""

    } // End kubernetes 
  } // End agent
  
  environment {
    ENV_NAME = "${BRANCH_NAME == "master" ? "uat" : "${BRANCH_NAME}"}"
    SCANNER_HOME = tool 'sonar-scanner-jenkins'
    PROJECT_KEY = "bookinfo-reviews"
    PROJECT_NAME = "bookinfo-reviews"
  }

//Start pipeline
  stages {
      // Clones the repository 
    stage('Clone source code') {
      steps {
        container('jnlp'){
          script {
            scmVars = git branch: '${BRANCH_NAME}',
                          credentialsId: 'bookinfo-reviews-git-deploy-key',
                          url: 'git@github.com:mercurial963/bookinfo-reviews.git'
                  }// end script
              }// end container
          }// end steps
      }// end stage
    stage('Sonarqube Scanner') {
      steps {
        container('java-node'){
          script {
            withSonarQubeEnv('Sonarqube-bookinfo'){

              sh '''${SCANNER_HOME}/bin/sonar-scanner \
              -D sonar.projectKey=${PROJECT_KEY} \
              -D sonar.projectName=${PROJECT_NAME} \
              -D sonar.projectVersion=${BRANCH_NAME}-${BUILD_NUMBER} \
              -D sonar.source=./src/main/java,./src/main/webapp'''
            } // end withSonarQubeEnv

            timeout(time: 1, unit: 'MINUTES') {//Just in case something goes wrong,
              def qg = waitForQualityGate() //Reuse TaskID
              if (qg.status != 'OK'){
                error = "Pipeline aborted due to quality gate failure: ${qg.status}"
              }
            } // end timeout
                  }// end script
              }// end container
          }// end steps
      }// end stage

      // ********** Stage sKan ********** 
    stage('sKan') {
      steps {
        container('helm'){
          script {
            // Generate K8s-manifest-deploy.yaml for scanning
            sh "helm template -f helm-values/values-bookinfo-${ENV_NAME}-reviews.yaml \
            --set extraEnv.COMMIT_ID=${scmVars.GIT_COMMIT} \
            --namespace ${ENV_NAME} bookinfo-${ENV_NAME}-reviews helm/ \
            > k8s-manifest-deploy.yaml"  
                  }// end script
              }// end container
         container('skan'){
          script {
            // Scanning with sKan
            sh "/skan manifest -f k8s-manifest-deploy.yaml"
            // Kepp report as artifacts
            archiveArtifacts artifacts: 'skan-result.html'  
            sh "rm k8s-manifest-deploy.yaml"
                  }// end script
              }// end container             
          }// end steps
      }// end stage  

    stage('OWASP Dependency Check') {

      steps {
        container('java-node'){
          script {
             // Install application dependency
            sh ''' cd src/ && npm install --package-lock && cd ../'''

            // Start OPASP Dependency Check
            dependencyCheck(
              additionalArguments: "--data /home/jenkins/dependency-check-data --out dependency-check-report.xml" ,
              odcInstallation: "dependency-check"
            )

            // Publish report to Jenkins
            dependencyCheckPublisher(
              pattern: 'dependency-check-report.xml'
            )

             // Remove application dependency
            sh '''rm -rf src/node_modules src/package-lock.json'''


                  }// end script
              }// end container
          }// end steps
      }// end stage

    //   // Build image Dockerfile and push 
    stage('Build and Push') {

      steps {
        container('docker'){
          script {
            docker.withRegistry('https://ghcr.io', 'registry-bookinfo'){ //registry-bookinfo is user with token
                          // build and push
              docker.build('ghcr.io/mercurial963/bookinfo-reviews:${ENV_NAME}').push()
              }// end docker.withRegistry

                  }// end script
              }// end container
          }// end steps
      }// end stage

    stage('Anchore Engine') {

      steps {
        container('jnlp'){
          script {
                   // Send Docker image to Anchor Analyzer
            // def image = 'ghcr.io/mercurial963/bookinfo-reviews:${ENV_NAME}'
                         
            writeFile file: 'anchore_images', text: "ghcr.io/mercurial963/bookinfo-reviews:${ENV_NAME}"
            anchore name: 'anchore_images', bailOnFail: false
            // anchore_name: 'anchore_images', bailOnFail: false

                  }// end script
              }// end container
          }// end steps
      }// end stage


      // Deploy
    stage('Deploy reviews with Helm Chart') {
      steps {
        container('helm'){
          script {
            // withKubeConfig([credentialsId: 'config']){ //add kubeconfig to secret file
              sh "helm upgrade --install -f helm-values/values-bookinfo-${ENV_NAME}-reviews.yaml --wait \
              --set extraEnv.COMMIT_ID=${scmVars.GIT_COMMIT} \
              --namespace ${ENV_NAME} bookinfo-${ENV_NAME}-reviews helm/"
              // }// withCredentials

                  }// end script
              }// end container
          }// end steps
      }// end stage          
  }// end stages
  }





