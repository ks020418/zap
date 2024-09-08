pipeline {
    agent any

    stages {
        stage('Upload Test File to S3') {
            steps {
                script {
                    // Create a test file
                    def testFileName = "test_file.txt"
                    writeFile file: testFileName, text: 'This is a test file for S3 upload.'

                    // Upload the test file to S3
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh '''
                        aws s3 cp ${testFileName} s3://securityreports1337/reports/test/ --region us-west-1
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Test file uploaded successfully.'
        }
        failure {
            echo 'Failed to upload the test file.'
        }
    }
}
