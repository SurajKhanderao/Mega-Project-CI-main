
pipeline {
    agent any

    tools {
        jdk 'jdk17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
        IMAGE_NAME = 'surajkhanderao/bankapp'
        TRIVY_CACHE = '/var/cache/trivy'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SurajKhanderao/Mega-Project-CI-main.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Testing') {
            steps {
                sh 'mvn clean test'
                sh 'ls -la target/site/jacoco/'
            }
        }

        stage('Trivy File Scanning') {
            steps {
               
                    sh "trivy fs --format table --output fs-report.txt ."
               
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        "$SCANNER_HOME/bin/sonar-scanner" \
                          -Dsonar.projectKey=BankApp \
                          -Dsonar.projectName=BankApp \
                          -Dsonar.sources=src/main \
                          -Dsonar.tests=src/test \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate( abortPipeline: false, credentialsId: 'sonar-token')
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'nexus-secrets', traceability: true) 
                {
                      sh 'mvn deploy'
                }
            }
        }

        stage('Build the Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-hub-cred') {
                        
                      sh " docker build -t $IMAGE_NAME:$IMAGE_TAG . "  
                       
                    }
                }
            }
        }


        stage('Image Scanning') {
            steps {
                sh '''
                    export TMPDIR=/var/tmp/trivy

                    trivy image \
                      --cache-dir "$TRIVY_CACHE" \
                      --format table \
                      --output image-report.txt \
                      "$IMAGE_NAME:$IMAGE_TAG"
                '''
            }
        }

        stage('Push the Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-hub-cred') {
                        sh '''
                            docker push "$IMAGE_NAME:$IMAGE_TAG"
                        '''
                    }
                }
            }
        }

        stage('Update Manifest File') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-cred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e
                        
                        git clone \
                      "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/SurajKhanderao/Mega-Project-CD.git"  Mega-Project-CD
                      

                        cd Mega-Project-CD/Manifests

                        sed -i \
                          "s|surajkhanderao/bankapp:.*|surajkhanderao/bankapp:${IMAGE_TAG}|" \
                          bankapp-deploy.yml

                        echo "Updated manifest:"
                        cat bankapp-deploy.yml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        git add bankapp-deploy.yml

                        if ! git diff --cached --quiet; then
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        else
                            echo "No manifest changes to commit."
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
