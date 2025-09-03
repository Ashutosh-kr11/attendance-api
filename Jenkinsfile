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
                    . ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install pip-audit safety
                '''
            }
        }
        
        stage('Scan Dependencies') {
            steps {
                script {
                    sh '''
                        echo "Python Dependency Scan Report - $(date '+%Y-%m-%d %H:%M:%S')" > ${SCAN_REPORT}
                        echo "=======================================" >> ${SCAN_REPORT}
                        
                        # Activate virtual environment
                        . ${VENV_DIR}/bin/activate
                        
                        # Check for requirements.txt
                        if [ -f requirements.txt ]; then
                            echo -e "\\n\\n## SCANNING REQUIREMENTS.TXT WITH PIP-AUDIT ##" >> ${SCAN_REPORT}
                            pip-audit -r requirements.txt --format json | python -m json.tool >> ${SCAN_REPORT} 2>&1 || true
                            
                            echo -e "\\n\\n## SCANNING REQUIREMENTS.TXT WITH SAFETY ##" >> ${SCAN_REPORT}
                            safety check -r requirements.txt --output text >> ${SCAN_REPORT} 2>&1 || true
                        else
                            echo -e "\\n\\nNo requirements.txt found" >> ${SCAN_REPORT}
                        fi
                        
                        # Check for pyproject.toml
                        if [ -f pyproject.toml ]; then
                            echo -e "\\n\\n## SCANNING PYPROJECT.TOML WITH PIP-AUDIT ##" >> ${SCAN_REPORT}
                            pip-audit -f pyproject.toml --format json | python -m json.tool >> ${SCAN_REPORT} 2>&1 || true
                            
                            # Extract dependencies from pyproject.toml for safety check
                            echo -e "\\n\\n## SCANNING PYPROJECT.TOML WITH SAFETY ##" >> ${SCAN_REPORT}
                            python -c "
import tomli
import sys
try:
    with open('pyproject.toml', 'rb') as f:
        data = tomli.load(f)
    deps = []
    # Get dependencies from different possible locations in pyproject.toml
    if 'dependencies' in data.get('project', {}):
        deps.extend(data['project']['dependencies'])
    if 'dependencies' in data.get('tool', {}).get('poetry', {}):
        deps.extend(data['tool']['poetry']['dependencies'].keys())
    
    # Write to temp requirements file
    if deps:
        with open('pyproject_deps.txt', 'w') as f:
            for dep in deps:
                if dep != 'python':  # Skip python version requirement
                    f.write(f'{dep}\\n')
        print('Dependencies extracted from pyproject.toml')
    else:
        print('No dependencies found in pyproject.toml')
except Exception as e:
    print(f'Error processing pyproject.toml: {e}')
            " >> ${SCAN_REPORT} 2>&1
                            
                            if [ -f pyproject_deps.txt ]; then
                                safety check -r pyproject_deps.txt --output text >> ${SCAN_REPORT} 2>&1 || true
                            fi
                        else
                            echo -e "\\n\\nNo pyproject.toml found" >> ${SCAN_REPORT}
                        fi
                        
                        # Add a comprehensive summary section
                        echo -e "\\n\\n## SUMMARY ##" >> ${SCAN_REPORT}
                        echo "Timestamp: $(date '+%Y-%m-%d %H:%M:%S')" >> ${SCAN_REPORT}
                        echo "Repository: $(git config --get remote.origin.url)" >> ${SCAN_REPORT}
                        
                        if grep -q "No vulnerable packages found" ${SCAN_REPORT} && ! grep -q "vulnerable package" ${SCAN_REPORT}; then
                            echo "RESULT: No vulnerabilities detected." >> ${SCAN_REPORT}
                        else
                            echo "RESULT: Vulnerabilities detected. Review report for details." >> ${SCAN_REPORT}
                            echo -e "\\n## VULNERABILITY HIGHLIGHTS ##" >> ${SCAN_REPORT}
                            grep -A 3 -B 1 "vulnerable package" ${SCAN_REPORT} || true
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: "${SCAN_REPORT}", allowEmptyArchive: true
            
            emailext (
                subject: "Python Dependency Scan Results - ${currentBuild.currentResult}",
                body: '''${FILE, path="${SCAN_REPORT}"}
                
                Jenkins Build: ${BUILD_URL}artifact/${SCAN_REPORT}
                ''',
                attachmentsPattern: "${SCAN_REPORT}",
                to: 'ashutosh.devpro@gmail.com',
                mimeType: 'text/plain'
            )
            
            sh 'rm -rf ${VENV_DIR} pyproject_deps.txt || true'
        }
    }
}
