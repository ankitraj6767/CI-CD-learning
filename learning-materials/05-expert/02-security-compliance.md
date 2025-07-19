# Module 18: Advanced Security & Compliance

## Learning Objectives

By the end of this module, you will be able to:

- Implement comprehensive security scanning in CI/CD pipelines
- Set up compliance frameworks (SOC 2, PCI DSS, HIPAA)
- Configure advanced secret management and rotation
- Implement security policies as code
- Set up vulnerability management and remediation
- Create security monitoring and incident response
- Implement zero-trust security principles
- Configure compliance reporting and auditing

## Prerequisites

- Completion of Module 17 (Infrastructure as Code)
- Understanding of security fundamentals
- Knowledge of compliance frameworks
- Experience with cloud security services
- Familiarity with container security

---

## 1. Security Scanning Integration

### Comprehensive Security Pipeline

```yaml
# .github/workflows/security-pipeline.yml
name: 'Security Pipeline'

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Daily security scan

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  secret-scanning:
    name: 'Secret Scanning'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history for secret scanning
    
    - name: Run TruffleHog
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD
        extra_args: --debug --only-verified
    
    - name: Run GitLeaks
      uses: gitleaks/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
    
    - name: Run Semgrep Secrets
      uses: returntocorp/semgrep-action@v1
      with:
        config: p/secrets
        generateSarif: "1"
    
    - name: Upload SARIF results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: semgrep.sarif

  sast-scanning:
    name: 'Static Application Security Testing'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Run CodeQL Analysis
      uses: github/codeql-action/init@v3
      with:
        languages: javascript, python, java
        queries: security-extended,security-and-quality
    
    - name: Autobuild
      uses: github/codeql-action/autobuild@v3
    
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
    
    - name: Run Semgrep SAST
      uses: returntocorp/semgrep-action@v1
      with:
        config: >
          p/security-audit
          p/owasp-top-ten
          p/cwe-top-25
        generateSarif: "1"
    
    - name: Run Bandit (Python)
      if: contains(github.event.repository.language, 'Python')
      run: |
        pip install bandit[toml]
        bandit -r . -f json -o bandit-report.json || true
    
    - name: Run ESLint Security (JavaScript)
      if: contains(github.event.repository.language, 'JavaScript')
      run: |
        npm install eslint-plugin-security
        npx eslint . --ext .js,.ts --format json --output-file eslint-security.json || true

  dependency-scanning:
    name: 'Dependency Vulnerability Scanning'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Run Snyk Test
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high --json-file-output=snyk-results.json
    
    - name: Run OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'security-pipeline'
        path: '.'
        format: 'JSON'
        out: 'dependency-check-report'
        args: >
          --enableRetired
          --enableExperimental
          --nvdApiKey ${{ secrets.NVD_API_KEY }}
    
    - name: Run Trivy Filesystem Scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-fs-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-fs-results.sarif'

  container-scanning:
    name: 'Container Security Scanning'
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Build Docker Image
      run: |
        docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
    
    - name: Run Trivy Container Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-container-results.sarif'
    
    - name: Run Grype Container Scan
      uses: anchore/scan-action@v3
      with:
        image: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
        format: sarif
        output-file: grype-results.sarif
    
    - name: Run Docker Scout
      uses: docker/scout-action@v1
      with:
        command: cves
        image: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
        format: sarif
        output: docker-scout-results.sarif
    
    - name: Upload container scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: |
          trivy-container-results.sarif
          grype-results.sarif
          docker-scout-results.sarif

  infrastructure-scanning:
    name: 'Infrastructure Security Scanning'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Run Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: terraform,dockerfile,kubernetes
        output_format: cli,json,sarif
        output_file_path: checkov-report.json,checkov-results.sarif
        check: CKV_AWS_20,CKV_AWS_57,CKV_AWS_61,CKV_K8S_*
        quiet: true
        soft_fail: false
    
    - name: Run TFSec
      uses: aquasecurity/tfsec-action@v1.0.3
      with:
        working_directory: infrastructure/
        format: sarif
        additional_args: --out tfsec-results.sarif
    
    - name: Run Kube-score
      if: contains(github.event.repository.topics, 'kubernetes')
      run: |
        wget https://github.com/zegl/kube-score/releases/latest/download/kube-score_linux_amd64.tar.gz
        tar xzf kube-score_linux_amd64.tar.gz
        find . -name '*.yaml' -o -name '*.yml' | grep -E 'k8s|kubernetes' | xargs ./kube-score score --output-format json > kube-score-results.json || true
    
    - name: Upload infrastructure scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: |
          checkov-results.sarif
          tfsec-results.sarif

  security-report:
    name: 'Generate Security Report'
    runs-on: ubuntu-latest
    needs: [secret-scanning, sast-scanning, dependency-scanning, container-scanning, infrastructure-scanning]
    if: always()
    
    steps:
    - name: Download all artifacts
      uses: actions/download-artifact@v4
    
    - name: Generate consolidated security report
      run: |
        python3 << 'EOF'
        import json
        import os
        from datetime import datetime
        
        report = {
            'scan_date': datetime.now().isoformat(),
            'repository': '${{ github.repository }}',
            'commit': '${{ github.sha }}',
            'branch': '${{ github.ref_name }}',
            'summary': {
                'total_vulnerabilities': 0,
                'critical': 0,
                'high': 0,
                'medium': 0,
                'low': 0
            },
            'scans': {}
        }
        
        # Process scan results
        scan_files = {
            'secrets': ['gitleaks.json', 'trufflehog.json'],
            'sast': ['semgrep.json', 'bandit-report.json', 'eslint-security.json'],
            'dependencies': ['snyk-results.json', 'dependency-check-report.json'],
            'containers': ['trivy-container-results.json', 'grype-results.json'],
            'infrastructure': ['checkov-report.json', 'tfsec-results.json']
        }
        
        for scan_type, files in scan_files.items():
            report['scans'][scan_type] = {'files_processed': 0, 'issues': []}
            
            for file in files:
                if os.path.exists(file):
                    try:
                        with open(file, 'r') as f:
                            data = json.load(f)
                            report['scans'][scan_type]['files_processed'] += 1
                            # Process based on tool format
                            # This is a simplified example
                    except Exception as e:
                        print(f"Error processing {file}: {e}")
        
        # Save report
        with open('security-report.json', 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"Security scan completed. Total vulnerabilities: {report['summary']['total_vulnerabilities']}")
        EOF
    
    - name: Upload security report
      uses: actions/upload-artifact@v4
      with:
        name: security-report
        path: security-report.json
    
    - name: Comment PR with security summary
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v7
      with:
        script: |
          const fs = require('fs');
          if (fs.existsSync('security-report.json')) {
            const report = JSON.parse(fs.readFileSync('security-report.json', 'utf8'));
            const summary = report.summary;
            
            const comment = `## 🔒 Security Scan Results
            
            **Total Vulnerabilities:** ${summary.total_vulnerabilities}
            - 🔴 Critical: ${summary.critical}
            - 🟠 High: ${summary.high}
            - 🟡 Medium: ${summary.medium}
            - 🟢 Low: ${summary.low}
            
            **Scans Performed:**
            ${Object.entries(report.scans).map(([type, data]) => 
              `- ${type}: ${data.files_processed} files processed`
            ).join('\n')}
            
            View detailed results in the [Security tab](${context.payload.repository.html_url}/security).`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
          }
```

---

## 2. Compliance Frameworks Implementation

### SOC 2 Compliance Pipeline

