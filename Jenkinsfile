pipeline {
    agent {
        node {
            label 'jenkins-agent'
        }
    }

    environment {
        AWS_CREDENTIALS_ID = 'aws-ecr-credentials' // ID de tus credenciales en Jenkins
        AWS_REGION         = 'us-east-1'
        AWS_ACCOUNT_ID     = '270876217576'
        IMAGE_NAME         = 'devops-enterprise-api'
        
        // Asignar tag dinámico según la rama
        BUILD_TAG          = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('1. Code Analysis & Unit Tests') {
            steps {
                container('docker-trivy') {
                    sh '''
                        echo "=== [CI] Iniciando análisis y pruebas unitarias en rama: ${BRANCH_NAME} ==="
                        cd app/
                        python3 -m venv venv
                        . venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        pytest tests/ --maxfail=1 --disable-warnings -v
                    '''
                }
            }
        }

        stage('2. Build Docker Image') {
            steps {
                container('docker-trivy') {
                    sh '''
                        echo "=== [CI] Construyendo imagen de Docker para la rama ${BRANCH_NAME} ==="
                        cd app/
                        docker build -t ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}:${BUILD_TAG} .
                    '''
                }
            }
        }

        stage('3. Security Scan (Trivy DevSecOps)') {
            steps {
                container('docker-trivy') {
                    sh '''
                        echo "=== [SECURITY] Escaneando vulnerabilidades con Trivy ==="
                        trivy image --severity HIGH,CRITICAL --exit-code 0 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}:${BUILD_TAG}
                    '''
                }
            }
        }

        stage('4. AWS ECR Authentication & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.AWS_CREDENTIALS_ID,
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    container('aws-cli') {
                        script {
                            sh '''
                                echo "=== [CD] Autenticando en AWS ECR ==="
                                aws ecr get-login-password --region ${AWS_REGION} > /shared-tmp/ecr_token.txt
                            '''
                        }
                    }
                    container('docker-trivy') {
                        sh '''
                            echo "=== [CD] Subiendo imagen a ECR ==="
                            ECR_TOKEN=$(cat /shared-tmp/ecr_token.txt)
                            rm -f /shared-tmp/ecr_token.txt
                            
                            docker login -u AWS -p "${ECR_TOKEN}" https://${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}:${BUILD_TAG}
                        '''
                    }
                }
            }
        }

        stage('5. Advanced Kubernetes Deployment') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.AWS_CREDENTIALS_ID,
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    container('aws-cli') {
                        sh 'aws ecr get-login-password --region ${AWS_REGION} > /shared-tmp/ecr_k8s_token.txt'
                    }
                    
                    container('docker-trivy') {
                        script {
                            // Configuración común de kubectl y secrets
                            sh '''
                                if ! command -v curl &> /dev/null; then apk add --no-cache curl; fi
                                if ! command -v kubectl &> /dev/null; then
                                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                                    chmod +x kubectl && mv kubectl /usr/local/bin/
                                fi
                            '''

                            // LÓGICA CONDICIONAL SEGÚN LA RAMA (GITFLOW)
                            if (env.BRANCH_NAME == 'develop') {
                                echo "=== [CD - DEV] Despliegue Estándar (Rolling Update) en namespace 'dev' ==="
                                sh '''
                                    ECR_TOKEN=$(cat /shared-tmp/ecr_k8s_token.txt) && rm -f /shared-tmp/ecr_k8s_token.txt
                                    kubectl create secret docker-registry regcred --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com --docker-username=AWS --docker-password="${ECR_TOKEN}" -n dev --dry-run=client -o yaml | kubectl apply -f -
                                    
                                    kubectl apply -f k8s/namespaces.yaml
                                    kubectl apply -f k8s/configmaps/configmap-dev.yaml
                                    
                                    # Despliegue estándar (puedes usar un deployment base o reutilizar el anterior)
                                    sed -i "s|<AWS_ACCOUNT_ID>|${AWS_ACCOUNT_ID}|g" k8s/blue-green/deployment-blue.yaml
                                    sed -i "s|BUILD_TAG|${BUILD_TAG}|g" k8s/blue-green/deployment-blue.yaml
                                    kubectl apply -f k8s/blue-green/deployment-blue.yaml -n dev
                                    kubectl rollout status deployment/devops-enterprise-api -n dev --timeout=300s
                                '''
                            } 
                            else if (env.BRANCH_NAME == 'staging') {
                                echo "=== [CD - STAGING] Despliegue Canary en namespace 'staging' ==="
                                sh '''
                                    ECR_TOKEN=$(cat /shared-tmp/ecr_k8s_token.txt) && rm -f /shared-tmp/ecr_k8s_token.txt
                                    kubectl create secret docker-registry regcred --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com --docker-username=AWS --docker-password="${ECR_TOKEN}" -n staging --dry-run=client -o yaml | kubectl apply -f -
                                    
                                    kubectl apply -f k8s/namespaces.yaml
                                    kubectl apply -f k8s/configmaps/configmap-staging.yaml
                                    
                                    # Desplegar versión Canary y Stable
                                    sed -i "s|<AWS_ACCOUNT_ID>|${AWS_ACCOUNT_ID}|g" k8s/canary/deployment-canary.yaml
                                    sed -i "s|BUILD_TAG|${BUILD_TAG}|g" k8s/canary/deployment-canary.yaml
                                    
                                    kubectl apply -f k8s/canary/deployment-stable.yaml -n staging
                                    kubectl apply -f k8s/canary/deployment-canary.yaml -n staging
                                    kubectl apply -f k8s/canary/service-canary.yaml -n staging
                                    
                                    kubectl rollout status deployment/devops-enterprise-api-canary -n staging --timeout=300s
                                '''
                            } 
                            else if (env.BRANCH_NAME == 'main') {
                                echo "=== [CD - PROD] Despliegue Blue/Green en namespace 'prod' ==="
                                sh '''
                                    ECR_TOKEN=$(cat /shared-tmp/ecr_k8s_token.txt) && rm -f /shared-tmp/ecr_k8s_token.txt
                                    kubectl create secret docker-registry regcred --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com --docker-username=AWS --docker-password="${ECR_TOKEN}" -n prod --dry-run=client -o yaml | kubectl apply -f -
                                    
                                    kubectl apply -f k8s/namespaces.yaml
                                    kubectl apply -f k8s/configmaps/configmap-prod.yaml
                                    
                                    # 1. Desplegar la versión Green (Nueva) en paralelo con Blue (Actual)
                                    sed -i "s|<AWS_ACCOUNT_ID>|${AWS_ACCOUNT_ID}|g" k8s/blue-green/deployment-green.yaml
                                    sed -i "s|BUILD_TAG|${BUILD_TAG}|g" k8s/blue-green/deployment-green.yaml
                                    kubectl apply -f k8s/blue-green/deployment-green.yaml -n prod
                                    
                                    kubectl rollout status deployment/devops-enterprise-api-green -n prod --timeout=300s
                                '''

                                // 2. Pausa para Aprobación Manual antes del Switch a Producción
                                timeout(time: 30, unit: 'MINUTES') {
                                    input message: "⚠️ ¿Deseas aprobar el cambio de tráfico a la versión GREEN en PRODUCCIÓN?",
                                          ok: "Aprobar y Conmutar Tráfico"
                                }

                                sh '''
                                    echo "=== [CD - PROD] Conmutando tráfico al entorno GREEN ==="
                                    kubectl apply -f k8s/blue-green/service-prod.yaml -n prod
                                    echo "¡Despliegue Blue/Green completado con éxito en Producción!"
                                '''
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ ¡El pipeline en la rama ${env.BRANCH_NAME} finalizó exitosamente!"
        }
        failure {
            echo "❌ El pipeline falló en la rama ${env.BRANCH_NAME}. Revisa los logs anteriores."
        }
    }
}