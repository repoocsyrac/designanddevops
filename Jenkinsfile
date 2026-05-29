pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: "build-${env.BUILD_NUMBER}", description: 'Docker image tag to build and push')
  }
  stages {
    stage('Docker cleanup') {
      steps {
        sh 'docker rm -f $(docker ps -aq) || true'
        sh 'docker rmi -f $(docker images -aq) || true'
        sh 'docker network rm my-network || true'
      }
    }
    stage('Run Trivy filesystem scan') {
      steps {
        sh "trivy fs --format json --output trivy-fs-report.json . || true"
      }
      post {
        always {
            archiveArtifacts artifacts: 'trivy-fs-report.json', onlyIfSuccessful: true
        }
      }
    }
    stage('Docker setup'){
      steps {
        sh "docker network create my-network || true"
      }
    }
    stage('Docker build flask-app') {
      steps {
        sh "docker build -t flask-app:${params.IMAGE_TAG} -f flask-app/Dockerfile flask-app"
      }
    }
    stage('Docker build nginx') {
      steps {
        sh "docker build -t nginx:${params.IMAGE_TAG} -f nginx/Dockerfile nginx"
      }
    }
    stage('Run Trivy image scan') {
      steps {
        sh "trivy image flask-app:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-flask-image-report.json || true"
        sh "trivy image nginx:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-nginx-image-report.json || true"
      }
      post {
        always {
            archiveArtifacts artifacts: 'trivy-flask-image-report.json', onlyIfSuccessful: true
            archiveArtifacts artifacts: 'trivy-nginx-image-report.json', onlyIfSuccessful: true
        }
      }
    }
    stage('Approve run containers') {
        steps {
            input "WARNING:don't approve unless scan results have been checked."
        }
    }
    stage('Docker run') {
      steps {
        sh "docker run -d --name flask-app --network my-network flask-app:${params.IMAGE_TAG}"
        sh "docker run -d --name nginx --network my-network -p 80:80 nginx:${params.IMAGE_TAG}"
      }
    }
    stage('Run unit tests') {
      steps {
          script {
              catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                  sh '''
                      python3 -m venv .venv
                      . .venv/bin/activate
                      pip install -r tests/requirements.txt
                      python3 -m unittest discover -s tests
                      deactivate
                  '''
              }
          }
      }
    }
    stage('Smoke test') {
      steps {
        sh "sleep 5"
        sh "curl -f http://localhost:80 || exit 1"
      }
    }
    stage('Approve push') {
        steps {
            input "Approve pushing images to docker hub?"
        }
    }
    stage('Docker push') {
        steps {
            sh "docker tag flask-app:${params.IMAGE_TAG} syraccc/flask-app:${params.IMAGE_TAG}"
            sh "docker tag nginx:${params.IMAGE_TAG} syraccc/nginx:${params.IMAGE_TAG}"
            withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                  usernameVariable: 'DOCKER_USERNAME',
                                  passwordVariable: 'DOCKER_PASSWORD')]) {
              sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
              sh "docker push syraccc/flask-app:${params.IMAGE_TAG}"
              sh "docker push syraccc/nginx:${params.IMAGE_TAG}"
              sh 'docker logout'
            }
        }
    }
    
  }
  post {
    always {
      sh 'docker rm -f $(docker ps -aq) || true'
      sh 'docker rmi -f $(docker images -aq) || true'
      sh 'docker network rm my-network || true'
    }
  }
}
