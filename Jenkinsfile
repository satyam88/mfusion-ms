pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        BRANCH_NAME = "${env.BRANCH_NAME}"
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        IMAGE_TAG = "${BRANCH_NAME}-mfusion-ms-v.1.${BUILD_NUMBER}"
        DEV_IMAGE_TAG = "dev-mfusion-ms-v.1.${BUILD_NUMBER}"
        PREPROD_IMAGE_TAG = "preprod-mfusion-ms-v.1.${BUILD_NUMBER}"
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
                        echo "Starting Building Docker Image: ${ECR_URL}/mfusion-ms:${DEV_IMAGE_TAG}"
                        sh "docker build -t ${ECR_URL}/mfusion-ms:${DEV_IMAGE_TAG} ."
                        echo 'Docker Image Build Completed'
                    }
                }

                stage('Docker Image Push to Amazon ECR') {
                    steps {
                        echo "Pushing Docker Image to ECR: ${ECR_URL}/mfusion-ms:${DEV_IMAGE_TAG}"
                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            sh "docker push ${ECR_URL}/mfusion-ms:${DEV_IMAGE_TAG}"
                        }
                        echo "Docker Image Push to ECR Completed"
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
                    if (env.BRANCH_NAME == 'preprod') {
                        def devImage = "${ECR_URL}/mfusion-ms:${DEV_IMAGE_TAG}"
                        def preprodImage = "${ECR_URL}/mfusion-ms:${PREPROD_IMAGE_TAG}"

                        echo "Pulling Dev Image: ${devImage}"
                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            sh "docker pull ${devImage}"
                            echo "Tagging Dev Image as Preprod: ${preprodImage}"
                            sh "docker tag ${devImage} ${preprodImage}"
                            echo "Pushing Preprod Image to ECR: ${preprodImage}"
                            sh "docker push ${preprodImage}"
                        }
                        echo "Deleting Local Images"
                        sh "docker rmi ${devImage} ${preprodImage} || true"
                    } else if (env.BRANCH_NAME == 'prod') {
                        def preprodImage = "${ECR_URL}/mfusion-ms:${PREPROD_IMAGE_TAG}"
                        def prodImage = "${ECR_URL}/mfusion-ms:prod-mfusion-ms-v.1.${BUILD_NUMBER}"

                        echo "Pulling Preprod Image: ${preprodImage}"
                        withDockerRegistry([credentialsId: 'ecr:ap-south-1:ecr-credentials', url: "https://${ECR_URL}"]) {
                            sh "docker pull ${preprodImage}"
                            echo "Tagging Preprod Image as Prod: ${prodImage}"
                            sh "docker tag ${preprodImage} ${prodImage}"
                            echo "Pushing Prod Image to ECR: ${prodImage}"
                            sh "docker push ${prodImage}"
                        }
                        echo "Deleting Local Images"
                        sh "docker rmi ${preprodImage} ${prodImage} || true"
                    }
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

                    sh """
                        sed -i 's|<latest>|${DEV_IMAGE_TAG}|g' ${yamlFile}
                        cat ${yamlFile} | grep ${DEV_IMAGE_TAG} || echo "Replacement failed in ${yamlFile}"
                    """
                    sh """
                        kubectl --kubeconfig=/var/lib/jenkins/.kube/config apply -f kubernetes/dev/
                    """

                    def configMapChanged = sh(script: "git diff --name-only HEAD~1 | grep -q 'kubernetes/dev/06-configmap.yaml'", returnStatus: true)
                    if (configMapChanged == 0) {
                        echo "ConfigMap changed, restarting pods"
                        sh """
                            kubectl --kubeconfig=/var/lib/jenkins/.kube/config rollout restart deployment dev-mfusion-ms-deployment -n dev
                        """
                    } else {
                        echo "No changes in ConfigMap, skipping pod restart"
                    }
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

                    sh """
                        sed -i 's|<latest>|${PREPROD_IMAGE_TAG}|g' ${yamlFile}
                        cat ${yamlFile} | grep ${PREPROD_IMAGE_TAG} || echo "Replacement failed in ${yamlFile}"
                    """
                    sh """
                        kubectl --kubeconfig=/var/lib/jenkins/.kube/config apply -f kubernetes/preprod/
                    """
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

                    sh """
                        sed -i 's|<latest>|prod-mfusion-ms-v.1.${BUILD_NUMBER}|g' ${yamlFile}
                        cat ${yamlFile} | grep prod-mfusion-ms-v.1.${BUILD_NUMBER} || echo "Replacement failed in ${yamlFile}"
                    """
                    sh """
                        kubectl --kubeconfig=/var/lib/jenkins/.kube/config apply -f kubernetes/prod/
                    """
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
