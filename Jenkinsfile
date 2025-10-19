// Jenkins Declarative Pipeline for CineVision Microservices Project
// This pipeline handles building all Java microservices, the React frontend,
// SonarQube analysis, and Docker image creation/push to JFrog Artifactory.

pipeline {
    // TEMPORARY FIX: Switched from 'agent docker' to 'agent any'
    // You MUST install the 'Pipeline: Declarative Agent Docker' plugin and restart Jenkins 
    // to use the original 'agent docker' configuration.
    agent any
    
    // Global parameters and configurations
    environment {
        // --- JFROG ARTIFactory Settings ---
        DOCKER_REPO_HOST = 'jfrog-repo.yourcompany.com' // Replace with your Artifactory hostname
        ARTY_REPO_KEY    = 'docker-virtual'              // Virtual repository key created in Artifactory
        // Based on the JFrog login, the username is likely the email, 
        // so the token/password should be stored as the credential ID below.
        DOCKER_CREDS_ID  = 'JFROG_DOCKER_CREDS'          // Jenkins Credential ID for Docker login (Username/Password)
        
        // --- SONARQUBE Settings ---
        SONAR_SERVER      = 'MyCloudSonar'                  // FIXED: Now correctly matches the name from the Jenkins configuration (image_f38447.png)
        SONAR_PROJECTKEY = 'cinevision-app'              // Project Key used in SonarQube UI
        SONAR_ORGANIZATION = 'amvdevopspoc'             // Organization Key if using SonarQube Cloud
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
                // This previous REMOVED comment is no longer relevant as we are using the tool block below.
            }
        }

        stage('Backend Build & Test') {
            // WARN: This stage still requires 'mvn' to be available on the 'any' agent.
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
            // WARN: This stage still requires 'mvn' to be available on the 'any' agent.
            steps {
                echo 'Running SonarQube analysis on all backend modules...'
                withSonarQubeEnv(env.SONAR_SERVER) {
                    // Execute SonarQube analysis from the root directory to analyze all modules
                    sh "mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=${env.SONAR_PROJECTKEY} -Dsonar.organization=${env.SONAR_ORGANIZATION}"
                }
            }
        }

        stage('Frontend Build') {
            // CRITICAL FIX: The Jenkins 'tool' mechanism is failing despite correct naming.
            // We are switching this stage to use a 'docker' agent to guarantee a clean environment 
            // with NodeJS available, bypassing the tool configuration issue entirely.
            // This stage builds the CineVision React application which uses Redux, Router, and Axios.
            agent {
                docker {
                    image 'node:18-alpine' // A light-weight image with NodeJS 18 and npm
                    args '-u root:root' // Ensures permissions are adequate for npm install
                }
            }
            steps {
                echo 'Building React frontend inside node:18-alpine Docker container...'
                // The 'tool' directive is no longer necessary as it's provided by the Docker image
                dir('frontend') { // Assumes frontend code is in a 'frontend' sub-directory
                    sh 'npm install'
                    sh 'npm run build' // Creates the production-ready build directory
                }
            }
        }
        
        stage('Docker Build & Push') {
            // WARN: This stage still requires the Docker daemon and client to be available on the 'any' agent.
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
                            def imagePath = "${env.DOCKER_REPO_HOST}/${env.ARTY_REPO_KEY}/${serviceName}"
                            
                            echo "--- Building and Pushing: ${imagePath}:${tagName} ---"
                            
                            // Build the image. For the 'frontend', this step uses the 'frontend/Dockerfile' 
                            // which should handle packaging the built React assets (from the previous stage) 
                            // into a production web server image (e.g., Nginx).
                            def dockerImage = docker.build("${imagePath}:${tagName}", "-f ${serviceName}/Dockerfile ${serviceName}")
                            
                            // Push the specific tag
                            dockerImage.push()
                            
                            // Also push as 'latest' for easy deployment updates (optional, but common)
                            dockerImage.push('latest')
                        }
                    }
                }
            }
        }
    }
}