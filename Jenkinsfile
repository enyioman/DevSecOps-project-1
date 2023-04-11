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
                        -Dsonar.host.url=http://54.91.40.119:9000 \
                        -Dsonar.login=sqp_c1821f49c254cb0756b6a5d3bdb484a92b77f6ea'
                }
            }
        }       
        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: true
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
   
        stage('List Files in Workspace') {
            steps {
                sh 'ls -R'
            }
        }
        stage('Copy to S3') {
            steps {
                script {
                    withAWS(region:'us-east-1', credentials:'aws-credentials') {
                        s3Upload(bucket: 'devsecops-jenkins-logs', pathStyleAccessEnabled: true, payloadSigningEnabled: true, workingDir: '.', file: 'report.html')
                    }
                }
            }
        }
        stage('Docker  Push') {
            steps {
                withVault(configuration: [skipSslVerification: true, timeout: 60, vaultCredentialId: 'vault-cred', vaultUrl: 'http://34.228.188.132:8200/'], vaultSecrets: [[path: 'secrets/creds/docker', secretValues: [[vaultKey: 'username'], [vaultKey: 'password']]]]) {
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