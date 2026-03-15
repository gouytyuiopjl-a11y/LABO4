# LABO4
DEP-MGPU
# Лабораторная работа Git - Вариант 12

## Студент
- **ФИО:** ДАНСУ КВЕНТИН
- **Группа:** БД-251м
- **Вариант:** 12
- **Дата:** 14.03.2026
pipeline {
    agent any
    
    environment {
        // Конфигурация
        IMAGE_NAME = "мой-app-аналитика"
        // 500 МБ в байтах (500 * 1024 * 1024)
        MAX_IMAGE_SIZE = "524288000"
        BUILD_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Подготовка') {
            steps {
                echo '📦 Установка зависимостей...'
                sh 'pip install --user pylint'
                sh 'pip install --user -r requirements.txt || true'
            }
        }
        
        stage('Линтинг') {
            steps {
                echo '🔍 Проверка кода с Pylint...'
                script {
                    try {
                        sh '~/.local/bin/pylint --fail-under=5.0 src/*.py'
                    } catch (Exception e) {
                        echo '⚠️  Внимание: Обнаружены проблемы стиля, но пайплайн продолжается'
                    }
                }
            }
        }
        
        stage('Сборка Docker образа') {
            steps {
                echo '🏗️  Сборка Docker образа...'
                sh "docker build -t ${IMAGE_NAME}:${BUILD_TAG} ."
                sh "docker tag ${IMAGE_NAME}:${BUILD_TAG} ${IMAGE_NAME}:latest"
            }
        }
        
        stage('Проверка размера образа') {
            steps {
                echo '📏 Проверка размера образа...'
                script {
                    // Получаем размер образа в байтах
                    def imageSize = sh(
                        script: "docker image inspect ${IMAGE_NAME}:${BUILD_TAG} --format='{{.Size}}'",
                        returnStdout: true
                    ).trim()
                    
                    // Конвертируем в МБ для удобного отображения
                    def imageSizeMB = imageSize.toLong() / (1024 * 1024)
                    def maxSizeMB = MAX_IMAGE_SIZE.toLong() / (1024 * 1024)
                    
                    echo "📊 Размер образа: ${imageSizeMB} МБ"
                    echo "📊 Допустимый лимит: ${maxSizeMB} МБ"
                    
                    // Проверяем, не превышает ли размер лимит
                    if (imageSize.toLong() > MAX_IMAGE_SIZE.toLong()) {
                        error("""
                            ╔══════════════════════════════════════════════════════════╗
                            ║     ❌ ОШИБКА: ОБРАЗ СЛИШКОМ БОЛЬШОЙ!                     ║
                            ╠══════════════════════════════════════════════════════════╣
                            ║  Текущий размер: ${imageSizeMB} МБ                         
                            ║  Максимальный лимит: ${maxSizeMB} МБ                       
                            ║                                                           
                            ║  ❌ Пайплайн остановлен из-за превышения                   
                            ║     допустимого размера Docker образа.                    
                            ║                                                           
                            ║  Рекомендации:                                             
                            ║  - Используйте более легкий базовый образ (alpine)        
                            ║  - Очищайте кэш в Dockerfile                               
                            ║  - Удаляйте ненужные файлы после установки                
                            ╚══════════════════════════════════════════════════════════╝
                        """)
                    } else {
                        echo "✅ УСПЕХ: Размер образа (${imageSizeMB} МБ) в пределах лимита (${maxSizeMB} МБ)"
                    }
                }
            }
        }
        
        stage('Тестирование') {
            steps {
                echo '🧪 Тестирование образа...'
                sh "docker run --rm ${IMAGE_NAME}:${BUILD_TAG} python src/app.py --version"
                sh "docker run --rm ${IMAGE_NAME}:${BUILD_TAG} python src/app.py"
            }
        }
        
        stage('Оптимизация (Опционально)') {
            when {
                expression { 
                    // Этот этап выполняется только если образ близок к лимиту
                    def imageSize = sh(
                        script: "docker image inspect ${IMAGE_NAME}:${BUILD_TAG} --format='{{.Size}}'",
                        returnStdout: true
                    ).trim().toLong()
                    def threshold = MAX_IMAGE_SIZE.toLong() * 0.8 // 80% от лимита
                    return imageSize > threshold
                }
            }
            steps {
                echo '⚠️  Образ приближается к лимиту размера. Советы по оптимизации:'
                echo '   - Используйте легковесный базовый образ (python:3.9-alpine)'
                echo '   - Очищайте кэш pip: RUN pip cache purge'
                echo '   - Объединяйте команды RUN для уменьшения слоев'
            }
        }
    }
    
    post {
        always {
            echo '🧹 Очистка...'
            sh "docker rmi ${IMAGE_NAME}:${BUILD_TAG} || true"
        }
        success {
            echo '''
╔══════════════════════════════════════════════════════════╗
║  ✅ ПАЙПЛАЙН УСПЕШНО ЗАВЕРШЕН!                             ║
║                                                            ║
║  Docker образ успешно создан и проверен.                  ║
║  Размер образа соответствует лимиту в 500 МБ.             ║
╚══════════════════════════════════════════════════════════╝
            '''
        }
        failure {
            echo '''
╔══════════════════════════════════════════════════════════╗
║  ❌ ПАЙПЛАЙН ЗАВЕРШИЛСЯ ОШИБКОЙ                            ║
║                                                            ║
║  Пайплайн завершился с ошибкой. Проверьте логи выше       ║
║  для получения детальной информации.                       ║
╚══════════════════════════════════════════════════════════╝
            '''
        }
    }
}
