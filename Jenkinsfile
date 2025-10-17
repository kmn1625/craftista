pipeline {
  agent none
  
  stages {
    stage('Checkout') {
      agent any
      steps {
        echo 'Checking out code from GitHub...'
        checkout scm
        
        script {
          // Debug: Show workspace contents
          sh 'pwd'
          sh 'ls -la'
          sh 'ls -la voting/'
          sh 'cat voting/pom.xml | head -20'
        }
      }
    }
    
    stage('Voting Build') {
      agent {
        docker {
          image 'maven:3.9.6-eclipse-temurin-17-alpine'
          args '-v $HOME/.m2:/root/.m2'
          reuseNode true
        }
      }
      steps {
        echo 'Compiling the voting app...'
        dir('voting') {
          sh 'pwd'
          sh 'ls -la'
          sh 'mvn compile'
        }
      }
    }
    
    stage('Voting Test') {
      agent {
        docker {
          image 'maven:3.9.6-eclipse-temurin-17-alpine'
          args '-v $HOME/.m2:/root/.m2'
          reuseNode true
        }
      }
      steps {
        echo 'Running tests...'
        dir('voting') {
          sh 'mvn clean test'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }
    
    stage('Voting Package') {
      parallel {
        stage('Maven Package') {
          agent {
            docker {
              image 'maven:3.9.6-eclipse-temurin-17-alpine'
              args '-v $HOME/.m2:/root/.m2'
              reuseNode true
            }
          }
          when { 
            branch 'main'
          }
          steps {
            echo 'Packaging the voting app...'
            dir('voting') {
              sh 'mvn package -DskipTests'
              sh 'ls -la target/'
            }
          }
          post {
            success {
              archiveArtifacts artifacts: 'voting/target/*.jar', fingerprint: true
            }
          }
        }
        
        stage('Voting Image B&P') {
          agent any
          when { 
            branch 'main'
          }
          steps {
            script {
              echo 'Building and pushing Docker image...'
              
              docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                def commitHash = env.GIT_COMMIT.take(7)
                def imageName = 'initcron/craftista-voting'
                
                echo "Building image: ${imageName}:${commitHash}"
                
                def dockerImage = docker.build("${imageName}:${commitHash}", "./voting")
                
                echo 'Pushing images to Docker Hub...'
                dockerImage.push()
                dockerImage.push('latest')
                dockerImage.push('dev')
                
                echo "Successfully pushed ${imageName}:${commitHash}, latest, and dev"
              }
            }
          }
        }
      }
    }
    
    stage('Deploy to Container') {
      agent any
      when { 
        branch 'main'
      }
      steps {
        script {
          echo 'Deploying voting app to container...'
          
          // Stop and remove old container if it exists
          sh '''
            docker stop craftista-voting 2>/dev/null || true
            docker rm craftista-voting 2>/dev/null || true
          '''
          
          // Run new container
          sh '''
            docker run -d \
              --name craftista-voting \
              -p 8081:8080 \
              --restart unless-stopped \
              initcron/craftista-voting:latest
          '''
          
          echo 'Waiting for application to start...'
          sleep 15
          
          // Verify container is running
          sh 'docker ps | grep craftista-voting'
          
          echo '✅ Application deployed successfully!'
          echo '📍 Access the voting app at: http://localhost:8081'
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
      echo '🚀 Application is running at: http://localhost:8081'
      echo '🐳 Docker image: initcron/craftista-voting:latest'
    }
    failure {
      echo '❌ BUILD FAILED!'
      echo 'Check the console output above for errors'
    }
  }
}
