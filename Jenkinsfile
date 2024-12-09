pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        BRANCH_NAME = "${env.BRANCH_NAME}"
        ECR_IMAGE_NAME = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/mfusion-ms:dev-mfusion-ms-v.1.${env.BUILD_NUMBER}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'maven_3.9.4'
        // sonarqubeScanner 'sonarqube-scanner'
    }

    stages {
        stage('Build and Test for Dev') {
            when {
                branch 'dev'
            }
            stages {
                stage('Code Compilation') {
                    steps {
                        echo 'Code Compilation is In Progress!'
                        sh 'mvn clean compile'
                        echo 'Code Compilation is Completed Successfully!'
                    }
                }

                stage('Code QA Execution') {
                    steps {
                        echo 'JUnit Test Case Check in Progress!'
                        sh 'mvn clean test'
                        echo 'JUnit Test Case Check Completed!'
                    }
                }

                stage('Code Package') {
                    steps {
                        echo 'Creating WAR Artifact'
                        sh 'mvn clean package'
                        echo 'Artifact Creation Completed'
                    }
                }

                stage('Building & Tag Docker Image') {
                    steps {
                        echo "Starting Building Docker Image: ${env.IMAGE_NAME}"
                        sh "docker build -t ${env.IMAGE_NAME} ."
                        echo 'Docker Image Build Completed'
                    }
                }

                stage('Docker Image Push to Amazon ECR') {
                    steps {
                        echo "Tagging Docker Image for ECR: ${env.ECR_IMAGE_NAME}"
                        sh "docker tag ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                        echo "Docker Image Tagging Completed"

                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            echo "Pushing Docker Image to ECR: ${env.ECR_IMAGE_NAME}"
                            sh "docker push ${env.ECR_IMAGE_NAME}"
                            echo "Docker Image Push to ECR Completed"
                        }
                    }
                }
            }
        }

        stage('Tag Docker Image for Preprod and Prod') {
            when {
                anyOf {
                    branch 'preprod'
                    branch 'prod'
                }
            }
            steps {
                script {
                    def devImage = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/mfusion-ms:dev-mfusion-ms-v.1.${env.BUILD_NUMBER}"
                    def preprodImage = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/mfusion-ms:preprod-mfusion-ms-v.1.${env.BUILD_NUMBER}"
                    def prodImage = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/mfusion-ms:prod-mfusion-ms-v.1.${env.BUILD_NUMBER}"

                    if (env.BRANCH_NAME == 'preprod') {
                        echo "Tagging and Pushing Docker Image for Preprod: ${preprodImage}"
                        sh "docker tag ${devImage} ${preprodImage}"
                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            sh "docker push ${preprodImage}"
                        }
                    } else if (env.BRANCH_NAME == 'prod') {
                        echo "Tagging and Pushing Docker Image for Prod: ${prodImage}"
                        sh "docker tag ${devImage} ${prodImage}"
                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            sh "docker push ${prodImage}"
                        }
                    }
                }
            }
        }

        stage('Delete Local Docker Images') {
            steps {
                script {
                    echo "Deleting Local Docker Images: ${env.IMAGE_NAME} ${env.ECR_IMAGE_NAME}"
                    // Ensure each image name is checked for null before attempting deletion
                    sh "docker rmi ${env.IMAGE_NAME} || true"
                    sh "docker rmi ${env.ECR_IMAGE_NAME} || true"
                    echo "Local Docker Images Deletion Completed"
                }
            }
        }

        stage('Deploy app to dev env') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Current Branch: ${env.BRANCH_NAME}"
                    def yamlFile = 'kubernetes/dev/05-deployment.yaml'
                    sh "sed -i 's/<latest>/dev-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlFile}"
                    kubernetesDeploy(configs: 'kubernetes/dev/*.yaml', kubeconfigId: 'k8s-credentials')
                }
            }
        }

        stage('Deploy app to preprod env') {
            when {
                branch 'preprod'
            }
            steps {
                script {
                    echo "Current Branch: ${env.BRANCH_NAME}"
                    def yamlFile = 'kubernetes/preprod/05-deployment.yaml'
                    sh "sed -i 's/<latest>/preprod-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlFile}"
                    kubernetesDeploy(configs: 'kubernetes/preprod/*.yaml', kubeconfigId: 'k8s-credentials')
                }
            }
        }

        stage('Deploy app to prod env') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    echo "Current Branch: ${env.BRANCH_NAME}"
                    def yamlFile = 'kubernetes/prod/05-deployment.yaml'
                    sh "sed -i 's/<latest>/prod-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlFile}"
                    kubernetesDeploy(configs: 'kubernetes/prod/*.yaml', kubeconfigId: 'k8s-credentials')
                }
            }
        }
    }

    post {
        success {
            echo "Deployment to ${env.BRANCH_NAME} environment completed successfully"
        }
        failure {
            echo "Deployment to ${env.BRANCH_NAME} environment failed. Check logs for details."
        }
    }
}