```yaml
# .github/workflows/soc2-compliance.yml
name: 'SOC 2 Compliance Checks'

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly on Monday
  workflow_dispatch:

jobs:
  access-controls:
    name: 'Access Control Validation'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Validate IAM Policies
      run: |
        python3 << 'EOF'
        import json
        import boto3
        from datetime import datetime, timedelta
        
        class SOC2AccessValidator:
            def __init__(self):
                self.iam = boto3.client('iam')
                self.results = []
            
            def check_mfa_enforcement(self):
                """Verify MFA is enforced for all users"""
                try:
                    users = self.iam.list_users()['Users']
                    
                    for user in users:
                        username = user['UserName']
                        mfa_devices = self.iam.list_mfa_devices(UserName=username)['MFADevices']
                        
                        if not mfa_devices:
                            self.results.append({
                                'control': 'CC6.1 - MFA Enforcement',
                                'status': 'FAIL',
                                'resource': username,
                                'message': 'User does not have MFA enabled'
                            })
                        else:
                            self.results.append({
                                'control': 'CC6.1 - MFA Enforcement',
                                'status': 'PASS',
                                'resource': username,
                                'message': 'MFA is properly configured'
                            })
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.1 - MFA Enforcement',
                        'status': 'ERROR',
                        'resource': 'all',
                        'message': f'Error checking MFA: {str(e)}'
                    })
            
            def check_password_policy(self):
                """Verify password policy meets SOC 2 requirements"""
                try:
                    policy = self.iam.get_account_password_policy()['PasswordPolicy']
                    
                    requirements = {
                        'MinimumPasswordLength': 12,
                        'RequireUppercaseCharacters': True,
                        'RequireLowercaseCharacters': True,
                        'RequireNumbers': True,
                        'RequireSymbols': True,
                        'MaxPasswordAge': 90,
                        'PasswordReusePrevention': 12
                    }
                    
                    for req, expected in requirements.items():
                        actual = policy.get(req)
                        if req == 'MinimumPasswordLength' and actual < expected:
                            self.results.append({
                                'control': 'CC6.1 - Password Policy',
                                'status': 'FAIL',
                                'resource': 'password_policy',
                                'message': f'Minimum password length is {actual}, should be {expected}'
                            })
                        elif req == 'MaxPasswordAge' and actual > expected:
                            self.results.append({
                                'control': 'CC6.1 - Password Policy',
                                'status': 'FAIL',
                                'resource': 'password_policy',
                                'message': f'Max password age is {actual}, should be {expected}'
                            })
                        elif isinstance(expected, bool) and actual != expected:
                            self.results.append({
                                'control': 'CC6.1 - Password Policy',
                                'status': 'FAIL',
                                'resource': 'password_policy',
                                'message': f'{req} is {actual}, should be {expected}'
                            })
                    
                    if not any(r['status'] == 'FAIL' for r in self.results if 'Password Policy' in r['control']):
                        self.results.append({
                            'control': 'CC6.1 - Password Policy',
                            'status': 'PASS',
                            'resource': 'password_policy',
                            'message': 'Password policy meets SOC 2 requirements'
                        })
                        
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.1 - Password Policy',
                        'status': 'ERROR',
                        'resource': 'password_policy',
                        'message': f'Error checking password policy: {str(e)}'
                    })
            
            def check_unused_access_keys(self):
                """Check for unused access keys (CC6.3)"""
                try:
                    users = self.iam.list_users()['Users']
                    cutoff_date = datetime.now() - timedelta(days=90)
                    
                    for user in users:
                        username = user['UserName']
                        access_keys = self.iam.list_access_keys(UserName=username)['AccessKeyMetadata']
                        
                        for key in access_keys:
                            key_id = key['AccessKeyId']
                            last_used = self.iam.get_access_key_last_used(AccessKeyId=key_id)
                            
                            last_used_date = last_used.get('AccessKeyLastUsed', {}).get('LastUsedDate')
                            
                            if not last_used_date or last_used_date < cutoff_date:
                                self.results.append({
                                    'control': 'CC6.3 - Access Key Management',
                                    'status': 'FAIL',
                                    'resource': f'{username}/{key_id}',
                                    'message': 'Access key unused for more than 90 days'
                                })
                            else:
                                self.results.append({
                                    'control': 'CC6.3 - Access Key Management',
                                    'status': 'PASS',
                                    'resource': f'{username}/{key_id}',
                                    'message': 'Access key is actively used'
                                })
                                
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.3 - Access Key Management',
                        'status': 'ERROR',
                        'resource': 'all',
                        'message': f'Error checking access keys: {str(e)}'
                    })
            
            def generate_report(self):
                """Generate SOC 2 compliance report"""
                report = {
                    'timestamp': datetime.now().isoformat(),
                    'framework': 'SOC 2 Type II',
                    'scope': 'Access Controls (CC6)',
                    'results': self.results,
                    'summary': {
                        'total_checks': len(self.results),
                        'passed': len([r for r in self.results if r['status'] == 'PASS']),
                        'failed': len([r for r in self.results if r['status'] == 'FAIL']),
                        'errors': len([r for r in self.results if r['status'] == 'ERROR'])
                    }
                }
                
                with open('soc2-access-report.json', 'w') as f:
                    json.dump(report, f, indent=2)
                
                return report
        
        # Run validation
        validator = SOC2AccessValidator()
        validator.check_mfa_enforcement()
        validator.check_password_policy()
        validator.check_unused_access_keys()
        
        report = validator.generate_report()
        
        print(f"SOC 2 Access Control Validation Complete")
        print(f"Total Checks: {report['summary']['total_checks']}")
        print(f"Passed: {report['summary']['passed']}")
        print(f"Failed: {report['summary']['failed']}")
        print(f"Errors: {report['summary']['errors']}")
        
        # Exit with error if any checks failed
        if report['summary']['failed'] > 0:
            exit(1)
        EOF
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ vars.AWS_REGION }}
    
    - name: Upload SOC 2 report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: soc2-access-report
        path: soc2-access-report.json

  data-protection:
    name: 'Data Protection Validation'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Validate Encryption Controls
      run: |
        python3 << 'EOF'
        import json
        import boto3
        
        class SOC2DataProtectionValidator:
            def __init__(self):
                self.s3 = boto3.client('s3')
                self.rds = boto3.client('rds')
                self.ec2 = boto3.client('ec2')
                self.results = []
            
            def check_s3_encryption(self):
                """Verify S3 bucket encryption (CC6.7)"""
                try:
                    buckets = self.s3.list_buckets()['Buckets']
                    
                    for bucket in buckets:
                        bucket_name = bucket['Name']
                        
                        try:
                            encryption = self.s3.get_bucket_encryption(Bucket=bucket_name)
                            self.results.append({
                                'control': 'CC6.7 - Data Encryption',
                                'status': 'PASS',
                                'resource': bucket_name,
                                'message': 'S3 bucket has encryption enabled'
                            })
                        except self.s3.exceptions.ClientError as e:
                            if e.response['Error']['Code'] == 'ServerSideEncryptionConfigurationNotFoundError':
                                self.results.append({
                                    'control': 'CC6.7 - Data Encryption',
                                    'status': 'FAIL',
                                    'resource': bucket_name,
                                    'message': 'S3 bucket does not have encryption enabled'
                                })
                            else:
                                raise e
                                
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.7 - Data Encryption',
                        'status': 'ERROR',
                        'resource': 'all_s3_buckets',
                        'message': f'Error checking S3 encryption: {str(e)}'
                    })
            
            def check_rds_encryption(self):
                """Verify RDS encryption (CC6.7)"""
                try:
                    instances = self.rds.describe_db_instances()['DBInstances']
                    
                    for instance in instances:
                        instance_id = instance['DBInstanceIdentifier']
                        encrypted = instance.get('StorageEncrypted', False)
                        
                        if encrypted:
                            self.results.append({
                                'control': 'CC6.7 - Data Encryption',
                                'status': 'PASS',
                                'resource': instance_id,
                                'message': 'RDS instance has encryption enabled'
                            })
                        else:
                            self.results.append({
                                'control': 'CC6.7 - Data Encryption',
                                'status': 'FAIL',
                                'resource': instance_id,
                                'message': 'RDS instance does not have encryption enabled'
                            })
                            
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.7 - Data Encryption',
                        'status': 'ERROR',
                        'resource': 'all_rds_instances',
                        'message': f'Error checking RDS encryption: {str(e)}'
                    })
            
            def check_ebs_encryption(self):
                """Verify EBS volume encryption (CC6.7)"""
                try:
                    volumes = self.ec2.describe_volumes()['Volumes']
                    
                    for volume in volumes:
                        volume_id = volume['VolumeId']
                        encrypted = volume.get('Encrypted', False)
                        
                        if encrypted:
                            self.results.append({
                                'control': 'CC6.7 - Data Encryption',
                                'status': 'PASS',
                                'resource': volume_id,
                                'message': 'EBS volume has encryption enabled'
                            })
                        else:
                            self.results.append({
                                'control': 'CC6.7 - Data Encryption',
                                'status': 'FAIL',
                                'resource': volume_id,
                                'message': 'EBS volume does not have encryption enabled'
                            })
                            
                except Exception as e:
                    self.results.append({
                        'control': 'CC6.7 - Data Encryption',
                        'status': 'ERROR',
                        'resource': 'all_ebs_volumes',
                        'message': f'Error checking EBS encryption: {str(e)}'
                    })
            
            def generate_report(self):
                """Generate data protection report"""
                report = {
                    'timestamp': datetime.now().isoformat(),
                    'framework': 'SOC 2 Type II',
                    'scope': 'Data Protection (CC6.7)',
                    'results': self.results,
                    'summary': {
                        'total_checks': len(self.results),
                        'passed': len([r for r in self.results if r['status'] == 'PASS']),
                        'failed': len([r for r in self.results if r['status'] == 'FAIL']),
                        'errors': len([r for r in self.results if r['status'] == 'ERROR'])
                    }
                }
                
                with open('soc2-data-protection-report.json', 'w') as f:
                    json.dump(report, f, indent=2)
                
                return report
        
        # Run validation
        from datetime import datetime
        validator = SOC2DataProtectionValidator()
        validator.check_s3_encryption()
        validator.check_rds_encryption()
        validator.check_ebs_encryption()
        
        report = validator.generate_report()
        
        print(f"SOC 2 Data Protection Validation Complete")
        print(f"Total Checks: {report['summary']['total_checks']}")
        print(f"Passed: {report['summary']['passed']}")
        print(f"Failed: {report['summary']['failed']}")
        print(f"Errors: {report['summary']['errors']}")
        
        # Exit with error if any checks failed
        if report['summary']['failed'] > 0:
            exit(1)
        EOF
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ vars.AWS_REGION }}
    
    - name: Upload data protection report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: soc2-data-protection-report
        path: soc2-data-protection-report.json

  monitoring-logging:
    name: 'Monitoring and Logging Validation'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Validate Logging Controls
      run: |
        python3 << 'EOF'
        import json
        import boto3
        from datetime import datetime
        
        class SOC2MonitoringValidator:
            def __init__(self):
                self.cloudtrail = boto3.client('cloudtrail')
                self.logs = boto3.client('logs')
                self.config = boto3.client('config')
                self.results = []
            
            def check_cloudtrail_enabled(self):
                """Verify CloudTrail is enabled (CC7.2)"""
                try:
                    trails = self.cloudtrail.describe_trails()['trailList']
                    
                    if not trails:
                        self.results.append({
                            'control': 'CC7.2 - System Monitoring',
                            'status': 'FAIL',
                            'resource': 'cloudtrail',
                            'message': 'No CloudTrail trails configured'
                        })
                        return
                    
                    for trail in trails:
                        trail_name = trail['Name']
                        status = self.cloudtrail.get_trail_status(Name=trail_name)
                        
                        if status['IsLogging']:
                            self.results.append({
                                'control': 'CC7.2 - System Monitoring',
                                'status': 'PASS',
                                'resource': trail_name,
                                'message': 'CloudTrail is actively logging'
                            })
                        else:
                            self.results.append({
                                'control': 'CC7.2 - System Monitoring',
                                'status': 'FAIL',
                                'resource': trail_name,
                                'message': 'CloudTrail is not logging'
                            })
                            
                except Exception as e:
                    self.results.append({
                        'control': 'CC7.2 - System Monitoring',
                        'status': 'ERROR',
                        'resource': 'cloudtrail',
                        'message': f'Error checking CloudTrail: {str(e)}'
                    })
            
            def check_config_enabled(self):
                """Verify AWS Config is enabled (CC7.2)"""
                try:
                    recorders = self.config.describe_configuration_recorders()['ConfigurationRecorders']
                    
                    if not recorders:
                        self.results.append({
                            'control': 'CC7.2 - System Monitoring',
                            'status': 'FAIL',
                            'resource': 'aws_config',
                            'message': 'AWS Config is not configured'
                        })
                        return
                    
                    for recorder in recorders:
                        recorder_name = recorder['name']
                        status = self.config.describe_configuration_recorder_status(
                            ConfigurationRecorderNames=[recorder_name]
                        )['ConfigurationRecordersStatus'][0]
                        
                        if status['recording']:
                            self.results.append({
                                'control': 'CC7.2 - System Monitoring',
                                'status': 'PASS',
                                'resource': recorder_name,
                                'message': 'AWS Config is actively recording'
                            })
                        else:
                            self.results.append({
                                'control': 'CC7.2 - System Monitoring',
                                'status': 'FAIL',
                                'resource': recorder_name,
                                'message': 'AWS Config is not recording'
                            })
                            
                except Exception as e:
                    self.results.append({
                        'control': 'CC7.2 - System Monitoring',
                        'status': 'ERROR',
                        'resource': 'aws_config',
                        'message': f'Error checking AWS Config: {str(e)}'
                    })
            
            def check_log_retention(self):
                """Verify log retention policies (CC7.3)"""
                try:
                    log_groups = self.logs.describe_log_groups()['logGroups']
                    
                    for log_group in log_groups:
                        group_name = log_group['logGroupName']
                        retention_days = log_group.get('retentionInDays')
                        
                        if not retention_days:
                            self.results.append({
                                'control': 'CC7.3 - Log Retention',
                                'status': 'FAIL',
                                'resource': group_name,
                                'message': 'Log group has no retention policy (logs never expire)'
                            })
                        elif retention_days < 365:  # SOC 2 typically requires 1 year retention
                            self.results.append({
                                'control': 'CC7.3 - Log Retention',
                                'status': 'FAIL',
                                'resource': group_name,
                                'message': f'Log retention is {retention_days} days, should be at least 365'
                            })
                        else:
                            self.results.append({
                                'control': 'CC7.3 - Log Retention',
                                'status': 'PASS',
                                'resource': group_name,
                                'message': f'Log retention is properly set to {retention_days} days'
                            })
                            
                except Exception as e:
                    self.results.append({
                        'control': 'CC7.3 - Log Retention',
                        'status': 'ERROR',
                        'resource': 'all_log_groups',
                        'message': f'Error checking log retention: {str(e)}'
                    })
            
            def generate_report(self):
                """Generate monitoring and logging report"""
                report = {
                    'timestamp': datetime.now().isoformat(),
                    'framework': 'SOC 2 Type II',
                    'scope': 'Monitoring and Logging (CC7)',
                    'results': self.results,
                    'summary': {
                        'total_checks': len(self.results),
                        'passed': len([r for r in self.results if r['status'] == 'PASS']),
                        'failed': len([r for r in self.results if r['status'] == 'FAIL']),
                        'errors': len([r for r in self.results if r['status'] == 'ERROR'])
                    }
                }
                
                with open('soc2-monitoring-report.json', 'w') as f:
                    json.dump(report, f, indent=2)
                
                return report
        
        # Run validation
        validator = SOC2MonitoringValidator()
        validator.check_cloudtrail_enabled()
        validator.check_config_enabled()
        validator.check_log_retention()
        
        report = validator.generate_report()
        
        print(f"SOC 2 Monitoring and Logging Validation Complete")
        print(f"Total Checks: {report['summary']['total_checks']}")
        print(f"Passed: {report['summary']['passed']}")
        print(f"Failed: {report['summary']['failed']}")
        print(f"Errors: {report['summary']['errors']}")
        
        # Exit with error if any checks failed
        if report['summary']['failed'] > 0:
            exit(1)
        EOF
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_DEFAULT_REGION: ${{ vars.AWS_REGION }}
    
    - name: Upload monitoring report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: soc2-monitoring-report
        path: soc2-monitoring-report.json

  generate-soc2-report:
    name: 'Generate SOC 2 Compliance Report'
    runs-on: ubuntu-latest
    needs: [access-controls, data-protection, monitoring-logging]
    if: always()
    
    steps:
    - name: Download all reports
      uses: actions/download-artifact@v4
    
    - name: Consolidate SOC 2 reports
      run: |
        python3 << 'EOF'
        import json
        import os
        from datetime import datetime
        
        # Consolidate all SOC 2 reports
        consolidated_report = {
            'timestamp': datetime.now().isoformat(),
            'framework': 'SOC 2 Type II',
            'assessment_period': '2024',
            'organization': '${{ github.repository_owner }}',
            'repository': '${{ github.repository }}',
            'controls': {},
            'overall_summary': {
                'total_checks': 0,
                'passed': 0,
                'failed': 0,
                'errors': 0,
                'compliance_percentage': 0
            }
        }
        
        # Process individual reports
        report_files = [
            'soc2-access-report/soc2-access-report.json',
            'soc2-data-protection-report/soc2-data-protection-report.json',
            'soc2-monitoring-report/soc2-monitoring-report.json'
        ]
        
        for report_file in report_files:
            if os.path.exists(report_file):
                with open(report_file, 'r') as f:
                    report_data = json.load(f)
                    
                    scope = report_data['scope']
                    consolidated_report['controls'][scope] = report_data
                    
                    # Update overall summary
                    summary = report_data['summary']
                    consolidated_report['overall_summary']['total_checks'] += summary['total_checks']
                    consolidated_report['overall_summary']['passed'] += summary['passed']
                    consolidated_report['overall_summary']['failed'] += summary['failed']
                    consolidated_report['overall_summary']['errors'] += summary['errors']
        
        # Calculate compliance percentage
        total = consolidated_report['overall_summary']['total_checks']
        passed = consolidated_report['overall_summary']['passed']
        if total > 0:
            consolidated_report['overall_summary']['compliance_percentage'] = (passed / total) * 100
        
        # Save consolidated report
        with open('soc2-compliance-report.json', 'w') as f:
            json.dump(consolidated_report, f, indent=2)
        
        # Generate executive summary
        exec_summary = f"""
        # SOC 2 Type II Compliance Assessment
        
        **Assessment Date:** {consolidated_report['timestamp']}
        **Organization:** {consolidated_report['organization']}
        **Repository:** {consolidated_report['repository']}
        
        ## Executive Summary
        
        This automated assessment evaluated the organization's compliance with SOC 2 Type II security controls.
        
        **Overall Compliance:** {consolidated_report['overall_summary']['compliance_percentage']:.1f}%
        
        - **Total Controls Tested:** {consolidated_report['overall_summary']['total_checks']}
        - **Controls Passed:** {consolidated_report['overall_summary']['passed']}
        - **Controls Failed:** {consolidated_report['overall_summary']['failed']}
        - **Errors Encountered:** {consolidated_report['overall_summary']['errors']}
        
        ## Control Areas Assessed
        
        """
        
        for scope, data in consolidated_report['controls'].items():
            summary = data['summary']
            compliance = (summary['passed'] / summary['total_checks']) * 100 if summary['total_checks'] > 0 else 0
            exec_summary += f"""
        ### {scope}
        - **Compliance:** {compliance:.1f}%
        - **Passed:** {summary['passed']}/{summary['total_checks']}
        - **Failed:** {summary['failed']}
        
        """
        
        exec_summary += """
        ## Recommendations
        
        1. **Address Failed Controls:** Review and remediate all failed control checks
        2. **Implement Continuous Monitoring:** Set up automated compliance monitoring
        3. **Regular Assessments:** Conduct quarterly compliance assessments
        4. **Documentation:** Maintain evidence of control implementation
        5. **Training:** Ensure staff are trained on SOC 2 requirements
        
        ## Next Steps
        
        1. Review detailed findings in the full compliance report
        2. Create remediation plan for failed controls
        3. Implement additional monitoring and alerting
        4. Schedule follow-up assessment
        
        ---
        
        *This report was generated automatically as part of the CI/CD pipeline.*
        """
        
        with open('soc2-executive-summary.md', 'w') as f:
            f.write(exec_summary)
        
        print(f"SOC 2 Compliance Assessment Complete")
        print(f"Overall Compliance: {consolidated_report['overall_summary']['compliance_percentage']:.1f}%")
        print(f"Total Checks: {consolidated_report['overall_summary']['total_checks']}")
        print(f"Passed: {consolidated_report['overall_summary']['passed']}")
        print(f"Failed: {consolidated_report['overall_summary']['failed']}")
        
        # Exit with error if compliance is below threshold
        if consolidated_report['overall_summary']['compliance_percentage'] < 95:
            print("❌ SOC 2 compliance below 95% threshold")
            exit(1)
        else:
            print("✅ SOC 2 compliance meets requirements")
        EOF
    
    - name: Upload consolidated SOC 2 report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: soc2-compliance-report
        path: |
          soc2-compliance-report.json
          soc2-executive-summary.md
```

