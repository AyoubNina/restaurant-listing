pipeline {
  agent any

  environment {
    DOCKERHUB_CREDENTIALS = credentials('DOCKER_HUB_CREDENTIAL')
    VERSION = "${env.BUILD_ID}"

  }
//it's our external tool 
  tools {
    maven "Maven"
  }

  stages {

    stage('Maven Build'){
        steps{ //we skipped the test step her coz we want just to check if there's no compilation error 
          //and just the build is successful or not
        sh 'mvn clean package  -DskipTests'
        }
    }

     stage('Run Tests') {
      steps {
        sh 'mvn test'
      }
    }
//execute sonarQube
    stage('SonarQube Analysis') {
  steps {
    sh 'mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.host.url=http://35.180.137.8:9000/ -Dsonar.login=squ_32789bcdadb6e4337e432d6cbc100c2a1a14fde5'
  }
}


   stage('Check code coverage') {
            steps {
                script {
                    def token = "squ_32789bcdadb6e4337e432d6cbc100c2a1a14fde5"
                    def sonarQubeUrl = "http://35.180.137.8:9000/api"
                    def componentKey = "com.sesame:restaurantlisting" //combination of group id and artifactid
                    def coverageThreshold = 80.0

                    def response = sh (
                        script: "curl -H 'Authorization: Bearer ${token}' '${sonarQubeUrl}/measures/component?component=${componentKey}&metricKeys=coverage'",
                        returnStdout: true
                    ).trim()

                    def coverage = sh (
                        script: "echo '${response}' | jq -r '.component.measures[0].value'",
                        returnStdout: true
                    ).trim().toDouble()

                    echo "Coverage: ${coverage}"

                    if (coverage < coverageThreshold) {
                        error "Coverage is below the threshold of ${coverageThreshold}%. Aborting the pipeline."
                    }
                }
            }
        } 
//use dockerhub credential to log in dockerhub ,then we are going to build this image..

      stage('Docker Build and Push') {
      steps {
          sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
          sh 'docker build -t jiheneayoub1/restaurant-listing-service:${VERSION} .'
          sh 'docker push jiheneayoub1/restaurant-listing-service:${VERSION}'
      }
    } 
//once our docker image is pushed and we clean the directory

     stage('Cleanup Workspace') {
      steps {
        deleteDir()
       
      }
    }

//from Jenkins when you updatr the version image and push it, the tag will be 'update image tag'
    // in the manifest file. yml (deployment-folder/aws/restaurant-manifest.yml), the version of the image will 
    //change and be same version as jenkins
    //!Bref, Jenkins does maven build, test, run sonar,code coverage,ckeck docker and build image
    //and Push this image to docker hub and after that it updates in the deployment folder!

    stage('Update Image Tag in GitOps') {
      steps {
         checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[ credentialsId: 'git-ssh', url: 'git@github.com:AyoubNina/deployment-folder.git']])
        script {
       sh '''
          sed -i "s/image:.*/image: jiheneayoub1\\/restaurant-listing-service:${VERSION}/" aws/restaurant-manifest.yml
        '''
          sh 'git checkout master'
          sh 'git add .'
          sh 'git commit -m "Update image tag"'
        sshagent(['git-ssh'])
            {
                  sh('git push')
            }//!!!THE NEW image after getting pushed ,will be PULLED by ArgoCD and DEPLOYED in our cluster!!
        }
      }
    }

  }

}


