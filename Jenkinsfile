pipeline {
    agent any
    tools { 
        maven 'Maven_3_5_2'  
    }
    stages {
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
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }

        stage('Build') { 
            steps { 
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script{
                        app = docker.build("asg")
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script{
                    docker.withRegistry('https://145988340565.dkr.ecr.us-west-2.amazonaws.com', 'ecr:us-west-2:aws-credentials') {
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
       
        stage ('Wait for Testing') {
            steps {
                sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
            }
        }
       
        stage('Run DAST Using ZAP') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    script {
                        sh 'zap.sh -cmd -quickurl http://$(kubectl get services/asgbuggy --namespace=devsecops -o json | jq -r ".status.loadBalancer.ingress[] | .hostname") -quickprogress -quickout ${WORKSPACE}/zap_report.html'
                        archiveArtifacts artifacts: 'zap_report.html'
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Get current date and time
                def timestamp = new Date().format("yyyyMMdd_HHmmss", TimeZone.getTimeZone("UTC"))
                def s3Folder = "reports/${timestamp}/"

                // Paths to the reports
                def sonarReportFile = "sonar-report.html"
                def snykReportFile = "snyk-report.json"
                def zapReportFile = "zap_report.html"

                // Upload reports to S3
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh """
                    aws s3 cp ${sonarReportFile} s3://securityreports1337/${s3Folder}${sonarReportFile} --region us-west-1
                    aws s3 cp ${snykReportFile} s3://securityreports1337/${s3Folder}${snykReportFile} --region us-west-1
                    aws s3 cp ${zapReportFile} s3://securityreports1337/${s3Folder}${zapReportFile} --region us-west-1
                    """
                }
            }
        }
    }
}