---

## 3. Advanced Secret Management

### Secret Rotation Pipeline

```python
# scripts/secret-rotation.py
import boto3
import json
import os
import time
from datetime import datetime, timedelta
from typing import Dict, List, Any
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SecretRotationManager:
    def __init__(self, region: str = 'us-west-2'):
        self.secrets_client = boto3.client('secretsmanager', region_name=region)
        self.ssm_client = boto3.client('ssm', region_name=region)
        self.rds_client = boto3.client('rds', region_name=region)
        self.region = region
        
    def list_secrets_for_rotation(self, max_age_days: int = 90) -> List[Dict[str, Any]]:
        """List secrets that need rotation based on age"""
        secrets_to_rotate = []
        cutoff_date = datetime.now() - timedelta(days=max_age_days)
        
        try:
            paginator = self.secrets_client.get_paginator('list_secrets')
            
            for page in paginator.paginate():
                for secret in page['SecretList']:
                    secret_name = secret['Name']
                    last_changed = secret.get('LastChangedDate', secret.get('CreatedDate'))
                    
                    if last_changed and last_changed.replace(tzinfo=None) < cutoff_date:
                        # Check if rotation is enabled
                        try:
                            rotation_config = self.secrets_client.describe_secret(
                                SecretId=secret_name
                            )
                            
                            if rotation_config.get('RotationEnabled', False):
                                secrets_to_rotate.append({
                                    'name': secret_name,
                                    'arn': secret['ARN'],
                                    'last_changed': last_changed.isoformat(),
                                    'days_old': (datetime.now() - last_changed.replace(tzinfo=None)).days,
                                    'rotation_enabled': True
                                })
                            else:
                                logger.warning(f"Secret {secret_name} is old but rotation is not enabled")
                                secrets_to_rotate.append({
                                    'name': secret_name,
                                    'arn': secret['ARN'],
                                    'last_changed': last_changed.isoformat(),
                                    'days_old': (datetime.now() - last_changed.replace(tzinfo=None)).days,
                                    'rotation_enabled': False
                                })
                        except Exception as e:
                            logger.error(f"Error checking rotation config for {secret_name}: {e}")
                            
        except Exception as e:
            logger.error(f"Error listing secrets: {e}")
            
        return secrets_to_rotate
    
    def rotate_database_secret(self, secret_name: str) -> bool:
        """Rotate database credentials"""
        try:
            logger.info(f"Starting rotation for database secret: {secret_name}")
            
            # Get current secret value
            current_secret = self.secrets_client.get_secret_value(SecretId=secret_name)
            secret_data = json.loads(current_secret['SecretString'])
            
            # Validate required fields
            required_fields = ['username', 'password', 'engine', 'host', 'port', 'dbname']
            if not all(field in secret_data for field in required_fields):
                logger.error(f"Secret {secret_name} missing required fields")
                return False
            
            # Start rotation
            response = self.secrets_client.rotate_secret(
                SecretId=secret_name,
                ForceRotateSecrets=False
            )
            
            rotation_id = response['VersionId']
            logger.info(f"Rotation started with version ID: {rotation_id}")
            
            # Wait for rotation to complete
            max_wait_time = 300  # 5 minutes
            wait_interval = 10
            elapsed_time = 0
            
            while elapsed_time < max_wait_time:
                try:
                    rotation_status = self.secrets_client.describe_secret(
                        SecretId=secret_name
                    )
                    
                    if rotation_status.get('RotationEnabled', False):
                        # Check if rotation is complete
                        versions = rotation_status.get('VersionIdsToStages', {})
                        if rotation_id in versions and 'AWSCURRENT' in versions[rotation_id]:
                            logger.info(f"Rotation completed successfully for {secret_name}")
                            return True
                    
                    time.sleep(wait_interval)
                    elapsed_time += wait_interval
                    
                except Exception as e:
                    logger.error(f"Error checking rotation status: {e}")
                    break
            
            logger.error(f"Rotation timed out for {secret_name}")
            return False
            
        except Exception as e:
            logger.error(f"Error rotating secret {secret_name}: {e}")
            return False
    
    def rotate_api_key_secret(self, secret_name: str, api_provider: str) -> bool:
        """Rotate API key secrets"""
        try:
            logger.info(f"Starting rotation for API key secret: {secret_name}")
            
            # Get current secret
            current_secret = self.secrets_client.get_secret_value(SecretId=secret_name)
            secret_data = json.loads(current_secret['SecretString'])
            
            # Generate new API key based on provider
            new_api_key = self._generate_new_api_key(api_provider, secret_data)
            
            if not new_api_key:
                logger.error(f"Failed to generate new API key for {api_provider}")
                return False
            
            # Update secret with new key
            updated_secret_data = secret_data.copy()
            updated_secret_data.update(new_api_key)
            
            self.secrets_client.update_secret(
                SecretId=secret_name,
                SecretString=json.dumps(updated_secret_data)
            )
            
            logger.info(f"API key rotated successfully for {secret_name}")
            return True
            
        except Exception as e:
            logger.error(f"Error rotating API key secret {secret_name}: {e}")
            return False
    
    def _generate_new_api_key(self, provider: str, current_data: Dict[str, Any]) -> Dict[str, Any]:
        """Generate new API key based on provider"""
        # This is a simplified example - in practice, you'd integrate with each provider's API
        if provider == 'github':
            # GitHub API integration would go here
            return {'api_key': 'new_github_token_' + str(int(time.time()))}
        elif provider == 'slack':
            # Slack API integration would go here
            return {'api_key': 'new_slack_token_' + str(int(time.time()))}
        elif provider == 'datadog':
            # Datadog API integration would go here
            return {'api_key': 'new_datadog_key_' + str(int(time.time()))}
        else:
            logger.warning(f"Unknown API provider: {provider}")
            return {}
    
    def validate_secret_access(self, secret_name: str) -> bool:
        """Validate that the secret can be accessed and is valid"""
        try:
            # Test secret retrieval
            secret_value = self.secrets_client.get_secret_value(SecretId=secret_name)
            
            # Validate JSON format
            secret_data = json.loads(secret_value['SecretString'])
            
            # Basic validation - ensure it's not empty
            if not secret_data:
                logger.error(f"Secret {secret_name} is empty")
                return False
            
            logger.info(f"Secret {secret_name} validation passed")
            return True
            
        except Exception as e:
            logger.error(f"Secret validation failed for {secret_name}: {e}")
            return False
    
    def generate_rotation_report(self, rotation_results: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Generate rotation summary report"""
        total_secrets = len(rotation_results)
        successful_rotations = len([r for r in rotation_results if r['success']])
        failed_rotations = len([r for r in rotation_results if not r['success']])
        
        report = {
            'timestamp': datetime.now().isoformat(),
            'summary': {
                'total_secrets': total_secrets,
                'successful_rotations': successful_rotations,
                'failed_rotations': failed_rotations,
                'success_rate': (successful_rotations / total_secrets * 100) if total_secrets > 0 else 0
            },
            'results': rotation_results,
            'recommendations': []
        }
        
        # Add recommendations based on results
        if failed_rotations > 0:
            report['recommendations'].append(
                "Review and remediate failed secret rotations"
            )
        
        if report['summary']['success_rate'] < 95:
            report['recommendations'].append(
                "Investigate rotation failures and improve automation"
            )
        
        if total_secrets == 0:
            report['recommendations'].append(
                "No secrets found for rotation - verify secret management setup"
            )
        
        return report

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Secret Rotation Manager')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--max-age-days', type=int, default=90, help='Maximum age for secrets before rotation')
    parser.add_argument('--dry-run', action='store_true', help='Show what would be rotated without actually rotating')
    parser.add_argument('--secret-name', help='Rotate specific secret by name')
    
    args = parser.parse_args()
    
    manager = SecretRotationManager(region=args.region)
    rotation_results = []
    
    if args.secret_name:
        # Rotate specific secret
        logger.info(f"Rotating specific secret: {args.secret_name}")
        
        if args.dry_run:
            logger.info(f"DRY RUN: Would rotate secret {args.secret_name}")
            success = True
        else:
            # Determine secret type and rotate accordingly
            success = manager.rotate_database_secret(args.secret_name)
        
        rotation_results.append({
            'secret_name': args.secret_name,
            'success': success,
            'timestamp': datetime.now().isoformat(),
            'dry_run': args.dry_run
        })
    else:
        # Find and rotate old secrets
        secrets_to_rotate = manager.list_secrets_for_rotation(args.max_age_days)
        
        logger.info(f"Found {len(secrets_to_rotate)} secrets that need rotation")
        
        for secret in secrets_to_rotate:
            secret_name = secret['name']
            
            if args.dry_run:
                logger.info(f"DRY RUN: Would rotate secret {secret_name} (age: {secret['days_old']} days)")
                success = True
            else:
                if secret['rotation_enabled']:
                    success = manager.rotate_database_secret(secret_name)
                else:
                    logger.warning(f"Skipping {secret_name} - rotation not enabled")
                    success = False
            
            rotation_results.append({
                'secret_name': secret_name,
                'success': success,
                'days_old': secret['days_old'],
                'rotation_enabled': secret['rotation_enabled'],
                'timestamp': datetime.now().isoformat(),
                'dry_run': args.dry_run
            })
    
    # Generate and save report
    report = manager.generate_rotation_report(rotation_results)
    
    with open('secret-rotation-report.json', 'w') as f:
        json.dump(report, f, indent=2)
    
    # Print summary
    print(f"\n{'='*50}")
    print("SECRET ROTATION SUMMARY")
    print(f"{'='*50}")
    print(f"Total Secrets: {report['summary']['total_secrets']}")
    print(f"Successful Rotations: {report['summary']['successful_rotations']}")
    print(f"Failed Rotations: {report['summary']['failed_rotations']}")
    print(f"Success Rate: {report['summary']['success_rate']:.1f}%")
    
    if report['recommendations']:
        print("\nRecommendations:")
        for rec in report['recommendations']:
            print(f"- {rec}")
    
    # Exit with error if rotations failed
    if report['summary']['failed_rotations'] > 0 and not args.dry_run:
        exit(1)

if __name__ == "__main__":
    main()
```

---

## 6. Audit Logging and Monitoring

### Centralized Audit Logging

