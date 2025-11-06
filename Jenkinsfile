pipeline {
    agent any

    environment {
        // Ortelius Configuration
        DHURL = "http://34.50.82.137"
        DHUSER = "admin"
        DHPASS = "admin"

        APP_NAME = "GLOBAL.JavaEcommerce"
        APP_VERSION = "v1.0.0"
        COMPONENT_NAME = "GLOBAL.JavaEcommerce.Backend"
        BUILD_NUM = "${env.BUILD_NUMBER}"

        // GitHub repo
        GITHUB_REPO_URL = "https://github.com/koojennie/ecommerce-springboot.git"
        GITHUB_AUTH_TOKEN = credentials('github-token')

        // NVD API
        NVD_API_KEY = credentials('NVD_API_KEY')

        // Path
        PATH = "/usr/local/bin:${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master2', url: "${GITHUB_REPO_URL}"
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                echo "📦 Installing Ortelius CLI..."
                curl -L https://github.com/ortelius/ortelius-cli/releases/download/v9.3.283/ortelius-linux-amd64.tar.gz -o dh.tar.gz
                tar -xvf dh.tar.gz
                chmod +x ortelius
                mv ortelius dh

                echo "📦 Installing Syft (SBOM generator)..."
                curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b $PWD

                echo "📦 Installing OWASP Dependency-Check..."
                curl -L https://github.com/jeremylong/DependencyCheck/releases/download/v12.1.0/dependency-check-12.1.0-release.zip -o depcheck.zip
                unzip depcheck.zip
                mv dependency-check dependency-check-tmp
                mv dependency-check-tmp/dependency-check ./dependency-check
                rm -rf dependency-check-tmp
                chmod +x dependency-check/bin/dependency-check.sh

                echo "📊 Installing OpenSSF Scorecard..."
                curl -L https://github.com/ossf/scorecard/releases/download/v5.3.0/scorecard_5.3.0_linux_amd64.tar.gz -o scorecard.tar.gz
                tar -xzf scorecard.tar.gz && chmod +x scorecard
                '''
            }
        }

        stage('Build & Test') {
            steps {
                dir('JtProject') {
                    sh '''
                    echo "🏗️ Building Java Spring Boot project..."
                    mvn clean package -DskipTests
                    '''
                }
            }
        }

        stage('Security Scanning & SBOM') {
            environment {
                OSS_INDEX_USERNAME = credentials('OSS_INDEX_CREDS_USR')
                OSS_INDEX_PASSWORD = credentials('OSS_INDEX_CREDS_PSW')
            }
            steps {
                dir('JtProject') {
                    sh '''
                    echo "🧩 Generating SBOM using Syft..."
                    ../syft . -o cyclonedx-json > sbom.json || true

                    echo "🛡️ Running OWASP Dependency-Check..."
                    ../dependency-check/bin/dependency-check.sh \
                        --project "JtSpringProject" \
                        --scan ./ \
                        --format JSON \
                        --out dependency-check-report \
                        --data ../dependency-check-data \
                        --nvdApiKey $NVD_API_KEY \
                        --disableAssembly \
                        --disableOssIndex \
                        --ossIndexUsername "$OSS_INDEX_USERNAME" \
                        --ossIndexPassword "$OSS_INDEX_PASSWORD"
                    '''
                }
            }
        }

        stage('Generate OSSF Scorecard') {
            steps {
                dir('JtProject') {
                    sh '''
                    echo "🔑 Running OSSF Scorecard..."
                    export GITHUB_AUTH_TOKEN=${GITHUB_AUTH_TOKEN}
                    ../scorecard --repo=${GITHUB_REPO_URL} --format json --show-details > scorecard_raw.json

                    echo "🧹 Normalizing Scorecard JSON..."
                    cat scorecard_raw.json | jq '{
                        repo: {name: .repo.name},
                        score: .score,
                        checks: [.checks[] | {name: .check, score: .score}]
                    }' > scorecard.json
                    '''
                }
            }
        }

        stage('Prepare Metadata') {
            steps {
                dir('JtProject') {
                    writeFile file: 'component.toml', text: """
Application = "${APP_NAME}"
Application_Version = "${APP_VERSION}"
Name = "${COMPONENT_NAME}"
Variant = "springboot-backend"
Version = "v${APP_VERSION}.${BUILD_NUM}"

[Attributes]
  ServiceOwner = "${DHUSER}"
  ServiceOwnerEmail = "aliyajenny14@gmail.com"
  SourceUrl = "${GITHUB_REPO_URL}"
"""
                }
            }
        }

        stage('Publish to Ortelius') {
            steps {
                dir('JtProject') {
                    sh '''
                    echo "🚀 Uploading component and reports to Ortelius..."
                    export DHURL=${DHURL}
                    export DHUSER=${DHUSER}
                    export DHPASS=${DHPASS}

                    ../dh updatecomp --rsp component.toml \
                        --deppkg "cyclonedx@sbom.json" \
                        --deppkg "dependencycheck@dependency-check-report/dependency-check-report.json" \
                        --deppkg "scorecard@scorecard.json" \
                        --deploydatasave component.json
                    '''
                }
            }
        }

        stage('Deploy to Application') {
            steps {
                sh '''
                echo "📦 Linking component into Ortelius Application..."
                ./dh deploy \
                    --dhurl ${DHURL} \
                    --dhuser ${DHUSER} \
                    --dhpass ${DHPASS} \
                    --appname ${APP_NAME} \
                    --appversion ${APP_VERSION} \
                    --deployenv "GLOBAL.JavaEcommerce.Dev" \
                    --deploydata JtProject/component.json \
                    --logdeployment
                '''
            }
        }

        stage('Upload Scorecard to Dashboard (Optional)') {
            steps {
                dir('JtProject') {
                    sh '''
                    echo "📤 Uploading Scorecard JSON to Ortelius metrics dashboard..."
                    curl -X POST -u ${DHUSER}:${DHPASS} \
                    -H "Content-Type: application/json" \
                    -d @scorecard.json \
                    ${DHURL}/msapi/metrics
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'JtProject/**/*.json', fingerprint: true
            cleanWs()
        }
        success {
            echo '✅ Build, SBOM, Vulnerability, and Ortelius upload successful!'
        }
        failure {
            echo '❌ Pipeline failed. Check Jenkins logs.'
        }
    }
}
