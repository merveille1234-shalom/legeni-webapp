// Lot 4 : intégration continue (qualité en parallèle, build, scan, publication)
pipeline {
    agent any

    environment {
        IMAGE = 'joslath/legeni-webapp'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    stages {
        stage('Préparation') {
            steps {
                script {
                    env.APP_VERSION = readFile('VERSION').trim()
                    env.TAG = "${env.APP_VERSION}-${env.BUILD_NUMBER}"
                }
                echo "Version ${env.TAG} - branche ${env.BRANCH_NAME}"
            }
        }

        stage('Qualité') {
            parallel {
                stage('Tests et couverture') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q -r requirements.txt -r requirements-dev.txt
                            mkdir -p reports
                            /tmp/venv/bin/pytest --junitxml=reports/junit.xml \
                                --cov=app --cov-report=xml:reports/coverage.xml \
                                --cov-report=term --cov-fail-under=80
                        '''
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                            recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                        }
                    }
                }
                stage('Lint Python') {
                    agent {
                        docker {
                            image 'python:3.12-slim'
                            args '-e HOME=/tmp'
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            python -m venv /tmp/venv
                            /tmp/venv/bin/pip install -q flake8==7.1.1
                            mkdir -p reports
                            /tmp/venv/bin/flake8 --output-file=reports/flake8.txt --tee .
                        '''
                    }
                    post {
                        always {
                            recordIssues(tools: [flake8(pattern: 'reports/flake8.txt')])
                        }
                    }
                }
                stage('Lint Dockerfile') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:v2.12.0 < Dockerfile'
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        sh 'cd ansible && ansible-playbook provision.yml deploy.yml --syntax-check'
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$TAG -t $IMAGE:$TAG .'
            }
        }

        stage('Scan Trivy') {
            steps {
                // échoue s'il reste une vulnérabilité CRITICAL corrigeable
                sh '''
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy_cache:/root/.cache \
                        aquasec/trivy:0.57.1 image --timeout 15m --scanners vuln \
                        --severity CRITICAL --ignore-unfixed --exit-code 1 $IMAGE:$TAG
                '''
            }
        }

        stage('Push registre') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Déploiement staging') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS_FILE')]) {
                    sh '''
                        export K8S_AUTH_KUBECONFIG="$KUBECONFIG_FILE"
                        cd ansible
                        ansible-playbook deploy.yml -e target_env=staging -e app_version=$TAG \
                            --vault-password-file "$VAULT_PASS_FILE"
                    '''
                }
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Déployer ${env.TAG} en production ?", ok: 'Déployer en production'
                }
            }
        }

        stage('Déploiement production') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS_FILE')]) {
                    sh '''
                        export K8S_AUTH_KUBECONFIG="$KUBECONFIG_FILE"
                        cd ansible
                        ansible-playbook deploy.yml -e target_env=prod -e app_version=$TAG \
                            --vault-password-file "$VAULT_PASS_FILE"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rmi $IMAGE:$TAG || true'
        }
        failure {
            script {
                if (env.BRANCH_NAME == 'main') {
                    try {
                        withCredentials([string(credentialsId: 'discord-webhook', variable: 'HOOK')]) {
                            httpRequest url: HOOK, httpMode: 'POST', contentType: 'APPLICATION_JSON',
                                        requestBody: "{\"content\": \"Pipeline en échec : ${env.JOB_NAME} #${env.BUILD_NUMBER}\"}",
                                        quiet: true
                        }
                    } catch (err) {
                        echo "Notification non envoyée : ${err.getMessage()}"
                    }
                }
            }
        }
        cleanup {
            cleanWs()
        }
    }
}
