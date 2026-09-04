pipeline {
    agent any
 
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
        choice(
        name: 'DEPLOY_MODE',
        choices: ['SELECTED', 'ALL'],
        description: 'Deploy selected bundles or all bundles'
        )

        string(
            name: 'BUNDLE_NAMES',
            defaultValue: '',
            description: 'For SELECTED mode, enter comma-separated bundle names. Example: CustomerAPI.bundle,PaymentAPI.bundle'
        )
    }
 
    environment {
        BUNDLE_DIR = 'bundles'
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
 
    stage('Validate Bundle Selection') {
    steps {
        script {
            if (params.DEPLOY_MODE == 'SELECTED') {

                if (!params.BUNDLE_NAMES?.trim()) {
                    error "BUNDLE_NAMES is required when DEPLOY_MODE=SELECTED"
                }

                def bundles = params.BUNDLE_NAMES
                    .split(',')
                    .collect { it.trim() }
                    .findAll { it }

                if (bundles.isEmpty()) {
                    error "No valid bundle names provided."
                }

                bundles.each { bundleName ->
                    def bundlePath = "${env.BUNDLE_DIR}\\${bundleName}"

                    if (!fileExists(bundlePath)) {
                        error "Bundle file not found: ${bundlePath}"
                    }

                    echo "Validated bundle: ${bundlePath}"
                }

                env.SELECTED_BUNDLES = bundles.join(',')

            } else {
                echo "All .bundle files in bundles folder will be deployed."
            }
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
 
 
stage('Deploy Bundles to Layer7') {
    steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'layer7-gateway-credentials',
                usernameVariable: 'GATEWAY_USERNAME',
                passwordVariable: 'GATEWAY_PASSWORD'
            )
        ]) {
            script {

                if (params.DEPLOY_MODE == 'SELECTED') {

                    def bundles = env.SELECTED_BUNDLES.split(',')

                    bundles.each { bundleName ->

                        def bundlePath = "${env.BUNDLE_DIR}\\${bundleName}"

                        echo "Deploying ${bundleName}"

                        bat """
                            set "PATH=%JAVA_HOME%\\bin;%PATH%"

                            "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                -h "%GATEWAY_HOST%" ^
                                -p "%GATEWAY_PORT%" ^
                                -u "%GATEWAY_USERNAME%" ^
                                --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                --trustCertificate ^
                                --trustHostname ^
                                -b "${bundlePath}" ^
                                -r "%WORKSPACE%\\gmu-results-${bundleName.replace('.bundle','')}.xml"

                            if errorlevel 1 (
                                echo ERROR: Deployment failed for ${bundleName}
                                exit /b 1
                            )
                        """
                    }

                } else {

                    bat '''
                        echo Deploying all bundles...

                        set "PATH=%JAVA_HOME%\\bin;%PATH%"

                        for %%F in ("${env.BUNDLE_DIR}\\*.bundle") do (

                            echo ========================================
                            echo Deploying %%~nxF
                            echo ========================================

                            "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                -h "%GATEWAY_HOST%" ^
                                -p "%GATEWAY_PORT%" ^
                                -u "%GATEWAY_USERNAME%" ^
                                --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                --trustCertificate ^
                                --trustHostname ^
                                -b "%%F" ^
                                -r "%WORKSPACE%\\gmu-results-%%~nF.xml"

                            if errorlevel 1 (
                                echo ERROR: Deployment failed for %%~nxF
                                exit /b 1
                            )
                        )
                    '''
                }
            }
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
