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
                    env.CURRENT_IP = sh(script: '''
                        TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
                        curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
                    ''', returnStdout: true).trim()
                    echo "Detected EC2 Public IP: ${env.CURRENT_IP}"
                    if (env.CURRENT_IP == "") {
                        error("Failed to detect EC2 public IP - aborting build")
                    }
                }
            }
        }

        stage("Update SonarQube Webhook"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'sonar-admin-creds', usernameVariable: 'SONAR_USER', passwordVariable: 'SONAR_PASS')]) {
                    sh """
                    # Delete old webhook if it exists (ignore failure if none found)
                    WEBHOOK_KEY=\$(curl -s -u \$SONAR_USER:\$SONAR_PASS "http://${env.CURRENT_IP}:9000/api/webhooks/list" | grep -o '\"key\":\"[^\"]*\"' | head -1 | cut -d'\"' -f4)
                    if [ ! -z "\$WEBHOOK_KEY" ]; then
                        curl -s -u \$SONAR_USER:\$SONAR_PASS -X POST "http://${env.CURRENT_IP}:9000/api/webhooks/delete" -d "webhook=\$WEBHOOK_KEY"
                    fi

                    # Create fresh webhook pointing to current Jenkins IP
                    curl -s -u \$SONAR_USER:\$SONAR_PASS -X POST "http://${env.CURRENT_IP}:9000/api/webhooks/create" \
                        -d "name=jenkins-wanderlust" \
                        -d "url=http://${env.CURRENT_IP}:8080/sonarqube-webhook/"
                    """
                }
            }
        }

        stage("Update GitHub Webhook"){
            steps{
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    script{
                        def repo = "pradeep-kumarl/wanderlust"
                        def newUrl = "http://${env.CURRENT_IP}:8080/github-webhook/"

                        writeFile file: 'webhook_payload.json', text: """{"name":"web","active":true,"events":["push"],"config":{"url":"${newUrl}","content_type":"json"}}"""

                        sh """
                        HOOK_ID=\$(curl -s -H "Authorization: token \$GITHUB_TOKEN" "https://api.github.com/repos/${repo}/hooks" | grep -B5 "github-webhook" | grep '"id"' | head -1 | grep -o '[0-9]*' || true)

                        if [ ! -z "\$HOOK_ID" ]; then
                            curl -s -X PATCH -H "Authorization: token \$GITHUB_TOKEN" "https://api.github.com/repos/${repo}/hooks/\$HOOK_ID" -d @webhook_payload.json
                        else
                            curl -s -X POST -H "Authorization: token \$GITHUB_TOKEN" "https://api.github.com/repos/${repo}/hooks" -d @webhook_payload.json
                        fi

                        rm -f webhook_payload.json
                        """
                    }
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
                sh "docker compose build --no-cache"
                sh "docker compose up -d"
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
