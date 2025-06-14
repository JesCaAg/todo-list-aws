pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                withCredentials([string(credentialsId: 'token', variable: 'tokengh')]) {
                    git branch: 'master', url: 'https://$tokengh@github.com/JesCaAg/todo-list-aws.git' // Traemos el codigo del repo autenticandonos
                }
                sh 'https://raw.githubusercontent.com/JesCaAg/todo-list-aws-config/refs/heads/production/samconfig.toml'
            }
        }
        
        stage ('Deploy') { // Despliegue del stack de production
            steps {
                sh '''
                    sam build --config-env production
                    sam validate --config-env production --region us-east-1
                    sam deploy --no-confirm-changeset --no-fail-on-empty-changeset --stack-name production-todo-list-aws --config-env production --resolve-s3
                '''
                sh '[ -f samconfig.toml ] && rm samconfig.toml'
            }
        }
        
        stage ('Rest Test') { // Test de integracion
            steps {
                script {
                    def URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name production-todo-list-aws --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()
                    withEnv(["BASE_URL=${URL}"]) {
                        def result = sh (script: 'python -m pytest -m soloLectura --junitxml=junit-rest.xml test/integration/todoApiTest.py', returnStatus: true)
                        junit 'junit-rest.xml'
                        sh '[ -f junit-rest.xml ] && rm junit-rest.xml'
                        if (result != 0) {
                            error "Fallo alguna prueba, abortando pipeline."
                        }
                    }
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
