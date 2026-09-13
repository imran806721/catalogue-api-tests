pipeline {
    agent {
        node {
            label 'ROBOSHOP'
        }
    }
    parameters {
        string(name: 'NAMESPACE', defaultValue: 'roboshop-dev', description: 'K8s namespace the target catalogue deployment lives in')
        string(name: 'COMMIT_ID', defaultValue: '', description: 'Commit SHA of the catalogue build under test (for traceability)')
    }
    options {
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
    }
    stages {
        stage('install-dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('resolve-catalogue-endpoint') {
            steps {
                script {
                    withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                        sh "aws eks update-kubeconfig --name roboshop --region us-east-1"
                        def podIp = sh(
                            script: "kubectl get pods -n ${params.NAMESPACE} -l project=roboshop,tier=backend,component=catalogue -o jsonpath='{.items[0].status.podIP}'",
                            returnStdout: true
                        ).trim()
                        if (!podIp) {
                            error("No running catalogue pod found in namespace ${params.NAMESPACE}")
                        }
                        env.CATALOGUE_URL = "http://${podIp}:8080"
                        echo "Testing against catalogue pod at ${env.CATALOGUE_URL}"
                    }
                }
            }
        }
        /* stage('api-tests') {
            steps {
                sh 'npm test'
            }
        } */

        stage('API Tests') {
    steps {
        script {
            withAWS(credentials: 'aws-cred', region: 'us-east-1') {
                sh '''
                    set -e

                    aws eks update-kubeconfig \
                      --name roboshop \
                      --region us-east-1

                    echo "Checking catalogue service..."
                    kubectl get svc catalogue -n roboshop-dev

                    echo "Starting port-forward..."
                    kubectl -n roboshop-dev port-forward \
                      svc/catalogue 18080:8080 \
                      > /tmp/catalogue-port-forward.log 2>&1 &

                    PF_PID=$!

                    cleanup() {
                        echo "Stopping port-forward..."
                        kill $PF_PID 2>/dev/null || true
                    }

                    trap cleanup EXIT

                    echo "Waiting for catalogue..."
                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:18080/health; then
                            echo
                            echo "Catalogue is reachable"
                            break
                        fi

                        if ! kill -0 $PF_PID 2>/dev/null; then
                            echo "Port-forward exited unexpectedly"
                            cat /tmp/catalogue-port-forward.log
                            exit 1
                        fi

                        sleep 2
                    done

                    echo "Running catalogue API tests..."

                    export CATALOGUE_URL=http://127.0.0.1:18080

                    npm test
                '''
            }
        }
    }
}
    }
    post {
        always {
            junit testResults: 'junit.xml', allowEmptyResults: true
            archiveArtifacts artifacts: 'junit.xml', allowEmptyArchive: true
        }
        success {
            echo "catalogue-api-tests passed against ${params.NAMESPACE} (commit ${params.COMMIT_ID})"
        }
        failure {
            echo "catalogue-api-tests failed against ${params.NAMESPACE} (commit ${params.COMMIT_ID}) — see junit report"
        }
    }
}