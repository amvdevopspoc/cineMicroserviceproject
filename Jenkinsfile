// Jenkins Declarative Pipeline for CineVision Microservices Project
// This pipeline handles building all Java microservices, the React frontend,
// SonarQube analysis, and Docker image creation/push to JFrog Artifactory.

pipeline {
    // Relying on the 'any' agent having Java/Maven/Git/Docker installed
    agent any
    
    // Global parameters and configurations
    environment {
        // --- JFROG ARTIFactory Settings ---
        DOCKER_REPO_HOST = 'akhil15.jfrog.io' 
        ARTY_REPO_KEY    = 'docker-virtual'  
        
        // Credential ID for Docker login
        DOCKER_CREDS_ID  = 'JFROG_DOCKER_CREDS' 
        
        // --- SONARQUBE Settings ---
        SONAR_SERVER     = 'MyCloudSonar'    
        SONAR_PROJECTKEY = 'cinevision-app'  
        SONAR_ORGANIZATION = 'amvdevopspoc'  
    }
    
    // Only execute the pipeline when pushed to the 'dev' branch
    options {
        skipDefaultCheckout() // Will be done manually in the SCM stage
    }

    stages {
        stage('Restrict Branch') {
            when { branch 'dev' }
            steps {
                echo "Pipeline is running on the restricted branch: ${env.BRANCH_NAME}"
            }
        }
        
        stage('SCM Checkout') {
            steps {
                // Checkout the code for the current branch
                checkout scm
            }
        }

        // ----------------------------------------------------------------
        // MOVED STAGE: Frontend Build is now before Backend Build & Test
        // ----------------------------------------------------------------
        stage('Frontend Build') {
            // Using Docker container for build since the NodeJS plugin (withNodeJS) is missing.
            steps {
                echo 'Building React frontend inside a temporary Docker container...'
                script {
                    docker.image('node:18-alpine').inside {
                        dir('frontend') {
                            // FIX: Force clean the cache and address the EACCES error by using a local cache dir.
                            // The path is relative to the dir('frontend') block's current working directory.
                            sh 'npm cache clean --force' 
                            // Using --cache to avoid the "EACCES: permission denied, mkdir '/.npm'" error
                            sh 'npm install --cache ./.npm-cache' 
                            sh 'npm run build'
                        }
                    }
                }
            }
        }
        // ----------------------------------------------------------------
        
        stage('Backend Build & Test') {
            steps {
                echo 'Building all Java microservices with Maven...'
                // Clean and compile all Java services
                sh 'mvn clean install -DskipTests'
                
                // Run unit tests
                sh 'mvn test'
                
                // Archive test results
                junit '**/target/surefire-reports/TEST-*.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis on all backend modules...'
                withSonarQubeEnv(env.SONAR_SERVER) {
                    // Execute SonarQube analysis from the root directory to analyze all modules
                    sh "mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=${env.SONAR_PROJECTKEY} -Dsonar.organization=${env.SONAR_ORGANIZATION}"
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def services = [
                        'eureka-server', 
                        'api-gateway', 
                        'movie-service', 
                        'user-service', 
                        'frontend' // CineVision React App (built previously)
                    ]
                    
                    // Get the short Git commit hash for the image tag
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    def tagName = "latest-${gitCommit}"
                    
                    // Use a common function to handle Docker login and push
                    withDockerRegistry(credentialsId: env.DOCKER_CREDS_ID, url: "https://${env.DOCKER_REPO_HOST}") {
                        for (int i = 0; i < services.size(); i++) {
                            def serviceName = services[i]
                            // Full repository path including the host and the virtual repo key
                            def imagePath = "${env.DOCKER_REPO_HOST}/${env.ARTY_REPO_KEY}/${serviceName}"
                            
                            echo "--- Building and Pushing: ${imagePath}:${tagName} ---"
                            
                            // Build the image. Context must be the service directory.
                            def dockerImage = docker.build("${imagePath}:${tagName}", "-f ${serviceName}/Dockerfile ${serviceName}")
                            
                            // Push the specific tag
                            dockerImage.push()
                            
                            // Also push as 'latest'
                            dockerImage.push('latest')
                        }
                    }
                }
            }
        }
    }
}