```python
# scripts/audit-logger.py
import boto3
import json
import logging
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
import hashlib
import hmac

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class AuditLogger:
    def __init__(self, region: str = 'us-west-2'):
        self.cloudtrail_client = boto3.client('cloudtrail', region_name=region)
        self.cloudwatch_client = boto3.client('cloudwatch', region_name=region)
        self.logs_client = boto3.client('logs', region_name=region)
        self.s3_client = boto3.client('s3', region_name=region)
        self.region = region
        
    def setup_audit_trail(self, trail_name: str, s3_bucket: str) -> Dict[str, Any]:
        """Setup CloudTrail for comprehensive audit logging"""
        logger.info(f"Setting up CloudTrail: {trail_name}")
        
        try:
            # Create CloudTrail
            response = self.cloudtrail_client.create_trail(
                Name=trail_name,
                S3BucketName=s3_bucket,
                S3KeyPrefix='cloudtrail-logs/',
                IncludeGlobalServiceEvents=True,
                IsMultiRegionTrail=True,
                EnableLogFileValidation=True,
                EventSelectors=[
                    {
                        'ReadWriteType': 'All',
                        'IncludeManagementEvents': True,
                        'DataResources': [
                            {
                                'Type': 'AWS::S3::Object',
                                'Values': ['arn:aws:s3:::*/*']
                            },
                            {
                                'Type': 'AWS::Lambda::Function',
                                'Values': ['arn:aws:lambda:*']
                            }
                        ]
                    }
                ],
                InsightSelectors=[
                    {
                        'InsightType': 'ApiCallRateInsight'
                    }
                ]
            )
            
            # Start logging
            self.cloudtrail_client.start_logging(Name=trail_name)
            
            logger.info(f"CloudTrail {trail_name} created and started")
            return response
            
        except Exception as e:
            logger.error(f"Error setting up CloudTrail: {e}")
            raise
    
    def create_log_groups(self) -> List[str]:
        """Create CloudWatch Log Groups for different audit categories"""
        log_groups = [
            '/aws/security/authentication',
            '/aws/security/authorization',
            '/aws/security/data-access',
            '/aws/security/configuration-changes',
            '/aws/security/network-activity',
            '/aws/compliance/soc2',
            '/aws/compliance/pci-dss',
            '/aws/compliance/hipaa'
        ]
        
        created_groups = []
        
        for log_group in log_groups:
            try:
                self.logs_client.create_log_group(
                    logGroupName=log_group,
                    retentionInDays=2555  # 7 years for compliance
                )
                
                # Add encryption
                self.logs_client.associate_kms_key(
                    logGroupName=log_group,
                    kmsKeyId='alias/aws/logs'
                )
                
                created_groups.append(log_group)
                logger.info(f"Created log group: {log_group}")
                
            except self.logs_client.exceptions.ResourceAlreadyExistsException:
                logger.info(f"Log group already exists: {log_group}")
                created_groups.append(log_group)
            except Exception as e:
                logger.error(f"Error creating log group {log_group}: {e}")
        
        return created_groups
    
    def setup_security_monitoring(self) -> List[Dict[str, Any]]:
        """Setup CloudWatch alarms for security events"""
        alarms = [
            {
                'name': 'RootAccountUsage',
                'description': 'Alert on root account usage',
                'metric_filter': {
                    'filterName': 'RootAccountUsageFilter',
                    'filterPattern': '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }',
                    'logGroupName': '/aws/cloudtrail'
                },
                'alarm': {
                    'AlarmName': 'RootAccountUsage',
                    'AlarmDescription': 'Root account usage detected',
                    'MetricName': 'RootAccountUsageCount',
                    'Namespace': 'Security/Authentication',
                    'Statistic': 'Sum',
                    'Period': 300,
                    'EvaluationPeriods': 1,
                    'Threshold': 1,
                    'ComparisonOperator': 'GreaterThanOrEqualToThreshold'
                }
            },
            {
                'name': 'UnauthorizedAPICalls',
                'description': 'Alert on unauthorized API calls',
                'metric_filter': {
                    'filterName': 'UnauthorizedAPICallsFilter',
                    'filterPattern': '{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }',
                    'logGroupName': '/aws/cloudtrail'
                },
                'alarm': {
                    'AlarmName': 'UnauthorizedAPICalls',
                    'AlarmDescription': 'Unauthorized API calls detected',
                    'MetricName': 'UnauthorizedAPICallsCount',
                    'Namespace': 'Security/Authorization',
                    'Statistic': 'Sum',
                    'Period': 300,
                    'EvaluationPeriods': 1,
                    'Threshold': 10,
                    'ComparisonOperator': 'GreaterThanThreshold'
                }
            },
            {
                'name': 'ConsoleSigninFailures',
                'description': 'Alert on console signin failures',
                'metric_filter': {
                    'filterName': 'ConsoleSigninFailuresFilter',
                    'filterPattern': '{ ($.eventName = ConsoleLogin) && ($.responseElements.ConsoleLogin = "Failure") }',
                    'logGroupName': '/aws/cloudtrail'
                },
                'alarm': {
                    'AlarmName': 'ConsoleSigninFailures',
                    'AlarmDescription': 'Console signin failures detected',
                    'MetricName': 'ConsoleSigninFailuresCount',
                    'Namespace': 'Security/Authentication',
                    'Statistic': 'Sum',
                    'Period': 300,
                    'EvaluationPeriods': 1,
                    'Threshold': 5,
                    'ComparisonOperator': 'GreaterThanThreshold'
                }
            },
            {
                'name': 'IAMPolicyChanges',
                'description': 'Alert on IAM policy changes',
                'metric_filter': {
                    'filterName': 'IAMPolicyChangesFilter',
                    'filterPattern': '{($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=AttachRolePolicy)||($.eventName=DetachRolePolicy)||($.eventName=AttachUserPolicy)||($.eventName=DetachUserPolicy)||($.eventName=AttachGroupPolicy)||($.eventName=DetachGroupPolicy)}',
                    'logGroupName': '/aws/cloudtrail'
                },
                'alarm': {
                    'AlarmName': 'IAMPolicyChanges',
                    'AlarmDescription': 'IAM policy changes detected',
                    'MetricName': 'IAMPolicyChangesCount',
                    'Namespace': 'Security/Configuration',
                    'Statistic': 'Sum',
                    'Period': 300,
                    'EvaluationPeriods': 1,
                    'Threshold': 1,
                    'ComparisonOperator': 'GreaterThanOrEqualToThreshold'
                }
            }
        ]
        
        created_alarms = []
        
        for alarm_config in alarms:
            try:
                # Create metric filter
                self.logs_client.put_metric_filter(
                    logGroupName=alarm_config['metric_filter']['logGroupName'],
                    filterName=alarm_config['metric_filter']['filterName'],
                    filterPattern=alarm_config['metric_filter']['filterPattern'],
                    metricTransformations=[
                        {
                            'metricName': alarm_config['alarm']['MetricName'],
                            'metricNamespace': alarm_config['alarm']['Namespace'],
                            'metricValue': '1'
                        }
                    ]
                )
                
                # Create alarm
                self.cloudwatch_client.put_metric_alarm(**alarm_config['alarm'])
                
                created_alarms.append(alarm_config)
                logger.info(f"Created security alarm: {alarm_config['name']}")
                
            except Exception as e:
                logger.error(f"Error creating alarm {alarm_config['name']}: {e}")
        
        return created_alarms
    
    def generate_audit_report(self, start_time: datetime, end_time: datetime) -> Dict[str, Any]:
        """Generate comprehensive audit report"""
        logger.info(f"Generating audit report from {start_time} to {end_time}")
        
        try:
            # Query CloudTrail events
            events = self.cloudtrail_client.lookup_events(
                LookupAttributes=[
                    {
                        'AttributeKey': 'EventTime',
                        'AttributeValue': start_time.isoformat()
                    }
                ],
                StartTime=start_time,
                EndTime=end_time,
                MaxItems=1000
            )
            
            # Analyze events
            event_analysis = self._analyze_events(events['Events'])
            
            # Get CloudWatch metrics
            metrics = self._get_security_metrics(start_time, end_time)
            
            # Generate report
            report = {
                'report_metadata': {
                    'generated_at': datetime.now().isoformat(),
                    'period_start': start_time.isoformat(),
                    'period_end': end_time.isoformat(),
                    'region': self.region
                },
                'executive_summary': {
                    'total_events': len(events['Events']),
                    'unique_users': len(event_analysis['unique_users']),
                    'unique_services': len(event_analysis['unique_services']),
                    'failed_events': event_analysis['failed_events_count'],
                    'high_risk_events': event_analysis['high_risk_events_count']
                },
                'event_analysis': event_analysis,
                'security_metrics': metrics,
                'compliance_status': self._assess_compliance_status(event_analysis),
                'recommendations': self._generate_audit_recommendations(event_analysis)
            }
            
            return report
            
        except Exception as e:
            logger.error(f"Error generating audit report: {e}")
            raise
    
    def _analyze_events(self, events: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Analyze CloudTrail events for security insights"""
        analysis = {
            'unique_users': set(),
            'unique_services': set(),
            'failed_events_count': 0,
            'high_risk_events_count': 0,
            'event_types': {},
            'source_ips': {},
            'user_agents': {},
            'high_risk_events': []
        }
        
        high_risk_event_names = {
            'CreateUser', 'DeleteUser', 'CreateRole', 'DeleteRole',
            'PutUserPolicy', 'DeleteUserPolicy', 'AttachUserPolicy', 'DetachUserPolicy',
            'CreateAccessKey', 'DeleteAccessKey', 'UpdateAccessKey',
            'ConsoleLogin', 'AssumeRole', 'GetSessionToken'
        }
        
        for event in events:
            # Extract user information
            user_identity = event.get('UserIdentity', {})
            if user_identity.get('userName'):
                analysis['unique_users'].add(user_identity['userName'])
            elif user_identity.get('arn'):
                analysis['unique_users'].add(user_identity['arn'])
            
            # Extract service information
            event_source = event.get('EventSource', 'unknown')
            analysis['unique_services'].add(event_source)
            
            # Count event types
            event_name = event.get('EventName', 'unknown')
            analysis['event_types'][event_name] = analysis['event_types'].get(event_name, 0) + 1
            
            # Check for failed events
            if event.get('ErrorCode') or event.get('ErrorMessage'):
                analysis['failed_events_count'] += 1
            
            # Check for high-risk events
            if event_name in high_risk_event_names:
                analysis['high_risk_events_count'] += 1
                analysis['high_risk_events'].append({
                    'event_name': event_name,
                    'event_time': event.get('EventTime', '').isoformat() if event.get('EventTime') else '',
                    'user': user_identity.get('userName') or user_identity.get('arn', 'unknown'),
                    'source_ip': event.get('SourceIPAddress', 'unknown'),
                    'user_agent': event.get('UserAgent', 'unknown')
                })
            
            # Track source IPs
            source_ip = event.get('SourceIPAddress', 'unknown')
            analysis['source_ips'][source_ip] = analysis['source_ips'].get(source_ip, 0) + 1
            
            # Track user agents
            user_agent = event.get('UserAgent', 'unknown')
            analysis['user_agents'][user_agent] = analysis['user_agents'].get(user_agent, 0) + 1
        
        # Convert sets to lists for JSON serialization
        analysis['unique_users'] = list(analysis['unique_users'])
        analysis['unique_services'] = list(analysis['unique_services'])
        
        return analysis
    
    def _get_security_metrics(self, start_time: datetime, end_time: datetime) -> Dict[str, Any]:
        """Get security-related CloudWatch metrics"""
        metrics = {}
        
        metric_queries = [
            {
                'name': 'RootAccountUsage',
                'namespace': 'Security/Authentication',
                'metric_name': 'RootAccountUsageCount'
            },
            {
                'name': 'UnauthorizedAPICalls',
                'namespace': 'Security/Authorization',
                'metric_name': 'UnauthorizedAPICallsCount'
            },
            {
                'name': 'ConsoleSigninFailures',
                'namespace': 'Security/Authentication',
                'metric_name': 'ConsoleSigninFailuresCount'
            },
            {
                'name': 'IAMPolicyChanges',
                'namespace': 'Security/Configuration',
                'metric_name': 'IAMPolicyChangesCount'
            }
        ]
        
        for query in metric_queries:
            try:
                response = self.cloudwatch_client.get_metric_statistics(
                    Namespace=query['namespace'],
                    MetricName=query['metric_name'],
                    StartTime=start_time,
                    EndTime=end_time,
                    Period=3600,  # 1 hour
                    Statistics=['Sum']
                )
                
                total_value = sum(point['Sum'] for point in response['Datapoints'])
                metrics[query['name']] = {
                    'total': total_value,
                    'datapoints': response['Datapoints']
                }
                
            except Exception as e:
                logger.warning(f"Could not get metric {query['name']}: {e}")
                metrics[query['name']] = {'total': 0, 'datapoints': []}
        
        return metrics
    
    def _assess_compliance_status(self, event_analysis: Dict[str, Any]) -> Dict[str, Any]:
        """Assess compliance status based on audit findings"""
        compliance_status = {
            'SOC2': {'compliant': True, 'issues': []},
            'PCI_DSS': {'compliant': True, 'issues': []},
            'HIPAA': {'compliant': True, 'issues': []},
            'ISO27001': {'compliant': True, 'issues': []}
        }
        
        # Check for compliance issues
        if event_analysis['failed_events_count'] > 100:
            issue = f"High number of failed events: {event_analysis['failed_events_count']}"
            for framework in compliance_status:
                compliance_status[framework]['compliant'] = False
                compliance_status[framework]['issues'].append(issue)
        
        if event_analysis['high_risk_events_count'] > 50:
            issue = f"High number of high-risk events: {event_analysis['high_risk_events_count']}"
            for framework in compliance_status:
                compliance_status[framework]['compliant'] = False
                compliance_status[framework]['issues'].append(issue)
        
        # Check for suspicious IP addresses
        suspicious_ips = [
            ip for ip, count in event_analysis['source_ips'].items()
            if count > 1000 and not ip.startswith(('10.', '172.', '192.168.'))
        ]
        
        if suspicious_ips:
            issue = f"Suspicious IP addresses detected: {', '.join(suspicious_ips[:5])}"
            compliance_status['SOC2']['compliant'] = False
            compliance_status['SOC2']['issues'].append(issue)
        
        return compliance_status
    
    def _generate_audit_recommendations(self, event_analysis: Dict[str, Any]) -> List[str]:
        """Generate recommendations based on audit findings"""
        recommendations = []
        
        if event_analysis['failed_events_count'] > 50:
            recommendations.append(
                "Investigate and reduce the number of failed API calls"
            )
        
        if event_analysis['high_risk_events_count'] > 20:
            recommendations.append(
                "Review high-risk events and implement additional controls"
            )
        
        if len(event_analysis['unique_users']) > 100:
            recommendations.append(
                "Consider implementing user access reviews and cleanup"
            )
        
        # Check for root account usage
        root_events = [e for e in event_analysis['high_risk_events'] if 'root' in str(e).lower()]
        if root_events:
            recommendations.append(
                "Eliminate root account usage and implement IAM users/roles"
            )
        
        recommendations.extend([
            "Implement automated security monitoring and alerting",
            "Regular review of CloudTrail logs for anomalies",
            "Establish incident response procedures for security events",
            "Consider implementing AWS Config for configuration compliance"
        ])
        
        return recommendations

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Audit Logger and Monitor')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--action', choices=['setup', 'report'], required=True, help='Action to perform')
    parser.add_argument('--trail-name', default='security-audit-trail', help='CloudTrail name')
    parser.add_argument('--s3-bucket', help='S3 bucket for CloudTrail logs')
    parser.add_argument('--days', type=int, default=7, help='Number of days for report')
    parser.add_argument('--output-file', default='audit-report.json', help='Output file for report')
    
    args = parser.parse_args()
    
    audit_logger = AuditLogger(region=args.region)
    
    if args.action == 'setup':
        if not args.s3_bucket:
            print("Error: --s3-bucket is required for setup")
            exit(1)
        
        # Setup audit infrastructure
        print("Setting up audit infrastructure...")
        
        # Create log groups
        log_groups = audit_logger.create_log_groups()
        print(f"Created {len(log_groups)} log groups")
        
        # Setup CloudTrail
        trail_response = audit_logger.setup_audit_trail(args.trail_name, args.s3_bucket)
        print(f"CloudTrail setup complete: {trail_response['TrailARN']}")
        
        # Setup security monitoring
        alarms = audit_logger.setup_security_monitoring()
        print(f"Created {len(alarms)} security alarms")
        
        print("Audit infrastructure setup complete!")
    
    elif args.action == 'report':
        # Generate audit report
        end_time = datetime.now()
        start_time = end_time - timedelta(days=args.days)
        
        print(f"Generating audit report for {args.days} days...")
        report = audit_logger.generate_audit_report(start_time, end_time)
        
        # Save report
        with open(args.output_file, 'w') as f:
            json.dump(report, f, indent=2)
        
        # Print summary
        print(f"\n{'='*60}")
        print("AUDIT REPORT SUMMARY")
        print(f"{'='*60}")
        print(f"Period: {start_time.date()} to {end_time.date()}")
        print(f"Total Events: {report['executive_summary']['total_events']}")
        print(f"Unique Users: {report['executive_summary']['unique_users']}")
        print(f"Failed Events: {report['executive_summary']['failed_events']}")
        print(f"High-Risk Events: {report['executive_summary']['high_risk_events']}")
        
        print(f"\nCompliance Status:")
        for framework, status in report['compliance_status'].items():
            status_text = "✅ COMPLIANT" if status['compliant'] else "❌ NON-COMPLIANT"
            print(f"  {framework}: {status_text}")
            if status['issues']:
                for issue in status['issues'][:3]:
                    print(f"    - {issue}")
        
        print(f"\nTop Recommendations:")
        for i, rec in enumerate(report['recommendations'][:5], 1):
            print(f"{i}. {rec}")
        
        print(f"\nDetailed report saved to: {args.output_file}")

if __name__ == "__main__":
    main()
```

