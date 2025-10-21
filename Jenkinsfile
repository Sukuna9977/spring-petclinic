pipeline {
    agent any
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
        SONAR_HOST_URL = 'http://172.17.0.1:9000'
        // Remove this line - credentials should only be used in withCredentials block
        // SONAR_AUTH_TOKEN = credentials('sonarqube-token')
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
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh """
                        echo "=== Starting SonarQube Analysis ==="
                        echo "SonarQube URL: ${env.SONAR_HOST_URL}"
                        echo "Project Key: ${env.SONAR_PROJECT_KEY}"
                        
                        # Test token first
                        echo "Testing SonarQube connection..."
                        curl -u "${SONAR_AUTH_TOKEN}": "${env.SONAR_HOST_URL}/api/projects/search" | head -c 100
                        
                        echo "Running SonarQube analysis..."
                        ./mvnw sonar:sonar \
                          -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                          -Dsonar.projectName='Spring PetClinic' \
                          -Dsonar.host.url=${env.SONAR_HOST_URL} \
                          -Dsonar.token=${SONAR_AUTH_TOKEN} \
                          -Dsonar.verbose=true \
                          -Denforcer.skip=true \
                          -Dcheckstyle.skip=true
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            when {
                expression { 
                    return env.SONAR_HOST_URL != null && env.SONAR_HOST_URL != '' 
                }
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
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
            cleanWs()
        }
        
        success {
            echo "🎉 Pipeline executed successfully!"
            sh '''
                echo "=== SUCCESS ==="
                echo "✅ Code compiled successfully"
                echo "✅ Tests passed" 
                echo "✅ Package built (66MB JAR)"
                echo "✅ Artifacts archived in Jenkins"
            '''
        }
        
        failure {
            echo "❌ Pipeline failed! Check the logs above."
        }
        
        unstable {
            echo "⚠️ Quality gate failed but build completed!"
        }
    }
}
