pipeline {
  agent { label 'docker' }

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME   = 'regapp'
    ANSIBLE_COLLECTIONS_PATHS = '/var/lib/jenkins/.ansible/collections:/usr/share/ansible/collections'
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
    
    stage('Build WAR') {
      steps {
        sh 'mvn clean package'
        sh 'cp webapp/target/*.war ./'
      }
    }
    
    stage('Build & Tag') {
      steps {
        sh '''
          echo "🔨 Building Docker image..."
          docker build -t $IMAGE_NAME:$VERSION .
          echo "🏷️ Tagging image with version and latest..."
          docker tag $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:$VERSION
          docker tag $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:latest
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-ecr-push'
        ]]) {
        sh '''
          docker push $ECR_REGISTRY/$IMAGE_NAME:$VERSION
          docker push $ECR_REGISTRY/$IMAGE_NAME:latest
        '''
        }
      }
    }

    stage('Ansible Deploy') {
      steps {
        sh '''
          echo '🚀 SSH into Ansible server and deploy...'
          ssh -o StrictHostKeyChecking=no -i /var/lib/jenkins/.ssh/id_rsa ansadmin@10.0.101.39 '
            export ANSIBLE_COLLECTIONS_PATHS=/home/ansadmin/.ansible/collections:/usr/share/ansible/collections
            cd /opt/docker
            ansible-playbook -i hosts regapp.yml -e app_tag=$VERSION
          '
        '''
      }
    }
  }

  post {
    success {
      echo "✅  $IMAGE_NAME:$VERSION pushed & deployed."
    }
    failure {
      echo "❌  Deployment failed – check the console log."
    }
  }
}


