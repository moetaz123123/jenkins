pipeline {
    agent any
    
    environment {
        // Configuration SonarQube
        SONAR_TOKEN = credentials('squ_957be0ba0db6a033a5a71511f65495804aca0645')
        SONAR_HOST_URL = 'http://localhost:9000'
        
        // Configuration Docker
        DOCKER_IMAGE = 'laravel-app'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        
        // Configuration PHP
        PHP_VERSION = '8.2'
        
        // Configuration de la base de données de test
        DB_CONNECTION = 'sqlite'
        DB_DATABASE = ':memory:'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installing PHP dependencies...'
                sh 'composer install --no-interaction --prefer-dist --optimize-autoloader'
            }
        }
        
        stage('Environment Setup') {
            steps {
                echo 'Setting up environment...'
                sh '''
                    cp .env.example .env
                    php artisan key:generate
                    php artisan config:cache
                '''
            }
        }
        
        stage('Code Quality - PHPStan') {
            steps {
                echo 'Running PHPStan analysis...'
                script {
                    try {
                        sh '''
                            if ! command -v phpstan &> /dev/null; then
                                composer require --dev phpstan/phpstan
                            fi
                            ./vendor/bin/phpstan analyse app --level=5 --no-progress
                        '''
                    } catch (Exception e) {
                        echo "PHPStan analysis failed: ${e.getMessage()}"
                        // Continue pipeline even if PHPStan fails
                    }
                }
            }
        }
        
        stage('Code Quality - Laravel Pint') {
            steps {
                echo 'Running Laravel Pint code style check...'
                sh './vendor/bin/pint --test'
            }
        }
        
        stage('Run Tests') {
            steps {
                echo 'Running PHPUnit tests...'
                sh 'php artisan test --coverage-clover=coverage.xml'
            }
            post {
                always {
                    // Publish test results
                    publishTestResults testResultsPattern: 'tests/**/test-results.xml'
                    
                    // Publish coverage report if exists
                    script {
                        if (fileExists('coverage.xml')) {
                            publishCoverage adapters: [cloverAdapter('coverage.xml')], sourceFileResolver: sourceFiles('NEVER_STORE')
                        }
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube analysis...'
                script {
                    // Install SonarQube Scanner if not available
                    sh '''
                        if ! command -v sonar-scanner &> /dev/null; then
                            wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.8.0.2856-linux.zip
                            unzip sonar-scanner-cli-4.8.0.2856-linux.zip
                            sudo mv sonar-scanner-4.8.0.2856-linux /opt/sonar-scanner
                            sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner
                        fi
                    '''
                    
                    // Run SonarQube analysis
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=laravel-multitenant \
                                -Dsonar.projectName="Laravel Multi-tenant Application" \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.sources=app,resources \
                                -Dsonar.tests=tests \
                                -Dsonar.php.coverage.reportPaths=coverage.xml \
                                -Dsonar.php.tests.reportPath=tests/phpunit-report.xml \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.login=${SONAR_TOKEN} \
                                -Dsonar.exclusions=vendor/**,node_modules/**,storage/**,bootstrap/cache/**
                        '''
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .'
            }
            post {
                always {
                    // Clean up Docker images to save space
                    sh 'docker image prune -f'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...'
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
    }
}
