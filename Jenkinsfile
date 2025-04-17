pipeline {
  agent any
  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    ECR_REGISTRY      = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
  }
  parameters {
    string(name: 'VERSION', defaultValue: 'v2.4', description: 'Docker image tag to deploy')
  }
  stages {
    stage('Checkout') {
      steps {
        // Pull latest from SCM
        checkout scm
      }
    }
    stage('Login to ECR') {
      steps {
        // Authenticate Docker to AWS ECR
        sh '''
          aws ecr get-login-password --region $AWS_DEFAULT_REGION \
            | docker login --username AWS --password-stdin $ECR_REGISTRY
        '''
      }
    }
    stage('Build & Tag') {
      steps {
        dir('/opt/docker') {
          sh 'docker build -t regapp:${VERSION} .'
          sh 'docker tag regapp:${VERSION} ${ECR_REGISTRY}/regapp:${VERSION}'
        }
      }
    }
    stage('Push to ECR') {
      steps {
        // Push image to ECR
        sh 'docker push ${ECR_REGISTRY}/regapp:${VERSION}'
      }
    }
    stage('Ansible Deploy') {
      steps {
        // Trigger Ansible playbook on your ansible server
        sh 'ansible-playbook /opt/docker/regapp.yml -e "app_tag=${VERSION}"'
      }
    }
  }
  post {
    success {
      echo "Deployment of regapp:${VERSION} completed successfully!"
    }
    failure {
      echo "Deployment failed. Check the logs above for errors."
    }
  }
}
