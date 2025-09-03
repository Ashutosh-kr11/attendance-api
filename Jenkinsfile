@Library('sonar-tools') _

pipeline {
    agent any
    
    stages {
        stage('Scan Dependencies') {
            steps {
                script {
                    def scanResults = pythonDependencyScan(
                        reportName: 'python_security_scan.txt',
                        emailRecipients: 'ashutosh.devpro@gmail.com'
                    )
                    
                    echo "Scan completed with ${scanResults.vulnerabilitiesFound} vulnerabilities found"
                    
                    if (scanResults.vulnerabilitiesFound > 0) {
                        echo "WARNING: Vulnerabilities detected! See report: ${scanResults.reportUrl}"
                    }
                }
            }
        }
    }
}
