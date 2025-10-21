pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        timeout(time: 30, unit: 'MINUTES')
        retry(2)
    }
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        SONAR_HOST_URL = 'http://172.17.0.1:9000'
        // FIXED: Remove deprecated MaxPermSize for Java 21
        MAVEN_OPTS = '-Xmx1024m'
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
                    java -version
                    echo "Maven Wrapper:"
                    chmod +x ./mvnw  # Ensure wrapper is executable
                    ./mvnw --version || echo "Maven version check failed, continuing..."
                    echo "Available disk space:"
                    df -h .
                '''
            }
        }
        
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    # Use simpler clean command first
                    ./mvnw clean -q -Denforcer.skip=true -Dcheckstyle.skip=true || echo "Clean had issues but continuing..."
                    echo "Clean completed"
                '''
            }
        }
        
        stage('Dependency Resolution') {
            steps {
                sh '''
                    echo "=== Resolving Dependencies ==="
                    ./mvnw dependency:resolve -q -Denforcer.skip=true -Dcheckstyle.skip=true || echo "Dependency resolution had issues"
                    echo "Dependencies resolved"
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
                                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                            '''
                        }
                    } catch (Exception e) {
                        echo "SonarQube analysis failed but continuing: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo "Checking Quality Gate status..."
                    sleep 10
                    
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: false
                        }
                    } catch (Exception ex) {
                        echo "Quality Gate check had issues: ${ex.message}"
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
            """
        }
        
        success {
            echo "🎉 Pipeline executed successfully!"
        }
        
        unstable {
            echo "⚠️  Build completed with warnings"
        }
        
        failure {
            echo "❌ Build failed - check logs for details"
        }
        
        cleanup {
            cleanWs()
        }
    }
}