### Compliance Reporting Dashboard

```python
# scripts/compliance-dashboard.py
import boto3
import json
from datetime import datetime, timedelta
from typing import Dict, List, Any
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ComplianceDashboard:
    def __init__(self, region: str = 'us-west-2'):
        self.config_client = boto3.client('config', region_name=region)
        self.cloudwatch_client = boto3.client('cloudwatch', region_name=region)
        self.securityhub_client = boto3.client('securityhub', region_name=region)
        self.region = region
    
    def generate_compliance_dashboard(self) -> Dict[str, Any]:
        """Generate comprehensive compliance dashboard"""
        logger.info("Generating compliance dashboard...")
        
        dashboard = {
            'generated_at': datetime.now().isoformat(),
            'region': self.region,
            'compliance_frameworks': {
                'SOC2': self._get_soc2_compliance(),
                'PCI_DSS': self._get_pci_dss_compliance(),
                'HIPAA': self._get_hipaa_compliance(),
                'ISO27001': self._get_iso27001_compliance()
            },
            'security_controls': self._get_security_controls_status(),
            'findings_summary': self._get_findings_summary(),
            'trends': self._get_compliance_trends(),
            'action_items': self._get_action_items()
        }
        
        return dashboard
    
    def _get_soc2_compliance(self) -> Dict[str, Any]:
        """Get SOC 2 compliance status"""
        soc2_controls = {
            'CC1.1': 'COSO Principles and Organization',
            'CC2.1': 'Communication and Information',
            'CC3.1': 'Risk Assessment',
            'CC4.1': 'Monitoring Activities',
            'CC5.1': 'Control Activities',
            'CC6.1': 'Logical and Physical Access Controls',
            'CC6.2': 'System Access Controls',
            'CC6.3': 'Data Access Controls',
            'CC7.1': 'System Operations',
            'CC8.1': 'Change Management'
        }
        
        compliance_status = {
            'overall_score': 85,
            'compliant_controls': 8,
            'total_controls': 10,
            'controls': {},
            'findings': []
        }
        
        # Simulate control assessments
        for control_id, description in soc2_controls.items():
            # In practice, this would query actual compliance data
            compliance_status['controls'][control_id] = {
                'description': description,
                'status': 'COMPLIANT' if control_id not in ['CC6.2', 'CC8.1'] else 'NON_COMPLIANT',
                'last_assessed': (datetime.now() - timedelta(days=30)).isoformat(),
                'evidence_count': 5 if control_id not in ['CC6.2', 'CC8.1'] else 2
            }
        
        # Add findings for non-compliant controls
        compliance_status['findings'] = [
            {
                'control_id': 'CC6.2',
                'severity': 'HIGH',
                'description': 'Insufficient access controls for privileged accounts',
                'remediation': 'Implement MFA for all privileged accounts'
            },
            {
                'control_id': 'CC8.1',
                'severity': 'MEDIUM',
                'description': 'Change management process not fully documented',
                'remediation': 'Document and implement formal change management procedures'
            }
        ]
        
        return compliance_status
    
    def _get_pci_dss_compliance(self) -> Dict[str, Any]:
        """Get PCI DSS compliance status"""
        pci_requirements = {
            'Req1': 'Install and maintain a firewall configuration',
            'Req2': 'Do not use vendor-supplied defaults',
            'Req3': 'Protect stored cardholder data',
            'Req4': 'Encrypt transmission of cardholder data',
            'Req5': 'Protect all systems against malware',
            'Req6': 'Develop and maintain secure systems',
            'Req7': 'Restrict access to cardholder data',
            'Req8': 'Identify and authenticate access',
            'Req9': 'Restrict physical access to cardholder data',
            'Req10': 'Track and monitor all access',
            'Req11': 'Regularly test security systems',
            'Req12': 'Maintain an information security policy'
        }
        
        compliance_status = {
            'overall_score': 78,
            'compliant_requirements': 9,
            'total_requirements': 12,
            'requirements': {},
            'findings': []
        }
        
        non_compliant = ['Req3', 'Req7', 'Req11']
        
        for req_id, description in pci_requirements.items():
            compliance_status['requirements'][req_id] = {
                'description': description,
                'status': 'NON_COMPLIANT' if req_id in non_compliant else 'COMPLIANT',
                'last_assessed': (datetime.now() - timedelta(days=45)).isoformat(),
                'sub_requirements_compliant': 8 if req_id not in non_compliant else 5,
                'sub_requirements_total': 10
            }
        
        compliance_status['findings'] = [
            {
                'requirement_id': 'Req3',
                'severity': 'CRITICAL',
                'description': 'Cardholder data not properly encrypted at rest',
                'remediation': 'Implement encryption for all cardholder data storage'
            },
            {
                'requirement_id': 'Req7',
                'severity': 'HIGH',
                'description': 'Access to cardholder data not restricted by business need',
                'remediation': 'Implement role-based access controls'
            },
            {
                'requirement_id': 'Req11',
                'severity': 'MEDIUM',
                'description': 'Vulnerability scanning not performed regularly',
                'remediation': 'Implement automated quarterly vulnerability scans'
            }
        ]
        
        return compliance_status
    
    def _get_hipaa_compliance(self) -> Dict[str, Any]:
        """Get HIPAA compliance status"""
        hipaa_safeguards = {
            '164.308': 'Administrative Safeguards',
            '164.310': 'Physical Safeguards',
            '164.312': 'Technical Safeguards',
            '164.314': 'Organizational Requirements',
            '164.316': 'Policies and Procedures'
        }
        
        compliance_status = {
            'overall_score': 92,
            'compliant_safeguards': 4,
            'total_safeguards': 5,
            'safeguards': {},
            'findings': []
        }
        
        for safeguard_id, description in hipaa_safeguards.items():
            compliance_status['safeguards'][safeguard_id] = {
                'description': description,
                'status': 'NON_COMPLIANT' if safeguard_id == '164.312' else 'COMPLIANT',
                'last_assessed': (datetime.now() - timedelta(days=60)).isoformat(),
                'implementation_specifications': {
                    'required': 8 if safeguard_id != '164.312' else 6,
                    'addressable': 4 if safeguard_id != '164.312' else 2,
                    'implemented': 12 if safeguard_id != '164.312' else 8
                }
            }
        
        compliance_status['findings'] = [
            {
                'safeguard_id': '164.312',
                'severity': 'HIGH',
                'description': 'Technical safeguards for PHI access not fully implemented',
                'remediation': 'Implement automatic logoff and encryption controls'
            }
        ]
        
        return compliance_status
    
    def _get_iso27001_compliance(self) -> Dict[str, Any]:
        """Get ISO 27001 compliance status"""
        iso_controls = {
            'A.5': 'Information Security Policies',
            'A.6': 'Organization of Information Security',
            'A.7': 'Human Resource Security',
            'A.8': 'Asset Management',
            'A.9': 'Access Control',
            'A.10': 'Cryptography',
            'A.11': 'Physical and Environmental Security',
            'A.12': 'Operations Security',
            'A.13': 'Communications Security',
            'A.14': 'System Acquisition, Development and Maintenance',
            'A.15': 'Supplier Relationships',
            'A.16': 'Information Security Incident Management',
            'A.17': 'Information Security Aspects of Business Continuity',
            'A.18': 'Compliance'
        }
        
        compliance_status = {
            'overall_score': 88,
            'compliant_controls': 12,
            'total_controls': 14,
            'controls': {},
            'findings': []
        }
        
        non_compliant = ['A.9', 'A.16']
        
        for control_id, description in iso_controls.items():
            compliance_status['controls'][control_id] = {
                'description': description,
                'status': 'NON_COMPLIANT' if control_id in non_compliant else 'COMPLIANT',
                'last_assessed': (datetime.now() - timedelta(days=90)).isoformat(),
                'maturity_level': 2 if control_id in non_compliant else 4,
                'target_maturity_level': 4
            }
        
        compliance_status['findings'] = [
            {
                'control_id': 'A.9',
                'severity': 'HIGH',
                'description': 'Access control management needs improvement',
                'remediation': 'Implement comprehensive access review process'
            },
            {
                'control_id': 'A.16',
                'severity': 'MEDIUM',
                'description': 'Incident management procedures not fully documented',
                'remediation': 'Document and test incident response procedures'
            }
        ]
        
        return compliance_status
    
    def _get_security_controls_status(self) -> Dict[str, Any]:
        """Get overall security controls status"""
        return {
            'encryption': {
                'data_at_rest': {'status': 'COMPLIANT', 'coverage': 95},
                'data_in_transit': {'status': 'COMPLIANT', 'coverage': 98},
                'key_management': {'status': 'COMPLIANT', 'coverage': 90}
            },
            'access_control': {
                'mfa_enabled': {'status': 'PARTIAL', 'coverage': 75},
                'rbac_implemented': {'status': 'COMPLIANT', 'coverage': 85},
                'privileged_access': {'status': 'NON_COMPLIANT', 'coverage': 60}
            },
            'monitoring': {
                'logging_enabled': {'status': 'COMPLIANT', 'coverage': 92},
                'alerting_configured': {'status': 'COMPLIANT', 'coverage': 88},
                'incident_response': {'status': 'PARTIAL', 'coverage': 70}
            },
            'vulnerability_management': {
                'scanning_enabled': {'status': 'COMPLIANT', 'coverage': 90},
                'patch_management': {'status': 'PARTIAL', 'coverage': 80},
                'remediation_tracking': {'status': 'COMPLIANT', 'coverage': 85}
            }
        }
    
    def _get_findings_summary(self) -> Dict[str, Any]:
        """Get summary of security findings"""
        return {
            'total_findings': 156,
            'by_severity': {
                'CRITICAL': 3,
                'HIGH': 12,
                'MEDIUM': 45,
                'LOW': 96
            },
            'by_category': {
                'Access Control': 34,
                'Encryption': 8,
                'Network Security': 23,
                'Logging & Monitoring': 15,
                'Vulnerability Management': 28,
                'Configuration': 48
            },
            'remediation_status': {
                'resolved': 89,
                'in_progress': 34,
                'planned': 23,
                'new': 10
            }
        }
    
    def _get_compliance_trends(self) -> Dict[str, Any]:
        """Get compliance trends over time"""
        return {
            'monthly_scores': {
                'SOC2': [78, 82, 85, 85],
                'PCI_DSS': [72, 75, 78, 78],
                'HIPAA': [88, 90, 92, 92],
                'ISO27001': [82, 85, 88, 88]
            },
            'findings_trend': {
                'total': [180, 165, 156, 156],
                'critical': [8, 5, 3, 3],
                'high': [25, 18, 12, 12]
            },
            'remediation_velocity': {
                'average_days_to_resolve': [15, 12, 10, 8],
                'sla_compliance': [85, 88, 92, 95]
            }
        }
    
    def _get_action_items(self) -> List[Dict[str, Any]]:
        """Get prioritized action items"""
        return [
            {
                'priority': 'CRITICAL',
                'title': 'Implement encryption for cardholder data',
                'framework': 'PCI_DSS',
                'due_date': (datetime.now() + timedelta(days=30)).isoformat(),
                'owner': 'Security Team',
                'estimated_effort': '2 weeks'
            },
            {
                'priority': 'HIGH',
                'title': 'Enable MFA for all privileged accounts',
                'framework': 'SOC2',
                'due_date': (datetime.now() + timedelta(days=45)).isoformat(),
                'owner': 'IT Team',
                'estimated_effort': '1 week'
            },
            {
                'priority': 'HIGH',
                'title': 'Implement automated vulnerability scanning',
                'framework': 'PCI_DSS',
                'due_date': (datetime.now() + timedelta(days=60)).isoformat(),
                'owner': 'DevOps Team',
                'estimated_effort': '3 weeks'
            },
            {
                'priority': 'MEDIUM',
                'title': 'Document incident response procedures',
                'framework': 'ISO27001',
                'due_date': (datetime.now() + timedelta(days=90)).isoformat(),
                'owner': 'Security Team',
                'estimated_effort': '2 weeks'
            }
        ]

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Compliance Dashboard Generator')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--output-file', default='compliance-dashboard.json', help='Output file')
    parser.add_argument('--format', choices=['json', 'html'], default='json', help='Output format')
    
    args = parser.parse_args()
    
    dashboard = ComplianceDashboard(region=args.region)
    report = dashboard.generate_compliance_dashboard()
    
    if args.format == 'json':
        with open(args.output_file, 'w') as f:
            json.dump(report, f, indent=2)
    elif args.format == 'html':
        # Generate HTML report (simplified)
        html_content = f"""
        <!DOCTYPE html>
        <html>
        <head>
            <title>Compliance Dashboard</title>
            <style>
                body {{ font-family: Arial, sans-serif; margin: 20px; }}
                .framework {{ margin: 20px 0; padding: 15px; border: 1px solid #ddd; }}
                .compliant {{ background-color: #d4edda; }}
                .non-compliant {{ background-color: #f8d7da; }}
                .partial {{ background-color: #fff3cd; }}
            </style>
        </head>
        <body>
            <h1>Compliance Dashboard</h1>
            <p>Generated: {report['generated_at']}</p>
            
            <h2>Compliance Frameworks</h2>
        """
        
        for framework, data in report['compliance_frameworks'].items():
            status_class = 'compliant' if data['overall_score'] >= 90 else 'partial' if data['overall_score'] >= 70 else 'non-compliant'
            html_content += f"""
            <div class="framework {status_class}">
                <h3>{framework}</h3>
                <p>Overall Score: {data['overall_score']}%</p>
                <p>Findings: {len(data.get('findings', []))}</p>
            </div>
            """
        
        html_content += """
        </body>
        </html>
        """
        
        with open(args.output_file.replace('.json', '.html'), 'w') as f:
            f.write(html_content)
    
    # Print summary
    print(f"\n{'='*60}")
    print("COMPLIANCE DASHBOARD SUMMARY")
    print(f"{'='*60}")
    
    for framework, data in report['compliance_frameworks'].items():
        score = data['overall_score']
        status = "✅" if score >= 90 else "⚠️" if score >= 70 else "❌"
        print(f"{framework}: {score}% {status}")
    
    print(f"\nTotal Findings: {report['findings_summary']['total_findings']}")
    print(f"Critical: {report['findings_summary']['by_severity']['CRITICAL']}")
    print(f"High: {report['findings_summary']['by_severity']['HIGH']}")
    
    print(f"\nTop Action Items:")
    for item in report['action_items'][:3]:
        print(f"- {item['title']} ({item['priority']})")
    
    print(f"\nReport saved to: {args.output_file}")

if __name__ == "__main__":
    main()
```

