pipeline{
    agent any
    environment{
        SONAR_HOME = tool "sonar"
    }
    stages{
        stage("Clone Code from GitHub"){
            steps{
                git url: "https://github.com/pradeep-kumarl/wanderlust.git", branch: "devops"
            }
        }

        stage("Detect Current Public IP"){
            steps{
                script{
                    env.CURRENT_IP = sh(script: "curl -s http://169.254.169.254/latest/meta-data/public-ipv4", returnStdout: true).trim()
                    echo "Detected EC2 Public IP: ${env.CURRENT_IP}"
                }
            }
        }

        stage("Prepare Env Files"){
            steps{
                script{
                    // Backend .env
                    writeFile file: 'backend/.env', text: """PORT=5000
MONGODB_URI="mongodb://mongo:27017/wanderlust"
REDIS_URL="redis://redis-container:6379"
FRONTEND_URL=http://${env.CURRENT_IP}:5173
BACKEND_URL=http://${env.CURRENT_IP}:5000
ACCESS_COOKIE_MAXAGE=120000
ACCESS_TOKEN_EXPIRES_IN='120s'
REFRESH_COOKIE_MAXAGE=120000
REFRESH_TOKEN_EXPIRES_IN='120s'
JWT_SECRET=7ddd8b38486eee723ce2505f6db06f1ee503fde5eb06fc04687191a0ed665f3f98776902d2c89f6b993b1c579a87fedaf584c693a106f7cbf16e8b4e67e9d6df
NODE_ENV=Development
GOOGLE_CLIENT_ID=your_actual_google_client_id_here
GOOGLE_CLIENT_SECRET=your_actual_google_client_secret_here
"""

                    // Frontend .env
                    writeFile file: 'frontend/.env', text: """VITE_API_PATH=http://${env.CURRENT_IP}:5000
"""
                }
            }
        }

        stage("SonarQube Quality Analysis"){
            steps{
                withSonarQubeEnv("sonar"){
                    sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=wanderlust -Dsonar.projectKey=wanderlust"
                }
            }
        }

        stage("Sonar Quality Gate Scan") {
            steps{
                timeout(time: 2, unit: "MINUTES"){
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage("OWASP Dependency Check"){
            steps{
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--scan ./ --nvdApiKey ${NVD_API_KEY} --format ALL", odcInstallation: 'dep-owasp'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File System Scan"){
            steps{
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage("Deploy with Docker Compose"){
            steps{
                sh "docker compose down"
                sh "docker compose up -d --build --no-cache"
            }
        }
        stage("Cleanup Old Images"){
            steps{
                sh "docker image prune -f"
                sh "docker builder prune -f"
       }
  }
    }
}
