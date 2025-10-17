pipeline {
    agent any
    
    environment {
        // Define environment variables
        SONAR_PROJECT_KEY = 'spring-petclinic'
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'  // Adjust path as needed
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
        
        // Stage 2: Verify Environment
        stage('Verify Environment') {
            steps {
                sh '''
                    echo "=== Checking Environment ==="
                    java -version || echo "Java not found"
                    mvn -version || echo "Maven not found"
                    which sonar-scanner || echo "SonarScanner not found"
                    pwd
                    ls -la
                '''
            }
        }
        
        // Stage 3: Dependency Resolution
        stage('Resolve Dependencies') {
            steps {
                sh '''
                    echo "=== Installing Dependencies ==="
                    mvn dependency:resolve -q || echo "Dependency resolution completed"
                '''
            }
        }
        
        // Stage 4: Code Compilation
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    mvn compile -q
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        // Stage 5: Unit Tests
        stage('Run Unit Tests') {
            steps {
                sh '''
                    echo "=== Running Unit Tests ==="
                    mvn test -q
                    echo "Tests completed"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        // Stage 6: Code Quality Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }
        
        // Stage 7: Build Artifact
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    mvn package -DskipTests -q
                    echo "Package built successfully"
                '''
                // Archive the built artifacts
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
    }
    
    post {
        always {
            // Clean workspace (optional)
            // cleanWs()
            
            // Send notification
            emailext (
                subject: "Build Result: ${currentBuild.currentResult} - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                Result: ${currentBuild.currentResult}
                URL: ${env.BUILD_URL}
                
                Check the console output for details.
                """,
                to: "dev-team@company.com"  // Update with your email
            )
        }
        success {
            echo "Build completed successfully! 🎉"
            // Archive test results
            archiveArtifacts artifacts: '**/target/*.jar, **/target/surefire-reports/*.xml', fingerprint: true
        }
        failure {
            echo "Build failed! Please check the logs. ❌"
        }
        unstable {
            echo "Build is unstable! Tests failed or quality gate not met. ⚠️"
        }
    }
}