---

## 7. Hands-on Exercises

### Exercise 1: Security Scanning Pipeline

**Objective**: Implement a comprehensive security scanning pipeline

**Steps**:
1. Set up the security scanning workflow
2. Configure secret scanning
3. Implement SAST and DAST scanning
4. Add container vulnerability scanning
5. Create security reports

**Deliverables**:
- Working GitHub Actions workflow
- Security scan reports
- Remediation plan for findings

### Exercise 2: Compliance Automation

**Objective**: Automate SOC 2 compliance checks

**Steps**:
1. Implement compliance validation scripts
2. Set up automated compliance reporting
3. Create compliance dashboard
4. Configure compliance monitoring
5. Document compliance procedures

**Deliverables**:
- Automated compliance checks
- Compliance dashboard
- Documentation of controls

### Exercise 3: Secret Management

**Objective**: Implement comprehensive secret management

**Steps**:
1. Set up AWS Secrets Manager
2. Implement secret rotation
3. Configure secret scanning
4. Create secret validation pipeline
5. Document secret management procedures

**Deliverables**:
- Secret management infrastructure
- Rotation automation
- Secret scanning pipeline

### Exercise 4: Policy as Code

**Objective**: Implement security policies using OPA

**Steps**:
1. Write OPA policies for Kubernetes
2. Create Terraform security policies
3. Set up policy enforcement pipeline
4. Configure Gatekeeper for Kubernetes
5. Test policy violations

**Deliverables**:
- OPA policy files
- Policy enforcement pipeline
- Policy violation reports

---

## 8. Troubleshooting Guide

### Common Security Scanning Issues

**Issue**: False positives in security scans
**Solution**: 
- Configure scan exclusions
- Tune scanning rules
- Implement baseline scanning
- Use multiple scanning tools for validation

**Issue**: Slow security scanning
**Solution**:
- Implement incremental scanning
- Use caching for scan results
- Parallelize scanning processes
- Optimize scan configurations

### Compliance Automation Issues

**Issue**: Compliance checks failing
**Solution**:
- Review compliance requirements
- Update validation scripts
- Check AWS service configurations
- Verify IAM permissions

**Issue**: Incomplete compliance reports
**Solution**:
- Ensure all required data sources are accessible
- Check API rate limits
- Verify service availability
- Implement retry mechanisms

### Secret Management Issues

**Issue**: Secret rotation failures
**Solution**:
- Check IAM permissions
- Verify secret dependencies
- Test rotation procedures
- Implement rollback mechanisms

**Issue**: Secret scanning false positives
**Solution**:
- Configure scanning patterns
- Implement allowlists
- Use entropy-based detection
- Regular pattern updates

---

## Summary

Module 18 covered advanced security and compliance automation, including:

- **Security Scanning Integration**: Comprehensive scanning pipelines with multiple tools
- **Compliance Frameworks**: Automated SOC 2, PCI DSS, HIPAA, and ISO 27001 compliance
- **Secret Management**: Advanced secret rotation, scanning, and validation
- **Security Policies as Code**: OPA integration and policy enforcement
- **Vulnerability Management**: Automated assessment and remediation planning
- **Audit Logging**: Comprehensive audit trails and compliance reporting

### Next Steps

1. **Practice**: Implement the hands-on exercises
2. **Customize**: Adapt examples to your specific compliance requirements
3. **Integrate**: Combine with existing security tools and processes
4. **Monitor**: Set up continuous monitoring and alerting
5. **Review**: Regular security and compliance reviews

### Key Takeaways

- Security and compliance must be automated and integrated into CI/CD
- Multiple scanning tools provide comprehensive coverage
- Policy as code enables consistent security enforcement
- Continuous monitoring is essential for maintaining compliance
- Documentation and audit trails are critical for compliance frameworks
```

### Secret Scanning and Remediation

```yaml
# .github/workflows/secret-management.yml
name: 'Secret Management'

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 3 * * 1'  # Weekly secret rotation
  workflow_dispatch:
    inputs:
      action:
        description: 'Action to perform'
        required: true
        type: choice
        options: [scan, rotate, validate]
      secret_name:
        description: 'Specific secret name (optional)'
        required: false

jobs:
  secret-scanning:
    name: 'Secret Scanning and Detection'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Run comprehensive secret scan
      run: |
        # Install secret scanning tools
        pip install detect-secrets
        
        # Run detect-secrets
        detect-secrets scan --all-files --baseline .secrets.baseline
        
        # Check for new secrets
        if detect-secrets audit .secrets.baseline; then
          echo "✅ No new secrets detected"
        else
          echo "❌ New secrets detected!"
          detect-secrets audit .secrets.baseline --report
          exit 1
        fi
    
    - name: Scan for hardcoded secrets in code
      run: |
        python3 << 'EOF'
        import re
        import os
        import json
        from pathlib import Path
        
        # Common secret patterns
        secret_patterns = {
            'aws_access_key': r'AKIA[0-9A-Z]{16}',
            'aws_secret_key': r'[0-9a-zA-Z/+]{40}',
            'github_token': r'ghp_[0-9a-zA-Z]{36}',
            'slack_token': r'xox[baprs]-[0-9a-zA-Z-]+',
            'private_key': r'-----BEGIN [A-Z ]+PRIVATE KEY-----',
            'api_key': r'[aA][pP][iI][_]?[kK][eE][yY].*[=:][\s]*["\']?[0-9a-zA-Z]{20,}["\']?',
            'password': r'[pP][aA][sS][sS][wW][oO][rR][dD].*[=:][\s]*["\']?[0-9a-zA-Z!@#$%^&*()]{8,}["\']?',
            'database_url': r'[a-zA-Z][a-zA-Z0-9+.-]*://[^\s]+',
            'jwt_secret': r'[jJ][wW][tT][_]?[sS][eE][cC][rR][eE][tT].*[=:][\s]*["\']?[0-9a-zA-Z]{32,}["\']?'
        }
        
        findings = []
        
        # Scan files
        for root, dirs, files in os.walk('.'):
            # Skip common directories
            dirs[:] = [d for d in dirs if d not in ['.git', 'node_modules', '.venv', '__pycache__']]
            
            for file in files:
                if file.endswith(('.py', '.js', '.ts', '.yaml', '.yml', '.json', '.env')):
                    file_path = os.path.join(root, file)
                    
                    try:
                        with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
                            content = f.read()
                            
                            for secret_type, pattern in secret_patterns.items():
                                matches = re.finditer(pattern, content, re.IGNORECASE)
                                for match in matches:
                                    line_num = content[:match.start()].count('\n') + 1
                                    findings.append({
                                        'file': file_path,
                                        'line': line_num,
                                        'type': secret_type,
                                        'match': match.group()[:50] + '...' if len(match.group()) > 50 else match.group()
                                    })
                    except Exception as e:
                        print(f"Error scanning {file_path}: {e}")
        
        # Save findings
        with open('secret-scan-results.json', 'w') as f:
            json.dump({
                'timestamp': '2024-01-01T00:00:00Z',
                'total_findings': len(findings),
                'findings': findings
            }, f, indent=2)
        
        if findings:
            print(f"❌ Found {len(findings)} potential secrets in code!")
            for finding in findings[:10]:  # Show first 10
                print(f"  {finding['file']}:{finding['line']} - {finding['type']}")
            exit(1)
        else:
            print("✅ No hardcoded secrets found")
        EOF
    
    - name: Upload secret scan results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: secret-scan-results
        path: secret-scan-results.json

  secret-rotation:
    name: 'Secret Rotation'
    runs-on: ubuntu-latest
    if: github.event.inputs.action == 'rotate' || github.event_name == 'schedule'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install boto3 requests
    
    - name: Run secret rotation
      run: |
        python scripts/secret-rotation.py \
          --region ${{ vars.AWS_REGION }} \
          --max-age-days 90 \
          ${{ github.event.inputs.secret_name && format('--secret-name {0}', github.event.inputs.secret_name) || '' }}
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    
    - name: Upload rotation report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: secret-rotation-report
        path: secret-rotation-report.json

  secret-validation:
    name: 'Secret Validation'
    runs-on: ubuntu-latest
    if: github.event.inputs.action == 'validate' || github.event_name == 'push'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Validate secrets accessibility
      run: |
        python3 << 'EOF'
        import boto3
        import json
        import os
        from datetime import datetime
        
        def validate_aws_secrets():
            """Validate AWS Secrets Manager secrets"""
            secrets_client = boto3.client('secretsmanager')
            validation_results = []
            
            try:
                # List all secrets
                paginator = secrets_client.get_paginator('list_secrets')
                
                for page in paginator.paginate():
                    for secret in page['SecretList']:
                        secret_name = secret['Name']
                        
                        try:
                            # Test secret retrieval
                            secret_value = secrets_client.get_secret_value(SecretId=secret_name)
                            
                            # Validate JSON format
                            if secret_value.get('SecretString'):
                                json.loads(secret_value['SecretString'])
                            
                            validation_results.append({
                                'secret_name': secret_name,
                                'status': 'VALID',
                                'message': 'Secret is accessible and properly formatted'
                            })
                            
                        except json.JSONDecodeError:
                            validation_results.append({
                                'secret_name': secret_name,
                                'status': 'WARNING',
                                'message': 'Secret is not valid JSON'
                            })
                        except Exception as e:
                            validation_results.append({
                                'secret_name': secret_name,
                                'status': 'ERROR',
                                'message': f'Cannot access secret: {str(e)}'
                            })
            
            except Exception as e:
                validation_results.append({
                    'secret_name': 'all',
                    'status': 'ERROR',
                    'message': f'Error listing secrets: {str(e)}'
                })
            
            return validation_results
        
        def validate_github_secrets():
            """Validate GitHub secrets are properly set"""
            required_secrets = [
                'AWS_ACCESS_KEY_ID',
                'AWS_SECRET_ACCESS_KEY',
                'SNYK_TOKEN',
                'SONAR_TOKEN'
            ]
            
            validation_results = []
            
            for secret_name in required_secrets:
                if os.getenv(secret_name):
                    validation_results.append({
                        'secret_name': secret_name,
                        'status': 'VALID',
                        'message': 'GitHub secret is set'
                    })
                else:
                    validation_results.append({
                        'secret_name': secret_name,
                        'status': 'ERROR',
                        'message': 'GitHub secret is not set'
                    })
            
            return validation_results
        
        # Run validations
        aws_results = validate_aws_secrets()
        github_results = validate_github_secrets()
        
        all_results = aws_results + github_results
        
        # Generate report
        report = {
            'timestamp': datetime.now().isoformat(),
            'total_secrets': len(all_results),
            'valid_secrets': len([r for r in all_results if r['status'] == 'VALID']),
            'warning_secrets': len([r for r in all_results if r['status'] == 'WARNING']),
            'error_secrets': len([r for r in all_results if r['status'] == 'ERROR']),
            'results': all_results
        }
        
        with open('secret-validation-report.json', 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"Secret Validation Complete")
        print(f"Total: {report['total_secrets']}")
        print(f"Valid: {report['valid_secrets']}")
        print(f"Warnings: {report['warning_secrets']}")
        print(f"Errors: {report['error_secrets']}")
        
        # Exit with error if any secrets are invalid
        if report['error_secrets'] > 0:
            exit(1)
        EOF
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    
    - name: Upload validation report
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: secret-validation-report
        path: secret-validation-report.json
```

---

## 4. Security Policies as Code

### Open Policy Agent (OPA) Integration

```rego
# policies/kubernetes-security.rego
package kubernetes.security

# Deny containers running as root
deny[msg] {
    input.kind == "Pod"
    input.spec.securityContext.runAsUser == 0
    msg := "Container must not run as root user"
}

deny[msg] {
    input.kind == "Deployment"
    input.spec.template.spec.securityContext.runAsUser == 0
    msg := "Container must not run as root user"
}

