// Jenkins Declarative Pipeline for CineVision Microservices Project
// This pipeline handles building all Java microservices, the React frontend,
// SonarQube analysis, and Docker image creation/push to JFrog Artifactory.

pipeline {
    // TEMPORARY FIX: Switched from 'agent docker' to 'agent any'
    // We are relying on the 'any' agent having Java/Maven/Git/Docker installed, 
    // and we will attempt to use the Jenkins tool step for NodeJS one last time.
    agent any
    
    // Enable tools for Maven and NodeJS (requires plugins configured in Global Tools)
    tools {
        maven 'Maven3'    // Configure in Manage Jenkins > Global Tool Configuration
        nodejs 'Node18'   // Configure in Manage Jenkins > Global Tool Configuration
    }
    
    // Global parameters and configurations
    environment {
        // --- JFROG ARTIFactory Settings ---
        // Based on the login URL from image_ff65e5.png (akhil15.jfrog.io)
        DOCKER_REPO_HOST = 'akhil15.jfrog.io' // Replace with your Artifactory hostname (if different)
        ARTY_REPO_KEY    = 'docker-virtual'  // Virtual repository key confirmed from image_fe6a5f.png
        
        // Credential ID for Docker login (Username/Password or Token)
        // This ID MUST match the Jenkins credential record storing 'vakhil.kumar@sas.ken.com' and the associated Identity Token (image_fe6246.png)
        DOCKER_CREDS_ID  = 'JFROG_DOCKER_CREDS' 
        
        // --- SONARQUBE Settings ---
        SONAR_SERVER     = 'MyCloudSonar'    // Name from Jenkins configuration (image_f38447.png)
        SONAR_PROJECTKEY = 'cinevision-app'  // Project Key used in SonarQube UI (image_fe04ca.png)
        SONAR_ORGANIZATION = 'amvdevopspoc'  // Organization Key (image_fe04ca.png)
    }
    
    // Only execute the pipeline when pushed to the 'dev' branch
    options {
        skipDefaultCheckout() // Will be done manually in the SCM stage
        buildDiscarder(logRotator(numToKeepStr: '10')) // Keep last 10 builds
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

        stage('Backend Build & Test') {
            steps {
                echo 'Building all Java microservices with Maven...'
                // Clean and compile all Java services (using tool for PATH)
                sh "${tool 'Maven3'}/bin/mvn clean install -DskipTests"
                
                // Run unit tests
                sh "${tool 'Maven3'}/bin/mvn test"
                
                // Archive test results
                junit '**/target/surefire-reports/TEST-*.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis on all backend modules...'
                withSonarQubeEnv(env.SONAR_SERVER) {
                    // Use sonar:sonar goal to avoid re-running tests (executes from root for multi-module)
                    sh "${tool 'Maven3'}/bin/mvn sonar:sonar -Dsonar.projectKey=${env.SONAR_PROJECTKEY} -Dsonar.organization=${env.SONAR_ORGANIZATION}"
                }
            }
        }

        stage('Frontend Build') {
            steps {
                echo 'Building React frontend with npm...'
                // Assumes package.json in 'frontend' dir; NodeJS tool adds to PATH
                sh '''
                    cd frontend  # Adjust if your React app is in root or elsewhere
                    npm install
                    npm run build  # Add this for production build (outputs to /build)
                '''
                // Optional: Archive artifacts
                archiveArtifacts artifacts: 'frontend/build/**', allowEmptyArchive: true
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
                        'frontend' // CineVision React App (built previously; serve static files)
                    ]
                    
                    // Get the short Git commit hash for the image tag
                    def gitCommit = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    def tagName = "${env.BRANCH_NAME}-${gitCommit}"  // e.g., dev-abc123 for better tracking
                    
                    // Use a common function to handle Docker login and push
                    // The DOCKER_CREDS_ID is used here to securely inject the JFrog token/password
                    withDockerRegistry(credentialsId: env.DOCKER_CREDS_ID, url: "https://${env.DOCKER_REPO_HOST}") {
                        for (int i = 0; i < services.size(); i++) {
                            def serviceName = services[i]
                            // Full repository path including the host and the virtual repo key
                            def imagePath = "${env.DOCKER_REPO_HOST}/${env.ARTY_REPO_KEY}/${serviceName}"
                            def fullImageTag = "${imagePath}:${tagName}"
                            
                            echo "--- Building and Pushing: ${fullImageTag} ---"
                            
                            // Build the image. Context must be the service directory.
                            // Ensure Docker socket is accessible (fix permissions if needed)
                            def dockerImage = docker.build(fullImageTag, "-f ${serviceName}/Dockerfile ${serviceName}")
                            
                            // Push the specific tag
                            dockerImage.push()
                            
                            // Push as 'latest' for dev branch
                            if (env.BRANCH_NAME == 'dev') {
                                dockerImage.push('latest')
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean workspace to save disk space
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully! Images pushed to Artifactory.'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
            // Optional: emailext or slackSend here
        }
    }
}