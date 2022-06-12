#!groovy
pipeline {
    agent {
        //To run the agent jenkins slave
        docker {
            //docker pull eclipse-temurin:8u332-b09-jdk-alpine
            image 'eclipse-temurin:8u332-b09-jdk-alpine'
            args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
        }
    }

    environment {
        ORG_NAME="eclipse-temurin"
        APP_NAME = 'myproject'
        APP_VERSION = '0.0.1-SNAPSHOT'
        APP_CONTEXT_ROOT = '/'
        APP_LISTENING_PORT = '8080'
        TEST_CONTAINER_NAME = "ci-${APP_NAME}-${BUILD_NUMBER}"
        DOCKER_HUB = credentials("${ORG_NAME}-docker-hub")
    }

    stages {
        stage('Code Checkout') {
        git(
          url:"git@github.com:natarajk49/hello-world-spring-boot.git",
          credentialsId: 'titaniumTest',
          branch: "master"
          )
      }

        stage('Compile') {
            steps {
                echo '-=- compiling project -=-'
                sh './mvnw clean compile'
            }
        }

        stage('Unit tests') {
            steps {
                echo '-=- execute unit tests -=-'
                sh './mvnw test org.jacoco:jacoco-maven-plugin:report'
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
            }
        }

        stage('Mutation tests') {
            steps {
                echo '-=- execute mutation tests -=-'
                sh './mvnw org.pitest:pitest-maven:mutationCoverage'
            }
        }

        stage('Package') {
            steps {
                echo '-=- packaging project -=-'
                sh './mvnw package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build Docker image') {
            steps {
                echo '-=- build Docker image -=-'
                sh "docker build -t ${ORG_NAME}/${APP_NAME}:${APP_VERSION} -t ${ORG_NAME}/${APP_NAME}:latest ."
            }
        }

        stage('Run Docker image') {
            steps {
                echo '-=- run Docker image -=-'
                sh "docker run --name ${TEST_CONTAINER_NAME} --detach --rm --network ci --expose 8080 ${ORG_NAME}/${APP_NAME}:latest"
            }
        }

       
        stage('Dependency vulnerability scan') {
            steps {
                echo '-=- run dependency vulnerability tests -=-'
                sh './mvnw dependency-check:check'
                dependencyCheckPublisher(
                    failedTotalCritical: qualityGates.security.dependencies.critical.failed,
                    unstableTotalCritical: qualityGates.security.dependencies.critical.unstable,
                    failedTotalHigh: qualityGates.security.dependencies.high.failed,
                    unstableTotalHigh: qualityGates.security.dependencies.high.unstable,
                    failedTotalMedium: qualityGates.security.dependencies.medium.failed,
                    unstableTotalMedium: qualityGates.security.dependencies.medium.unstable)
                script {
                    if (currentBuild.result == 'FAILURE') {
                        error('Dependency vulnerabilities exceed the configured threshold')
                    }
                }
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                echo '-=- run code inspection & check quality gate -=-'
                withSonarQubeEnv('ci-sonarqube') {
                    sh './mvnw sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Push Docker image') {
            steps {
                echo '-=- push Docker image -=-'
                withDockerRegistry([ credentialsId: "${ORG_NAME}-docker-hub", url: "" ]) {
                    sh "docker tag ${ORG_NAME}/${APP_NAME}:${APP_VERSION} ${ORG_NAME}/${APP_NAME}:latest"
                    sh "docker push ${ORG_NAME}/${APP_NAME}:${APP_VERSION}"
                    sh "docker push ${ORG_NAME}/${APP_NAME}:latest"
                }
            }
        }
        //Implement a script in your language of choice for the following:
        //Pull the above docker image on the host
        stage('Pull Docker image') {
            steps {
                echo '-=- push Docker image -=-'
                withDockerRegistry([ credentialsId: "${ORG_NAME}-docker-hub", url: "" ]) {
                    sh "docker pull ${ORG_NAME}/${APP_NAME}:${APP_VERSION}"
                    sh "docker pull ${ORG_NAME}/${APP_NAME}:latest"
                }
            }
        }
        //Shut down container, if running
        stage('stop Docker container') {
            steps {
                    echo '-=- stop Docker image -=-'
                    sh "docker ps"
                    sh "docker stop ${TEST_CONTAINER_NAME}"
                }   
        }
        //Restart with the version of the image downloaded in step above
        stage('restart Docker container') {
            steps {
                    echo '-=- restart Docker image -=-'
                    sh "docker ps -a"
                    sh "docker restart ${TEST_CONTAINER_NAME}"
                }   
        }
        //Verify that the container started successfully
        stage('verify Docker container') {
            steps {
                    echo '-=- verify Docker container running -=-'
                    sh "sudo systemctl status docker"
                    sh "docker ps | grep ${TEST_CONTAINER_NAME}"
                    sh "docker exec -ti ${TEST_CONTAINER_NAME} sh -c "echo a && echo b""
                }   
        }
        //Verify application is running successfully
        stage("Application Availability"){
            status = sh (
            script: "curl -iks http://localhost:8080 | head -n 1",
            returnStdout: true
            ) .trim()
            if(status == "HTTP/1.1 200 OK") {
                echo "${env.JOB_NAME} application status : ${status}"
                print "application is running successfully"
                }
            else {
                echo "${env.JOB_NAME} application status : ${status}"
                print "application is unavailable"
                currentBuild.result="FAILURE"  
        }
      }
    }

    post {
        always {
            echo '-=- stop deployment -=-'
            sh "docker stop ${TEST_CONTAINER_NAME}"
        }
    }
}
