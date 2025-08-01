pipeline {
  agent any

  environment {
    // AWS access
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    AWS_DEFAULT_REGION    = 'us-east-1'

    // Azure access
    ARM_CLIENT_ID       = credentials('AZURE_CLIENT_ID')
    ARM_CLIENT_SECRET   = credentials('AZURE_CLIENT_SECRET')
    ARM_TENANT_ID       = credentials('AZURE_TENANT_ID')
    ARM_SUBSCRIPTION_ID = credentials('AZURE_SUBSCRIPTION_ID')

    // Terraform backend
    BUCKET_NAME       = "my-terraform-remote-state-bucket-007"
    S3_FOLDER         = "markerFile"
    MARKER_FILE_NAME  = ".backend_done"

    // Docker and AKS
    REGISTRY          = 'myregistry.azurecr.io'
    IMAGE             = 'myapp'
    
    // Define Terraform vars used later
    TF_VAR_resource_group_name = 'myResourceGroup'
    TF_VAR_prefix              = 'myakscluster'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Generate SSH Key (for Azure AKS)') {
      steps {
        dir('terraform') {
          sh '''
            if [ ! -f id_rsa ]; then
              ssh-keygen -t rsa -b 2048 -f id_rsa -C "terraform@devops" -N ""
            fi
          '''
        }
      }
    }

    stage('Terraform Init & Plan') {
      steps {
        dir('terraform') {
          sh 'terraform init'
          sh 'terraform plan -out=plan.out'
        }
      }
    }

    stage('Terraform Apply') {
      steps {
        dir('terraform') {
          sh 'terraform apply -auto-approve plan.out'
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        sh "docker build -t $REGISTRY/$IMAGE:${BUILD_NUMBER} ."
        sh "az acr login --name myregistry"
        sh "docker push $REGISTRY/$IMAGE:${BUILD_NUMBER}"
      }
    }

    stage('Deploy to Blue & Green') {
      steps {
        script {
          def clusters = ['blue', 'green']
          clusters.each { color ->
            sh "az aks get-credentials -g ${TF_VAR_resource_group_name} -n ${TF_VAR_prefix}-${color} --overwrite-existing"
            sh "kubectl apply -f k8s-manifests/deployment.yaml"
            sh "kubectl set image deployment/myapp myapp=$REGISTRY/$IMAGE:${BUILD_NUMBER}"
            sh "kubectl apply -f k8s-manifests/service.yaml"
          }
        }
      }
    }
  }}