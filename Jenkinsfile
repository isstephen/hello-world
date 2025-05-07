pipeline {
  agent { label 'docker' }

  environment {
    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME   = 'regapp'
    ANSIBLE_COLLECTIONS_PATHS = '/var/lib/jenkins/.ansible/collections:/usr/share/ansible/collections'
    ANSIBLE_SERVER = '10.0.101.241'
    ANSIBLE_USER   = 'ec2-user'
    SSH_KEY_PATH   = '/var/lib/jenkins/.ssh/id_rsa'
  }

  parameters {
    string(name: 'VERSION', defaultValue: 'v3.4', description: 'Docker image tag')
  }

  stages {
    stage('Checkout Code') {
      steps {
        checkout scm
      }
    }

    stage('Login to AWS ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-ecr-eks'
        ]]) {
          sh '''
            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    stage('Build WAR and Docker Image') {
      steps {
        sh '''
          mvn clean package
          cp webapp/target/*.war .
          echo "🔨 Building Docker image..."
          docker build -t $IMAGE_NAME:$VERSION .
          docker tag $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:$VERSION
          #docker tag $IMAGE_NAME:$VERSION $ECR_REGISTRY/$IMAGE_NAME:latest
        '''
      }
    }

    stage('Push Docker Image to ECR') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-ecr-eks'
        ]]) {
          sh '''
            docker push $ECR_REGISTRY/$IMAGE_NAME:$VERSION
            #docker push $ECR_REGISTRY/$IMAGE_NAME:latest
          '''
        }
      }
    }

    stage('Deploy to EKS via Ansible') {
      steps {
        sh '''
          echo "🚀 Deploying via Ansible server..."
          ssh -i $SSH_KEY_PATH -o StrictHostKeyChecking=no $ANSIBLE_USER@$ANSIBLE_SERVER '
            aws eks update-kubeconfig --region us-east-1 --name webapp-cluster &&
            ansible-playbook -i /opt/docker/hosts /opt/docker/regapp.yml -e app_tag=$VERSION
          '
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployment of $IMAGE_NAME:$VERSION to EKS completed successfully!"
    }
    failure {
      echo "❌ Deployment failed — Check console output."
    }
  }
}
