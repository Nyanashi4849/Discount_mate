pipeline {
    agent any

    environment {
        IMAGE_NAME = "discountmate-api"
        CONTAINER_NAME = "discountmate-api"
        PORT = "5000"
    }

    stages {

        stage('Install & Build') {
            steps {
                bat '''
                echo Installing dependencies...
                npm install
                '''
            }
        }

        stage('Test') {
            steps {
                bat '''
                echo Running tests...
                npm test
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                echo Building Docker image...
                docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .
                '''
            }
        }

        stage('Code Quality - SonarCloud') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    script {
                        def scannerHome = tool 'sonar-scanner'

                        bat """
                        echo Running SonarCloud Analysis...

                        ${scannerHome}\\bin\\sonar-scanner ^
                        -Dsonar.projectKey=Nyanashi4849_DiscountMateFINAL ^
                        -Dsonar.organization=nyanashi4849 ^
                        -Dsonar.sources=. ^
                        -Dsonar.host.url=https://sonarcloud.io ^
                        -Dsonar.login=%SONAR_TOKEN%
                        """
                    }
                }
            }
        }

        stage('Security Scan - Snyk') {
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    bat '''
                    echo Installing Snyk CLI...
                    npm install -g snyk

                    echo Authenticating Snyk...
                    snyk auth %SNYK_TOKEN%

                    echo Scanning dependencies...
                    snyk test --severity-threshold=high

                    echo Scanning Docker image...
                    snyk container test %IMAGE_NAME%:%BUILD_NUMBER% --severity-threshold=high
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                bat '''
                echo =====================================
                echo DEPLOYMENT STARTING
                echo =====================================

                set IMAGE=%IMAGE_NAME%:%BUILD_NUMBER%

                echo Checking existing container...
                docker ps -a -q -f name=%CONTAINER_NAME% > container.txt
                set /p OLD_CONTAINER=<container.txt

                if not "%OLD_CONTAINER%"=="" (
                    echo Removing old container...
                    docker stop %CONTAINER_NAME%
                    docker rm %CONTAINER_NAME%
                )

                echo Running new container...
                docker run -d ^
                    --name %CONTAINER_NAME% ^
                    -p %PORT%:5000 ^
                    --restart unless-stopped ^
                    %IMAGE%

                echo Waiting for startup...
                timeout /t 5 >nul

                echo Performing health check...
                curl -f http://localhost:%PORT%/ >nul

                if %ERRORLEVEL% NEQ 0 (
                    echo Deployment failed!
                    exit /b 1
                )

                echo Deployment successful!
                '''
            }
        }

        stage('Release') {
            steps {
                withCredentials([string(credentialsId: 'github-creds', variable: 'GIT_PASS')]) {
                    bat """
                    echo Creating Git tag...

                    set TAG=v%BUILD_NUMBER%

                    git config user.email "jenkins@ci.com"
                    git config user.name "Jenkins CI"

                    git tag -d %TAG% 2>nul
                    git tag -a %TAG% -m "Release %TAG%"

                    git push origin :refs/tags/%TAG%
                    git push https://Nyanashi4849:%GIT_PASS%@github.com/Nyanashi4849/DiscountMateFINAL.git %TAG%

                    echo Generating release metadata...

                    (
                        echo {
                        echo   "version": "%TAG%",
                        echo   "image": "%IMAGE_NAME%:%BUILD_NUMBER%",
                        echo   "status": "released",
                        echo   "timestamp": "%DATE% %TIME%"
                        echo }
                    ) > release.json
                    """
                }
            }
        }

        stage('Monitoring') {
            steps {
                bat '''
                echo =====================================
                echo MONITORING STARTED
                echo =====================================

                set URL=http://localhost:%PORT%
                set COUNT=0

                :loop
                curl -s %URL%/ >nul

                if %ERRORLEVEL% EQU 0 (
                    echo Service is running ✔
                ) else (
                    echo Service not responding ⚠
                )

                set /a COUNT+=1
                if %COUNT% GEQ 5 goto end

                timeout /t 5 >nul
                goto loop

                :end
                echo Monitoring completed
                '''
            }
        }
    }
}
