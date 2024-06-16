def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]
pipeline {
    agent any
    environment {
        SCANNER_HOME=tool 'SonarScanner'
        SNYK_HOME   = tool name: 'Snyk'
    }
    tools {
        snyk 'Snyk'
    }
    stages {
        // Checkout To The Service Branch
        stage('Checkout To Mcroservice Branch'){
            steps{
                git branch: 'app-email-service', url: 'https://github.com/awanmbandi/realworld-microservice-project.git'
            }
        }
        // SonarQube SAST Code Analysis
        stage("SonarQube SAST Analysis"){
            steps{
                withSonarQubeEnv('SonarScanner') {
                    sh ''' $SCANNER_HOME/bin/sonarscanner -Dsonar.projectName=app-email-service \
                    -Dsonar.projectKey=app-email-service '''
                }
            }
        }
        // Providing Snyk Access
        stage('Authenticate & Authorize Snyk') {
            steps {
                withCredentials([string(credentialsId: 'Snyk-API-Token', variable: 'SNYK_TOKEN')]) {
                    sh "${SNYK_HOME}/snyk-linux auth $SNYK_TOKEN"
                }
            }
        }
        // Scan Service Dockerfile With Open Policy Agent (OPA)
        stage('OPA Dockerfile Vulnerability Scan') {
            steps {
                sh "docker run --rm -v ${WORKSPACE}:/project openpolicyagent/conftest test --policy docker-opa-security.rego Dockerfile || true"
            }
        }
        // Build and Tag Service Docker Image
        stage('Build & Tag Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker build -t sylvanus2022/emailservice:latest ."
                    }
                }
            }
        }
        // Execute SCA/Dependency Test on Service Docker Image
        stage('Snyk SCA Test | Dependencies') {
            steps {
                sh "${SNYK_HOME}/snyk-linux test --docker sylvanus2022/emailservice:latest || true" 
            }
        }
        // Push Service Image to DockerHub
        stage('Push Microservice Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'DockerHub-Credential', toolName: 'docker') {
                        sh "docker push sylvanus2022/emailservice:latest "
                    }
                }
            }
        }
        // // Deploy to The Staging/Test Environment
        // stage('Deploy Microservice To The Stage/Test Env'){
        //     steps{
        //         script{
        //             withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'Kubernetes-Credential', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
        //                sh 'kubectl apply -f deploy-envs/test-env/test-namespace.yaml'
        //                sh 'kubectl apply -f deploy-envs/test-env/deployment.yaml'
        //                sh 'kubectl apply -f deploy-envs/test-env/service.yaml'  //ClusterIP Service
        //            }
        //         }
        //     }
        // }
        // // Production Deployment Approval
        // stage('Approve Prod Deployment') {
        //     steps {
        //             input('Do you want to proceed?')
        //     }
        // }
        // // // Deploy to The Production Environment
        // stage('Deploy Microservice To The Prod Env'){
        //     steps{
        //         script{
        //             withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'Kubernetes-Credential', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
        //                sh 'kubectl apply -f deploy-envs/prod-env/prod-namespace.yaml'
        //                sh 'kubectl apply -f deploy-envs/prod-env/deployment.yaml'
        //                sh 'kubectl apply -f deploy-envs/prod-env/service.yaml'  //ClusterIP Service
        //             }
        //         }
        //     }
        // }
    }
    post {
    always {
        echo 'Slack Notifications.'
        slackSend channel: 'szm2-multi-microservices-alerts', //update and provide your channel name
        color: COLOR_MAP[currentBuild.currentResult],
        message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info at: ${env.BUILD_URL}"
    }
  }
}
