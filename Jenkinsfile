@Library('Shared') _

pipeline {
    agent any
    
    environment {
        // Updated image names for GigaShop project
        DOCKER_IMAGE_NAME = 'anuragstark/gigashop-app'
        DOCKER_MIGRATION_IMAGE_NAME = 'anuragstark/gigashop-migration'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CREDENTIALS = credentials('github-credentials')
        GIT_BRANCH = "main"
    }
    
    stages {

        stage('Cleanup Workspace') {
            steps {
                script {
                    clean_ws()
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    clone("https://github.com/anuragstark/gigashop.git", "main")
                }
            }
        }

        //  NEW STAGE: Cleanup old Docker images to avoid disk full issues
        stage('Cleanup Old Docker Images') {
            steps {
                script {
                    echo "Cleaning up old Docker images, containers & volumes..."

                    // Remove dangling images
                    sh "docker image prune -f"

                    // Remove unused images older than 12 hours
                    sh "docker image prune -a --force --filter \"until=12h\""

                    // Remove stopped containers
                    sh "docker container prune -f"

                    // Remove unused volumes (safe)
                    sh "docker volume prune -f"

                    echo "Cleanup completed successfully!"
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                
                stage('Build Main App Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'Dockerfile',
                                context: '.'
                            )
                        }
                    }
                }
                
                stage('Build Migration Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'scripts/Dockerfile.migration',
                                context: '.'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                script {
                    run_tests()
                }
            }
        }
        
        stage('Security Scan with Trivy') {
            steps {
                script {
                    trivy_scan()
                }
            }
        }
        
        stage('Push Docker Images') {
            parallel {
                
                stage('Push Main App Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
                
                stage('Push Migration Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                    echo "Updating Kubernetes manifests with image tag: ${env.DOCKER_IMAGE_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git config user.name "Jenkins CI"
                            git config user.email "jenkins@ci.local"
                            
                            # Update deployment image
                            sed -i "s|image: ${env.DOCKER_IMAGE_NAME}:.*|image: ${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}|g" kubernetes/08-gigashop-deployment.yaml
                            
                            # Update migration job image
                            sed -i "s|image: ${env.DOCKER_MIGRATION_IMAGE_NAME}:.*|image: ${env.DOCKER_MIGRATION_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}|g" kubernetes/12-migration-job.yaml
                            
                            git add kubernetes/08-gigashop-deployment.yaml kubernetes/12-migration-job.yaml
                            git commit -m "Jenkins automatically updated K8s manifests to tag \${DOCKER_IMAGE_TAG}" || echo "No changes to commit"
                            git push https://\${GIT_USERNAME}:\${GIT_PASSWORD}@github.com/anuragstark/gigashop.git HEAD:main
                        """
                    }
            }
        }
    }
}
