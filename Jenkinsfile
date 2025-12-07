## Jenkinsfile (Pipeline)

```groovy
pipeline {
  agent any 

  environment {
    AWS_DEFAULT_REGION = 'ap-south-1'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Terraform Init') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh 'terraform init -input=false'
        }
      }
    }

    stage('Terraform Validate') {
      steps { sh 'terraform validate' }
    }

    stage('Terraform Plan') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh 'terraform plan -out=tfplan -input=false'
        }
      }
    }

    stage('Approve') {
      steps {
        input message: 'Apply Terraform changes?'
      }
    }

    stage('Terraform Apply') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh 'terraform apply -input=false -auto-approve tfplan'
        }
      }
    }
  }

  post {
    always { cleanWs() }
  }
}
```
