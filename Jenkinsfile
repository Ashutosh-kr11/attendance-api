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
                    pip install pip-audit safety tomli
                '''
            }
        }
        
        stage('Scan Dependencies') {
            steps {
                sh '''
                    # Activate virtual environment
                    . ${VENV_DIR}/bin/activate
                    
                    # Create report file
                    echo "Python Dependency Scan Report - $(date '+%Y-%m-%d %H:%M:%S')" > ${SCAN_REPORT}
                    echo "=======================================" >> ${SCAN_REPORT}
                    
                    # Check for requirements.txt
                    if [ -f requirements.txt ]; then
                        echo -e "\\n\\n## REQUIREMENTS.TXT FOUND ##" >> ${SCAN_REPORT}
                        echo -e "Content of requirements.txt:" >> ${SCAN_REPORT}
                        cat requirements.txt >> ${SCAN_REPORT}
                        
                        echo -e "\\n\\n## SCANNING WITH PIP-AUDIT ##" >> ${SCAN_REPORT}
                        pip-audit --requirement requirements.txt --format text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nPip-audit scan completed with issues" >> ${SCAN_REPORT}
                        
                        echo -e "\\n\\n## SCANNING WITH SAFETY ##" >> ${SCAN_REPORT}
                        safety check -r requirements.txt --output text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nSafety scan completed with issues" >> ${SCAN_REPORT}
                    else
                        echo -e "\\n\\nNo requirements.txt found" >> ${SCAN_REPORT}
                    fi
                    
                    # Check for pyproject.toml
                    if [ -f pyproject.toml ]; then
                        echo -e "\\n\\n## PYPROJECT.TOML FOUND ##" >> ${SCAN_REPORT}
                        echo -e "Content of pyproject.toml:" >> ${SCAN_REPORT}
                        cat pyproject.toml >> ${SCAN_REPORT}
                        
                        # Create a temporary file with dependencies extracted from pyproject.toml
                        python -c "
import tomli
import sys
try:
    with open('pyproject.toml', 'rb') as f:
        data = tomli.load(f)
    deps = []
    # Try to find dependencies in various locations
    if 'dependencies' in data.get('project', {}):
        deps.extend(data['project']['dependencies'])
        print('Found PEP 621 dependencies')
    elif 'dependencies' in data.get('tool', {}).get('poetry', {}):
        deps = list(data['tool']['poetry']['dependencies'].keys())
        deps = [d for d in deps if d != 'python']
        print('Found Poetry dependencies')
    elif 'dependencies' in data:
        deps = list(data['dependencies'].keys())
        print('Found direct dependencies')
    
    # Write to temp requirements file
    if deps:
        with open('pyproject_deps.txt', 'w') as f:
            for dep in deps:
                if not dep.startswith('python'):
                    f.write(f'{dep}\\n')
        print(f'Dependencies extracted: {deps}')
    else:
        print('No dependencies found in pyproject.toml')
except Exception as e:
    print(f'Error processing pyproject.toml: {e}')
" >> ${SCAN_REPORT} 2>&1
                        
                        # Scan the extracted dependencies
                        if [ -f pyproject_deps.txt ]; then
                            echo -e "\\n\\n## SCANNING EXTRACTED DEPENDENCIES WITH PIP-AUDIT ##" >> ${SCAN_REPORT}
                            pip-audit --requirement pyproject_deps.txt --format text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nPip-audit scan completed with issues" >> ${SCAN_REPORT}
                            
                            echo -e "\\n\\n## SCANNING EXTRACTED DEPENDENCIES WITH SAFETY ##" >> ${SCAN_REPORT}
                            safety check -r pyproject_deps.txt --output text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nSafety scan completed with issues" >> ${SCAN_REPORT}
                        fi
                    else
                        echo -e "\\n\\nNo pyproject.toml found" >> ${SCAN_REPORT}
                    fi
                    
                    # Scan installed packages in the environment
                    echo -e "\\n\\n## SCANNING ALL INSTALLED PACKAGES ##" >> ${SCAN_REPORT}
                    echo -e "Installed packages:" >> ${SCAN_REPORT}
                    pip freeze >> ${SCAN_REPORT}
                    
                    echo -e "\\n\\n## SCANNING INSTALLED PACKAGES WITH PIP-AUDIT ##" >> ${SCAN_REPORT}
                    pip-audit --format text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nPip-audit scan completed with issues" >> ${SCAN_REPORT}
                    
                    echo -e "\\n\\n## SCANNING INSTALLED PACKAGES WITH SAFETY ##" >> ${SCAN_REPORT}
                    safety check --output text >> ${SCAN_REPORT} 2>&1 || echo -e "\\nSafety scan completed with issues" >> ${SCAN_REPORT}
                    
                    # Add a comprehensive summary section
                    echo -e "\\n\\n## SUMMARY ##" >> ${SCAN_REPORT}
                    echo "Timestamp: $(date '+%Y-%m-%d %H:%M:%S')" >> ${SCAN_REPORT}
                    echo -e "Repository: $(git config --get remote.origin.url)" >> ${SCAN_REPORT}
                    
                    # Count vulnerabilities for summary
                    echo -e "\\n## VULNERABILITY FINDINGS ##" >> ${SCAN_REPORT}
                    VULN_COUNT=$(grep -i -c "vulnerability" ${SCAN_REPORT} || echo "0")
                    echo "Found approximately ${VULN_COUNT} references to vulnerabilities in the scan report." >> ${SCAN_REPORT}
                    
                    # Extract vulnerability highlights
                    if [ "${VULN_COUNT}" -gt "0" ]; then
                        echo -e "\\n## VULNERABILITY HIGHLIGHTS ##" >> ${SCAN_REPORT}
                        grep -i -A 2 -B 1 "vulnerability" ${SCAN_REPORT} | head -n 20 >> ${SCAN_REPORT} || true
                        echo -e "\\n(See full report for complete details)" >> ${SCAN_REPORT}
                    else
                        echo -e "\\nNo obvious vulnerabilities were detected in the scan." >> ${SCAN_REPORT}
                    fi
                '''
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
