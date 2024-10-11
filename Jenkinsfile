pipeline {
    agent any
    environment {
        CONFIG_FILE = '/tmp/kubeconfig'
        RESOURCE_VERSION_FILE = 'resource_version.txt'
        AWS_DEFAULT_REGION = 'us-west-2'
        CLUSTER_NAME = 'certs'
    }
    stages {
        stage('Setup') {
            steps {
                script {
                    // Configure AWS CLI and update kubeconfig
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_DEFAULT_REGION} --kubeconfig ${CONFIG_FILE}"
                        env.EKS_ENDPOINT = sh(script: "aws eks describe-cluster --name ${CLUSTER_NAME} --query cluster.endpoint --output text", returnStdout: true).trim()
                    }
                }
            }
        }
        stage('Process Configuration') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'jenkins-sa-token', serverUrl: env.EKS_ENDPOINT, kubeconfigId: CONFIG_FILE]) {
                        // Load and parse the YAML configuration
                        def config = readYaml file: CONFIG_FILE

                        config.origins.each { origin ->
                            def sourceNamespace = origin.namespace

                            origin.secrets.each { secret ->
                                def secretName = secret.secret
                                def storedVersion = secret.resourceVersion

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
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
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
            git push origin main
        """
    }
}
