pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        IMAGE_NAME = "satyam88/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        ECR_IMAGE_NAME = "${ECR_URL}/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
        KUBECONFIG_ID = 'kubeconfig-aws-aks-k8s-cluster'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'maven_3.9.4'
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

                stage('Docker Push to Docker Hub') {
                    steps {
                        withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CRED', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                            echo "Pushing Docker Image to DockerHub: ${env.IMAGE_NAME}"
                            sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                            sh "docker push ${env.IMAGE_NAME}"
                            echo "Docker Image Push to DockerHub Completed"
                        }
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
                    def devImage = "satyam88/mfusion-ms:mfusion-ms-v.1.${env.BUILD_NUMBER}"
                    def preprodImage = "${ECR_URL}/mfusion-ms:preprod-mfusion-ms-v.1.${env.BUILD_NUMBER}"
                    def prodImage = "${ECR_URL}/mfusion-ms:prod-mfusion-ms-v.1.${env.BUILD_NUMBER}"

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
                    sh "docker rmi ${env.IMAGE_NAME} || true"
                    sh "docker rmi ${env.ECR_IMAGE_NAME} || true"
                    echo "Local Docker Images Deletion Completed"
                }
            }
        }

        stage('Deploy to Development Environment') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    echo "Deploying to Dev Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '05-deployment.yaml', '06-configmap.yaml', '09.hpa.yaml']
                    def yamlDir = 'kubernetes/dev/'
                    // Replace <latest> in dev environment only
                    sh "sed -i 's/<latest>/mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlDir}05-deployment.yaml"

                    withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KUBECONFIG'),
                                     [$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-credentials',
                                      accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        yamlFiles.each { yamlFile ->
                            sh """
                                aws configure set aws_access_key_id \$AWS_ACCESS_KEY_ID
                                aws configure set aws_secret_access_key \$AWS_SECRET_ACCESS_KEY
                                aws configure set region ${REGION}

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n dev --validate=false
                            """
                        }
                    }
                    echo "Deployment to Dev Completed"
                }
            }
        }

        stage('Deploy to Preprod Environment') {
            when {
                branch 'preprod'
            }
            steps {
                script {
                    echo "Deploying to Preprod Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '05-deployment.yaml', '06-configmap.yaml', '09.hpa.yaml']
                    def yamlDir = 'kubernetes/preprod/'

                    // No sed command for preprod, manual update will be applied

                    withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KUBECONFIG'),
                                     [$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-credentials',
                                      accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        yamlFiles.each { yamlFile ->
                            sh """
                                aws configure set aws_access_key_id \$AWS_ACCESS_KEY_ID
                                aws configure set aws_secret_access_key \$AWS_SECRET_ACCESS_KEY
                                aws configure set region ${REGION}

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n preprod --validate=false
                            """
                        }
                    }
                    echo "Deployment to Preprod Completed"
                }
            }
        }

        stage('Deploy to Production Environment') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    echo "Deploying to Prod Environment"
                    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '05-deployment.yaml', '06-configmap.yaml', '09.hpa.yaml']
                    def yamlDir = 'kubernetes/prod/'

                    // No sed command for prod, manual update will be applied

                    withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KUBECONFIG'),
                                     [$class: 'AmazonWebServicesCredentialsBinding',
                                      credentialsId: 'aws-credentials',
                                      accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        yamlFiles.each { yamlFile ->
                            sh """
                                aws configure set aws_access_key_id \$AWS_ACCESS_KEY_ID
                                aws configure set aws_secret_access_key \$AWS_SECRET_ACCESS_KEY
                                aws configure set region ${REGION}

                                kubectl apply -f ${yamlDir}${yamlFile} --kubeconfig=\$KUBECONFIG -n prod --validate=false
                            """
                        }
                    }
                    echo "Deployment to Prod Completed"
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