# Require security context
deny[msg] {
    input.kind in ["Pod", "Deployment"]
    not input.spec.securityContext
    msg := "Security context must be defined"
}

# Deny privileged containers
deny[msg] {
    input.kind == "Pod"
    input.spec.containers[_].securityContext.privileged == true
    msg := "Privileged containers are not allowed"
}

# Require resource limits
deny[msg] {
    input.kind in ["Pod", "Deployment"]
    container := input.spec.containers[_]
    not container.resources.limits
    msg := sprintf("Container '%s' must have resource limits defined", [container.name])
}

# Deny host network
deny[msg] {
    input.kind == "Pod"
    input.spec.hostNetwork == true
    msg := "Host network access is not allowed"
}

# Require non-root filesystem
deny[msg] {
    input.kind in ["Pod", "Deployment"]
    container := input.spec.containers[_]
    not container.securityContext.readOnlyRootFilesystem
    msg := sprintf("Container '%s' must use read-only root filesystem", [container.name])
}

# Deny host path volumes
deny[msg] {
    input.kind == "Pod"
    volume := input.spec.volumes[_]
    volume.hostPath
    msg := sprintf("Host path volume '%s' is not allowed", [volume.name])
}

# Require image pull policy
deny[msg] {
    input.kind in ["Pod", "Deployment"]
    container := input.spec.containers[_]
    not container.imagePullPolicy
    msg := sprintf("Container '%s' must specify imagePullPolicy", [container.name])
}

# Deny latest tag
deny[msg] {
    input.kind in ["Pod", "Deployment"]
    container := input.spec.containers[_]
    endswith(container.image, ":latest")
    msg := sprintf("Container '%s' must not use 'latest' tag", [container.name])
}

# Require specific image registries
allowed_registries := {
    "gcr.io",
    "us-docker.pkg.dev",
    "your-company-registry.com"
}

deny[msg] {
    input.kind in ["Pod", "Deployment"]
    container := input.spec.containers[_]
    registry := split(container.image, "/")[0]
    not registry in allowed_registries
    msg := sprintf("Container '%s' uses unauthorized registry '%s'", [container.name, registry])
}
```

```rego
# policies/terraform-security.rego
package terraform.security

# AWS S3 Security Policies
deny[msg] {
    input.resource_type == "aws_s3_bucket"
    not input.config.server_side_encryption_configuration
    msg := sprintf("S3 bucket '%s' must have encryption enabled", [input.address])
}

deny[msg] {
    input.resource_type == "aws_s3_bucket_public_access_block"
    input.config.block_public_acls != true
    msg := sprintf("S3 bucket '%s' must block public ACLs", [input.address])
}

# AWS EC2 Security Policies
deny[msg] {
    input.resource_type == "aws_instance"
    input.config.associate_public_ip_address == true
    msg := sprintf("EC2 instance '%s' must not have public IP", [input.address])
}

deny[msg] {
    input.resource_type == "aws_security_group"
    rule := input.config.ingress[_]
    rule.cidr_blocks[_] == "0.0.0.0/0"
    rule.from_port == 22
    msg := sprintf("Security group '%s' must not allow SSH from 0.0.0.0/0", [input.address])
}

# AWS RDS Security Policies
deny[msg] {
    input.resource_type == "aws_db_instance"
    input.config.storage_encrypted != true
    msg := sprintf("RDS instance '%s' must have encryption enabled", [input.address])
}

deny[msg] {
    input.resource_type == "aws_db_instance"
    input.config.publicly_accessible == true
    msg := sprintf("RDS instance '%s' must not be publicly accessible", [input.address])
}

# AWS IAM Security Policies
deny[msg] {
    input.resource_type == "aws_iam_policy"
    statement := input.config.policy.Statement[_]
    statement.Effect == "Allow"
    statement.Action[_] == "*"
    statement.Resource[_] == "*"
    msg := sprintf("IAM policy '%s' must not allow all actions on all resources", [input.address])
}

# Require MFA for IAM users
deny[msg] {
    input.resource_type == "aws_iam_user"
    not input.config.force_destroy
    msg := sprintf("IAM user '%s' should have force_destroy enabled for security", [input.address])
}
```

### Policy Enforcement Pipeline

```yaml
# .github/workflows/policy-enforcement.yml
name: 'Security Policy Enforcement'

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  opa-policy-test:
    name: 'OPA Policy Testing'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup OPA
      run: |
        curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
        chmod +x opa
        sudo mv opa /usr/local/bin/
    
    - name: Test OPA policies
      run: |
        # Test Kubernetes policies
        opa test policies/ --verbose
        
        # Validate policy syntax
        opa fmt --diff policies/
    
    - name: Run policy against Kubernetes manifests
      run: |
        # Find all Kubernetes YAML files
        find . -name '*.yaml' -o -name '*.yml' | grep -E 'k8s|kubernetes' > k8s-files.txt
        
        if [ -s k8s-files.txt ]; then
          echo "Evaluating Kubernetes manifests against security policies..."
          
          while read -r file; do
            echo "Checking $file"
            opa eval -d policies/kubernetes-security.rego -i "$file" "data.kubernetes.security.deny[x]"
          done < k8s-files.txt
        else
          echo "No Kubernetes manifests found"
        fi
    
    - name: Run policy against Terraform plans
      run: |
        # This would typically run against Terraform plan JSON output
        if [ -d "infrastructure" ]; then
          echo "Evaluating Terraform configuration against security policies..."
          
          # Convert Terraform to JSON for OPA evaluation
          cd infrastructure
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json
          
          # Evaluate against policies
          opa eval -d ../policies/terraform-security.rego -i tfplan.json "data.terraform.security.deny[x]"
        fi

  conftest-validation:
    name: 'Conftest Policy Validation'
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup Conftest
      run: |
        wget https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
        tar xzf conftest_0.46.0_Linux_x86_64.tar.gz
        sudo mv conftest /usr/local/bin/
    
    - name: Validate Kubernetes manifests
      run: |
        find . -name '*.yaml' -o -name '*.yml' | grep -E 'k8s|kubernetes' | while read -r file; do
          echo "Validating $file with Conftest"
          conftest verify --policy policies/ "$file" || exit 1
        done
    
    - name: Validate Dockerfile
      run: |
        if [ -f "Dockerfile" ]; then
          conftest verify --policy policies/ Dockerfile
        fi

  gatekeeper-policies:
    name: 'Gatekeeper Policy Deployment'
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Deploy Gatekeeper policies
      run: |
        # Convert OPA policies to Gatekeeper ConstraintTemplates
        python3 << 'EOF'
        import yaml
        import os
        
        # Create Gatekeeper ConstraintTemplate for security policies
        constraint_template = {
            'apiVersion': 'templates.gatekeeper.sh/v1beta1',
            'kind': 'ConstraintTemplate',
            'metadata': {
                'name': 'k8ssecuritypolicy'
            },
            'spec': {
                'crd': {
                    'spec': {
                        'names': {
                            'kind': 'K8sSecurityPolicy'
                        },
                        'validation': {
                            'type': 'object',
                            'properties': {
                                'message': {
                                    'type': 'string'
                                }
                            }
                        }
                    }
                },
                'targets': [{
                    'target': 'admission.k8s.gatekeeper.sh',
                    'rego': open('policies/kubernetes-security.rego').read()
                }]
            }
        }
        
        with open('gatekeeper-constraint-template.yaml', 'w') as f:
            yaml.dump(constraint_template, f)
        
        # Create Constraint instance
        constraint = {
            'apiVersion': 'constraints.gatekeeper.sh/v1beta1',
            'kind': 'K8sSecurityPolicy',
            'metadata': {
                'name': 'security-policy-constraint'
            },
            'spec': {
                'match': {
                    'kinds': [{
                        'apiGroups': [''],
                        'kinds': ['Pod']
                    }, {
                        'apiGroups': ['apps'],
                        'kinds': ['Deployment']
                    }]
                },
                'parameters': {
                    'message': 'Security policy violation detected'
                }
            }
        }
        
        with open('gatekeeper-constraint.yaml', 'w') as f:
            yaml.dump(constraint, f)
        EOF
        
        # Apply to cluster (if kubectl is configured)
        if kubectl cluster-info &> /dev/null; then
          kubectl apply -f gatekeeper-constraint-template.yaml
          kubectl apply -f gatekeeper-constraint.yaml
          echo "Gatekeeper policies deployed successfully"
        else
          echo "Kubectl not configured, skipping deployment"
        fi
