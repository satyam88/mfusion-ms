pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "533267238276"
        REGION = "ap-south-1"
        ECR_URL = "${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
        DEV_IMAGE_NAME = "dev-mfusion-ms-v1.${BUILD_NUMBER}"
        PREPROD_IMAGE_NAME = "preprod-mfusion-ms-v1.${BUILD_NUMBER}"
        PROD_IMAGE_NAME = "prod-mfusion-ms-v1.${BUILD_NUMBER}"
        DEV_ECR_IMAGE_NAME = "${ECR_URL}/${DEV_IMAGE_NAME}"
        PREPROD_ECR_IMAGE_NAME = "${ECR_URL}/${PREPROD_IMAGE_NAME}"
        PROD_ECR_IMAGE_NAME = "${ECR_URL}/${PROD_IMAGE_NAME}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    tools {
        maven 'maven_3.9.4'
    }

    stages {
        stage('Build and Test') {
            when { branch 'dev' }
            stages {
                stage('Code Compilation') {
                    steps {
                        echo 'Code Compilation in Progress...'
                        sh 'mvn clean compile'
                        echo 'Code Compilation Completed!'
                    }
                }

                stage('Code QA Execution') {
                    steps {
                        echo 'Running JUnit Tests...'
                        sh 'mvn clean test'
                        echo 'JUnit Tests Completed!'
                    }
                }

                stage('Package Application') {
                    steps {
                        echo 'Creating WAR Artifact...'
                        sh 'mvn clean package'
                        echo 'WAR Artifact Created!'
                    }
                }

                stage('Build Docker Image') {
                    steps {
                        echo "Building Docker Image: ${env.DEV_IMAGE_NAME}"
                        sh "docker build -t ${env.DEV_IMAGE_NAME} ."
                        echo "Docker Image Built Successfully!"
                    }
                }
            }
        }

        stage('Docker Image Push to Amazon ECR') {
            steps {
                echo "Tagging Docker Image for ECR: ${env.DEV_ECR_IMAGE_NAME}"
                sh "docker tag ${env.DEV_IMAGE_NAME} ${env.DEV_ECR_IMAGE_NAME}"
                echo "Docker Image Tagging Completed"

                withDockerRegistry([credentialsId: 'ecr-credentials', url: "https://${ECR_URL}"]) {
                    echo "Pushing Docker Image to ECR: ${env.DEV_ECR_IMAGE_NAME}"
                    sh "docker push ${env.DEV_ECR_IMAGE_NAME}"
                    echo "Docker Image Push to ECR Completed"
                }
            }
        }

        stage('Kubernetes Deployment') {
            parallel {
                stage('Deploy to Development') {
                    when { branch 'dev' }
                    steps {
                        script {
                            deployToKubernetes('dev', 'kubernetes/dev', env.DEV_ECR_IMAGE_NAME)
                        }
                    }
                }

                stage('Deploy to Preprod') {
                    when { branch 'preprod' }
                    steps {
                        script {
                            deployToKubernetes('preprod', 'kubernetes/preprod', env.PREPROD_ECR_IMAGE_NAME)
                        }
                    }
                }

                stage('Deploy to Prod') {
                    when { branch 'prod' }
                    steps {
                        script {
                            deployToKubernetes('prod', 'kubernetes/prod', env.PROD_ECR_IMAGE_NAME)
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline executed successfully on branch ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed on branch ${env.BRANCH_NAME}. Check logs for details."
        }
    }
}

def deployToKubernetes(namespace, yamlDir, imageName) {
    echo "Deploying to ${namespace} environment..."
    def yamlFiles = ['00-ingress.yaml', '02-service.yaml', '03-service-account.yaml', '05-deployment.yaml', '06-configmap.yaml', '09.hpa.yaml']
    sh "sed -i 's|<latest>|${imageName}|g' ${yamlDir}/05-deployment.yaml"

    withCredentials([file(credentialsId: 'kubeconfig-aws-aks-k8s-cluster', variable: 'KUBECONFIG'),
                     [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        yamlFiles.each { yamlFile ->
            sh """
                aws configure set aws_access_key_id \$AWS_ACCESS_KEY_ID
                aws configure set aws_secret_access_key \$AWS_SECRET_ACCESS_KEY
                aws configure set region ${env.REGION}

                kubectl apply -f ${yamlDir}/${yamlFile} --kubeconfig=\$KUBECONFIG -n ${namespace} --validate=false
            """
        }
    }
    echo "Deployment to ${namespace} Completed!"
}
