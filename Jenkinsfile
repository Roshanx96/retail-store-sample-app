pipeline {
    agent any

    parameters {
        booleanParam(name: 'MANUAL_TRIGGER', defaultValue: false, description: 'Build all services manually')
    }

    environment {
        AWS_REGION = credentials('aws-region')
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_ACCOUNT_ID = credentials('aws-account-id')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changed Services') {
            steps {
                script {
                    def services = ["ui", "catalog", "cart", "checkout", "orders"]
                    def changedServices = []

                    if (params.MANUAL_TRIGGER) {
                        changedServices = services
                    } else {
                        def diff = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim()

                        services.each { service ->
                            if (diff.contains("src/${service}/")) {
                                changedServices.add(service)
                            }
                        }
                    }

                    if (changedServices.isEmpty()) {
                        currentBuild.result = "SUCCESS"
                        error("No services changed. Exiting.")
                    }

                    env.CHANGED_SERVICES = changedServices.join(",")
                    echo "Services to deploy: ${env.CHANGED_SERVICES}"
                }
            }
        }

        stage('Deploy Services') {
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(",")

                    parallel services.collectEntries { service ->
                        ["Deploy-${service}": {
                            stage("Build & Push ${service}") {

                                def tag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                                def repo = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/retail-store-${service}"

                                sh """
                                aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                                aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                                aws configure set region ${AWS_REGION}

                                aws ecr describe-repositories --repository-names retail-store-${service} || \
                                aws ecr create-repository --repository-name retail-store-${service}

                                docker build -t ${repo}:${tag} -t ${repo}:latest src/${service}/
                                docker push ${repo}:${tag}
                                docker push ${repo}:latest
                                """

                                sh """
                                VALUES_FILE=src/${service}/chart/values.yaml
                                sed -i "s|repository:.*|repository: ${repo}|g" \$VALUES_FILE
                                sed -i "s|tag:.*|tag: \\"${tag}\\"|g" \$VALUES_FILE
                                """

                                sh """
                                git config user.email "gitops@jenkins.com"
                                git config user.name "Jenkins Bot"
                                git add src/${service}/chart/values.yaml
                                git commit -m "Update ${service} image to ${tag}" || true
                                git push origin gitops
                                """
                            }
                        }]
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Deployment finished."
        }
        success {
            echo "Deployment successful."
        }
        failure {
            echo "Deployment failed."
        }
    }
}
