pipeline {
    agent any

    tools {
        // Define the Maven version to use
        maven 'Maven_3.2.5'
    }

    stages {
        stage('Clone Repository') {
            steps {
                // Cloning the specified repository
                git branch: 'patch-1', url: 'https://github.com/tupppi/asecurityguru-devsecops-jenkins-k8s-tf-sast-sca-build-push-and-deploy-to-k8-repo.git'
            }
        }

        stage('Compile and Run Sonar Analysis') {
            steps {
                script {
                    def timestamp = new Date().format("ddMMMyyyy_HHmmss", TimeZone.getTimeZone("UTC"))
                    def sonarReportFile = "sonar_report_${timestamp}.html"

                    // Run SonarQube analysis with credentials
                    withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=asgbuggywebapp1337 \
                            -Dsonar.organization=asgbuggywebapp1337 \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.token=${SONAR_TOKEN}
                        '''
                    }

                    // Archive SonarQube report
                    sh "curl -o ${sonarReportFile} https://sonarcloud.io/api/measures/component?component=asgbuggywebapp1337&metricKeys=alert_status"

                    // Upload to S3
                    // s3Upload(file: sonarReportFile, bucket: 'securityreports1337', path: 'reports/sonar/')
                    s3Upload(
                        bucket: 'securityreports1337',
                        includePathPattern: 'path/to/report/*',
                        workingDir: 'path_to_reports',
                        path: 'reports/sonar/'
                    )
                }
            }
        }

        stage('Run SCA Analysis Using Snyk') {
            steps {
                script {
                    def timestamp = new Date().format("ddMMMyyyy_HHmmss", TimeZone.getTimeZone("UTC"))
                    def snykReportFile = "snyk_report_${timestamp}.html"

                    // Use Snyk token for SCA analysis
                    withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                        sh 'mvn snyk:test -fn'
                    }

                    // Archive Snyk report
                    sh "snyk monitor --json > ${snykReportFile}"

                    // Upload to S3
                    s3Upload(file: snykReportFile, bucket: 'securityreports1337', path: 'reports/snyk/')
                }
            }
        }

        stage('Build') {
            steps {
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        app = docker.build("asg")
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://851725186362.dkr.ecr.us-west-1.amazonaws.com', 'ecr:us-west-1:aws-credentials') {
                        app.push("latest")
                    }
                }
            }
        }

        stage('Kubernetes Deployment of ASG Buggy Web Application') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh 'kubectl delete all --all -n devsecops'
                    sh 'kubectl apply -f deployment.yaml --namespace=devsecops'
                }
            }
        }

        stage('Wait for Testing') {
            steps {
                sh 'pwd; sleep 180; echo "Application has been deployed on K8S"'
            }
        }

        stage('Run DAST Using ZAP') {
            steps {
                script {
                    def timestamp = new Date().format("ddMMMyyyy_HHmmss", TimeZone.getTimeZone("UTC"))
                    def zapReportFile = "zap_report_${timestamp}.html"

                    withKubeConfig([credentialsId: 'kubelogin']) {
                        sh '''
                        zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/${zapReportFile}
                        '''

                        // Upload ZAP report to S3
                        s3Upload(file: zapReportFile, bucket: 'securityreports1337', path: 'reports/zap/')
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
