pipeline {
  agent any
  environment {
    DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
    DOCKER_NETWORK = "my-network"
    NGINX_CONTAINER_NAME = "nginx"
    FLASK_CONTAINER_NAME = "flask-app"
    NGINX_PORT = 80
    FLASK_PORT = 5000
  }
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: "build-${env.BUILD_NUMBER}", description: 'Docker image tag to build and push')
    booleanParam(name: 'USE_SLIM', defaultValue: false, description: 'Build slim image?')
  }
  stages {
    stage('Initialise env vars') {
      steps {
        script {
          env.FLASK_IMAGE_NAME = "flask-app:${params.IMAGE_TAG}"
          env.NGINX_IMAGE_NAME = "nginx:${params.IMAGE_TAG}"
          if (params.USE_SLIM) {
            env.FLASK_IMAGE_NAME = "flask-app:${params.IMAGE_TAG}-slim"
            env.NGINX_IMAGE_NAME = "nginx:${params.IMAGE_TAG}-slim"
          }
        }
      }
    }
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
      when {
        expression { return !params.USE_SLIM }
      }
      parallel {
        stage('Build and scan flask-app') {
          steps {
            sh "docker build -t flask-app:${params.IMAGE_TAG} -f flask-app/Dockerfile flask-app"
            sh "trivy image flask-app:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-flask-image-report.json || true"
            //checkImageSize("flask-app:${params.IMAGE_TAG}")
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
            //checkImageSize("nginx:${params.IMAGE_TAG}")
          }
          post {
            always {
                archiveArtifacts artifacts: 'trivy-nginx-image-report.json', onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Build and scan slim images') {
      when {
        expression { return params.USE_SLIM }
      }
      parallel {
        stage('Build and scan slim flask-app') {
          steps {
            sh "slim build -t flask-app:${params.IMAGE_TAG} -f flask-app/Dockerfile flask-app"
            sh "trivy image flask-app:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-flask-image-report.json || true"
            //checkImageSize("flask-app:${params.IMAGE_TAG}")
          }
          post {
            always {
                archiveArtifacts artifacts: 'trivy-flask-image-report.json', onlyIfSuccessful: true
            }
          }
        }
        stage('Build and scan slim nginx') {
          steps {
            sh "slim build -t nginx:${params.IMAGE_TAG} -f nginx/Dockerfile nginx"
            sh "trivy image nginx:${params.IMAGE_TAG} --severity HIGH,CRITICAL --format json --output trivy-nginx-image-report.json || true"
            //checkImageSize("nginx:${params.IMAGE_TAG}")
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
        sh "docker run -d --name ${env.FLASK_CONTAINER_NAME} --network ${env.DOCKER_NETWORK} ${env.FLASK_IMAGE_NAME}"
        sh "docker run -d --name ${env.NGINX_CONTAINER_NAME} --network ${env.DOCKER_NETWORK} -p ${env.NGINX_PORT}:${env.NGINX_PORT} ${env.NGINX_IMAGE_NAME}"
      }
    }
    stage('Smoke test') {
      steps {
        sh "sleep 5"
        sh "curl -f http://localhost:${env.NGINX_PORT} || exit 1"
      }
    }
    stage('Approve push') {
        steps {
            input "Approve pushing images to docker hub?"
        }
    }
    stage('Docker push') {
        steps {
            sh "docker tag flask-app:${params.IMAGE_TAG} $DOCKERHUB_CREDENTIALS_USR/flask-app:${params.IMAGE_TAG}"
            sh "docker tag nginx:${params.IMAGE_TAG} $DOCKERHUB_CREDENTIALS_USR/nginx:${params.IMAGE_TAG}"
            sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            sh "docker push $DOCKERHUB_CREDENTIALS_USR/flask-app:${params.IMAGE_TAG}"
            sh "docker push $DOCKERHUB_CREDENTIALS_USR/nginx:${params.IMAGE_TAG}"
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
  def limitBytes = 200L * 1024 * 1024
  def size = sh(
    returnStdout: true,
    script: "docker image inspect ${imageName} --format='{{.Size}}'"
  ).trim().toLong()

  echo "${imageName} size: ${size} bytes"

  if (size > limitBytes) {
    error "${imageName} is over 200MB"
  }
}
