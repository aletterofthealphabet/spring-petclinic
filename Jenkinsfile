pipeline {
    agent any

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        SONAR_HOST_URL = 'http://sonarqube:9000'
        ANSIBLE_CONFIG = '/ansible/ansible.cfg'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests -Dspring-javaformat.skip=true'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw test -Dspring-javaformat.skip=true'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """./mvnw sonar:sonar \
                        -Dsonar.projectKey=spring-petclinic \
                        -Dsonar.projectName='Spring PetClinic' \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dspring-javaformat.skip=true"""
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                sh """ansible-playbook /ansible/deploy.yml \
                    -e jar_source=\$(find \${WORKSPACE}/target -name 'spring-petclinic-*.jar' -not -name '*-sources.jar' | head -1)"""
            }
        }

        stage('ZAP Security Scan') {
            steps {
                sh '''
                    docker exec zap zap-baseline.py \
                        -t http://production-vm:8080 \
                        -r /zap/reports/zap-report.html \
                        -I || true
                '''
                sh 'docker cp zap:/zap/reports/zap-report.html ${WORKSPACE}/zap-report.html || true'
            }
        }
    }

    post {
        always {
            publishHTML(target: [
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'zap-report.html',
                reportName: 'ZAP Security Report'
            ])
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
