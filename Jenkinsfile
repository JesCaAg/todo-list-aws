pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                withCredentials([string(credentialsId: 'token', variable: 'tokengh')]) {
                    git branch: 'develop', url: 'https://$tokengh@github.com/JesCaAg/todo-list-aws.git' // Traemos el codigo del repo autenticandonos                    
                }
                sh 'wget https://raw.githubusercontent.com/JesCaAg/todo-list-aws-config/refs/heads/staging/samconfig.toml'
            }
        }
        
        stage('Static Test') {
            parallel { 
                stage('Static'){ // Ejecucion de pruebas de analisis de codigo estatico, usando flake8
                    steps {
                        sh 'python -m flake8 --format=pylint --exit-zero src > flake8.out'
                        recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]
                        sh '[ -f flake8.out ] && rm flake8.out'
                    }
                }
                
                stage('Security test'){ // Ejecucion de pruebas de seguridad, usando bandit
                    steps {
                        sh 'python -m bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                        recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
                        sh '[ -f bandit.out ] && rm bandit.out'
                    }
                }
            }
        }
        
        stage ('Deploy') { // Despliegue del stack de staging
            steps {
                sh '''
                    sam build --config-env staging
                    sam validate --config-env staging --region us-east-1
                    sam deploy --no-confirm-changeset --no-fail-on-empty-changeset --stack-name staging-todo-list-aws --config-env staging --resolve-s3
                '''
                sh '[ -f samconfig.toml ] && rm samconfig.toml'
            }
        }
        
        stage ('Rest Test') { // Test de integracion
            steps {
                script {
                    def URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name staging-todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()
                    withEnv(["BASE_URL=${URL}"]) {
                        def result = sh (script: 'python -m pytest --junitxml=junit-rest.xml test/integration/todoApiTest.py', returnStatus: true)
                        junit 'junit-rest.xml'
                        sh '[ -f junit-rest.xml ] && rm junit-rest.xml'
                        if (result != 0) {
                            error "Fallo alguna prueba, abortando pipeline."
                        }
                    }
                }
            }
        }
        
        stage ('Promote') {
            steps {
                withCredentials([string(credentialsId: 'token', variable: 'tokengh')]) {
                    sh '''
                        git remote set-url origin https://$tokengh@github.com/JesCaAg/todo-list-aws.git
                        git fetch origin
                        git checkout master
                        git merge origin/develop --no-commit --no-ff || true
                        git checkout origin/master -- Jenkinsfile_agentes Jenkinsfile
                        git add .
                        git commit -m "Mergeo de develop en master manteniendo Jenkinsfile original" || echo "Nada que commitear"
                        git push origin master
                    '''
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
