pipeline {
    agent any
    
    // Removed the incorrect tools section
    
    environment {
        SCAN_REPORT = 'dependency_scan_report.txt'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                // Using system Python instead of tool installation
                sh 'python3 -m pip install --upgrade pip'
                sh 'pip3 install pip-audit'
            }
        }
        
        stage('Scan Dependencies') {
            steps {
                script {
                    // Create report file
                    sh "echo 'Python Dependency Scan Report - ${BUILD_TIMESTAMP}' > ${SCAN_REPORT}"
                    sh "echo '=======================================' >> ${SCAN_REPORT}"
                    
                    // Check for requirements.txt
                    sh '''
                        if [ -f requirements.txt ]; then
                            echo "\\n\\nScanning requirements.txt:" >> ${SCAN_REPORT}
                            pip-audit -r requirements.txt --format text >> ${SCAN_REPORT} || true
                        else
                            echo "\\n\\nNo requirements.txt found" >> ${SCAN_REPORT}
                        fi
                    '''
                    
                    // Check for pyproject.toml
                    sh '''
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
                        grep -c "No known vulnerabilities found" ${SCAN_REPORT} >> ${SCAN_REPORT} || echo "Vulnerabilities detected. Review report for details." >> ${SCAN_REPORT}
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
        }
    }
}
