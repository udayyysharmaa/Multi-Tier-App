pipeline {
    agent any
    tools {
        maven 'maven'
        jdk 'jdk17'
    }
    environment {
        SONAR_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "onlinelearningofficial/fullstackwebsite"
        TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Clone the Project') {
            steps {
                git branch: 'main', url: 'https://github.com/udayyysharmaa/Multi-Tier-App.git'
            }
        }

        stage('Project Compile') {
            steps {
                sh 'mvn compile -DskipTests=true'
            }
        }

        stage('Project Test') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('Scan the File with Trivy') {
            steps {
                sh 'trivy fs --format json -o fs.json .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SONAR_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=websiteproject \
                    -Dsonar.projectKey=websiteproject \
                    -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh 'docker build -t ${IMAGE_NAME}:${TAG} .'
                    }
                }
            }
        }

        stage('Scan Docker Image with Trivy') {
            steps {
                sh 'trivy image --scanners vuln --skip-db-update --format json -o dockerimagereport.json ${IMAGE_NAME}:${TAG}'
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh 'docker push ${IMAGE_NAME}:${TAG}'
                    }
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    // Replace the Docker image tag in line 58 of the ds.yml file
                    sh """
                    sed -i '58s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${TAG}|' ds.yml
                    """
                }
            }
        }

        stage('Commit and Push Changes') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'GITCRED', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                        // Use a here document for the Git commands to avoid interpolation warnings
                        sh """
                        git config --global user.email "udaysharmaniit12345@gmail.com"
                        git config --global user.name "udayyysharmaa"
                        git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/udayyysharmaa/Multi-Tier-App.git
                        git pull origin main
                        git add ds.yml dockerimagereport.json fs.json
                        git commit -m "Update image to ${IMAGE_NAME}:${TAG}" || echo "No changes to commit"
                        git push origin main
                        """
                    }
                }
            }
        }
        stage('Website Deploy to K8S') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8-token', 
                    namespace: 'webapps', 
                    serverUrl: 'https://87074A0EDCB85C20BD1EE835ECCA102F.gr7.ap-south-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl apply -f ds.yml -n webapps"
                    sleep 30
                }
            }
        }
        stage('verify K8 Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[
                    caCertificate: '', 
                    clusterName: 'devopsshack-cluster', 
                    contextName: '', 
                    credentialsId: 'k8-token', 
                    namespace: 'webapps', 
                    serverUrl: 'https://87074A0EDCB85C20BD1EE835ECCA102F.gr7.ap-south-1.eks.amazonaws.com'
                ]]) {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
}
