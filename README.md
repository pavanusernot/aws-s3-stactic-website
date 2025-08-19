pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = "ap-south-1"   // Change to your AWS region
        S3_BUCKET = "my-static-site-bucket" // Change to your bucket name
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/username/static-website.git'
            }
        }

        stage('Build Website') {
            steps {
                echo "No build needed for static website. Just preparing files..."
                sh 'ls -al'
            }
        }

        stage('Deploy to S3') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_DEFAULT_REGION}") {
                    sh """
                        aws s3 sync . s3://${S3_BUCKET} \
                        --exclude '.git/*' \
                        --delete
                    """
                }
            }
        }

        stage('Invalidate CloudFront (Optional)') {
            when {
                expression { return env.CLOUDFRONT_DISTRIBUTION_ID != null }
            }
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_DEFAULT_REGION}") {
                    sh "aws cloudfront create-invalidation --distribution-id ${CLOUDFRONT_DISTRIBUTION_ID} --paths '/*'"
                }
            }
        }
    }

    post {
        success {
            echo "✅ Website deployed successfully to S3!"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
