pipeline {
    agent any
 
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
    }
 
    environment {
        BUNDLE_FILE = 'bundles/Configuration-Cache-Demo.bundle'
        GMU_HOME = 'C:\\layer7\\GMU'
        JAVA_HOME = 'C:\\Users\\9854564\\Softwares\\jdk-21.0.12.1'
    }
 
    stages {
 
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
 
        stage('Load Environment Configuration') {
            steps {
                script {
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
 
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol
 
                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"
                }
            }
        }
 
        stage('Validate Bundle') {
            steps {
                script {
                    if (!fileExists(env.BUNDLE_FILE)) {
                        error "Bundle file not found: ${env.BUNDLE_FILE}"
                    }
 
                    echo "Bundle found: ${env.BUNDLE_FILE}"
                }
            }
        }
 
        stage('Validate GMU') {
            steps {
                bat '''
            echo ========================================
            echo Verifying Java
            echo ========================================

            echo JAVA_HOME=%JAVA_HOME%
            "%JAVA_HOME%\\bin\\java.exe" -version

            if errorlevel 1 (
                echo ERROR: Java execution failed
                exit /b 1
            )

            echo ========================================
            echo Verifying Layer7 GMU
            echo ========================================

            if not exist "%GMU_HOME%\\GatewayMigrationUtility.bat" (
                echo ERROR: GatewayMigrationUtility.bat not found
                exit /b 1
            )

            echo GMU launcher found successfully.
            echo GMU verification completed successfully.
        '''
            }
        }
stage('Test Layer7 Connectivity') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'layer7-gateway-credentials',
                usernameVariable: 'GATEWAY_USERNAME',
                passwordVariable: 'GATEWAY_PASSWORD'
            )
        ]) {
            bat '''
                echo ========================================
                echo Testing Layer7 Gateway Connectivity
                echo ========================================

                set "PATH=%JAVA_HOME%\\bin;%PATH%"

                "%GMU_HOME%\\GatewayMigrationUtility.bat" list ^
                    -h "%GATEWAY_HOST%" ^
                    -p "%GATEWAY_PORT%" ^
                    -u "%GATEWAY_USERNAME%" ^
                    --plaintextPassword "%GATEWAY_PASSWORD%" ^
                    --trustCertificate ^
                    --trustHostname ^
                    -t POLICY

                if errorlevel 1 (
                    echo ERROR: Unable to connect to Layer7 Gateway
                    exit /b 1
                )

                echo ========================================
                echo Layer7 Gateway connectivity successful.
                echo ========================================
            '''
        }
    }
}
 stage('Test Layer7 Bundle Import') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'layer7-gateway-credentials',
                usernameVariable: 'GATEWAY_USERNAME',
                passwordVariable: 'GATEWAY_PASSWORD'
            )
        ]) {
            bat '''
                echo ========================================
                echo Testing Layer7 Bundle Import
                echo ========================================

                set "PATH=%JAVA_HOME%\\bin;%PATH%"

                "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                    -h "%GATEWAY_HOST%" ^
                    -p "%GATEWAY_PORT%" ^
                    -u "%GATEWAY_USERNAME%" ^
                    --plaintextPassword "%GATEWAY_PASSWORD%" ^
                    --trustCertificate ^
                    --trustHostname ^
                    --bundle "%BUNDLE_FILE%" ^
                    -r "%WORKSPACE%\\gmu-migrate-test-results.xml" ^
                    --test

                if errorlevel 1 (
                    echo ERROR: Layer7 bundle test import failed
                    exit /b 1
                )

                echo Layer7 bundle test import successful.
            '''
        }
    }
}
 
stage('Deploy Bundle to Layer7') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'layer7-gateway-credentials',
                usernameVariable: 'GATEWAY_USERNAME',
                passwordVariable: 'GATEWAY_PASSWORD'
            )
        ]) {
            bat '''
                echo ========================================
                echo Deploying bundle to Layer7 Gateway
                echo ========================================

                set "PATH=%JAVA_HOME%\\bin;%PATH%"

                "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                    -h "%GATEWAY_HOST%" ^
                    -p "%GATEWAY_PORT%" ^
                    -u "%GATEWAY_USERNAME%" ^
                    --plaintextPassword "%GATEWAY_PASSWORD%" ^
                    --trustCertificate ^
                    --trustHostname ^
                    -b "%BUNDLE_FILE%" ^
                    -r "%WORKSPACE%\\gmu-migrate-results.xml"

                if errorlevel 1 (
                    echo ERROR: Layer7 bundle deployment failed.
                    exit /b 1
                )

                echo ========================================
                echo Layer7 bundle deployment successful.
                echo Results file:
                echo %WORKSPACE%\\gmu-migrate-results.xml
                echo ========================================
            '''
        }
    }
}
 
        stage('Deployment Verification') {
            steps {
                echo "GMU deployment completed successfully."
            }
        }
    }
 
post {
 
        success {
            echo """
            ==========================================
            Layer7 Deployment SUCCESS
            Environment : ${params.ENVIRONMENT}
            Bundle      : ${env.BUNDLE_FILE}
            ==========================================
            """
        }
 
        failure {
            echo """
            ==========================================
            Layer7 Deployment FAILED
            Environment : ${params.ENVIRONMENT}
            ==========================================
            """
        }
 
        always {
            archiveArtifacts artifacts: 'results.xml, gmu-migration-results.xml', allowEmptyArchive: true
            cleanWs()
        }
    }
}
