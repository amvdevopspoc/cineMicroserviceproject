// Jenkins Declarative Pipeline for CineVision Microservices Project
// This pipeline handles building all Java microservices, the React frontend,
// SonarQube analysis, and Docker image creation/push to JFrog Artifactory.
// Updated: Uses existing 'NodeJS' tool; manual Maven install for compatibility.
// Assumes NodeJS Plugin installed; Maven downloaded if missing on agent.

pipeline {
    // Using 'agent any' for flexibility (requires Docker/Git/Java on agent)
    agent any
    
    // Tools: Use existing NodeJS config (no Maven tool; handle manually)
    tools {
        nodejs 'NodeJS'  // Matches Jenkins suggestion; adds npm to PATH
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
        
        // Maven path (set after install)
        MAVEN_HOME = '/opt/maven'
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
                // Manual Maven setup (idempotent; only if not present)
                sh '''
                    if ! command -v mvn &> /dev/null; then
                        echo "Maven not found; installing..."
                        wget -q https://archive.apache.org/dist/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
                        tar xzvf apache-maven-3.9.6-bin.tar.gz
                        sudo mkdir -p /opt/maven || true
                        sudo mv apache-maven-3.9.6/* /opt/maven/ || true
                        sudo chown -R $(whoami) /opt/maven || true
                        export PATH=${MAVEN_HOME}/bin:$PATH
                        echo "Maven version: $(mvn --version)"
                    else
                        echo "Maven already available: $(mvn --version)"
                        export PATH=${MAVEN_HOME}/bin:$PATH  # Ensure path if custom install
                    fi
                '''
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
                    // Ensure Maven PATH (from previous stage or repeat)
                    sh '''
                        export PATH=${MAVEN_HOME}/bin:$PATH
                        # Use sonar:sonar goal to avoid re-running tests (executes from root for multi-module)
                        mvn sonar:sonar -Dsonar.projectKey=${SONAR_PROJECTKEY} -Dsonar.organization=${SONAR_ORGANIZATION}
                    '''
                }
            }
        }

        stage('Frontend Build') {
            steps {
                echo 'Building React frontend with npm...'
                // NodeJS tool adds to PATH; assumes package.json in 'frontend' dir
                sh '''
                    cd frontend  # Adjust if your React app is in root or elsewhere
                    npm install
                    npm run build  # Production build (outputs to /build)
                '''
                // Archive frontend artifacts
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
                    
                    // Use withDockerRegistry for secure login
                    withDockerRegistry(credentialsId: env.DOCKER_CREDS_ID, url: "https://${env.DOCKER_REPO_HOST}") {
                        for (int i = 0; i < services.size(); i++) {
                            def serviceName = services[i]
                            // Full repository path including the host and the virtual repo key
                            def imagePath = "${env.DOCKER_REPO_HOST}/${env.ARTY_REPO_KEY}/${serviceName}"
                            def fullImageTag = "${imagePath}:${tagName}"
                            
                            echo "--- Building and Pushing: ${fullImageTag} ---"
                            
                            // Build the image from service directory (assumes Dockerfile exists)
                            def dockerImage = docker.build(fullImageTag, "-f ${serviceName}/Dockerfile ${serviceName}")
                            
                            // Push the specific tag
                            dockerImage.push()
                            
                            // Push as 'latest' for dev branch
                            if (env.BRANCH_NAME == 'dev') {
                                dockerImage.push('latest')
                            }
                            
                            // Cleanup local image to save space
                            dockerImage.inside { sh 'rm -rf /tmp/*' }  // Optional: Clean inside if needed
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
            // Optional: Remove manual Maven install if in /opt
            sh 'sudo rm -rf /opt/maven || true'
        }
        success {
            echo 'Pipeline completed successfully! Images pushed to Artifactory.'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
            // Optional: Add emailext body: 'Build failed: ${BUILD_URL}', to: 'team@example.com'
        }
    }
}