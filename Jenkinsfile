pipeline {
    agent any 

    environment {
        NAMESPACE_SOURCE = 'namespace1'  // Replace with your source namespace
        NAMESPACES_TARGET = ['namespace2', 'namespace3']  // Replace with your target namespaces
        SECRET_NAME = 'my-tls-secret'  // Replace with your secret name
        RESOURCE_VERSION_FILE = 'resource_version.txt'  // File to store the resource version
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    // Create the resource_version.txt if it doesn't exist
                    if (!fileExists(RESOURCE_VERSION_FILE)) {
                        writeFile file: RESOURCE_VERSION_FILE, text: '0'  // Initial value
                    }
                }
            }
        }
        
        stage('Get Resource Version') {
            steps {
                script {
                    // Get the current resource version of the secret in the source namespace
                    def secret = sh(script: "kubectl get secret ${SECRET_NAME} -n ${NAMESPACE_SOURCE} -o jsonpath='{.metadata.resourceVersion}'", returnStdout: true).trim()
                    env.CURRENT_RESOURCE_VERSION = secret
                }
            }
        }

        stage('Check Resource Version') {
            steps {
                script {
                    // Read the stored resource version from the file
                    def storedResourceVersion = readFile(RESOURCE_VERSION_FILE).trim()
                    
                    // Check if the current resource version is different from the stored one
                    if (env.CURRENT_RESOURCE_VERSION != storedResourceVersion) {
                        echo "Resource version has changed. Updating secrets in target namespaces..."
                        env.NEED_UPDATE = true
                    } else {
                        echo "No changes detected in resource version."
                        env.NEED_UPDATE = false
                    }
                }
            }
        }

        stage('Update Secrets') {
            when {
                expression { env.NEED_UPDATE == 'true' }
            }
            steps {
                script {
                    // Download the secret from the source namespace
                    sh "kubectl get secret ${SECRET_NAME} -n ${NAMESPACE_SOURCE} -o yaml | kubectl apply -f - -n ${NAMESPACE_SOURCE}" 
                    
                    // Apply the secret to each target namespace
                    for (namespace in env.NAMESPACES_TARGET) {
                        sh "kubectl get secret ${SECRET_NAME} -n ${NAMESPACE_SOURCE} -o yaml | kubectl apply -f - -n ${namespace}"
                    }

                    // Update and commit the new resource version
                    writeFile file: RESOURCE_VERSION_FILE, text: env.CURRENT_RESOURCE_VERSION
                    sh '''
                        git config --global user.email "jenkins@example.com"  # Replace with your email
                        git config --global user.name "Jenkins"
                        git add ${RESOURCE_VERSION_FILE}
                        git commit -m "Updated resource version to ${CURRENT_RESOURCE_VERSION}"
                        git push origin main  # Replace with your branch name if necessary
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished."
        }
    }
}
