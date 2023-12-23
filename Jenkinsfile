pipeline {
    agent any

    stages {
      stage('Build') {
        steps {
          script {
            dockerImage = docker.build("taimoorrkhan/distance-converter:${env.BUILD_ID}")
        }
    }
}
        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Test') {
            steps {
                sh 'ls -l index.html' // Simple check for index.html
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Deploy the new version
                    sshPublisher(
                        publishers: [
                            sshPublisherDesc(
                                configName: "MyUbuntuServer", 
                                transfers: [sshTransfer(
                                    execCommand: """
                                        docker pull taimoorrkhan/distance-converter:${env.BUILD_ID}
                                        docker stop distance-converter-container || true
                                        docker rm distance-converter-container || true
                                        docker run -d --name distance-converter-container -p 400:80 taimoorrkhan/distance-converter:${env.BUILD_ID}
                                    """
                                )]
                            )
                        ]
                    )

                    // Check if deployment is successful
                    boolean isDeploymentSuccessful = sh(script: 'curl -s -o /dev/null -w "%{http_code}" http://51.20.126.236:400', returnStdout: true).trim() == '200'

                    if (!isDeploymentSuccessful) {
                        // Rollback to the previous version
                        def previousSuccessfulTag = readFile('previous_successful_tag.txt').trim()
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: "MyUbuntuServer",
                                    transfers: [sshTransfer(
                                        execCommand: """
                                            docker pull taimoorrkhan/distance-converter:${previousSuccessfulTag}
                                            docker stop distance-converter-container || true
                                            docker rm distance-converter-container || true
                                            docker run -d --name distance-converter-container -p 400:80 taimoorrkhan/distance-converter:${previousSuccessfulTag}
                                        """
                                    )]
                                )
                            ]
                        )
                    } else {
                        // Update the last successful tag
                        writeFile file: 'previous_successful_tag.txt', text: "${env.BUILD_ID}"
                    }
                }
            }
        }
    }

    post {
        failure {
            mail(
                to: 'sp20-bcs-026@cuiatk.edu.pk',
                subject: "Failed Pipeline: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body: 
                """
                The pipeline job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has failed.
                Please check the job for more details.
                
                URL: ${env.BUILD_URL}

                Regards,
                
                Jenkins
                """
            )
        }
    }
}
