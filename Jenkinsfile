pipeline {
    agent { label 'docker-builder' }

    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        DEPLOY_CONFIG = "deploy-config.yml"
    }

    stages {

        stage('Read Deploy Config') {
    steps {
        echo "=== GitOps: читаем конфигурацию деплоя ==="
        checkout scm

        script {
            def config = readFile('deploy-config.yml')
            echo "=== Текущий конфиг ==="
            echo config

            // Читаем каждую строку и ищем нужные поля
            config.eachLine { line ->
                if (line.trim().startsWith('image:')) {
                    env.DOCKER_IMAGE = line.trim().replace('image:', '').trim()
                }
                if (line.trim().startsWith('tag:')) {
                    env.DOCKER_TAG = line.trim().replace('tag:', '').trim()
                }
            }

            echo "=== Деплоим: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} ==="
        }
    }
}

        stage('Deploy') {
            steps {
                echo "=== Останавливаем старый контейнер ==="
                sh 'docker stop hw34-app || true'
                sh 'docker rm   hw34-app || true'

                echo "=== Запускаем новый контейнер ==="
                sh '''
                    docker run -d \
                        --name hw34-app \
                        -p 5000:5000 \
                        --restart unless-stopped \
                        $DOCKER_IMAGE:$DOCKER_TAG
                '''
                echo "=== ✅ Задеплоено: $DOCKER_IMAGE:$DOCKER_TAG ==="
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "Ждём запуска контейнера..."
                    sleep 3
                    for i in $(seq 1 5); do
                        if docker exec hw34-app python -c \
                            "import urllib.request; urllib.request.urlopen('http://localhost:5000/health'); print('ok')"; then
                            echo "✅ Приложение отвечает!"
                            exit 0
                        fi
                        echo "Попытка $i/5..."
                        sleep 2
                    done
                    echo "❌ Health check не прошёл"
                    docker logs hw34-app --tail 20
                    exit 1
                '''
            }
        }

    }

    post {
        success {
            echo "✅ GitOps Deploy успешен: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}"
        }
        failure {
            echo "❌ GitOps Deploy упал! Проверь логи."
        }
    }
}