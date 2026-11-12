pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'prod'], description: 'Target environment')
        string(name: 'NODE_IP', defaultValue: '192.168.67.147', description: 'Node IP for the smoke test')
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
         TARGET_ENV = "${params.ENV}"
	 NS = "grafana-${params.ENV}"
	 NODE_IP    = "${params.NODE_IP}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Prepare Namespace') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"'
                }
            }
        }

        stage('Validate') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl kustomize overlays/$TARGET_ENV > /dev/null
                        kubectl apply -k overlays/$TARGET_ENV --dry-run=server
                    '''
                }
            }
        }

        stage('Create Secret') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                    string(credentialsId: 'grafana-admin-password', variable: 'GF_PASS')
                ]) {
                    sh '''
                        kubectl -n "$NS" create secret generic grafana-admin \
                          --from-literal=admin-user=admin \
                          --from-literal=admin-password="$GF_PASS" \
                          --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh 'kubectl apply -k overlays/$TARGET_ENV'
                }
            }
        }

        stage('Verify Rollout') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl -n "$NS" rollout status deployment/grafana --timeout=180s
                        kubectl -n "$NS" get pods,svc,pvc
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        PORT=$(kubectl -n "$NS" get svc grafana -o jsonpath='{.spec.ports[0].nodePort}')
                        curl -fsS --retry 10 --retry-delay 5 --retry-connrefused \
                          "http://$NODE_IP:$PORT/api/health"
                    '''
                }
            }
        }
    }

    post {
        failure {
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh '''
                    kubectl -n "$NS" get pods || true
                    kubectl -n "$NS" describe deployment grafana || true
                    kubectl -n "$NS" rollout undo deployment/grafana || true
                '''
            }
        }
        success {
            echo "Grafana deployed to ${NS}"
        }
    }
}
