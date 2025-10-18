pipeline {
    agent {
        dockerContainer {
            image 'maven:3.9.9-eclipse-temurin-25-alpine'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    
    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic'
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
                    echo "Current directory:"
                    pwd
                '''
            }
        }
        
        // Stage 3: Clean and Build
        stage('Clean and Build') {
            steps {
                sh '''
                    echo "=== Cleaning and Building ==="
                    mvn clean compile -q
                    echo "Build completed successfully"
                '''
            }
        }
        
        // Stage 4: Run Tests
        stage('Run Tests') {
            steps {
                sh '''
                    echo "=== Running Tests ==="
                    mvn test jacoco:report -q
                    echo "Tests completed"
                '''
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        // Stage 5: SonarQube Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        echo "=== Starting SonarQube Analysis ==="
                        mvn sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='Spring PetClinic' \
                        -Dsonar.host.url=\${SONAR_HOST_URL} \
                        -Dsonar.login=\${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }
        
        // Stage 6: Quality Gate
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        // Stage 7: Build Package
        stage('Build Package') {
            steps {
                sh '''
                    echo "=== Building Package ==="
                    mvn package -DskipTests -q
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
            """
        }
    }
}
