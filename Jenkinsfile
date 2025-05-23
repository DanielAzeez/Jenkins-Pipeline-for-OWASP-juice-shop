pipeline {
    agent any 
    
    environment {
        SEMGREP_APP_TOKEN = credentials('Semgrep-app')
        DEFECTDOJO_API_KEY = credentials('DEFECTDOJO_API_KEY')
        DEFECTDOJO_URL = 'https://demo.defectdojo.org'
        ENGAGEMENT_ID = '25'
    }
        
    stages {
        stage ('Checkout Code') {
            steps {
                checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/juice-shop/juice-shop.git']])
            }
        }

        stage ('Run GitLeaks Scan on Code') {
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "${REPORT_DIR}"
                /usr/local/bin/docker pull zricethezav/gitleaks:latest
                
                set +e
                /usr/local/bin/docker run --rm \
                    -v "${WORKSPACE}:/OWASP" \
                    zricethezav/gitleaks:latest detect --source=/OWASP \
                    --report-path=gitleaks-report.json \
                    --report-format=json 
                EXIT_CODE=$?
                set -e
                
                if [ "$EXIT_CODE" -ne 0 ]; then
                    echo "GitLeaks scan detected secrets. Please review gitleaks-report.json"
                else
                    echo "GitLeaks scan passed with no secrets detected."
                fi
                '''
            }
        }

        stage ('Static Code Analysis with Semgrep') {
            environment {
                SEMGREP_APP_TOKEN = credentials('Semgrep-app')
            }
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "$REPORT_DIR"
                /usr/local/bin/docker pull semgrep/semgrep:latest
                /usr/local/bin/docker run \
                    -e SEMGREP_APP_TOKEN="$SEMGREP_APP_TOKEN" \
                    --rm -v "${WORKSPACE}:/src" semgrep/semgrep semgrep ci \
                    --json --output /src/$REPORT_DIR/semgrep-report.json
                '''
            }
        }

        stage ('Code Analysis with NJSSCAN') {
            steps {
                sh '''
                REPORT_DIR="reports"
                mkdir -p "$REPORT_DIR"
                /usr/local/bin/docker pull opensecurity/njsscan
                /usr/local/bin/docker run --rm -v "${WORKSPACE}:/src" opensecurity/njsscan /src --json | tee "$REPORT_DIR/njsscan-output.json"
                '''
            }
        }

        stage ('Upload Scan results to DefectDojo') {
            steps {
                sh '''
                echo "Uploading Gitleaks report to DefectDojo..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                    -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "scan_type=Gitleaks Scan" \
                    -F "file=@${WORKSPACE}/reports/gitleaks-report.json" \
                    -F "active=true" -F "verified=false"

                echo "Uploading Semgrep report to DefectDojo..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                    -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "scan_type=Semgrep JSON Report" \
                    -F "file=@${WORKSPACE}/reports/semgrep-report.json" \
                    -F "active=true" -F "verified=false"
                
                echo "Uploading NJSSCAN report to DefectDojo..."
                curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
                    -H "Authorization: Token $DEFECTDOJO_API_KEY" \
                    -F "engagement=$ENGAGEMENT_ID" \
                    -F "scan_type=Node Security Platform Scan" \
                    -F "file=@${WORKSPACE}/reports/njsscan-output.json" \
                    -F "active=true" -F "verified=false"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*json', fingerprint: true, allowEmptyArchive: true
        }
    }
}