pipeline {
    agent any
    environment {
        KUBE_CONTEXT = 'arn:aws:eks:us-west-2:340752821725:cluster/certs'
        CONFIG_FILE = 'config.yaml' // Assume this is in your workspace
        RESOURCE_VERSION_FILE = 'resource_version.txt'
        SECRET_NAME = 'my-tls-secret'
        AWS_DEFAULT_REGION = 'us-west-2'
    }
    stages {
        stage('Setup') {
            steps {
                script {
                    // Configure AWS CLI
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh "aws eks get-token --cluster-name certs | kubectl apply -f -"
                    }
                }
            }
        }
        stage('Process Configuration') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'jenkins-sa-token', serverUrl: 'https://REPLACE_WITH_YOUR_EKS_API_SERVER_URL']) {
                        // Load and parse the YAML configuration
                        def config = readYaml file: CONFIG_FILE

                        config.origins.each { origin ->
                            def sourceNamespace = origin.namespace

                            origin.secrets.each { secret ->
                                def secretName = secret.secret
                                def storedVersion = secret.resourceVersion

                                try {
                                    def currentVersion = sh(
                                        script: "kubectl get secret ${secretName} -n ${sourceNamespace} -o jsonpath='{.metadata.resourceVersion}' || echo 'not_found'",
                                        returnStdout: true
                                    ).trim()

                                    if (currentVersion != 'not_found' && currentVersion != storedVersion) {
                                        echo "Secret ${secretName} resourceVersion changed. Triggering refresh."

                                        def secretYaml = sh(
                                            script: "kubectl get secret ${secretName} -n ${sourceNamespace} -o yaml",
                                            returnStdout: true
                                        ).trim()

                                        origin.destinations.each { destination ->
                                            echo "Applying secret ${secretName} to namespace ${destination}."
                                            writeFile file: 'temp_secret.yaml', text: secretYaml.replaceAll("namespace: ${sourceNamespace}", "namespace: ${destination}")
                                            sh "kubectl apply -f temp_secret.yaml -n ${destination}"
                                        }

                                        updateStoredResourceVersion(secretName, currentVersion)
                                    } else {
                                        echo "Secret ${secretName} resourceVersion unchanged. Skipping refresh."
                                    }
                                } catch (Exception e) {
                                    echo "Error processing secret ${secretName} in namespace ${sourceNamespace}: ${e.message}"
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

def updateStoredResourceVersion(secretName, newVersion) {
    def resourceVersionContent = readFile(RESOURCE_VERSION_FILE)
    def updatedContent = resourceVersionContent.replaceAll(/(?<=name: ${secretName}\\n\\s*resourceVersion: )".*"/, "\"${newVersion}\"")
    writeFile file: RESOURCE_VERSION_FILE, text: updatedContent

    withCredentials([usernamePassword(credentialsId: 'gitCredentials', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
        sh """
            git config --local user.name "${GIT_USER}"
            git config --local user.email "${GIT_USER}@example.com"
            git add ${RESOURCE_VERSION_FILE}
            git commit -m 'Updated resourceVersion for ${secretName} on \$(date)'
            git push origin main // Consider using a variable for branch
        """
    }
}