```

---

## 5. Vulnerability Management

### Automated Vulnerability Assessment

```python
# scripts/vulnerability-manager.py
import boto3
import json
import requests
import time
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class VulnerabilityManager:
    def __init__(self, region: str = 'us-west-2'):
        self.ec2_client = boto3.client('ec2', region_name=region)
        self.ssm_client = boto3.client('ssm', region_name=region)
        self.inspector_client = boto3.client('inspector2', region_name=region)
        self.region = region
        self.vulnerabilities = []
        
    def scan_ec2_instances(self) -> List[Dict[str, Any]]:
        """Scan EC2 instances for vulnerabilities"""
        logger.info("Starting EC2 vulnerability scan...")
        
        try:
            # Get all running instances
            response = self.ec2_client.describe_instances(
                Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
            )
            
            instances = []
            for reservation in response['Reservations']:
                for instance in reservation['Instances']:
                    instances.append({
                        'instance_id': instance['InstanceId'],
                        'instance_type': instance['InstanceType'],
                        'launch_time': instance['LaunchTime'].isoformat(),
                        'vpc_id': instance.get('VpcId'),
                        'subnet_id': instance.get('SubnetId'),
                        'security_groups': [sg['GroupId'] for sg in instance.get('SecurityGroups', [])],
                        'tags': {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
                    })
            
            # Check for outdated AMIs
            for instance in instances:
                self._check_ami_vulnerabilities(instance)
                self._check_security_group_vulnerabilities(instance)
                self._check_patch_compliance(instance)
            
            return instances
            
        except Exception as e:
            logger.error(f"Error scanning EC2 instances: {e}")
            return []
    
    def _check_ami_vulnerabilities(self, instance: Dict[str, Any]):
        """Check if instance is using outdated AMI"""
        try:
            instance_id = instance['instance_id']
            
            # Get instance details
            response = self.ec2_client.describe_instances(InstanceIds=[instance_id])
            ami_id = response['Reservations'][0]['Instances'][0]['ImageId']
            
            # Get AMI details
            ami_response = self.ec2_client.describe_images(ImageIds=[ami_id])
            if ami_response['Images']:
                ami = ami_response['Images'][0]
                creation_date = datetime.fromisoformat(ami['CreationDate'].replace('Z', '+00:00'))
                
                # Check if AMI is older than 90 days
                if datetime.now().replace(tzinfo=creation_date.tzinfo) - creation_date > timedelta(days=90):
                    self.vulnerabilities.append({
                        'resource_type': 'EC2_INSTANCE',
                        'resource_id': instance_id,
                        'vulnerability_type': 'OUTDATED_AMI',
                        'severity': 'MEDIUM',
                        'description': f'Instance using AMI older than 90 days (created: {creation_date.date()})',
                        'recommendation': 'Update to latest AMI version',
                        'cve_ids': [],
                        'discovered_at': datetime.now().isoformat()
                    })
                    
        except Exception as e:
            logger.error(f"Error checking AMI vulnerabilities for {instance['instance_id']}: {e}")
    
    def _check_security_group_vulnerabilities(self, instance: Dict[str, Any]):
        """Check security group configurations for vulnerabilities"""
        try:
            for sg_id in instance['security_groups']:
                response = self.ec2_client.describe_security_groups(GroupIds=[sg_id])
                
                for sg in response['SecurityGroups']:
                    # Check for overly permissive rules
                    for rule in sg['IpPermissions']:
                        for ip_range in rule.get('IpRanges', []):
                            if ip_range.get('CidrIp') == '0.0.0.0/0':
                                severity = 'HIGH' if rule.get('FromPort') == 22 else 'MEDIUM'
                                
                                self.vulnerabilities.append({
                                    'resource_type': 'SECURITY_GROUP',
                                    'resource_id': sg_id,
                                    'vulnerability_type': 'OVERLY_PERMISSIVE_RULE',
                                    'severity': severity,
                                    'description': f'Security group allows access from 0.0.0.0/0 on port {rule.get("FromPort", "all")}',
                                    'recommendation': 'Restrict source IP ranges to specific networks',
                                    'cve_ids': [],
                                    'discovered_at': datetime.now().isoformat(),
                                    'affected_instances': [instance['instance_id']]
                                })
                                
        except Exception as e:
            logger.error(f"Error checking security group vulnerabilities: {e}")
    
    def _check_patch_compliance(self, instance: Dict[str, Any]):
        """Check patch compliance using SSM"""
        try:
            instance_id = instance['instance_id']
            
            # Check if instance is managed by SSM
            response = self.ssm_client.describe_instance_information(
                Filters=[{'Key': 'InstanceIds', 'Values': [instance_id]}]
            )
            
            if not response['InstanceInformationList']:
                self.vulnerabilities.append({
                    'resource_type': 'EC2_INSTANCE',
                    'resource_id': instance_id,
                    'vulnerability_type': 'SSM_NOT_MANAGED',
                    'severity': 'MEDIUM',
                    'description': 'Instance is not managed by Systems Manager',
                    'recommendation': 'Install SSM agent and configure IAM role',
                    'cve_ids': [],
                    'discovered_at': datetime.now().isoformat()
                })
                return
            
            # Check patch compliance
            try:
                compliance_response = self.ssm_client.list_compliance_items(
                    ResourceId=instance_id,
                    ResourceType='ManagedInstance',
                    Filters=[{'Key': 'ComplianceType', 'Values': ['Patch']}]
                )
                
                non_compliant_patches = [
                    item for item in compliance_response['ComplianceItems']
                    if item['Status'] == 'NON_COMPLIANT'
                ]
                
                if non_compliant_patches:
                    self.vulnerabilities.append({
                        'resource_type': 'EC2_INSTANCE',
                        'resource_id': instance_id,
                        'vulnerability_type': 'MISSING_PATCHES',
                        'severity': 'HIGH',
                        'description': f'Instance has {len(non_compliant_patches)} missing patches',
                        'recommendation': 'Apply missing patches using Systems Manager Patch Manager',
                        'cve_ids': [],
                        'discovered_at': datetime.now().isoformat(),
                        'patch_details': non_compliant_patches[:5]  # Include first 5 patches
                    })
                    
            except Exception as e:
                logger.warning(f"Could not check patch compliance for {instance_id}: {e}")
                
        except Exception as e:
            logger.error(f"Error checking patch compliance for {instance['instance_id']}: {e}")
    
    def scan_container_images(self, image_uris: List[str]) -> List[Dict[str, Any]]:
        """Scan container images for vulnerabilities using Inspector"""
        logger.info(f"Scanning {len(image_uris)} container images...")
        
        scan_results = []
        
        for image_uri in image_uris:
            try:
                # Trigger ECR scan if it's an ECR image
                if 'amazonaws.com' in image_uri:
                    self._trigger_ecr_scan(image_uri)
                
                # Use external tools for comprehensive scanning
                trivy_results = self._scan_with_trivy(image_uri)
                scan_results.extend(trivy_results)
                
            except Exception as e:
                logger.error(f"Error scanning image {image_uri}: {e}")
                
        return scan_results
    
    def _trigger_ecr_scan(self, image_uri: str):
        """Trigger ECR vulnerability scan"""
        try:
            # Parse ECR image URI
            parts = image_uri.split('/')
            repository_name = parts[-1].split(':')[0]
            
            ecr_client = boto3.client('ecr', region_name=self.region)
            
            # Start image scan
            ecr_client.start_image_scan(
                repositoryName=repository_name,
                imageId={'imageTag': 'latest'}
            )
            
            logger.info(f"ECR scan triggered for {repository_name}")
            
        except Exception as e:
            logger.warning(f"Could not trigger ECR scan for {image_uri}: {e}")
    
    def _scan_with_trivy(self, image_uri: str) -> List[Dict[str, Any]]:
        """Scan image with Trivy (simulated - would use actual Trivy API)"""
        # This is a simulation - in practice, you'd integrate with Trivy API
        vulnerabilities = []
        
        # Simulate some common vulnerabilities
        simulated_vulns = [
            {
                'cve_id': 'CVE-2023-1234',
                'severity': 'HIGH',
                'package': 'openssl',
                'version': '1.1.1f',
                'fixed_version': '1.1.1g',
                'description': 'Buffer overflow in OpenSSL'
            },
            {
                'cve_id': 'CVE-2023-5678',
                'severity': 'MEDIUM',
                'package': 'curl',
                'version': '7.68.0',
                'fixed_version': '7.70.0',
                'description': 'Information disclosure in curl'
            }
        ]
        
        for vuln in simulated_vulns:
            vulnerabilities.append({
                'resource_type': 'CONTAINER_IMAGE',
                'resource_id': image_uri,
                'vulnerability_type': 'PACKAGE_VULNERABILITY',
                'severity': vuln['severity'],
                'description': f"{vuln['description']} in {vuln['package']} {vuln['version']}",
                'recommendation': f"Update {vuln['package']} to version {vuln['fixed_version']} or later",
                'cve_ids': [vuln['cve_id']],
                'discovered_at': datetime.now().isoformat(),
                'package_details': vuln
            })
        
        return vulnerabilities
    
    def prioritize_vulnerabilities(self) -> List[Dict[str, Any]]:
        """Prioritize vulnerabilities based on severity and exploitability"""
        logger.info("Prioritizing vulnerabilities...")
        
        # Define priority scoring
        severity_scores = {
            'CRITICAL': 10,
            'HIGH': 8,
            'MEDIUM': 5,
            'LOW': 2,
            'INFO': 1
        }
        
        # Add priority scores
        for vuln in self.vulnerabilities:
            base_score = severity_scores.get(vuln['severity'], 1)
            
            # Increase priority for certain vulnerability types
            if vuln['vulnerability_type'] in ['MISSING_PATCHES', 'OVERLY_PERMISSIVE_RULE']:
                base_score += 2
            
            # Increase priority if CVE IDs are present
            if vuln.get('cve_ids'):
                base_score += 1
            
            # Increase priority for public-facing resources
            if 'public' in str(vuln).lower():
                base_score += 3
            
            vuln['priority_score'] = min(base_score, 10)  # Cap at 10
        
        # Sort by priority score (highest first)
        prioritized = sorted(
            self.vulnerabilities,
            key=lambda x: x['priority_score'],
            reverse=True
        )
        
        return prioritized
    
    def generate_remediation_plan(self) -> Dict[str, Any]:
        """Generate automated remediation plan"""
        logger.info("Generating remediation plan...")
        
        remediation_plan = {
            'timestamp': datetime.now().isoformat(),
            'total_vulnerabilities': len(self.vulnerabilities),
            'critical_count': len([v for v in self.vulnerabilities if v['severity'] == 'CRITICAL']),
            'high_count': len([v for v in self.vulnerabilities if v['severity'] == 'HIGH']),
            'medium_count': len([v for v in self.vulnerabilities if v['severity'] == 'MEDIUM']),
            'low_count': len([v for v in self.vulnerabilities if v['severity'] == 'LOW']),
            'remediation_actions': []
        }
        
        # Group vulnerabilities by type for batch remediation
        vuln_groups = {}
        for vuln in self.vulnerabilities:
            vuln_type = vuln['vulnerability_type']
            if vuln_type not in vuln_groups:
                vuln_groups[vuln_type] = []
            vuln_groups[vuln_type].append(vuln)
        
        # Generate remediation actions
        for vuln_type, vulns in vuln_groups.items():
            if vuln_type == 'MISSING_PATCHES':
                remediation_plan['remediation_actions'].append({
                    'action_type': 'PATCH_MANAGEMENT',
                    'priority': 'HIGH',
                    'affected_resources': [v['resource_id'] for v in vulns],
                    'automation_available': True,
                    'estimated_time': '2-4 hours',
                    'steps': [
                        'Create maintenance window in Systems Manager',
                        'Run patch baseline scan',
                        'Apply patches during maintenance window',
                        'Verify patch installation',
                        'Restart instances if required'
                    ]
                })
            
            elif vuln_type == 'OVERLY_PERMISSIVE_RULE':
                remediation_plan['remediation_actions'].append({
                    'action_type': 'SECURITY_GROUP_HARDENING',
                    'priority': 'HIGH',
                    'affected_resources': list(set([v['resource_id'] for v in vulns])),
                    'automation_available': True,
                    'estimated_time': '30 minutes',
                    'steps': [
                        'Review current security group rules',
                        'Identify legitimate source IP ranges',
                        'Update security group rules',
                        'Test application connectivity',
                        'Monitor for access issues'
                    ]
                })
            
            elif vuln_type == 'OUTDATED_AMI':
                remediation_plan['remediation_actions'].append({
                    'action_type': 'AMI_UPDATE',
                    'priority': 'MEDIUM',
                    'affected_resources': [v['resource_id'] for v in vulns],
                    'automation_available': False,
                    'estimated_time': '4-8 hours',
                    'steps': [
                        'Identify latest AMI version',
                        'Create launch template with new AMI',
                        'Test new AMI in staging environment',
                        'Schedule maintenance window',
                        'Replace instances with new AMI',
                        'Verify application functionality'
                    ]
                })
        
        return remediation_plan
    
    def generate_vulnerability_report(self) -> Dict[str, Any]:
        """Generate comprehensive vulnerability report"""
        prioritized_vulns = self.prioritize_vulnerabilities()
        remediation_plan = self.generate_remediation_plan()
        
        report = {
            'scan_metadata': {
                'timestamp': datetime.now().isoformat(),
                'region': self.region,
                'scan_type': 'COMPREHENSIVE',
                'total_resources_scanned': len(set([v['resource_id'] for v in self.vulnerabilities]))
            },
            'executive_summary': {
                'total_vulnerabilities': len(self.vulnerabilities),
                'critical_vulnerabilities': len([v for v in self.vulnerabilities if v['severity'] == 'CRITICAL']),
                'high_vulnerabilities': len([v for v in self.vulnerabilities if v['severity'] == 'HIGH']),
                'medium_vulnerabilities': len([v for v in self.vulnerabilities if v['severity'] == 'MEDIUM']),
                'low_vulnerabilities': len([v for v in self.vulnerabilities if v['severity'] == 'LOW']),
                'risk_score': self._calculate_risk_score()
            },
            'vulnerabilities': prioritized_vulns,
            'remediation_plan': remediation_plan,
            'compliance_impact': self._assess_compliance_impact(),
            'recommendations': self._generate_recommendations()
        }
        
        return report
    
    def _calculate_risk_score(self) -> float:
        """Calculate overall risk score"""
        if not self.vulnerabilities:
            return 0.0
        
        severity_weights = {
            'CRITICAL': 10,
            'HIGH': 7,
            'MEDIUM': 4,
            'LOW': 1
        }
        
        total_score = sum(
            severity_weights.get(v['severity'], 0)
            for v in self.vulnerabilities
        )
        
        # Normalize to 0-100 scale
        max_possible_score = len(self.vulnerabilities) * 10
        risk_score = (total_score / max_possible_score) * 100 if max_possible_score > 0 else 0
        
        return round(risk_score, 2)
    
    def _assess_compliance_impact(self) -> Dict[str, Any]:
        """Assess impact on compliance frameworks"""
        compliance_impact = {
            'SOC2': {'affected': False, 'controls': []},
            'PCI_DSS': {'affected': False, 'controls': []},
            'HIPAA': {'affected': False, 'controls': []},
            'ISO27001': {'affected': False, 'controls': []}
        }
        
        for vuln in self.vulnerabilities:
            if vuln['vulnerability_type'] in ['MISSING_PATCHES', 'OVERLY_PERMISSIVE_RULE']:
                compliance_impact['SOC2']['affected'] = True
                compliance_impact['SOC2']['controls'].append('CC6.1 - Logical Access')
                
                compliance_impact['ISO27001']['affected'] = True
                compliance_impact['ISO27001']['controls'].append('A.9.1.2 - Access to networks and network services')
            
            if vuln['severity'] in ['CRITICAL', 'HIGH']:
                compliance_impact['PCI_DSS']['affected'] = True
                compliance_impact['PCI_DSS']['controls'].append('Requirement 6 - Develop and maintain secure systems')
                
                compliance_impact['HIPAA']['affected'] = True
                compliance_impact['HIPAA']['controls'].append('164.312(a)(1) - Access control')
        
        return compliance_impact
    
    def _generate_recommendations(self) -> List[str]:
        """Generate high-level recommendations"""
        recommendations = []
        
        critical_count = len([v for v in self.vulnerabilities if v['severity'] == 'CRITICAL'])
        high_count = len([v for v in self.vulnerabilities if v['severity'] == 'HIGH'])
        
        if critical_count > 0:
            recommendations.append(
                f"Immediately address {critical_count} critical vulnerabilities"
            )
        
        if high_count > 0:
            recommendations.append(
                f"Prioritize remediation of {high_count} high-severity vulnerabilities"
            )
        
        patch_vulns = [v for v in self.vulnerabilities if v['vulnerability_type'] == 'MISSING_PATCHES']
        if patch_vulns:
            recommendations.append(
                "Implement automated patch management using Systems Manager"
            )
        
        sg_vulns = [v for v in self.vulnerabilities if v['vulnerability_type'] == 'OVERLY_PERMISSIVE_RULE']
        if sg_vulns:
            recommendations.append(
                "Review and harden security group configurations"
            )
        
        recommendations.extend([
            "Implement continuous vulnerability scanning",
            "Set up automated alerting for new vulnerabilities",
            "Establish regular security review processes",
            "Consider implementing infrastructure as code for consistent security"
        ])
        
        return recommendations

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Vulnerability Management Scanner')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--scan-type', choices=['ec2', 'containers', 'all'], default='all', help='Type of scan to perform')
    parser.add_argument('--output-file', default='vulnerability-report.json', help='Output file for report')
    
    args = parser.parse_args()
    
    manager = VulnerabilityManager(region=args.region)
    
    if args.scan_type in ['ec2', 'all']:
        logger.info("Scanning EC2 instances...")
        manager.scan_ec2_instances()
    
    if args.scan_type in ['containers', 'all']:
        logger.info("Scanning container images...")
        # Example image URIs - in practice, these would be discovered dynamically
        image_uris = [
            '123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app:latest',
            'nginx:latest',
            'postgres:13'
        ]
        container_vulns = manager.scan_container_images(image_uris)
        manager.vulnerabilities.extend(container_vulns)
    
    # Generate comprehensive report
    report = manager.generate_vulnerability_report()
    
    # Save report
    with open(args.output_file, 'w') as f:
        json.dump(report, f, indent=2)
    
    # Print summary
    print(f"\n{'='*60}")
    print("VULNERABILITY ASSESSMENT SUMMARY")
    print(f"{'='*60}")
    print(f"Total Vulnerabilities: {report['executive_summary']['total_vulnerabilities']}")
    print(f"Critical: {report['executive_summary']['critical_vulnerabilities']}")
    print(f"High: {report['executive_summary']['high_vulnerabilities']}")
    print(f"Medium: {report['executive_summary']['medium_vulnerabilities']}")
    print(f"Low: {report['executive_summary']['low_vulnerabilities']}")
    print(f"Risk Score: {report['executive_summary']['risk_score']}/100")
    
    print(f"\nTop Recommendations:")
    for i, rec in enumerate(report['recommendations'][:5], 1):
        print(f"{i}. {rec}")
    
    print(f"\nDetailed report saved to: {args.output_file}")
    
    # Exit with error if critical vulnerabilities found
    if report['executive_summary']['critical_vulnerabilities'] > 0:
        exit(1)

if __name__ == "__main__":
    main()
```