pipeline {
    agent any
    environment {
        DOCKER_REGISTRY = "your-registry.example.com"
        IMAGE_NAME = "${DOCKER_REGISTRY}/coturn"
        TAG = "${env.BUILD_NUMBER}"
        HELM_CHART_PATH = "coturn-chart"
        AKS_CLUSTER = "your-aks-cluster"
        AKS_RESOURCE_GROUP = "your-resource-group"
        ANCHORE_CLI_URL = "http://your-anchore-engine:8228"
        OPENSCAP_PROFILE = "xccdf_org.ssgproject.content_profile_stig"
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/coturn/coturn.git', branch: 'master'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${TAG}", "--build-arg VERSION=${env.BUILD_NUMBER} coturn/docker/coturn/debian/.")
                }
            }
        }
        
        stage('Anchore Scan') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'anchore-creds', 
                                   usernameVariable: 'ANCHORE_USER', 
                                   passwordVariable: 'ANCHORE_PASS')]) {
                        sh """
                            anchore-cli --url ${ANCHORE_CLI_URL} \
                              --u ${ANCHORE_USER} \
                              --p ${ANCHORE_PASS} \
                              image add ${IMAGE_NAME}:${TAG}
                            
                            anchore-cli --url ${ANCHORE_CLI_URL} \
                              --u ${ANCHORE_USER} \
                              --p ${ANCHORE_PASS} \
                              wait --timeout 300 ${IMAGE_NAME}:${TAG}
                            
                            anchore-cli --url ${ANCHORE_CLI_URL} \
                              --u ${ANCHORE_USER} \
                              --p ${ANCHORE_PASS} \
                              evaluate check ${IMAGE_NAME}:${TAG} --tag vulnerability
                        """
                    }
                }
            }
        }
        
        stage('OpenSCAP Scan') {
            steps {
                script {
                    sh """
                        oscap-docker image-cve ${IMAGE_NAME}:${TAG} \
                          --results-arf results-${TAG}.xml \
                          --profile ${OPENSCAP_PROFILE}
                        
                        # Fail build if critical vulnerabilities found
                        if grep -q 'severity=\"high\"' results-${TAG}.xml; then
                            echo "Critical vulnerabilities found!"
                            exit 1
                        fi
                    """
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://${DOCKER_REGISTRY}', 'docker-registry-creds') {
                        docker.image("${IMAGE_NAME}:${TAG}").push()
                    }
                }
            }
        }
        
        stage('Deploy to AKS') {
            steps {
                script {
                    withCredentials([azureServicePrincipal('azure-creds')]) {
                        sh """
                            az login --service-principal \
                              -u ${AZURE_CLIENT_ID} \
                              -p ${AZURE_CLIENT_SECRET} \
                              --tenant ${AZURE_TENANT_ID}
                            
                            az aks get-credentials \
                              --resource-group ${AKS_RESOURCE_GROUP} \
                              --name ${AKS_CLUSTER}
                            
                            helm upgrade --install coturn ${HELM_CHART_PATH} \
                              --set image.tag=${TAG} \
                              --set image.repository=${IMAGE_NAME} \
                              --set static-auth-secret=${env.STATIC_AUTH_SECRET} \
                              --namespace coturn \
                              --create-namespace
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(color: 'good', message: "Deployment successful: ${env.BUILD_URL}")
        }
        failure {
            slackSend(color: 'danger', message: "Build failed: ${env.BUILD_URL}")
        }
    }
}
