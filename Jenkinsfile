pipeline {
    agent any

    tools {
        maven '3.8.2' // Ensure Maven is configured in Jenkins
    }

    environment {
        NEXUS_URL = 'http://3.110.77.23:8081/repository/springboot-app-releases/' // Update with Nexus URL
        NEXUS_CREDENTIALS = credentials('nexus-credentials') // Jenkins credentials for Nexus
        SONARQUBE_SERVER = 'SonarQube' // SonarQube server configured in Jenkins
    }

    parameters {
        booleanParam(name: 'PROD_BUILD', defaultValue: true, description: 'Enable this as a production build')
        string(name: 'SERVER_IP', defaultValue: '13.201.85.177', description: 'Provide production server IP Address.')
    }

    stages {
        stage('Source') {
            steps {
                git branch: 'raghu', changelog: false, credentialsId: 'github', poll: false, url: 'https://github.com/Raghu2563/spring-boot-jsp'
            }
        }
        
        stage('Build and SonarQube Analysis') {
    steps {
        script {
            withSonarQubeEnv('SonarQube') {
                // Compile the project and skip tests
                sh "mvn clean install"
                // Run SonarQube analysis
                sh "mvn sonar:sonar -Dsonar.java.binaries=target/classes"
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
                    // Extract the version from the Maven project
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    echo "Artifact Version: ${version}"

                    // Use the nexusArtifactUploader to deploy to Nexus
                    nexusArtifactUploader artifacts: [[artifactId: 'testapp', classifier: '', file: 'target/news-v*.jar', type: 'jar']], credentialsId: 'nexus-credentials', groupId: 'org.springframework.boot', nexusUrl: '3.110.77.23:8081/repository/springboot-app-releases/', nexusVersion: 'nexus3', protocol: 'http', repository: 'springboot-app-releases', version: '2.3.2.RELEASE'
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression { return params.PROD_BUILD }
            }
            steps {
                     withCredentials([sshUserPrivateKey(credentialsId: 'pk_jv_app', keyFileVariable: 'SSHKEY', usernameVariable: 'USER')]) {
                      sh '''
                      # Extract the version from pom.xml
                     version=$(perl -nle 'print "$2" if /<(version>)(v(\\d\\.){2}\\d)<\\/\\1/' pom.xml)
                      echo "Extracted Version: $version"

                     rsync -avzP -e "ssh -o StrictHostKeyChecking=no -i $SSHKEY" target/news-${version}.jar ${SSHUSER}@${params.SERVER_IP}:/home/deploy/java-app/
ssh -o StrictHostKeyChecking=no -i ${SSHUSER}@${params.SERVER_IP} "sudo /usr/bin/systemctl restart java-app.service"
                      '''
                      }
            

                }
            }
        }

    
}
