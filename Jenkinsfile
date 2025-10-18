pipeline {
    agent any
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        // Let Maven wrapper handle Java version
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
                    echo "System Java Version:"
                    java -version
                    echo "Maven Wrapper:"
                    ./mvnw --version || echo "Using system Maven"
                    echo "Current directory:"
                    pwd
                '''
            }
        }
        
        // Stage 3: Clean Workspace - Use Maven Wrapper
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    ./mvnw clean -q
                    echo "Clean completed"
                '''
            }
        }
        
        // Stage 4: Dependency Resolution - Use Maven Wrapper
        stage('Resolve Dependencies') {
            steps {
                sh '''
                    echo "=== Resolving Dependencies ==="
                    ./mvnw dependency:resolve -q
                    echo "Dependencies resolved successfully"
                '''
            }
        }
        
        // Stage 5: Code Compilation - Use Maven Wrapper
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    ./mvnw compile -q
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        // Stage 6: Run Tests with Coverage - Use Maven Wrapper
        stage('Run Tests with Coverage') {
            steps {
                sh '''
                    echo "=== Running Tests with Coverage ==="
                    ./mvnw test jacoco:report -q
                    echo "Tests and coverage report completed"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                    archiveArtifacts artifacts: '**/target/surefire-reports/*.txt, **/target/site/jacoco/*', fingerprint: false
                }
            }
        }
        
        // Stage 7: Code Quality Analysis - Use Maven Wrapper
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        echo "=== Starting SonarQube Analysis ==="
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='Spring PetClinic' \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN}
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
        
        // Stage 9: Build Package - Use Maven Wrapper
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    ./mvnw package -DskipTests -q
                    echo "Package built successfully"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
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
                echo "Workspace: ${env.WORKSPACE}"
            """
        }
        
        success {
            echo "🎉 Pipeline executed successfully!"
        }
        
        failure {
            echo "❌ Pipeline failed! Check the logs above."
        }
        
        unstable {
            echo "⚠️ Pipeline completed but quality gate failed!"
        }
    }
}
