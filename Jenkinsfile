pipeline {

  agent none

  stages{

    // Worker app
    stage("worker-build"){
      agent{
        docker{
          image 'maven:3.9.8-sapmachine-21'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      when{
          changeset "**/worker/**"
      }
      steps{
        echo 'Compiling worker app..'
        dir(path: 'worker'){
          sh 'mvn compile'
        }
      }
    }

    stage("worker-test"){
      agent{
        docker{
          image 'maven:3.9.8-sapmachine-21'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      when{
        changeset "**/worker/**"
      }
      steps{
        echo 'Running Unit Tets on worker app..'
        dir(path: 'worker'){
          sh 'mvn clean test'
        }
      }
    }

    stage('worker-package') {
      agent {
        docker {
          image 'maven:3.9.8-sapmachine-21'
          args '-v $HOME/.m2:/root/.m2'
        }
      }
      when {
        branch 'master'
        changeset '**/worker/**'
      }
      steps {
        echo 'Packaging worker app'
        dir(path: 'worker') {
          sh 'mvn package -DskipTests'
          archiveArtifacts(artifacts: '**/target/*.jar', fingerprint: true)
        }
      }
    }

    stage('worker-docker-package'){
      agent any
      when{
        changeset "**/worker/**"
        branch 'master'
      }
      steps{
        echo 'Packaging worker app with docker'
        script{
          try {
            docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
              def workerImage = docker.build(
                "jcampos15/worker:v${env.BUILD_ID}", "--no-cache ./worker"
              )
              workerImage.push()
              workerImage.push("${env.BRANCH_NAME}")
              workerImage.push("latest")
            }
          } catch (Exception e) {
            echo "❌ Docker build/push failed: ${e.getMessage()}"
            currentBuild.result = 'FAILURE'
          }
        }
      }
    }

    // Vote app
    stage('vote-build'){ 
      agent{
        docker{
          image 'python:3.11-slim'
          args '--user root'
        }
      }
      when {
        changeset '**/vote/**'
      }
      steps{ 
        echo 'Compiling vote app.' 
        dir(path: 'vote'){
          sh "pip install -r requirements.txt"
        } 
      } 
    }

    stage('vote-test'){ 
      agent {
        docker{
          image 'python:3.11-slim'
          args '--user root'
        }
      }
      when {
        changeset '**/vote/**'
      }
      steps{ 
        echo 'Running Unit Tests on vote app.' 
        dir(path: 'vote'){   
          sh "pip install -r requirements.txt"
          sh 'nosetests -v'
        } 
      } 
    }

    stage('vote-integration'){ 
      agent any 
      when{ 
        changeset "**/vote/**" 
        branch 'master'
      } 
      steps{ 
        echo 'Running Integration Tests on vote app' 
        dir(path: 'vote'){ 
          sh 'sh integration_test.sh' 
        }
      } 
    } 

    stage('vote-docker-package'){
      agent any
      steps{
        echo 'Packaging vote app with docker'
        script{
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
            def voteImage = docker.build("jcampos15/vote:v${env.BUILD_ID}", "./vote")
            voteImage.push()
            voteImage.push("${env.BRANCH_NAME}")
            voteImage.push("latest")
          }
        }
      }
    }

    // Result app
    stage('result-build'){
      agent {
        docker {
          image 'node:22.4.0-alpine'
        }
      }
      when {
        changeset "**/result/**"
      }
      steps{ 
        echo 'Compiling result app' 
        dir(path: 'result'){ 
          sh 'npm install'
        } 
      } 
    }

    stage('result-test'){ 
      agent {
        docker {
          image 'node:22.4.0-alpine'
        }
      }
      when {
        changeset "**/result/**"
      }
      steps{ 
        echo 'running unit tests on result app' 
        dir(path: 'result'){ 
          sh 'npm install'
          sh 'npm test'
        } 
      } 
    }

    stage('result-docker-package') {
      agent any
      when {
        changeset '**/result/**'
        branch 'master'
      }
      steps {
        echo 'Packaging result app with docker'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def resultImage = docker.build("jcampos15/result:v${env.BUILD_ID}", './result')
            resultImage.push()
            resultImage.push("${env.BRANCH_NAME}")
            resultImage.push('latest')
          }
        }
      }
    }

    // Deploy all to dev
    stage('deploy to dev'){
      agent any
      when{
        branch 'master'
      }
      steps{
        echo 'Deploy instavote app with docker compose'
        sh 'docker-compose up -d'
      }
    }
  }

  post {
    always{ 
      echo 'Building mono pipeline for voting app is completed.' 
    }
    failure{
      slackSend (channel: "#ci-cd", message: "Mono Pipeline Deploy Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
    }
    success{
      slackSend (channel: "#ci-cd", message: "Mono Pipeline Deploy Success: ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
    }
  }
}
