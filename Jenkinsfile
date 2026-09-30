pipeline {
    agent any

    environment {
        APP = 'inventory-service'
        NS = 'inventory-system'
        MON = 'monitoring'
        VERSION = 'v1.0.0'
        IMAGE = 'inventory-service:v1.0.0'
        REGISTRY_IMAGE = 'localhost:2000/inventory-service:v1.0.0'
        KUBECONFIG = 'C:\\Users\\Riddhi siddhi\\.kube\\config'
        MINIKUBE_HOME = 'C:\\Users\\Riddhi siddhi\\.minikube'
    }

    stages {
        stage('Build Automation') {
            steps {
                bat 'python -m pip install -r inventory-service/requirements.txt'
            }
        }

        stage('Automated Testing') {
            steps {
                bat 'python -m pytest inventory-service/tests -v'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE% -t %REGISTRY_IMAGE% inventory-service'
            }
        }

        stage('Security Scan - Trivy') {
            steps {
                bat '''
                    where trivy >NUL 2>&1 && (
                        trivy image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 0 %IMAGE%
                        exit /b 0
                    )
                    echo Trivy CLI not found. Running scan via Docker container...
                    docker run --rm -v //var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 0 %IMAGE% || (
                        echo Trivy scan completed or scanner not available.
                        exit /b 0
                    )
                '''
            }
        }

        stage('Container Registry') {
            steps {
                bat '''
                    docker ps -q -f name=inventory-registry -f status=running | findstr . >NUL 2>&1 || (
                        docker rm -f inventory-registry 2>NUL || ver >NUL
                        docker run -d -p 2000:5000 --restart unless-stopped --name inventory-registry registry:2
                        timeout /t 3 /nobreak >NUL
                    )
                    docker push %REGISTRY_IMAGE%
                '''
            }
        }

        stage('Load Image to Kubernetes') {
            steps {
                bat '''
                    echo Loading image into Minikube...
                    minikube image load %IMAGE%
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat '''
                    minikube update-context >NUL 2>&1 || ver >NUL
                    kubectl apply -f kubernetes/namespace.yaml
                    kubectl apply -f kubernetes/monitoring/namespace.yaml
                    kubectl apply -f kubernetes -R
                    kubectl rollout status deployment/%APP% -n %NS% --timeout=120s
                    kubectl get pods -n %NS%
                '''
            }
        }

        stage('Start Services') {
            steps {
                bat '''
                    taskkill /F /IM kubectl.exe 2>NUL || ver >NUL
                    set JENKINS_NODE_COOKIE=dontKillMe
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& kubectl port-forward service/prometheus 1000:1000 -n %MON% > prometheus-pf.log 2>&1"
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& kubectl port-forward service/inventory-service 2001:2001 -n %NS% > inventory-pf.log 2>&1"
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& kubectl port-forward service/grafana 2002:2002 -n %MON% > grafana-pf.log 2>&1"
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& kubectl port-forward service/grafana 1002:1002 -n %MON% > grafana-1002-pf.log 2>&1"
                    timeout /t 5 /nobreak >NUL
                    exit /b 0
                '''
            }
        }
    }

    post {
        success {
            echo '======================================================='
            echo 'INVENTORY CI/CD PIPELINE COMPLETED SUCCESSFULLY'
            echo 'Prometheus: http://localhost:1000'
            echo 'API Docs:   http://localhost:2001/docs'
            echo 'Grafana:    http://localhost:2002 or http://localhost:1002'
            echo 'Registry:   localhost:2000/inventory-service:v1.0.0'
            echo '======================================================='
        }
        failure {
            echo 'Pipeline failed. Check stage logs for details.'
        }
    }
}
