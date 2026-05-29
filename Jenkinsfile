pipeline {
  agent any
  environment {
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
  }
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: "build-${env.BUILD_NUMBER}", description: 'Docker image tag to build and push')
  }
  stages {
    stage('Run tests') {
      parallel {
        stage('Run unit tests') {
          steps {
              script {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                      sh '''
                          python3 -m venv .venv
                          . .venv/bin/activate
                          cd flask-app
                          pip install -r requirements.txt
                          python3 -m unittest discover -s tests
                          deactivate
                      '''
                }
              }
          }
        }
        stage('Run trivy filesystem scan') {
          steps {
            sh "trivy fs --format json --output trivy-fs-report.json . || true"
          }
          post {
            always {
                archiveArtifacts artifacts: 'trivy-fs-report.json', onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Docker cleanup') {
      steps {
        sh 'docker rm -f $(docker ps -aq) || true'
        sh 'docker rmi -f $(docker images -aq) || true'
        sh 'docker network rm my-network || true'
      }
    }
    stage('Docker setup'){
      steps {
        sh "docker network create my-network || true"
      }
    }
    stage('Build and scan images') {
      parallel {
        stage('Build and scan flask-app') {
          steps {
            sh "docker build -t flask-app:${params.IMAGE_TAG} -f flask-app/Dockerfile flask-app"
            sh "trivy image flask-app:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-flask-image-report.json || true"
            checkImageSize("flask-app:${params.IMAGE_TAG}")
          }
          post {
            always {
                archiveArtifacts artifacts: 'trivy-flask-image-report.json', onlyIfSuccessful: true
            }
          }
        }
        stage('Build and scan nginx') {
          steps {
            sh "docker build -t nginx:${params.IMAGE_TAG} -f nginx/Dockerfile nginx"
            sh "trivy image nginx:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-nginx-image-report.json || true"
            checkImageSize("nginx:${params.IMAGE_TAG}")
          }
          post {
            always {
                archiveArtifacts artifacts: 'trivy-nginx-image-report.json', onlyIfSuccessful: true
            }
          }
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
            sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            sh "docker push syraccc/flask-app:${params.IMAGE_TAG}"
            sh "docker push syraccc/nginx:${params.IMAGE_TAG}"
            sh 'docker logout'

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

def checkImageSize = { imageName ->
  def limitBytes = 200 * 1024 * 1024
  def size = sh(
    returnStdout: true,
    script: "docker image inspect ${imageName} --format='{{.Size}}'"
  ).trim().toInteger()

  echo "${imageName} size: ${size} bytes"

  if (size > limitBytes) {
    error "${imageName} is over 200MB"
  }
}
