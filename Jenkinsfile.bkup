pipeline {
    agent {
        node {
            label 'ROBOSHOP'
        }
    }
    environment {
        appVersion = ""
        ACC_ID     = "764694154057"
        REGION     = "us-east-1"
        AWS_CREDS  = "aws-creds"
        ECR_REGISTRY = "764694154057.dkr.ecr.us-east-1.amazonaws.com"
    }
    options {
        // disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
    }
    /* parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
        booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Toggle this value')
        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    } */
    stages {
        stage ('Read version'){
            steps {
                script {
                    // Read and parse the JSON file
                    def packageJson = readJSON file: 'package.json'
                    
                    // Access fields directly
                    appVersion = packageJson.version
                    
                    echo "Building version ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                    sh """
                        npm install
                    """
                }
            }
        }
        /* stage ('SonarQube Analysis'){
            steps{
                script{
                    def scannerHome = tool name: 'sonar-8' 
                    withSonarQubeEnv('sonar-server') { //analysing and uploading to server
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }
        stage('Quality Gate') {
            steps {
                // Set a timeout so the pipeline doesn't hang if SonarQube is unresponsive
                timeout(time: 2, unit: 'MINUTES') { 
                    waitForQualityGate abortPipeline: true
                }
            }
        } */
        stage('Check Dependabot Alerts') {
            environment {
                REPO_OWNER = 'e-shekharreddy'
                REPO_NAME  = 'catalogue'
            }
            steps {
                script {
                    // Retrieve token securely from Jenkins Credentials ID 'GITHUB_TOKEN'
                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        
                        // Fetch count of open HIGH and CRITICAL alerts
                        def alertCount = sh(
                            script: """
                                curl -s -H "Accept: application/vnd.github+json" \
                                    -H "Authorization: Bearer ${GH_TOKEN}" \
                                    -H "X-GitHub-Api-Version: 2022-11-28" \
                                    "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/dependabot/alerts?state=open&severity=high,critical" \
                                | jq '. | length'
                            """,
                            returnStdout: true
                        ).trim()

                        // Validate API response (ensures jq returned a valid integer)
                        if (!alertCount.isInteger()) {
                            error("Failed to fetch Dependabot alerts. Ensure github-token has 'security_events' read permissions.")
                        }

                        echo "Open High/Critical Dependabot Alerts Found: ${alertCount}"

                        if (alertCount.toInteger() > 0) {
                            // Display detailed alert information in execution output before failing
                            sh """
                                curl -s -H "Accept: application/vnd.github+json" \
                                    -H "Authorization: Bearer ${GH_TOKEN}" \
                                    -H "X-GitHub-Api-Version: 2022-11-28" \
                                    "https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/dependabot/alerts?state=open&severity=high,critical" \
                                | jq '.[] | {number: .number, severity: .security_advisory.severity, summary: .security_advisory.summary, package: .dependency.package.name}'
                            """
                            error("Pipeline failed: Found ${alertCount} open HIGH or CRITICAL Dependabot alert(s) in ${REPO_OWNER}/${REPO_NAME}.")
                        } else {
                            echo "No open HIGH or CRITICAL Dependabot vulnerabilities found. Proceeding."
                        }
                    }
                }
            }
        }
        stage('Build Image') {
            steps {
                script{
                    sh """
                        docker build -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion} .
                    """
                }
                
            }
        }
        stage('Trivy OS Scan') {
            steps {
                script {
                    // Generate table report
                    sh """
                        trivy image \
                        --scanners vuln \
                        --pkg-types os \
                        --severity HIGH,MEDIUM \
                        --format table \
                        --output trivy-os-report.txt \
                        --exit-code 0 \
                        ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion}
                    """

                    // Print table to console
                    sh 'cat trivy-os-report.txt'

                    // Fail pipeline if vulnerabilities found
                    def scanResult = sh(
                        script: """
                            trivy image \
                            --scanners vuln \
                            --pkg-types os \
                            --severity HIGH,MEDIUM \
                            --format table \
                            --exit-code 1 \
                            --quiet \
                            ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """,
                        returnStatus: true
                    )

                    if (scanResult != 0) {
                        error "🚨 Trivy found HIGH/MEDIUM OS vulnerabilities. Pipeline failed."
                    } else {
                        echo "✅ No HIGH or MEDIUM OS vulnerabilities found. Pipeline continues."
                    }
                }
            }
        }
        stage('Push Image to ECR') {
            steps {
                script{
                    withAWS(credentials: "${AWS_CREDS}" , region: "${REGION}") {
                    // Commands here have AWS auth
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                            docker push ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """
                    }
                }
                
            }
        }
    }
    

    // post build
    post { 
        always { 
            echo 'I will always say Hello again!'
        }
        success {
            echo "pipeline success"
        }
        failure {
            echo "pipeline failure"
        }
    }
}