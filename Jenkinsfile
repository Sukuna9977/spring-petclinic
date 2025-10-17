pipeline {
    agent any
    
    tools {
        jdk 'jdk17'  // Adjust to your JDK tool name in Jenkins
    }
    
    environment {
        // Define environment variables
        SONAR_PROJECT_KEY = 'my-java-app'
        DOCKER_REGISTRY = 'your-docker-registry'  // Optional for Docker builds
    }
    
    stages {
        // Stage 1: Code Checkout
        stage('Checkout SCM') {
            steps {
                checkout scm
                sh 'git branch --show-current'
                sh 'git log -1 --oneline'
            }
        }
        
        // Stage 2: Dependency Resolution
        stage('Resolve Dependencies') {
            steps {
                sh '''
                    echo "=== Installing Dependencies ==="
                    # For Maven projects
                    mvn dependency:resolve || echo "Maven not available"
                    
                    # For Gradle projects
                    ./gradlew dependencies || echo "Gradle not available"
                    
                    # For Node.js projects
                    npm install || echo "NPM not available"
                '''
            }
        }
        
        // Stage 3: Code Compilation
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    # Maven compilation
                    mvn compile -q || echo "Maven compilation failed or not available"
                    
                    # Gradle compilation
                    ./gradlew classes -q || echo "Gradle compilation failed or not available"
                    
                    # Check if compiled classes exist
                    find . -name "*.class" -o -name "target" -o -name "build" | head -5
                '''
            }
        }
        
        // Stage 4: Unit Tests
        stage('Run Unit Tests') {
            steps {
                sh '''
                    echo "=== Running Unit Tests ==="
                    # Maven tests
                    mvn test -q || echo "Maven tests failed or not available"
                    
                    # Gradle tests
                    ./gradlew test -q || echo "Gradle tests failed or not available"
                    
                    # Node.js tests
                    npm test || echo "NPM tests failed or not available"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'  // Maven test reports
                    junit '**/build/test-results/**/*.xml'   // Gradle test reports
                }
            }
        }
        
        // Stage 5: Code Quality Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN} \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    """
                }
            }
        }
        
        // Stage 6: Build Artifact
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    # Maven package
                    mvn package -DskipTests -q || echo "Maven package failed"
                    
                    # Gradle build
                    ./gradlew build -x test -q || echo "Gradle build failed"
                    
                    # List generated artifacts
                    find . -name "*.jar" -o -name "*.war" -o -name "*.zip" | head -10
                '''
                // Archive the built artifacts
                archiveArtifacts artifacts: '**/target/*.jar, **/build/libs/*.jar', fingerprint: true
            }
        }
        
        // Stage 7: Security Scan (Optional)
        stage('Security Scan') {
            steps {
                sh '''
                    echo "=== Running Security Checks ==="
                    # OWASP Dependency Check (if installed)
                    mvn org.owasp:dependency-check-maven:check -q || echo "Dependency check not available"
                    
                    # npm audit for Node.js
                    npm audit --audit-level moderate || echo "NPM audit not available"
                '''
            }
        }
    }
    
    post {
        always {
            // Always clean workspace
            cleanWs()
            
            // Send notification
            emailext (
                subject: "Build Result: ${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                Result: ${currentBuild.currentResult}
                URL: ${env.BUILD_URL}
                
                Check the console output for details.
                """,
                to: "dev-team@company.com"
            )
        }
        success {
            echo "Build completed successfully! 🎉"
            // Optional: Deploy to dev environment
        }
        failure {
            echo "Build failed! Please check the logs. ❌"
        }
        unstable {
            echo "Build is unstable! Tests failed or quality gate not met. ⚠️"
        }
    }
}
