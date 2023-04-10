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
                        -Dsonar.host.url=http://54.162.28.43:9000 \
                        -Dsonar.login=sqp_678c2cafbba2cc11a9a2fbdbc3c843d26c4652f2
                }
            }
        }
        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        
        stage('Docker  Build') {
            steps {
      	        sh 'docker build -t fynewily/sprint-boot-app:v1.$BUILD_ID .'
                sh 'docker image tag fynewily/sprint-boot-app:v1.$BUILD_ID fynewily/sprint-boot-app:latest'
            }
        }
        stage('Image Scan') {
            steps {
      	        sh ' trivy image --format template --template "@/usr/local/share/trivy/templates/html.tpl" -o report.html fynewily/sprint-boot-app:latest '
            }
        }
        stage('Upload Scan report to AWS S3') {
              steps {
                  sh 'aws s3 cp report.html s3://devsecops-jenkins-logs/'
              }
         }
        stage('Docker  Push') {
            steps {
                withVault(configuration: [skipSslVerification: true, timeout: 60, vaultCredentialId: 'vault-cred', vaultUrl: 'http://your-vault-server-ip:8200'], vaultSecrets: [[path: 'secrets/creds/docker', secretValues: [[vaultKey: 'username'], [vaultKey: 'password']]]]) {
                    sh "docker login -u ${username} -p ${password} "
                    sh 'docker push fynewily/sprint-boot-app:v1.$BUILD_ID'
                    sh 'docker push fynewily/sprint-boot-app:latest'
                    sh 'docker rmi fynewily/sprint-boot-app:v1.$BUILD_ID fynewily/sprint-boot-app:latest'
                }
            }
        }
        stage('Deploy to k8s') {
            steps {
                script{
                    kubernetesDeploy configs: 'spring-boot-deployment.yaml', kubeconfigId: 'kubernetes'
                }
            }
        }
        
 
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
