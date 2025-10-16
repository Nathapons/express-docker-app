pipeline {
    agent any // <--- Pipeline นี้ถูกออกแบบให้รันบน Agent/Server ที่มี Docker ติดตั้ง

    // กำหนด environment variables
    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-cred'
        DOCKER_REPO               = "iamsamitdev/express-docker-app-jenkins"
        APP_NAME                  = "express-docker-app-jenkins"
    }

    // กำหนด stages ของ Pipeline
    stages {

        // Stage 1: ดึงโค้ดล่าสุดจาก Git
        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        // Stage 2: ติดตั้ง dependencies และรันเทสต์ใน Node.js container
        stage('Install & Test') {
            steps {
                script {
                    // เพิ่มการตรวจสอบ PATH และแสดงผล
                    sh 'echo "Current PATH is: $PATH"'
                    sh 'echo "Searching for docker executable..."'
                    
                    def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0
                    
                    if (dockerAvailable) {
                        // ... (โค้ดเดิม)
                    } else {
                        echo "Docker not available, skipping tests. Please install Docker on Jenkins agent."
                        // ... (โค้ดเดิม)
                    }
                }
            }
        }

        // Stage 3: Build Docker Image (ใช้ docker build)
        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image: ${DOCKER_REPO}:${BUILD_NUMBER}"
                    docker build --target production -t ${DOCKER_REPO}:${BUILD_NUMBER} -t ${DOCKER_REPO}:latest . // <-- คำสั่งนี้หาไม่เจอ
                """
            }
        }

        // Stage 4: Push Image ไปยัง Docker Hub
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "Logging into Docker Hub..."
                        echo "\${DOCKER_PASS}" | docker login -u "\${DOCKER_USER}" --password-stdin
                        echo "Pushing image to Docker Hub..."
                        docker push ${DOCKER_REPO}:${BUILD_NUMBER}
                        docker push ${DOCKER_REPO}:latest
                        docker logout
                    """
                }
            }
        }

        // Stage 5: เคลียร์ Docker images บน agent
        stage('Cleanup Docker') {
            steps {
                sh """
                    echo "Cleaning up local Docker images/cache on agent..."
                    docker image rm -f ${DOCKER_REPO}:${BUILD_NUMBER} || true
                    docker image rm -f ${DOCKER_REPO}:latest || true
                    docker image prune -af || true
                    docker builder prune -af || true
                """
            }
        }

        // Stage 6: Deploy ไปยังเครื่อง local
        stage('Deploy Local') {
            steps {
                sh """
                    echo "Deploying container ${APP_NAME} from latest image..."
                    docker pull ${DOCKER_REPO}:latest
                    docker stop ${APP_NAME} || true
                    docker rm ${APP_NAME} || true
                    docker run -d --name ${APP_NAME} -p 3000:3000 ${DOCKER_REPO}:latest
                    docker ps --filter name=${APP_NAME} --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}"
                """
            }
        }

        // Stage 7: Deploy ไปยังเครื่อง remote server (ถ้ามี)
        // ต้องตั้งค่า SSH Key และอนุญาตให้ Jenkins เข้าถึง server
        // stage('Deploy to Server') {
        //     steps {
        //         script {
        //             def isWindows = isUnix() ? false : true
        //             echo "Deploying to remote server..."
        //             if (isWindows) {
        //                 bat """
        //                     ssh -o StrictHostKeyChecking=no user@your-server-ip \\
        //                     'docker pull ${DOCKER_REPO}:latest && \\
        //                     docker stop ${APP_NAME} || echo ignore && \\
        //                     docker rm ${APP_NAME} || echo ignore && \\
        //                     docker run -d --name ${APP_NAME} -p 3000:3000 ${DOCKER_REPO}:latest && \\
        //                     docker ps --filter name=${APP_NAME} --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"'
        //                 """
        //             } else {
        //                 sh """
        //                     ssh -o StrictHostKeyChecking=no user@your-server-ip \\
        //                     'docker pull ${DOCKER_REPO}:latest && \\
        //                     docker stop ${APP_NAME} || true && \\
        //                     docker rm ${APP_NAME} || true && \\
        //                     docker run -d --name ${APP_NAME} -p 3000:3000 ${DOCKER_REPO}:latest && \\
        //                     docker ps --filter name=${APP_NAME} --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"'
        //                 """
        //             }
        //         }
        //     }
        // }

    }
}