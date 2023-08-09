pipeline {
  agent any
  tools {
    maven 'M3'
    jdk 'JDK11'
  }

  environment {
    AWS_CREDENTIALS_NAME = "AWSCredentials"
    REGION = "ap-northeast-2"
    DOCKER_IMAGE_NAME = "project02-spring-petclinic"
    DOCKER_TAG = "1.0"
    ECR_REPOSITORY = "257307634175.dkr.ecr.ap-northeast-2.amazonaws.com"
    APPLICATION_NAME = "project02-production-in-place"
    DEPLOYMENT_GROUP_NAME = "project02-production-in-place"
    ECR_DOCKER_IMAGE = "${ECR_REPOSITORY}/${DOCKER_IMAGE_NAME}"
    ECR_DOCKER_TAG = "${DOCKER_TAG}"
    DEPLOY_CONFIG = "CodeDeployDefault.OneAtATime"
  }
  
  stages {
    stage('Git Clone') {
      steps {
        git url: 'https://github.com/sfunzbob7/spring-petclinic.git', branch: 'efficient-webjars', credentialsId: 'project02_git_accept'
      }
    }
    stage('mvn build') {
      steps {
        sh 'mvn -Dmaven.test.failure.ignore=true install'
      }
      post {
        success {
          junit '**/target/surefire-reports/TEST-*.xml'
        }
      }
    }
    stage('Docker Image Build') {
      steps {
        dir("${env.WORKSPACE}") {
          sh 'docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_TAG} .'
        }
      }
    }
    stage('Push Docker Image') {
      steps {
        script {
          sh 'rm -f ~/.dockercfg ~/.docker/config.json || true'
          docker.withRegistry("https://${ECR_REPOSITORY}", "ecr:${REGION}:${AWS_CREDENTIALS_NAME}") {
            docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_TAG}").push()
          }
        }
      }
    }
    stage('Upload to S3') {
      steps {
        dir("${env.WORKSPACE}") {
          sh 'zip -r deploy-1.0.zip ./scripts appspec.yml'
          sh 'aws s3 cp --region ap-northeast-2 --acl private ./deploy-1.0.zip s3://project02-terraform-status'
          sh 'rm -rf ./deploy-1.0.zip'
        }
      }
    }
    stage('CodeDeploy Deploy') {
      steps {
        script {
          sh 'aws deploy delete-application --application-name project02-production-in-place'
          sh 'aws deploy create-application --application-name project02-production-in-place'
          sh 'aws deploy create-deployment-group \
                  --application-name project02-production-in-place \
                  --auto-scaling-groups PROJECT02-AUTOSCALING-GROUP \
                  --deployment-config-name CodeDeployDefault.OneAtATime \
                  --deployment-group-name project02-production-in-place \
                  --service-role-arn arn:aws:iam::257307634175:role/project02-code-deploy-service-role \
                  --load-balancer-info elbInfoList="arn:aws:elasticloadbalancing:ap-northeast-2:257307634175:loadbalancer/app/project02-lb/09ae2513820d4021",targetGroupInfoList="arn:aws:elasticloadbalancing:ap-northeast-2:257307634175:targetgroup/project02-target-group/b4acc958d48b58f6"'
          sh 'aws deploy create-deployment \
                  --application-name project02-production-in-place \
                  --deployment-config-name CodeDeployDefault.OneAtATime \
                  --deployment-group-name project02-production-in-place \
                  --description "My deployment" \
                  --github-location repository=sfunzbob7/spring-petclinic,commitId="08ce42a70a168bcd76c7a57dacf694366ced3aaf"'
        }
      }
    }
  }
}
