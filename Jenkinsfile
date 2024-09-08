pipeline {
    agent any

    stages {
        stage('Upload Test File to S3') {
            steps {
                script {
                    // Define file name and content
                    def testFileName = "test_file.txt"
                    def testFileContent = "This is a test file for S3 upload."
                    
                    // Create a test file
                    writeFile file: testFileName, text: testFileContent

                    // Upload the test file to S3
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh """
                        aws s3 cp ${testFileName} s3://securityreports1337/reports/test/ --region us-west-1
                        """
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
