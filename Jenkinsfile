pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        DEV_IMAGE_NAME = "dev-mfusion-ms-v.1.${env.BUILD_NUMBER}"
        PREPROD_IMAGE_NAME = "preprod-mfusion-ms-v.1.${env.BUILD_NUMBER}"
        PROD_IMAGE_NAME = "prod-mfusion-ms-v.1.${env.BUILD_NUMBER}"
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

                stage('Build and Push Docker Image for Dev') {
                    steps {
                        echo "Building Docker Image for Dev: ${env.DEV_IMAGE_NAME}"
                        sh "docker build -t ${env.DEV_IMAGE_NAME} ."
                        echo "Pushing Docker Image to DockerHub: ${env.DEV_IMAGE_NAME}"
                        withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CRED', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                            sh "docker push ${env.DEV_IMAGE_NAME}"
                        }
                    }
                }
            }
        }

        stage('Tag and Push Docker Image for Preprod') {
            when {
                branch 'preprod'
            }
            steps {
                script {
                    echo "Tagging and Pushing Docker Image for Preprod: ${env.PREPROD_IMAGE_NAME}"
                    sh "docker tag ${env.DEV_IMAGE_NAME} ${env.PREPROD_IMAGE_NAME}"
                    withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                        sh "docker push ${env.PREPROD_IMAGE_NAME}"
                    }
                }
            }
        }

        stage('Tag and Push Docker Image for Prod') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    echo "Tagging and Pushing Docker Image for Prod: ${env.PROD_IMAGE_NAME}"
                    sh "docker tag ${env.DEV_IMAGE_NAME} ${env.PROD_IMAGE_NAME}"
                    withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                        sh "docker push ${env.PROD_IMAGE_NAME}"
                    }
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
                    sh "sed -i 's/<latest>/dev-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlDir}05-deployment.yaml"

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
                    sh "sed -i 's/<latest>/preprod-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlDir}05-deployment.yaml"

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
                    sh "sed -i 's/<latest>/prod-mfusion-ms-v.1.${BUILD_NUMBER}/g' ${yamlDir}05-deployment.yaml"

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
