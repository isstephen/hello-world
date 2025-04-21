pipeline {
  /* 1️⃣  Make sure we land on a node that has the Docker CLI installed
         and permission to run it (usually by being in the docker group). */
  agent { label 'docker' }          // create/assign a “docker”‑labelled agent

  /* 2️⃣  Global settings */
  environment {
    AWS_REGION     = 'us‑east‑1'
    ECR_REGISTRY   = '732583169994.dkr.ecr.us-east-1.amazonaws.com'
    IMAGE_NAME     = 'regapp'
  }

  /* 3️⃣  Parameterise the tag you want to deploy */
  parameters {
    string(name: 'VERSION', defaultValue: 'v1.0',
           description: 'Docker image tag to deploy')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    /* 4️⃣  Inject AWS creds just for this block */
    stage('Login to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                          credentialsId: 'aws-ecr-push']]) {      // <‑‑‑ add in UI
          sh '''
            aws ecr get-login-password --region $AWS_REGION \
              | docker login --username AWS --password-stdin $ECR_REGISTRY
          '''
        }
      }
    }

    /* 5️⃣  Build & tag — use the workspace as context unless you
           really need /opt/docker on the agent. */
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
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                          credentialsId: 'aws‑ecr‑push']]) {
          sh 'docker push $ECR_REGISTRY/$IMAGE_NAME:$VERSION'
        }
      }
    }

    /* 6️⃣  Trigger the deployment playbook */
    stage('Ansible Deploy') {
      steps {
        // assumes ssh key / inventory already set up on this agent
        sh 'ansible-playbook /opt/docker/regapp.yml -e "app_tag=${VERSION}"'
      }
    }
  }

  post {
    success { echo "✅  Deployment of $IMAGE_NAME:$VERSION completed!" }
    failure { echo "❌  Deployment failed – check the console log." }
  }
}

