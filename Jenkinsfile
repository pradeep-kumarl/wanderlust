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
MONGO_URI=mongodb://mongo:27017/wanderlust
REDIS_URL=redis://redis:6379
FRONTEND_URL=http://${env.CURRENT_IP}:5173
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
                sh "docker compose up -d --build"
            }
        }
    }
}
