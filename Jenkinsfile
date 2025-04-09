pipeline {
    agent any 

    triggers {
        githubPush()
    }
    
    tools {
        nodejs 'nodejs'
    }
    
    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'
        AWS_ACCOUNT_ID = credentials('Account_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO1')
        AWS_DEFAULT_REGION = 'us-east-1'
        REPOSITORY_URI = '${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/'
        NVD_API_KEY = '53c118cc-372b-4b1b-9286-2fd55765b27d'
        NVD_CACHE_DIR = '/var/lib/jenkins/owasp-data'
    }
    
    stages {
        
        stage('Cleaning Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout from Git') {
            steps {
                echo "Git checking out the frontend code.."
                git credentialsId: 'GITHUB-APP', url: 'https://github.com/ShantanuMakkar/DevOps-Project-App.git', branch: 'main'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo "Beginning sonarqube scan.."
                dir('frontend') {
                    withSonarQubeEnv('sonar-server') { // Use withSonarQubeEnv wrapper
                        sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=frontend \
                        -Dsonar.projectKey=frontend \
                        -Dsonar.sources=.
                        '''
                    }
                }
            }
        }
        
        stage('Quality Check') {
            steps {
                script {
                    echo "Receiving quality gate reports.."
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        //stage('OWASP Dependency-Check Scan') {
            //steps {
                //dir('Application-Code/backend') {
                    //dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=${env.NVD_API_KEY}", odcInstallation: 'DP-Check'
                    //dependencyCheckPublisher pattern: "**/dependency-check-report.xml"
                //}
            //}
        //}
        
        stage('Trivy File Scan') {
            steps {
                echo "Beginning Trivy file scan.."
                dir('frontend') {
                    sh 'trivy fs . > trivyfs.txt'
                }
            }
        }
        
        stage('Docker Image Build') {
            steps {
                echo "Beginning docker image build.."
                script {
                    dir('frontend') {
                        sh 'docker system prune -f'
                        sh 'docker container prune -f'
                        sh 'docker build -t ${AWS_ECR_REPO_NAME} .'
                    }
                }
            }
        }
        
        stage('ECR Image Pushing') {
            steps {
                echo "Pushing the image to ECR.."
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}"
                    sh "docker tag ${AWS_ECR_REPO_NAME} ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                echo "Beginning trivy image scan.."
                sh "trivy image ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER} > trivyimage.txt"
            }
        }
        
        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "DevOps-Project-App"
                GIT_USER_NAME = "shantanumakkar"
            }
            steps {
                echo "Updating k8s deployment manifests.."
                dir('k8s-manifests/frontend/manifests') {
                    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                        git config user.email "shantanu0015@gmail.com"
                        git config user.name "shantanumakkar"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        imageTag=$(grep -oP '(?<=frontend:)[^ ]+' deployment.yaml)
                        sed -i "s/${AWS_ECR_REPO_NAME}:${imageTag}/${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}/" deployment.yaml
                        git add deployment.yaml
                        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        '''
                    }
                }
            }
        }
    }
}
