pipeline {
  agent none

  stages {

    stage('Checkout') {
      agent any
      steps {
        echo '📦 Checking out code from GitHub...'
        checkout scm
        sh 'pwd && ls -la'
      }
    }

    stage('Build & Test Voting App') {
      agent {
        docker {
          image 'maven:3.9.6-eclipse-temurin-17-alpine'
          args '-v $HOME/.m2:/root/.m2'
          reuseNode true
        }
      }
      steps {
        dir('voting') {
          echo '⚙️ Building and testing app...'
          sh 'mvn clean package -DskipTests=false'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Build & Push Docker Image') {
      agent any
      when { branch 'main' }
      steps {
        script {
          // Update image name with your Docker Hub username
          def commitHash = env.GIT_COMMIT?.take(7) ?: "local"
          def imageName = "kmn1624/craftista-voting"

          echo "🐳 Building Docker image: ${imageName}:${commitHash}"

          // Build the Docker image
          sh """
            docker build -t ${imageName}:${commitHash} -t ${imageName}:latest -f voting/Dockerfile voting
          """

          // Push image to Docker Hub using Jenkins credentials
          withDockerRegistry([credentialsId: 'dockerlogin', url: 'https://index.docker.io/v1/']) {
            sh """
              docker push ${imageName}:${commitHash}
              docker push ${imageName}:latest
            """
          }

          echo "✅ Successfully pushed Docker image: ${imageName}:${commitHash}"
        }
      }
    }

    stage('Deploy to Local Container') {
      agent any
      when { branch 'main' }
      steps {
        script {
          echo '🚀 Deploying the app container on this Jenkins EC2 instance...'

          // Stop and remove any previous container
          sh '''
            docker stop craftista-voting 2>/dev/null || true
            docker rm craftista-voting 2>/dev/null || true
          '''

          // Run the latest container
          sh '''
            docker run -d \
              --name craftista-voting \
              -p 8081:8080 \
              --restart unless-stopped \
              kmn1624/craftista-voting:latest
          '''

          echo '🕒 Waiting for the container to start...'
          sleep 15

          // Verify the container is running
          sh 'docker ps | grep craftista-voting || (echo "❌ Container not running!" && exit 1)'

          echo '✅ Deployment successful! Access your app below:'
          echo "🌐 http://<your-public-ip>:8081"
        }
      }
    }
  }

  tools {
    maven 'Maven 3.9.6'
  }

  post {
    always {
      echo '=========================================='
      echo 'Pipeline completed for Craftista Voting!'
      echo '=========================================='
    }
    success {
      echo '✅ BUILD SUCCESSFUL!'
      echo '🌐 Application running at: http://<your-public-ip>:8081'
      echo '🐳 Docker image: kmn1624/craftista-voting:latest'
    }
    failure {
      echo '❌ BUILD FAILED! Check the console output for details.'
    }
  }
}
