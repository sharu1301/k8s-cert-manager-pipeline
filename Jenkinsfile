pipeline {
    agent any
    environment {
        KUBE_CONTEXT = 'arn:aws:eks:us-west-2:340752821725:cluster/certs' // Replace with your actual Kubernetes context
        CONFIG_FILE = '/root/.kube/config.yaml' // The configuration file in the repository
        RESOURCE_VERSION_FILE = 'resource_version.txt' // File to store resource versions
        SECRET_NAME = 'my-tls-secret' // Replace with your actual secret name if fixed
    }
    stages {
        stage('Setup') {
            steps {
                script {
                    // Set the Kubernetes context
                    sh "kubectl config use-context ${KUBE_CONTEXT}"
                }
            }
        }
        stage('Process Configuration') {
            steps {
                script {
                    // Load and parse the YAML configuration
                    def config = readYaml file: CONFIG_FILE

                    // Iterate through each origin in the configuration
                    config.origins.each { origin ->
                        def sourceNamespace = origin.namespace

                        // Iterate through each secret defined in the origin
                        origin.secrets.each { secret ->
                            def secretName = secret.secret
                            def storedVersion = secret.resourceVersion

                            // Get the current resource version from the cluster
                            def currentVersion = sh(
                                script: "kubectl get secret ${secretName} -n ${sourceNamespace} -o jsonpath='{.metadata.resourceVersion}' || echo 'not_found'",
                                returnStdout: true
                            ).trim()

                            // Compare versions and update if there's a change
                            if (currentVersion != 'not_found' && currentVersion != storedVersion) {
                                echo "Secret ${secretName} resourceVersion changed. Triggering refresh."

                                // Download the secret YAML
                                def secretYaml = sh(
                                    script: "kubectl get secret ${secretName} -n ${sourceNamespace} -o yaml",
                                    returnStdout: true
                                ).trim()

                                // Apply the secret to each destination namespace
                                origin.destinations.each { destination ->
                                    echo "Applying secret ${secretName} to namespace ${destination}."
                                    sh """
                                        echo '${secretYaml}' | sed 's/namespace: ${sourceNamespace}/namespace: ${destination}/g' | kubectl apply -f - -n ${destination}
                                    """
                                }

                                // Update the stored resource version in the file
                                updateStoredResourceVersion(secretName, currentVersion)
                            } else {
                                echo "Secret ${secretName} resourceVersion unchanged. Skipping refresh."
                            }
                        }
                    }
                }
            }
        }
    }
}

// Function to update the resourceVersion in the resource version file
def updateStoredResourceVersion(secretName, newVersion) {
    def resourceVersionContent = readFile(RESOURCE_VERSION_FILE)
    
    // Update the resourceVersion for the specified secret
    def updatedContent = resourceVersionContent.replaceAll(/(?<=name: ${secretName}\\n\\s*resourceVersion: )".*"/, "\"${newVersion}\"")
    
    // Write the updated content back to the resource version file
    writeFile file: RESOURCE_VERSION_FILE, text: updatedContent

    // Commit the updated resource version file to the repository
    withCredentials([usernamePassword(credentialsId: 'gitCredentials', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
        sh """
            git config --local user.name "${GIT_USER}"
            git config --local user.email "${GIT_USER}@example.com"
            git add ${RESOURCE_VERSION_FILE}
            git commit -m 'Updated resourceVersion for ${secretName} on \$(date)'
            git push origin main // Adjust branch name if different
        """
    }
}
