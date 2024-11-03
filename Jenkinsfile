pipeline {
    agent any

    tools {
        maven '3.8.2' // Ensure Maven is configured in Jenkins
    }

    environment {
        NEXUS_URL = 'http://65.1.132.91:8081/repository/springboot-app-releases/' // Update with Nexus URL
        NEXUS_CREDENTIALS = credentials('nexus-credentials') // Jenkins credentials for Nexus
        SONARQUBE_SERVER = 'SonarQube' // SonarQube server configured in Jenkins
    }

    parameters {
        booleanParam(name: 'PROD_BUILD', defaultValue: true, description: 'Enable this as a production build')
        string(name: 'SERVER_IP', defaultValue: '65.0.21.175', description: 'Provide production server IP Address.')
    }

    stages {
        stage('Source') {
            steps {
                git branch: 'raghu', changelog: false, credentialsId: 'github', poll: false, url: 'https://github.com/Raghu2563/spring-boot-jsp'
            }
        }
        
        stage('Code Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar -DskipTests'
                    }
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Test') {
                    steps {
                        sh 'mvn test'
                    }
                }
                stage('Integration Test') {
                    steps {
                        echo 'Running integration tests' // Placeholder for actual integration test commands
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    echo "Artifact Version: ${version}"
                    
                    // Deploy artifact to Nexus
                    sh """
                        mvn deploy:deploy-file \
                            -DgroupId=com.example \
                            -DartifactId=news-app \
                            -Dversion=${version} \
                            -Dpackaging=jar \
                            -Dfile=target/news-${version}.jar \
                            -DrepositoryId=nexus \
                            -Durl=${NEXUS_URL}
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression { return params.PROD_BUILD }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'deployment-key', keyFileVariable: 'SSHKEY', usernameVariable: 'SSHUSER')]) {
                    sh '''
                        version=$(perl -nle 'print "$2" if /<(version>)(v(\\d\\.){2}\\d)<\\/\\1/' pom.xml)
                        rsync -avzP -e "ssh -o StrictHostKeyChecking=no -i ${SSHKEY}" target/news-${version}.jar ${SSHUSER}@${params.SERVER_IP}:/home/headless-newsapp/newsapp/
                        ssh -o StrictHostKeyChecking=no -i ${SSHKEY} ${SSHUSER}@${params.SERVER_IP} sudo /usr/bin/systemctl restart newsapp.service
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend color: 'good', message: "The Build #${env.BUILD_NUMBER} was successful: ${env.BUILD_URL}"
        }
        failure {
            slackSend color: 'danger', message: "The Build #${env.BUILD_NUMBER} has failed: ${env.BUILD_URL}"
        }
    }
}
