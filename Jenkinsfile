pipeline {
    
    agent none

    stages {

        stage ("worker build") {
            when {
                changeset "**/worker/**"
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps{
                echo 'Compiling worker app'
                dir ('worker') {
                    sh 'mvn compile'
                }
            }
        }

        stage ("worker test") {
            when {
                changeset "**/worker/**"
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps{
                echo 'Running Unit Tests on worker app'
                dir ('worker') {
                    sh 'mvn clean test'
                }
            }
        }

        stage ("worker package") {
            when {
                branch 'master'
                changeset "**/worker/**"
            }
            agent {
                docker {
                    image 'maven:3.9.8-sapmachine-21'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Packaging worker app into a jar file'
                dir ('worker') {
                    sh 'mvn package -DskipTests'
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }

        stage ("worker docker-package") {
            agent any
            when {
                branch 'master'
                changeset "**/worker/**"
            }
            steps {
                echo 'Packaging worker app with docker'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        def workerImage = docker.build("valandegh1/worker:v${env.BUILD_ID}", "./worker")
                        workerImage.push()
                        workerImage.push("${env.BRANCH_NAME}")
                        workerImage.push("latest")
                    }
                }
            }
        }

        stage("result build") {
            when{
                changeset '**/result/**'
            }
            agent {
                docker {
                    image 'node:22.4.0-slim'
                }
            }
            steps {
                echo 'Compiling result app..'
                dir('result') {
                    sh 'npm install'
                    sh 'npm ls'
                }
            }
        }

        stage("result test") {
            when{
                changeset '**/result/**'
            }
            agent {
                docker {
                    image 'node:22.4.0-slim'
                }
            }
            steps {
                echo 'Running Unit Tests on result app..'
                dir('result') {
                    sh 'npm install'
                    sh 'npm ls'
                    sh 'npm test'
                }
            }
        }

        stage('result docker-package'){
            agent any
            when {
                branch 'master'
                changeset "**/result/**"
            }
            steps{
                echo 'Packaging result app with docker'
                script{
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        // ./result is the path to the Dockerfile that Jenkins will find from the Github repo
                        def resultImage = docker.build("valandegh1/result:v${env.BUILD_ID}", "./result")
                        resultImage.push()
                        resultImage.push("${env.BRANCH_NAME}")
                        resultImage.push("latest")
                    }
                }
            }
        }

        stage('vote build'){ 
            agent{
                docker{
                    image 'python:3.11-slim'
                    args '--user root'
                }
            }
            steps{ 
                echo 'Compiling vote app.' 
                dir('vote'){
                    sh "pip install -r requirements.txt"
                } 
            } 
        } 

        stage('vote test'){ 
            agent {
                docker {
                    image 'python:3.11-slim'
                    args '--user root'
                }
            }
            steps{ 
                echo 'Running Unit Tests on vote app.' 
                dir('vote') {
                    sh "pip install -r requirements.txt"
                    sh 'nosetests -v'
                } 
            } 
        } 

        stage('vote docker-package'){
            agent any
            when {
                branch 'master'
                changeset "**/vote/**"
            }
            steps{
                echo 'Packaging vote app with docker'
                script{
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
                        // ./vote is the path to the Dockerfile that Jenkins will find from the Github repo
                        def voteImage = docker.build("valandegh1/vote:v${env.BUILD_ID}", "./vote")
                        voteImage.push()
                        voteImage.push("${env.BRANCH_NAME}")
                        voteImage.push("latest")
                    }
                }
            }
        }

        stage('Sonarqube') {
            agent any
            when {
                branch 'master'
            }
            environment {
                sonarpath = tool 'SonarScanner'
            }
            steps {
                echo 'Running Sonarqube Analysis...'
                withSonarQubeEnv('conar-instavote') {
                    sh "$(sonarpath)/bin/sonnar-scanner 
                            -Dproject.settings=sonar-project.properties 
                            -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                }
            }
        }

        stage("Quality Gate") {
            agent any
            when {
                branch 'master'
            }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('deploy to dev') {
            agent any
            when {
                branch 'master'
            }
            steps {
                echo 'Deploy instavote app with docker compose'
                sh 'docker compose up -d'
            }
        }

    }

    post {
        always {
            echo 'The job is complete.'
            slackSend (channel: "#ci-cd-jenkins", message: "Build Finished: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
        failure{
            slackSend (channel: "#ci-cd-jenkins", message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
        
        success{
            slackSend (channel: "#ci-cd-jenkins", message: "Build Success: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
    }
}
