pipeline {
  agent { label 'docker' }

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME   = 'regapp'
    ANSIBLE_COLLECTIONS_PATHS = '/root/.ansible/collections:/usr/share/ansible/collections'
  }

  parameters {
    string(name: 'VERSION', defaultValue: 'v3.2', description: 'Docker image tag')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

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
          docker tag $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:$VERSION
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
        sh '''
          export ANSIBLE_COLLECTIONS_PATHS=/var/lib/jenkins/.ansible/collections:/usr/share/ansible/collections
          ansible-playbook /var/lib/jenkins/workspace/newdeployment/regapp.yml \
            -e "app_tag=${VERSION}" \
            -u ansadmin \
            --private-key=/var/lib/jenkins/.ssh/id_rsa
        '''
      }
    }
  }

  post {
    success { echo "✅  $IMAGE_NAME:$VERSION pushed & deployed." }
    failure { echo "❌  Deployment failed – check the console log." }
  }
}

