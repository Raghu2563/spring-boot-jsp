pipeline {
    agent any

    tools {
        maven 'v3.8.2'
    }
    parameters {
        booleanParam(name: 'PROD_BUILD', defaultValue: true, description: 'Enable this as a production build')
        string(name: 'SERVER_IP', defaultValue: '127.0.0.1', description: 'Provide production server IP Address.')
    }

    stages {
        stage('Source') {
            steps {
                git branch: 'batch8', changelog: false, credentialsId: 'github', poll: false, url: 'https://github.com/Raghu2563/spring-boot-jsp/'
            }
        }
        stage('Validate') {
            steps {
                sh 'mvn validate'
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
                        echo 'Doing integration test' // Just for demonstration of the parallel job.
                    }
                }
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Publishing and Deployment') {
            stages {
                stage('Publishing Artifacts') {
                    steps {
                        archiveArtifacts artifacts: 'target/news-v*.jar', fingerprint: true, onlyIfSuccessful: true
                    }
                }
                stage('Deploying to EC2') {
                    when {
                        expression { return params.PROD_BUILD }
                    }
                    steps {
                        // credentialsId is different from your current key, please change it accordingly
                        withCredentials([sshUserPrivateKey(credentialsId: 'deployment-key', keyFileVariable: 'SSHKEY', usernameVariable: 'SSHUSER')]) {
                            sh '''
                                version=$(perl -nle 'print "$2" if /<(version>)(v(\\d\\.){2}\\d)<\\/\\1/' pom.xml)
                                rsync -avzP -e "ssh -o StrictHostKeyChecking=no -i ${SSHKEY}" target/news-${version}.jar ${SSHUSER}@${SERVER_IP}:/home/headless-newsapp/newsapp/
                                ssh -o StrictHostKeyChecking=no -i ${SSHKEY} ${SSHUSER}@${SERVER_IP} sudo /usr/bin/systemctl restart newsapp.service
                            '''
                        }
                    }
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
