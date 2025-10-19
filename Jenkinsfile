// Jenkins Declarative Pipeline for CineVision Microservices Project
// This pipeline handles building all Java microservices, the React frontend,
// SonarQube analysis, and Docker image creation/push to JFrog Artifactory.

pipeline {
    // TEMPORARY FIX: Switched from 'agent docker' to 'agent any'
    // The Frontend Build stage uses a robust Docker container 'inside' block to bypass
    // the broken Jenkins NodeJS plugin and system PATH issues on the agent.
    agent any
    
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
            // FINAL ROBUST FIX: Using docker.image().inside() to execute the Node build reliably.
            agent any
            steps {
                echo 'Building React frontend inside a temporary Docker container...'
                
                // Use the docker.image().inside() block to execute commands within a clean Node environment
                script {
                    docker.image('node:18-alpine').inside('-u root') {
                        // Move into the frontend directory
                        dir('frontend') {
                            // Install dependencies
                            sh 'npm install'
                            // Run the build command
                            sh 'npm run build'
                            // The build artifacts remain in the workspace, ready for the Docker stage.
                        }
                    }
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
                    // The DOCKER_CREDS_ID is used here to securely inject the JFrog token/password
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