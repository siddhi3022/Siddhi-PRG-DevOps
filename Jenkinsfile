pipeline {
    agent any

    environment {
        APP = 'inventory-service'
        NS = 'inventory-system'
        MON = 'monitoring'
        VERSION = 'v1.0.0'
        IMAGE = 'inventory-service:v1.0.0'
        REGISTRY_IMAGE = 'localhost:2000/inventory-service:v1.0.0'
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
                        trivy image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 %IMAGE%
                    ) || (
                        where docker >NUL 2>&1 && (
                            echo Trivy CLI not found on host PATH. Running Trivy via Docker container...
                            docker run --rm -v //var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:latest image --scanners vuln --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 %IMAGE%
                        ) || (
                            echo Trivy and Docker not found. Skipping Trivy scan.
                        )
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
                    )
                    docker push %REGISTRY_IMAGE%
                '''
            }
        }

        stage('Load Image to Kubernetes') {
            steps {
                bat '''
                    where minikube >NUL 2>&1 && (
                        echo Loading image via minikube CLI...
                        minikube image load %IMAGE% && exit /b 0
                    )

                    docker inspect minikube >NUL 2>&1 && (
                        echo Loading image into Minikube container...
                        docker save -o k8s.tar %IMAGE%
                        docker cp k8s.tar minikube:/k8s.tar
                        docker exec minikube ctr -n k8s.io images import /k8s.tar
                        docker exec minikube rm -f /k8s.tar
                        del /f /q k8s.tar
                        exit /b 0
                    )

                    docker inspect desktop-control-plane >NUL 2>&1 && (
                        echo Loading image into desktop-control-plane...
                        docker save -o k8s.tar %IMAGE%
                        docker cp k8s.tar desktop-control-plane:/k8s.tar
                        docker exec desktop-control-plane ctr -n k8s.io images import /k8s.tar
                        docker exec desktop-control-plane rm -f /k8s.tar
                        del /f /q k8s.tar
                        exit /b 0
                    )

                    echo Using local image cache...
                    exit /b 0
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                bat '''
                    minikube status >NUL 2>&1 || kubectl cluster-info >NUL 2>&1 || (
                        where minikube >NUL 2>&1 && (
                            echo Starting Minikube cluster...
                            minikube start || (
                                echo Minikube cluster corrupted. Resetting Minikube cluster...
                                minikube delete
                                minikube start
                            )
                        )
                    )
                    kubectl apply -f kubernetes/namespace.yaml 2>NUL || ver >NUL
                    kubectl apply -f kubernetes/monitoring/namespace.yaml 2>NUL || ver >NUL
                    kubectl apply -f kubernetes -R
                    kubectl rollout status deployment/%APP% -n %NS% --timeout=120s || (
                        echo Deployment rollout failed. Performing automated rollback...
                        kubectl rollout undo deployment/%APP% -n %NS%
                        exit /b 1
                    )
                    kubectl get pods -n %NS%
                '''
            }
        }

        stage('Start Services') {
            steps {
                bat '''
                    taskkill /F /IM kubectl.exe 2>NUL || ver >NUL
                    set JENKINS_NODE_COOKIE=dontKillMe
                    set BUILD_ID=dontKillMe
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& set BUILD_ID=dontKillMe&& kubectl port-forward service/prometheus 1000:1000 -n %MON% > prometheus-pf.log 2>&1"
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& set BUILD_ID=dontKillMe&& kubectl port-forward service/inventory-service 2001:2001 -n %NS% > inventory-pf.log 2>&1"
                    start "" /B cmd /c "set JENKINS_NODE_COOKIE=dontKillMe&& set BUILD_ID=dontKillMe&& kubectl port-forward service/grafana 2002:2002 -n %MON% > grafana-pf.log 2>&1"
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
            echo 'Grafana:    http://localhost:2002'
            echo 'Registry:   localhost:2000/inventory-service:v1.0.0'
            echo '======================================================='
        }
        failure {
            echo 'Pipeline failed. Check stage logs for details.'
        }
    }
}
