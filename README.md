Pipeline Script
```

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    parameters {
        string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'Docker tag')
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
               git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/1995mars/BankApp-CI.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=True"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analsyis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=bankapp -Dsonar.projectKey=bankapp \
                            -Dsonar.java.binaries=target '''
                }
            }
        }
        
        // stage('Quality Gate') {
        //     steps {
        //         script {
        //           waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
        //         }
        //     }
        // }
        
        stage('Publish To Nexus') {
            steps {
              withMaven(globalMavenSettingsConfig: 'maven-config', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=True"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
              script {
                  withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t 1995mars/bankapp:${params.DOCKER_TAG} ."
                    }
              }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o dimage.html 1995mars/bankapp:${params.DOCKER_TAG}"
            }
        }
        
        stage('Push Docker Image') {
            steps {
              script {
                  withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push 1995mars/bankapp:${params.DOCKER_TAG}"
                    }
              }
            }
        }
        
        stage('dummy') {
            steps {
              echo "hi"
            }
        }
        
    }
}
```
