pipeline {
    agent any

    environment {
        JIRA_SITE = 'JIRA-SITE' // Your Jira site name
        JIRA_PROJECT_KEY = 'PROJECT-KEY' // Your Jira project key
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                git branch: 'main', url: 'https://github.com/{github-username}/Flask_Application_Kurbenetes.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                bat 'pip install -r requirements.txt'
            }
        }

        stage('Check Docker Daemon') {
            steps {
                script {
                    bat 'docker info || echo "Docker daemon is not running"'
                }
            }
        }
        
        stage('Build and Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'dockerKey', usernameVariable: 'dockerUser')]) {
                        // Build Docker image
                        bat """
                        echo "Building Docker image..."
                        docker build -t ${dockerUser}/flask-app:latest .
                        echo "Logging in to Docker..."
                        echo \${dockerKey} | docker login --username \${dockerUser} --password-stdin
                        echo "Pushing Docker image..."
                        docker push ${dockerUser}/flask-app:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat """
                kubectl apply -f deployment.yaml --validate=false
                kubectl apply -f service.yaml
                """
            }
        }

        stage('Monitor Pods') {
            steps {
                script {
                    def podStatus = bat(returnStdout: true, script: 'kubectl get pods').trim()

                    // Check for any pods in Error/CrashLoopBackOff status
                    if (podStatus.contains('Error') || podStatus.contains('CrashLoopBackOff')) {
                        currentBuild.result = 'UNSTABLE'

                        // Create a Jira ticket if any pod is unhealthy
                        def issueFailure = [
                            fields: [
                                project: [key: env.JIRA_PROJECT_KEY],  // Use project key
                                summary: "Pod Failure Detected in Kubernetes Cluster",
                                description: """
                                There was an issue detected with the following pod(s):
                                ${podStatus}
                                Please investigate the issue immediately.
                                """,
                                issuetype: [name: 'Bug']  // Use the issue type name
                            ]
                        ]

                        responseFailure = jiraNewIssue(issue: issueFailure)

                        // Log the response for the issue creation
                        echo "Jira issue creation for pod failure successful: ${responseFailure.successful}"
                        echo "Response data: ${responseFailure.data}"

                    } else {
                        echo "All Pods are running fine."

                        // Create a Jira issue indicating all pods are running fine
                        def issueSuccess = [
                            fields: [
                                project: [key: env.JIRA_PROJECT_KEY],  // Use project key
                                summary: "All Pods Running Fine",
                                description: "All pods are up and running without issues.",
                                issuetype: [name: 'Task']  // Use the issue type name
                            ]
                        ]

                        responseSuccess = jiraNewIssue(issue: issueSuccess)

                        // Log the response for the issue creation
                        echo "Jira issue creation for pod status successful: ${responseSuccess.successful}"
                        echo "Response data: ${responseSuccess.data}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
    }
}
