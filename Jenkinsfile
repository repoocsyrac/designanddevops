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
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Docker cleanup') {
      steps {
        sh 'docker rm -f $(docker ps -aq) || true'
        sh 'docker rmi -f $(docker images -aq) || true'
        sh 'docker network rm my-network || true'
      }
    }
    stage('Run tests') {
      steps {
        sh "pwd"
      }
    }
    stage('Run security check') {
      steps {
        sh "pwd"
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
    stage('Docker run') {
      steps {
        sh "docker run -d --name flask-app --network my-network flask-app:${params.IMAGE_TAG}"
        sh "docker run -d --name nginx --network my-network -p 80:80 nginx:${params.IMAGE_TAG}"
      }
    }
    stage('Smoke test') {
      steps {
        sh "sleep 5"
        sh "curl -f http://localhost:5500 || exit 1"
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
            sh "docker push syraccc/flask-app:${params.IMAGE_TAG}"
            sh "docker push syraccc/nginx:${params.IMAGE_TAG}"
        }
    }
    
  }
}
