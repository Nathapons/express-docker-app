pipeline {
    // ใช้ agent any เพราะ Jenkins ไม่รองรับ Docker agent
    agent any

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
                    // เพิ่มการตรวจสอบ PATH และ Docker ให้ละเอียดขึ้นสำหรับการดีบั๊ก
                    sh 'echo "Current PATH is: $PATH"'
                    sh 'echo "Attempting to locate Docker..."'
                    
                    // ใช้ 'which docker' เพื่อตรวจสอบว่าคำสั่ง Docker อยู่ใน PATH หรือไม่
                    def dockerAvailable = sh(script: 'which docker', returnStatus: true) == 0
                    
                    if (dockerAvailable) {
                        echo "Docker is available. Running npm install and tests inside Node.js container."
                        sh '''
                            docker run --rm -v "$PWD":/app -w /app node:18-alpine sh -c "npm install && npm test"
                        '''
                    } else {
                        // ถ้า Docker ไม่พร้อมใช้งาน จะข้ามส่วนนี้และแจ้งเตือน
                        echo "========================================================================="
                        echo "🚫 Docker NOT available. Please install Docker and ensure the 'jenkins' user"
                        echo "   has permission (e.g., added to 'docker' group) on the Jenkins Agent/Server."
                        echo "   Tests will be skipped in this build. Subsequent Docker stages may fail."
                        echo "========================================================================="
                    }
                }
            }
        }

        // Stage 3: สร้าง Docker Image
        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image: ${DOCKER_REPO}:${BUILD_NUMBER}"
                    // Docker build จะล้มเหลวที่นี่ถ้า Docker ยังเข้าถึงไม่ได้
                    docker build --target production -t ${DOCKER_REPO}:${BUILD_NUMBER} -t ${DOCKER_REPO}:latest .
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

        // Stage 6: Deploy ไปยังเครื่อง local (Jenkins Agent)
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

        // Stage 7: Deploy ไปยังเครื่อง remote server (ถูกคอมเมนต์ไว้)
        // ต้องตั้งค่า SSH Key และอนุญาตให้ Jenkins เข้าถึง server
        /*
        stage('Deploy to Server') {
             steps {
                 // ... (โค้ดเดิม)
             }
        }
        */
    }
}