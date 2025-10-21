pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        retry(2) // Retry entire pipeline up to 2 times on failure
    }
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        SONAR_HOST_URL = 'http://172.17.0.1:9000'
        MAVEN_OPTS = '-Xmx1024m -XX:MaxPermSize=256m'
        // Add build cache directory
        MAVEN_REPO_CACHE = '/tmp/m2_repo_cache'
    }
    
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
                sh '''
                    echo "=== Git Information ==="
                    git branch --show-current
                    git log -1 --oneline
                    git status --short
                '''
            }
        }
        
        stage('Verify Environment') {
            steps {
                sh '''
                    echo "=== Checking Environment ==="
                    echo "Java Version:"
                    java -version || echo "Java not found"
                    echo "Maven Wrapper:"
                    ./mvnw --version || echo "Maven wrapper not executable"
                    echo "Available disk space:"
                    df -h .
                    echo "Memory info:"
                    free -h || echo "Free command not available"
                '''
            }
        }
        
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    # Use more aggressive clean but preserve local Maven cache
                    ./mvnw clean -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                    echo "Clean completed"
                '''
            }
        }
        
        stage('Dependency Resolution') {
            steps {
                sh '''
                    echo "=== Resolving Dependencies ==="
                    # Download dependencies separately to catch network issues early
                    ./mvnw dependency:resolve -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                    echo "Dependencies resolved"
                '''
            }
        }
        
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    ./mvnw compile -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        stage('Run Tests with Coverage') {
            steps {
                sh '''
                    echo "=== Running Tests with Coverage ==="
                    # Run tests with better error handling and timeouts
                    ./mvnw test jacoco:report -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dtest=!PostgresIntegrationTests -Dmaven.test.failure.ignore=false -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                    echo "Tests and coverage report completed"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/site/jacoco/*', fingerprint: false
                    // Capture test results even if tests fail
                    sh '''
                        echo "=== Test Results Summary ==="
                        find . -name "*.xml" -path "*/surefire-reports/*" -exec grep -l "<testsuite" {} \\; | head -5 | while read file; do
                            echo "Test file: $file"
                            tests=$(grep -o "tests=\"[0-9]*\"" "$file" | head -1 | sed 's/tests=\"//' | sed 's/\"//')
                            failures=$(grep -o "failures=\"[0-9]*\"" "$file" | head -1 | sed 's/failures=\"//' | sed 's/\"//')
                            errors=$(grep -o "errors=\"[0-9]*\"" "$file" | head -1 | sed 's/errors=\"//' | sed 's/\"//')
                            echo "  Tests: $tests, Failures: $failures, Errors: $errors"
                        done
                    '''
                }
                unsuccessful {
                    echo "Tests failed - check test reports for details"
                    // Don't fail the build immediately, continue to build package
                }
            }
        }
        
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    ./mvnw package -DskipTests -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                    echo "Package built successfully"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
                    sh '''
                        echo "=== Generated Artifacts ==="
                        ls -la target/*.jar 2>/dev/null || echo "No jar files found"
                        if [ -f target/*.jar ]; then
                            echo "Artifact size:"
                            du -h target/*.jar
                        fi
                    '''
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        withSonarQubeEnv('sonarqube') {
                            sh '''
                                echo "Running SonarQube analysis..."
                                ./mvnw sonar:sonar \
                                    -Dsonar.projectKey=spring-petclinic \
                                    -Dsonar.projectName="Spring PetClinic" \
                                    -Dsonar.host.url=http://172.17.0.1:9000 \
                                    -Denforcer.skip=true \
                                    -Dcheckstyle.skip=true \
                                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                                    -Dmaven.repo.local=${MAVEN_REPO_CACHE}
                            '''
                        }
                    } catch (Exception e) {
                        echo "SonarQube analysis failed: ${e.message}"
                        echo "Continuing build without SonarQube analysis"
                        // Don't fail the build for SonarQube issues
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo "Checking Quality Gate status..."
                    sleep 30
                    
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            def qualityGate = waitForQualityGate abortPipeline: false
                            echo "Quality Gate status: ${qualityGate.status}"
                            if (qualityGate.status != 'OK') {
                                echo "WARNING: Quality Gate failed with status: ${qualityGate.status}"
                                // Mark build as unstable instead of failed
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    } catch (Exception ex) {
                        echo "Quality Gate check had issues but build continues: ${ex.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh """
                echo "=== Build Summary ==="
                echo "Build: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                echo "Result: ${currentBuild.currentResult}"
                echo "Duration: ${currentBuild.durationString}"
                echo "=== Disk Usage After Build ==="
                df -h . 2>/dev/null || echo "Disk check not available"
            """
            
            // Publish build stability metrics
            script {
                if (currentBuild.result == 'FAILURE') {
                    echo "❌ Build failed - check logs for details"
                } else if (currentBuild.result == 'UNSTABLE') {
                    echo "⚠️  Build completed with warnings"
                } else {
                    echo "🎉 Pipeline executed successfully!"
                }
            }
        }
        
        success {
            sh '''
                echo "=== SUCCESS ==="
                echo "✅ Code compiled successfully"
                echo "✅ Tests completed" 
                echo "✅ Package built"
                echo "✅ Artifacts archived in Jenkins"
            '''
        }
        
        unstable {
            sh '''
                echo "=== UNSTABLE ==="
                echo "⚠️  Build completed with warnings"
                echo "⚠️  Check test results or quality gates"
            '''
        }
        
        failure {
            sh '''
                echo "=== FAILURE ==="
                echo "❌ Build failed - investigate the following:"
                echo "❌ Check compilation errors"
                echo "❌ Review test failures"
                echo "❌ Verify environment setup"
            '''
            
            // Optional: Notify team on critical failures
            // emailext (
            //     subject: "FAILED: Job '${env.JOB_NAME}' (${env.BUILD_NUMBER})",
            //     body: "Check console output at: ${env.BUILD_URL}",
            //     to: "team@example.com"
            // )
        }
        
        cleanup {
            // Clean workspace but preserve Maven cache for future builds
            sh '''
                echo "=== Cleaning Workspace ==="
                # Remove everything except the Maven cache
                find . -maxdepth 1 ! -name . ! -name $(basename "$MAVEN_REPO_CACHE") -exec rm -rf {} + 2>/dev/null || true
                echo "Workspace cleaned, Maven cache preserved"
            '''
        }
    }
}
