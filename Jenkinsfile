pipeline {
    agent any

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-2', description: 'Región de AWS donde están tus recursos')
        string(name: 'ECR_REPO', defaultValue: 'nginx-ecs-demo', description: 'Nombre de tu repositorio en ECR')
        string(name: 'ECS_CLUSTER', defaultValue: 'ecs-lab-cluster', description: 'Nombre de tu clúster de ECS')
        string(name: 'ECS_SERVICE', defaultValue: 'nginx-lab-svc', description: 'Nombre de tu servicio en ECS')
        string(name: 'TASK_FAMILY', defaultValue: 'nginx-lab-task', description: 'Nombre (familia) de tu Task Definition')
        string(name: 'ACCOUNT_ID', defaultValue: '324441770509', description: 'El ID de tu cuenta de AWS')
    }

    environment {
        ECR_URL = "${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com/${params.ECR_REPO}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push to ECR') {
            steps {
                withAWS(credentials: 'aws-lab', region: params.AWS_REGION) {
                    script {
                        echo "1. Autenticando Docker con el registro de ECR..."
                        sh "aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com"

                        echo "2. Construyendo la imagen de Docker..."
                        sh "docker build -t ${env.ECR_URL}:latest ."

                        echo "3. Subiendo la imagen a ECR..."
                        sh "docker push ${env.ECR_URL}:latest"
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withAWS(credentials: 'aws-lab', region: params.AWS_REGION) {
                    script {
                        echo "1. Obteniendo la definición de tarea actual..."
                        def taskDefJson = sh(script: "aws ecs describe-task-definition --task-definition ${params.TASK_FAMILY} --region ${params.AWS_REGION}", returnStdout: true).trim()

                        echo "2. Creando una nueva revisión de la Task Definition..."
                        def newTaskDefPayload = sh(
                            script: """
                                echo '${taskDefJson}' | jq '.taskDefinition | .containerDefinitions[0].image = "${env.ECR_URL}:latest" | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)'
                            """,
                            returnStdout: true
                        ).trim()

                        def newTaskDefArn = sh(
                            script: "aws ecs register-task-definition --cli-input-json '${newTaskDefPayload}' --region ${params.AWS_REGION} | jq -r '.taskDefinition.taskDefinitionArn'",
                            returnStdout: true
                        ).trim()

                        echo "3. Actualizando el servicio de ECS para usar la nueva tarea: ${newTaskDefArn}"
                        sh "aws ecs update-service --cluster ${params.ECS_CLUSTER} --service ${params.ECS_SERVICE} --task-definition ${newTaskDefArn} --force-new-deployment --region ${params.AWS_REGION}"
                    }
                }
            }
        }
    }
}
// Fin del archivo Jenkinsfile
