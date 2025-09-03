pipeline {
    agent any
    
    environment {
        SCAN_REPORT = 'dependency_scan_report.txt'
        VENV_DIR = 'venv'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Virtual Environment') {
            steps {
                sh 'python3 -m venv ${VENV_DIR}'
                sh '''
                    # Activate virtual environment and install tools
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install pip-audit
                '''
            }
        }
        
        stage('Scan Dependencies') {
            steps {
                script {
                    // Create report file with current date using shell command
                    sh '''
                        echo "Python Dependency Scan Report - $(date '+%Y-%m-%d %H:%M:%S')" > ${SCAN_REPORT}
                        echo "=======================================" >> ${SCAN_REPORT}
                    '''
                    
                    // Check for requirements.txt
                    sh '''
                        # Activate virtual environment
                        . ${VENV_DIR}/bin/activate
                        
                        if [ -f requirements.txt ]; then
                            echo "\\n\\nScanning requirements.txt:" >> ${SCAN_REPORT}
                            pip-audit -r requirements.txt --format text >> ${SCAN_REPORT} || true
                        else
                            echo "\\n\\nNo requirements.txt found" >> ${SCAN_REPORT}
                        fi
                    '''
                    
                    // Check for pyproject.toml
                    sh '''
                        # Activate virtual environment
                        . ${VENV_DIR}/bin/activate
                        
                        if [ -f pyproject.toml ]; then
                            echo "\\n\\nScanning pyproject.toml:" >> ${SCAN_REPORT}
                            pip-audit -f pyproject.toml --format text >> ${SCAN_REPORT} || true
                        else
                            echo "\\n\\nNo pyproject.toml found" >> ${SCAN_REPORT}
                        fi
                    '''
                    
                    // Add summary
                    sh '''
                        echo "\\n\\nSummary:" >> ${SCAN_REPORT}
                        if grep -q "No known vulnerabilities found" ${SCAN_REPORT}; then
                            echo "No vulnerabilities detected." >> ${SCAN_REPORT}
                        else
                            echo "Vulnerabilities may be present. Review report for details." >> ${SCAN_REPORT}
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // Archive report as artifact
            archiveArtifacts artifacts: "${SCAN_REPORT}", allowEmptyArchive: true
            
            // Send email with report
            emailext (
                subject: "Python Dependency Scan Results - ${currentBuild.currentResult}",
                body: '''${FILE, path="${SCAN_REPORT}"}
                
                Jenkins Build: ${BUILD_URL}
                ''',
                attachmentsPattern: "${SCAN_REPORT}",
                to: 'ashutosh.devpro@gmail.com',
                mimeType: 'text/plain'
            )
            
            // Clean up virtual environment
            sh 'rm -rf ${VENV_DIR}'
        }
    }
}
