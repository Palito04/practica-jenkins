pipeline {
    agent any

    parameters {
        string(name: 'AWS_REGION', defaultValue: 'us-east-2', description: 'Región de AWS donde están tus recursos')
        string(name: 'ECR_REPO', defaultValue: 'nginx-ecs-demo', description: 'Nombre de tu repositorio en ECR')
        string(name: 'ECS_CLUSTER', defaultValue: 'ecs-lab-cluster', description: 'Nombre de tu clúster de ECS')
        string(name: 'ECS_SERVICE', defaultValue: 'nginx-lab-svc', description: 'Nombre de tu servicio en ECS')
        string(name: 'TASK_FAMILY', defaultValue: 'nginx-lab-task', description: 'Nombre (familia) de tu Task Definition')
        // 🚨 ¡No te olvides de cambiar esto!
        string(name: 'ACCOUNT_ID', defaultValue: '324441770509', description: 'El ID de tu cuenta de AWS')
    }

    environment {
        // Define la URL completa del repositorio de ECR para usarla en los comandos
        ECR_URL = "324441770509.dkr.ecr.us-east-2.amazonaws.com/nginx-ecs-demo"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push to ECR') {
            steps {
                // Envuelve los comandos en withAWS para usar las credenciales guardadas
                withAWS(credentials: 'aws-lab', region: params.AWS_REGION) {
                    script {
                        echo "1. Autenticando Docker con el registro de ECR..."
                        // Obtiene el comando de login de ECR y lo ejecuta
                        sh "aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${params.ACCOUNT_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com"

                        echo "2. Construyendo la imagen de Docker..."
                        // Construye la imagen y la etiqueta como 'latest'
                        sh "docker build -t ${env.ECR_URL}:latest ."

                        echo "3. Subiendo la imagen a ECR..."
                        // Sube la imagen al repositorio
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
                        // Obtiene la definición de la tarea en formato JSON
                        def taskDefJson = sh(script: "aws ecs describe-task-definition --task-definition ${params.TASK_FAMILY} --region ${params.AWS_REGION}", returnStdout: true).trim()

                        echo "2. Creando una nueva revisión de la Task Definition..."
                        // Crea una nueva definición de tarea actualizando la imagen del contenedor
                        // Se usa la herramienta 'jq' para manipular el JSON
                        def newTaskDefJson = sh(script: """
                            echo '${taskDefJson}' | jq --arg IMAGE_URL "${env.ECR_URL}:latest" '.taskDefinition.containerDefinitions[0].image = \$IMAGE_URL' | \
                            jq 'del(.taskDefinition.taskDefinitionArn, .taskDefinition.revision, .taskDefinition.status, .taskDefinition.requiresAttributes, .taskDefinition.compatibilities, .taskDefinition.registeredAt, .taskDefinition.registeredBy)'
                        """, returnStdout: true).trim()

                        def newTaskDef = sh(script: "aws ecs register-task-definition --cli-input-json '${newTaskDefJson}' --region ${params.AWS_REGION} | jq -r '.taskDefinition.taskDefinitionArn'", returnStdout: true).trim()

                        echo "3. Actualizando el servicio de ECS para usar la nueva tarea: ${newTaskDef}"
                        // Fuerza un nuevo despliegue del servicio con la última revisión de la tarea
                        sh "aws ecs update-service --cluster ${params.ECS_CLUSTER} --service ${params.ECS_SERVICE} --task-definition ${newTaskDef} --force-new-deployment --region ${params.AWS_REGION}"
                    }
                }
            }
        }
    }
}
