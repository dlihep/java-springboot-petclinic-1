pipeline {
    agent any
    tools {
        maven 'maven3.8'
        jdk 'jdk8'
    }
    environment { 
        AWS_REGION = 'us-east-1'
        ECRREGISTRY = '579036934425.dkr.ecr.us-east-1.amazonaws.com'
        IMAGENAME = 'dokoimage'
        IMAGE_TAG = 'latest'
        ECS_CLUSTER = 'dokocluster'
        ECS_SERVICE = 'dokoservice'
    }
    stages {
       stage ('Clone') {
          steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
        }
        stage('Unit Tests') {
            steps {
                sh 'mvn surefire:test'
            }
        }
        stage("build & SonarQube analysis") {
            agent any
            steps {
              withSonarQubeEnv('sonarserver') {
                sh 'mvn clean package sonar:sonar'
              }
            }
          }

         stage('Deployment Approval') {
            steps {
              script {
                timeout(time: 10, unit: 'MINUTES') {
                 input(id: 'Deploy Gate', message: 'Deploy Application to Dev ?', ok: 'Deploy')
                 }
               }
            }
         }   
        
         stage('AWS ecr login') {
            steps {
                sh 'alias aws='docker run --rm -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY amazon/aws-cli' aws ecr get-login-password \ --region <region> \docker login \ --username AWS \
    --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com'
            }
        }        
         stage('docker build and tag') {
            steps {
                sh 'docker build -t ${IMAGENAME}:${IMAGE_TAG} .'
                sh 'docker tag ${IMAGENAME}:${IMAGE_TAG} ${ECRREGISTRY}/${IMAGENAME}:${IMAGE_TAG}'
            }
        }  
         stage('docker push') {
            steps {
                sh 'docker push ${ECRREGISTRY}/${IMAGENAME}:${IMAGE_TAG}'
            }
        }                
       
        
         stage('update ecs service') {
            steps {
                sh 'aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment --region ${AWS_REGION}'
            }
        }            
        
         stage('wait ecs service stable') {
            steps {
                sh 'aws ecs wait services-stable --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --region ${AWS_REGION}'
            }
        }                    
    }
    post {
        always {
            junit 'target/surefire-reports/TEST-*.xml'
            deleteDir()
        }
    }
}
