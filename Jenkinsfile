pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        SONAR_AUTH_TOKEN = credentials('sonarqube-token')
        PATH = "/opt/sonar-scanner/bin:${env.PATH}" 
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                        -Dsonar.projectKey=testPipeLine \
                        -Dsonar.sources=. \
                        -Dsonar.php.version=8.0 \
                        -Dsonar.host.url=http://10.30.212.77:9000/ \
                        -Dsonar.login=${SONAR_AUTH_TOKEN} \
                        -Dsonar.python.version=3.x  # Especifica la versión de Python si es necesario
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                // Aumenta el tiempo de espera si el análisis tarda más de lo esperado
               timeout(time: 5, unit: 'HOURS') {
                   waitForQualityGate abortPipeline: true
               }
            }
        }
        stage('DAST con OWASP ZAP') {
            steps {
                script {
                    // Remove any existing container named 'zap_scan'
                    sh 'docker rm -f zap_scan || true'

                    // Run OWASP ZAP container
                    sh '''
                    docker run --user root --name zap_scan -v zap_volume:/zap/wrk/ -t ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py -t http://10.30.212.61 \
                    -r reporte_zap.html -I
                    '''

                    // Copy the report from the 'zap_scan' container to Jenkins workspace
                    sh 'docker cp zap_scan:/zap/wrk/reporte_zap.html ./reporte_zap.html'

                    // Remove the 'zap_scan' container
                    sh 'docker rm zap_scan'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reporte_zap.html', fingerprint: true
                }
            }
        }
        stage('Deploy to Web Server') {
            steps {
                sshagent(['webserver_ssh_credentials_id']) {
                    sh '''
                        echo "Verificando clave SSH"
                        ssh-add -l

                        ssh -o StrictHostKeyChecking=no root@10.30.212.61 '
                            # Añadir el directorio Proyecto_web a la lista de directorios seguros
                            git config --global --add safe.directory /var/www/Proyecto_web

                            # Verificar si la carpeta Proyecto_web existe en /var/www
                            if [ -d /var/www/Proyecto_web ]; then
                                echo "El directorio Proyecto_web ya existe. Realizando git pull..."
                                cd /var/www/Proyecto_web && git reset --hard HEAD && git pull
                            else
                                echo "El directorio Proyecto_web no existe. Realizando git clone..."
                                mkdir -p /var/www/Proyecto_web
                                git clone https://github.com/Albert-Xicola/Proyecto_web.git /var/www/Proyecto_web
                            fi
                        '
                    '''
                }
            }
        }
    }
}
