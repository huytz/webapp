pipeline {

  agent any

  parameters {
    string (
      defaultValue: 'origin/main',
      description: 'Git branch to build and deploy.',
      name : 'GIT_BRANCH_NAME')
  }

  environment {

      CI_REGISTRY = "369537814977.dkr.ecr.ap-southeast-1.amazonaws.com"

      REPOSITORY = "webapp"

      CI_REGISTRY_USER = credentials('aws-id')
      
      CI_REGISTRY_PASSWORD =  credentials('aws-key')

      CI_PROJECT_PATH = "webapp"

      IMAGE = "${CI_REGISTRY}/${CI_PROJECT_PATH}"

      NAMESPACE = "default"
      
      DEPLOYMENT_NAME = "webapp"

      KUBE_CONFIG = credentials('kubeconfig')

  }

  stages {
    
    stage("Build") {
      steps {
            sh """
              docker login -u ${CI_REGISTRY_USER} --password ${CI_REGISTRY_PASSWORD} ${CI_REGISTRY}
              echo 'Running docker...'
              docker pull ${IMAGE}:latest || true
              docker build --cache-from ${IMAGE}:latest -f Dockerfile -t ${IMAGE}:latest -t ${IMAGE}:${env.GIT_BRANCH}_${env.GIT_COMMIT} .
              docker push ${IMAGE}:${env.GIT_BRANCH}_${env.GIT_COMMIT}
              docker push ${IMAGE}:latest
            """
        }
    }

    stage("Publish Helm chart") {

      agent {
          docker { 
            // jenkins start container with jenkins user, we must force to run as root
            image 'dtzar/helm-kubectl:3.1.2'
            args '-u 0:0'
          }
      }      
      steps {
          sh '''
              apk add git py-pip
              pip install yamllint=='1.8.1'
              export VERSION="newest"
              export YAMLLINT_PATH=$(pwd)
              cd k8s/charts/webapp
              HELM_EXPERIMENTAL_OCI=1 helm chart save . ${CI_REGISTRY}/webapp/${REPOSITORY}:${VERSION}

          '''
        }
    }

    stage("Deploy") {

      agent {
          docker { 
            // jenkins start container with jenkins user, we must force to run as root
            image 'dtzar/helm-kubectl:3.1.2'
            args '-u 0:0'
          }
      }
      steps {
          sh '''
              apk add --update curl
              curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
              chmod +x ./kubectl
              mv ./kubectl /usr/local/bin/kubectl
              curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.7/2019-03-27/bin/linux/amd64/aws-iam-authenticator
              chmod +x ./aws-iam-authenticator
              mv ./aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
              mkdir -p ~/.kube
              echo "${KUBE_CONFIG}" > ~/.kube/config
              echo 'Deploy Helm release ...'
              |
              helm upgrade --install \
                           --namespace=$NAMESPACE \
                           --set image.repository="$CI_REGISTRY/webapp/$DEPLOYMENT_NAME" \
                           --set image.tag=${BRANCH_NAME}_${GIT_COMMIT} \
                           --set imagePullSecrets[0].name=harbor-secret \
                           --timeout=600s \
                           $DEPLOYMENT_NAME \
                           ./$DEPLOYMENT_NAME
          '''
      }
    }
  }
}