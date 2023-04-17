pipeline {
    agent any
    tools { 
        maven 'maven-3.9.1' 
    }
    stages {
        stage('Checkout git') {
            steps {
               git branch: 'main', url: 'https://github.com/enyioman/DevSecOps-project-1'
            }
        }
        
        stage ('Build & JUnit Test') {
            steps {
                sh 'mvn install' 
            }
            post {
               success {
                    junit 'target/surefire-reports/**/*.xml'
                }   
            }
        }
        stage('SonarQube Analysis'){
            steps{
                withSonarQubeEnv('SonarQube-server') {
                        sh 'mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=demo \
                        -Dsonar.host.url=http://54.144.110.218:9000 \
                        -Dsonar.login=sqp_c1821f49c254cb0756b6a5d3bdb484a92b77f6ea'
                }
            }
        }       
        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    try {
                        sh 'docker build -t fynewily/sprint-boot-app:v1.$BUILD_ID .'
                        sh 'docker image tag fynewily/sprint-boot-app:v1.$BUILD_ID fynewily/sprint-boot-app:latest'
                    } catch (Exception ex) {
                        currentBuild.result = 'FAILURE'
                        error("Failed to build Docker image: ${ex.getMessage()}")
                    }
                }
            }
        }
        stage('Image Scan') {
            steps {
      	        sh ' trivy image --format template --template "@/usr/local/share/trivy/templates/html.tpl" -o report.html fynewily/sprint-boot-app:latest '
            }
        }
        stage('List Files in Workspace') {
            steps {
                sh 'ls -R'
            }
        }
        stage('Copy to S3') {
            steps {
                script {
                    withAWS(region:'us-east-1', credentials:'jenkins-aws') {
                        s3Upload(bucket: 'devsecops-jenkins-logs', pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: '.', file: 'report.html')
                    }
                }
            }
        }
        stage('Docker Push') {
            steps {
                withVault(configuration: [skipSslVerification: true, timeout: 60, vaultCredentialId: 'vault-jenkins-role', vaultUrl: 'http://34.228.188.132:8200'], vaultSecrets: [[path: 'secrets/creds/docker', secretValues: [[vaultKey: 'username'], [vaultKey: 'password']]]]) {
                    sh "docker login -u ${username} -p ${password} "
                    sh 'docker push fynewily/sprint-boot-app:v1.$BUILD_ID'
                    sh 'docker push fynewily/sprint-boot-app:latest'
                    sh 'docker rmi fynewily/sprint-boot-app:v1.$BUILD_ID fynewily/sprint-boot-app:latest'
                }
            }
        }

        // stage('Docker Push') {
        //     steps {
        //         script {
        //             try {
        //                 withVault(configuration: [skipSslVerification: true, timeout: 60, vaultCredentialId: 'jenkins-docker', vaultUrl: 'http://34.228.188.132:8200'], vaultSecrets: [[path: 'secrets/creds/docker', secretValues: [[vaultKey: 'username'], [vaultKey: 'password']]]]) {
        //                     script {
        //                         def username = vaultSecrets.get("secrets/creds/docker").get("username")
        //                         def password = vaultSecrets.get("secrets/creds/docker").get("password")
        //                         echo "Docker credentials retrieved from Vault: username=${username}, password=${password}"
        //                         sh "docker login -u ${username} -p ${password}"
        //                         sh 'docker push fynewily/sprint-boot-app:v1.$BUILD_ID'
        //                         sh 'docker push fynewily/sprint-boot-app:latest'
        //                         sh 'docker rmi fynewily/sprint-boot-app:v1.$BUILD_ID fynewily/sprint-boot-app:latest'
        //                     }
        //                 }
        //             } catch (Exception e) {
        //             echo "Error retrieving Docker credentials from Vault: ${e.getMessage()}"
        //         }
        //         }
        //     }
        // }
    }
    post{
        always{
            sendSlackNotifcation()
        }
    }
}

def sendSlackNotifcation()
{
    if ( currentBuild.currentResult == "SUCCESS" ) {
        buildSummary = "Job_name: ${env.JOB_NAME}\n Build_id: ${env.BUILD_ID} \n Status: *SUCCESS*\n Build_url: ${BUILD_URL}\n Job_url: ${JOB_URL} \n"
        slackSend( channel: "#devsecops", token: 'slack-token', color: 'good', message: "${buildSummary}")
    }
    else {
        buildSummary = "Job_name: ${env.JOB_NAME}\n Build_id: ${env.BUILD_ID} \n Status: *FAILURE*\n Build_url: ${BUILD_URL}\n Job_url: ${JOB_URL}\n  \n "
        slackSend( channel: "#devsecops", token: 'slack-token', color : "danger", message: "${buildSummary}")
    }
}