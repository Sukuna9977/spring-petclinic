pipeline {
    agent any
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
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
                    echo "Current directory:"
                    pwd
                    echo "Project structure:"
                    ls -la
                '''
            }
        }
        
        stage('Clean Project') {
            steps {
                sh '''
                    echo "=== Cleaning Project ==="
                    ./mvnw clean -q -Denforcer.skip=true
                    echo "Clean completed"
                '''
            }
        }
        
        stage('Compile Code') {
            steps {
                sh '''
                    echo "=== Compiling Code ==="
                    ./mvnw compile -q -Denforcer.skip=true
                    echo "Compilation completed successfully"
                '''
            }
        }
        
        stage('Run Tests with Coverage') {
            steps {
                sh '''
                    echo "=== Running Tests with Coverage ==="
                    ./mvnw test jacoco:report -q -Denforcer.skip=true
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
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        echo "=== Starting SonarQube Analysis ==="
                        ./mvnw sonar:sonar -q -Denforcer.skip=true \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='Spring PetClinic' \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    ./mvnw package -DskipTests -q -Denforcer.skip=true
                    echo "Package built successfully"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                    sh '''
                        echo "=== Generated Artifact ==="
                        ls -la target/*.jar
                    '''
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
    }
}
