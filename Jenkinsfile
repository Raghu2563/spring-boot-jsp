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
        // string(name: 'SERVER_IP', defaultValue: '13.201.85.177', description: 'Provide production server IP Address.')
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

            // Construct the dynamic artifactId and file name
            def artifactId = "news-${version}.jar"
            def nexusFilePath = "target/${artifactId}"

            // Use the nexusArtifactUploader to deploy to Nexus
            nexusArtifactUploader artifacts: [[
                artifactId: 'testapp',  // This can also be made dynamic if needed
                classifier: '',
                file: nexusFilePath,
                type: 'jar'
            ]],
            credentialsId: 'nexus-credentials',
            groupId: 'org.springframework.boot',
            nexusUrl: '3.110.77.23:8081/repository/springboot-app-releases/',  // Ensure http is included
            nexusVersion: 'nexus3',
            protocol: 'http',
            repository: 'springboot-app-releases',
            version: version  // Use the dynamic version from the Maven project
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
                # Extract version without the leading 'v'
                version=$(perl -nle 'print "$2" if /<version>(v?((\\d\\.){2}\\d))<\\/version>/' pom.xml | head -n 1)
                echo "Extracted Version: $version"

                # Use rsync to copy the artifact
                rsync -avzP -e "ssh -o StrictHostKeyChecking=no -i $SSHKEY" target/news-${version}.jar ${USER}@13.201.85.177:/home/deploy/java-app/

                # Restart the service on the remote server
                ssh -o StrictHostKeyChecking=no -i $SSHKEY ${USER}@13.201.85.177 "sudo /usr/bin/systemctl restart java-app.service"
            '''
        }
    }
}





}

    
}
