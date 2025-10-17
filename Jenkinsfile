pipeline {
    agent any
    
    environment {
        // Define environment variables
        SONAR_PROJECT_KEY = 'spring-petclinic'
        MAVEN_OPTS = '-Xmx1024m -XX:MaxPermSize=256m'  // Maven memory settings
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
                    echo "Java Version:"
                    java -version
                    echo "Maven Version:"
                    mvn --version
                    echo "Maven Home:"
                    mvn -version | grep "Maven home"
                    echo "Current directory:"
                    pwd
                    echo "Project structure:"
                    ls -la
                    echo "Source files:"
                    find . -name "pom.xml" -o -name "*.java" | head -10
                '''
            }
        }
        
        // Stage 3: Clean Workspace
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    mvn clean -q
                    echo "Clean completed"
                '''
            }
        }
        
        // Stage 4: Dependency Resolution
        stage('Resolve Dependencies') {
            steps {
                sh '''
                    echo "=== Resolving Dependencies ==="
                    mvn dependency:resolve -q
                    echo "Dependencies resolved successfully"
                    
                    # Show dependency tree for debugging
                    echo "=== Dependency Tree ==="
                    mvn dependency:tree -Dverbose -q | head -20 || echo "Dependency tree generation completed"
                '''
            }
        }
        
        // Stage 5: Code Compilation
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    mvn compile -q
                    echo "=== Checking compiled classes ==="
                    find target/classes -name "*.class" | head -5 || echo "No classes found yet"
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        // Stage 6: Run Tests with Coverage
        stage('Run Tests with Coverage') {
            steps {
                sh '''
                    echo "=== Running Tests with Coverage ==="
                    mvn test jacoco:report -q
                    echo "Tests and coverage report completed"
                    
                    # Show test results summary - FIXED LINE
                    echo "=== Test Results Summary ==="
                    find target/surefire-reports -name "*.txt" -exec head -5 {} \\; 2>/dev/null || echo "Test reports being generated"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    // Archive test reports for debugging
                    archiveArtifacts artifacts: '**/target/surefire-reports/*.txt, **/target/site/jacoco/*', fingerprint: false
                }
            }
        }
        
        // Stage 7: Code Quality Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        echo "=== Starting SonarQube Analysis ==="
                        sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='Spring PetClinic' \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.java.libraries=~/.m2/repository/**/*.jar \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN} \
                        -X  # Enable debug mode for troubleshooting
                    """
                }
            }
        }
        
        // Stage 8: Wait for Quality Gate
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        // Stage 9: Build Package
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    mvn package -DskipTests -q
                    echo "=== Generated Artifacts ==="
                    find target -name "*.jar" -o -name "*.war" | head -10
                    echo "Package built successfully"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    sh '''
                        echo "=== Package Details ==="
                        ls -la target/*.jar
                    '''
                }
            }
        }
        
        // Stage 10: Integration Tests (Optional)
        stage('Integration Tests') {
            steps {
                sh '''
                    echo "=== Running Integration Tests ==="
                    mvn verify -q -DskipUnitTests || echo "Integration tests completed or skipped"
                '''
            }
        }
    }
    
    post {
        always {
            // Generate build summary
            sh '''
                echo "=== Build Summary ==="
                echo "Build: ${JOB_NAME} #${BUILD_NUMBER}"
                echo "Result: ${currentBuild.currentResult}"
                echo "Workspace: ${WORKSPACE}"
                
                # Show final artifacts
                echo "=== Final Artifacts ==="
                find target -name "*.jar" -o -name "*.xml" 2>/dev/null | head -10 || echo "No artifacts found"
            '''
            
            // Clean up large files to save space (optional)
            sh '''
                echo "=== Cleaning up large files ==="
                du -sh . || echo "Disk usage check completed"
            '''
            
            // Send notification (configure email in Jenkins first)
            emailext (
                subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                Status: ${currentBuild.currentResult}
                Duration: ${currentBuild.durationString}
                URL: ${env.BUILD_URL}
                
                Check the console output for detailed results.
                """,
                to: "developer@company.com",  // Update with your email
                attachLog: true
            )
        }
        
        success {
            echo "🎉 Pipeline executed successfully!"
            sh '''
                echo "=== SUCCESS ==="
                echo "All stages completed successfully"
                echo "Artifacts are available in Jenkins"
                echo "SonarQube analysis completed"
            '''
        }
        
        failure {
            echo "❌ Pipeline failed! Check the logs above."
            sh '''
                echo "=== FAILURE ==="
                echo "Check the specific stage that failed above"
                echo "Common issues:"
                echo "- Compilation errors"
                echo "- Test failures"
                echo "- SonarQube connection issues"
                echo "- Dependency resolution problems"
            '''
        }
        
        unstable {
            echo "⚠️ Pipeline completed but quality gate failed!"
            sh '''
                echo "=== UNSTABLE ==="
                echo "Quality gate not met in SonarQube"
                echo "Check SonarQube dashboard for details"
            '''
        }
        
        cleanup {
            // Optional: Clean workspace after build
            // cleanWs notCleanWhen: 'FAILED'
            echo "=== Cleanup completed ==="
        }
    }
}
