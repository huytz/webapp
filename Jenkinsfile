pipeline {

  agent any

  parameters {
    string (
      defaultValue: 'origin/master',
      description: 'Git branch to build and deploy.',
      name : 'GIT_BRANCH_NAME')
  }

  environment {

      REPOSITORY = "communication-api"

      CI_REGISTRY_USER = credentials('docker-hub-user')
      
      CI_REGISTRY_PASSWORD =  credentials('docker-hub-pass')

      CI_PROJECT_PATH = "communication-api"
      
      CI_REGISTRY = "http://chart-registry"

      USERNAME_HARBOR = ""
      
      PASSWORD_HARBOR = ""

      IMAGE = "${CI_REGISTRY}/${CI_PROJECT_PATH}"

      NAMESPACE = "default"
      
      DEPLOYMENT_NAME = "communication-api"

      KUBE_CONFIG = credentials('kubeconfig')

  }

  stages {
    
    stage("CodeAnalytics") {
      agent {
          docker { 
            // jenkins start container with jenkins user, we must force to run as root
            image 'mcr.microsoft.com/dotnet/core/sdk:3.1.301'
            args '-u 0:0 --link sonarqube:sonarqube'
          }
      }
      steps {
        sh '''
          export PATH="$PATH:/root/.dotnet/tools"
          apt-get update -qq && apt-get install -y software-properties-common openjdk-11-jdk
          dotnet --info
          dotnet restore
          dotnet tool install --global dotnet-sonarscanner
          dotnet tool install --global dotnet-reportgenerator-globaltool
          
          cd src/Communication/S5E.Communication.Api
          dotnet sonarscanner begin /k:"$projectKey" /d:sonar.host.url="$sonarUrl" /d:sonar.login="$sonarLogin" /d:sonar.cs.opencover.reportsPaths=$(pwd)/coverage/coverage.opencover-*.xml /d:sonar.coverage.exclusions="**Tests*.csproj"
          dotnet build
          dotnet add package coverlet.msbuild --version 2.8.0
          dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover /p:CoverletOutput=$(pwd)/coverage/coverage.opencover-communication.xml --no-build
          ls -al $(pwd)/coverage
          reportgenerator "-reports:$(pwd)/coverage/coverage.opencover-*.xml" "-targetdir:coverage/Cobertura" "-reporttypes:Cobertura;HTMLInline;HTMLChart"
          dotnet sonarscanner end /d:sonar.login="$sonarLogin"
        '''
      }
                
    }
    stage("Build") {
      steps {
            sh """
              docker login -u ${CI_REGISTRY_USER} --password ${CI_REGISTRY_PASSWORD} ${CI_REGISTRY}
              echo 'Running build-communication-api...'
              docker pull ${IMAGE}:latest || true
              docker build --cache-from ${IMAGE}:latest -f src/Communication/S5E.Communication.Api/Dockerfile -t ${IMAGE}:latest -t ${IMAGE}:${env.GIT_BRANCH}_${env.GIT_COMMIT} .
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
              cd k8s/charts/communication-api
              helm lint .
              yamllint -c ${YAMLLINT_PATH}/k8s/.yamllint.yml -s $(find . -type f -name "Chart.yaml") || true
              yamllint -c ${YAMLLINT_PATH}/k8s/.yamllint.yml -s $(find . -type f -name "values.yaml") || true
              HELM_EXPERIMENTAL_OCI=1 helm chart save . ${CI_REGISTRY}/stepone/${REPOSITORY}:${VERSION}
              HELM_EXPERIMENTAL_OCI=1 helm chart push ${CI_REGISTRY}/stepone/${REPOSITORY}:${VERSION}
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
              HELM_EXPERIMENTAL_OCI=1 helm registry login $CI_REGISTRY -u $USERNAME_HARBOR -p $PASSWORD_HARBOR
              echo 'Deploy Helm release ...'
              HELM_EXPERIMENTAL_OCI=1 helm chart pull $CI_REGISTRY/stepone/$DEPLOYMENT_NAME:newest
              HELM_EXPERIMENTAL_OCI=1 helm chart export $CI_REGISTRY/stepone/$DEPLOYMENT_NAME:newest
              |
              helm upgrade --install \
                           --namespace=$NAMESPACE \
                           --set image.repository="$CI_REGISTRY/stepone/$DEPLOYMENT_NAME" \
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