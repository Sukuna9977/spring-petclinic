pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
    }
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        SONAR_HOST_URL = 'http://172.17.0.1:9000'
    }
    
    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
                sh 'git branch --show-current'
                sh 'git log -1 --oneline'
            }
        }
        
        stage('Verify Environment') {
            steps {
                sh '''
                    echo "=== Checking Environment ==="
                    echo "Java Version:"
                    java -version
                    echo "Maven Wrapper:"
                    ./mvnw --version
                '''
            }
        }
        
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    ./mvnw clean -q -Denforcer.skip=true -Dcheckstyle.skip=true
                    echo "Clean completed"
                '''
            }
        }
        
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    ./mvnw compile -q -Denforcer.skip=true -Dcheckstyle.skip=true
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        stage('Run Tests with Coverage') {
            steps {
                sh '''
                    echo "=== Running Tests with Coverage ==="
                    ./mvnw test jacoco:report -q -Denforcer.skip=true -Dcheckstyle.skip=true -Dtest=!PostgresIntegrationTests
                    echo "Tests and coverage report completed"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/site/jacoco/*', fingerprint: false
                }
            }
        }
        
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    ./mvnw package -DskipTests -q -Denforcer.skip=true -Dcheckstyle.skip=true
                    echo "Package built successfully"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    sh '''
                        echo "=== Generated Artifact ==="
                        ls -la target/*.jar
                        echo "Artifact size:"
                        du -h target/*.jar
                    '''
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        echo "Running SonarQube analysis..."
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=spring-petclinic \
                            -Dsonar.projectName="Spring PetClinic" \
                            -Dsonar.host.url=http://172.17.0.1:9000 \
                            -Denforcer.skip=true \
                            -Dcheckstyle.skip=true \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo "Waiting for Quality Gate result..."
                    sleep 30
                    
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate abortPipeline: false
                            
                            if (qg.status == 'OK') {
                                echo "✅ Quality Gate PASSED"
                                currentBuild.description = "✅ Quality Gate: PASSED"
                            } else {
                                echo "⚠️ Quality Gate status: ${qg.status}"
                                // Don't set result to UNSTABLE - this fucks up weather report
                                currentBuild.description = "⚠️ Quality Gate: ${qg.status}"
                            }
                        }
                    } catch (Exception ex) {
                        echo "⚠️ Quality Gate timeout or error: ${ex.message}"
                        currentBuild.description = "⚠️ Quality Gate: TIMEOUT"
                        // Don't change build result - keep it SUCCESS
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
            """
            // Don't clean workspace in post-always - it fucks up artifacts
        }
        
        success {
            echo "🎉 Pipeline executed successfully!"
            sh '''
                echo "=== SUCCESS ==="
                echo "✅ Code compiled successfully"
                echo "✅ Tests passed" 
                echo "✅ Package built"
                echo "✅ Artifacts archived in Jenkins"
            '''
        }
        
        failure {
            echo "❌ Pipeline failed! Check the logs above."
        }
        
        cleanup {
            cleanWs()  // Clean workspace only at the very end
        }
    }
}
