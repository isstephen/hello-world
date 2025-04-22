pipeline {
  agent { label 'docker' }             // make sure this node has Docker

  environment {
    AWS_REGION   = 'us-east-1'         // re‑typed ASCII dashes
    ECR_REGISTRY = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME   = 'regapp'
  }

  parameters {
    string(name: 'VERSION', defaultValue: 'v2.4', description: 'Docker image tag')
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Login to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-ecr-push'
        ]]) {
          sh '''
            aws ecr get-login-password --region $AWS_REGION | \
              docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    stage('Build & Tag') {
      steps {
        sh '''
          docker build -t $IMAGE_NAME:$VERSION .
          docker tag    $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:$VERSION
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-ecr-push'
        ]]) {
          sh 'docker push $ECR_REGISTRY/$IMAGE_NAME:$VERSION'
        }
      }
    }

    stage('Ansible Deploy') {
      steps {
        sh 'ansible-playbook /var/lib/jenkins/workspace/newdeployment/regapp.yml -e "app_tag=${VERSION}"'
      }
    }
  }

  post {
    success { echo "✅  $IMAGE_NAME:$VERSION pushed & deployed." }
    failure { echo "❌  Deployment failed – check the console log." }
  }
}
