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
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    sh '''
                    mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=asgbuggywebapp1337 \
                        -Dsonar.organization=asgbuggywebapp1337 \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }


        stage('Run SCA Analysis Using Snyk') {
            steps {
                // Use Snyk token for SCA analysis
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
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
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        def timestamp = new Date().format("ddMMMyyyy_HHmmss", TimeZone.getTimeZone("UTC"))
                        def zapReportFile = "zap_report_${timestamp}.html"

                        sh '''
                        zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html
                        '''
                        archiveArtifacts artifacts: zapReportFile
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
