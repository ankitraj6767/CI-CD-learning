# Module 20: Enterprise Patterns

## Learning Objectives

By the end of this module, you will be able to:

- Design and implement enterprise-scale CI/CD architectures
- Implement advanced deployment patterns for large organizations
- Set up multi-tenant CI/CD systems
- Design disaster recovery and business continuity strategies
- Implement governance and compliance frameworks
- Create scalable monitoring and observability solutions
- Design cost optimization strategies for enterprise CI/CD
- Implement advanced security patterns for enterprise environments

## Prerequisites

- Completion of Modules 17-19
- Experience with enterprise software development
- Understanding of organizational governance
- Knowledge of cloud architecture patterns
- Familiarity with enterprise security requirements

---

## 1. Enterprise CI/CD Architecture

### Multi-Tenant Platform Design

```yaml
# .github/workflows/enterprise-platform.yml
name: Enterprise CI/CD Platform

on:
  workflow_call:
    inputs:
      tenant_id:
        required: true
        type: string
      environment:
        required: true
        type: string
      service_name:
        required: true
        type: string
    secrets:
      TENANT_SECRETS:
        required: true

env:
  TENANT_ID: ${{ inputs.tenant_id }}
  ENVIRONMENT: ${{ inputs.environment }}
  SERVICE_NAME: ${{ inputs.service_name }}
  REGISTRY: ${{ vars.ENTERPRISE_REGISTRY }}

jobs:
  tenant-validation:
    runs-on: ubuntu-latest
    outputs:
      tenant-config: ${{ steps.validate.outputs.config }}
      resource-limits: ${{ steps.validate.outputs.limits }}
    steps:
      - name: Validate tenant
        id: validate
        run: |
          # Validate tenant exists and has permissions
          python scripts/validate-tenant.py \
            --tenant-id "${{ inputs.tenant_id }}" \
            --environment "${{ inputs.environment }}"
  
  resource-allocation:
    runs-on: ubuntu-latest
    needs: tenant-validation
    steps:
      - name: Allocate resources
        run: |
          # Allocate compute and storage resources based on tenant tier
          python scripts/allocate-resources.py \
            --tenant-id "${{ inputs.tenant_id }}" \
            --limits '${{ needs.tenant-validation.outputs.resource-limits }}'
  
  security-scan:
    runs-on: ubuntu-latest
    needs: tenant-validation
    steps:
      - name: Tenant-specific security scan
        run: |
          # Run security scans with tenant-specific policies
          python scripts/tenant-security-scan.py \
            --tenant-id "${{ inputs.tenant_id }}" \
            --config '${{ needs.tenant-validation.outputs.tenant-config }}'
  
  build-and-deploy:
    runs-on: ubuntu-latest
    needs: [tenant-validation, resource-allocation, security-scan]
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build with tenant isolation
        run: |
          # Build with tenant-specific configuration
          docker build \
            --build-arg TENANT_ID="${{ inputs.tenant_id }}" \
            --build-arg ENVIRONMENT="${{ inputs.environment }}" \
            -t ${{ env.REGISTRY }}/${{ inputs.tenant_id }}/${{ inputs.service_name }}:${{ github.sha }} .
      
      - name: Deploy to tenant environment
        run: |
          # Deploy to tenant-specific namespace
          kubectl apply -f k8s/tenant-deployment.yaml \
            --namespace="tenant-${{ inputs.tenant_id }}-${{ inputs.environment }}"
```

### Enterprise Platform Management

```python
# scripts/enterprise-platform-manager.py
import json
import yaml
import boto3
import kubernetes
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
import logging
from datetime import datetime, timedelta

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class TenantConfig:
    tenant_id: str
    tier: str  # bronze, silver, gold, platinum
    resource_limits: Dict[str, Any]
    security_policies: List[str]
    compliance_requirements: List[str]
    cost_center: str
    contact_email: str
    environments: List[str]

@dataclass
class ResourceAllocation:
    cpu_limit: str
    memory_limit: str
    storage_limit: str
    network_bandwidth: str
    concurrent_builds: int
    retention_days: int

class EnterprisePlatformManager:
    def __init__(self):
        self.aws_client = boto3.client('sts')
        self.k8s_client = kubernetes.client.ApiClient()
        self.tenants = {}
        self.load_tenant_configurations()
    
    def load_tenant_configurations(self):
        """Load tenant configurations from central store"""
        try:
            # Load from S3 or database
            s3 = boto3.client('s3')
            response = s3.get_object(
                Bucket='enterprise-cicd-config',
                Key='tenants/configurations.json'
            )
            
            tenant_data = json.loads(response['Body'].read())
            
            for tenant_info in tenant_data['tenants']:
                tenant = TenantConfig(**tenant_info)
                self.tenants[tenant.tenant_id] = tenant
                
            logger.info(f"Loaded {len(self.tenants)} tenant configurations")
            
        except Exception as e:
            logger.error(f"Failed to load tenant configurations: {e}")
            raise
    
    def validate_tenant(self, tenant_id: str, environment: str) -> Dict[str, Any]:
        """Validate tenant and return configuration"""
        if tenant_id not in self.tenants:
            raise ValueError(f"Tenant {tenant_id} not found")
        
        tenant = self.tenants[tenant_id]
        
        if environment not in tenant.environments:
            raise ValueError(f"Environment {environment} not allowed for tenant {tenant_id}")
        
        # Get resource limits based on tier
        resource_limits = self.get_resource_limits(tenant.tier)
        
        return {
            'tenant_config': asdict(tenant),
            'resource_limits': asdict(resource_limits),
            'validation_timestamp': datetime.now().isoformat()
        }
    
    def get_resource_limits(self, tier: str) -> ResourceAllocation:
        """Get resource limits based on tenant tier"""
        tier_limits = {
            'bronze': ResourceAllocation(
                cpu_limit='2',
                memory_limit='4Gi',
                storage_limit='10Gi',
                network_bandwidth='100Mbps',
                concurrent_builds=2,
                retention_days=30
            ),
            'silver': ResourceAllocation(
                cpu_limit='4',
                memory_limit='8Gi',
                storage_limit='50Gi',
                network_bandwidth='500Mbps',
                concurrent_builds=5,
                retention_days=90
            ),
            'gold': ResourceAllocation(
                cpu_limit='8',
                memory_limit='16Gi',
                storage_limit='100Gi',
                network_bandwidth='1Gbps',
                concurrent_builds=10,
                retention_days=180
            ),
            'platinum': ResourceAllocation(
                cpu_limit='16',
                memory_limit='32Gi',
                storage_limit='500Gi',
                network_bandwidth='10Gbps',
                concurrent_builds=20,
                retention_days=365
            )
        }
        
        return tier_limits.get(tier, tier_limits['bronze'])
    
    def allocate_resources(self, tenant_id: str, limits: ResourceAllocation) -> bool:
        """Allocate resources for tenant"""
        try:
            # Create Kubernetes ResourceQuota
            quota_manifest = {
                'apiVersion': 'v1',
                'kind': 'ResourceQuota',
                'metadata': {
                    'name': f'tenant-{tenant_id}-quota',
                    'namespace': f'tenant-{tenant_id}'
                },
                'spec': {
                    'hard': {
                        'requests.cpu': limits.cpu_limit,
                        'requests.memory': limits.memory_limit,
                        'persistentvolumeclaims': '10',
                        'requests.storage': limits.storage_limit,
                        'count/jobs.batch': str(limits.concurrent_builds)
                    }
                }
            }
            
            # Apply quota
            kubernetes.utils.create_from_dict(
                self.k8s_client,
                quota_manifest
            )
            
            # Create NetworkPolicy for bandwidth limiting
            network_policy = {
                'apiVersion': 'networking.k8s.io/v1',
                'kind': 'NetworkPolicy',
                'metadata': {
                    'name': f'tenant-{tenant_id}-network-policy',
                    'namespace': f'tenant-{tenant_id}'
                },
                'spec': {
                    'podSelector': {},
                    'policyTypes': ['Ingress', 'Egress'],
                    'ingress': [{
                        'from': [{
                            'namespaceSelector': {
                                'matchLabels': {
                                    'tenant': tenant_id
                                }
                            }
                        }]
                    }],
                    'egress': [{
                        'to': []
                    }]
                }
            }
            
            kubernetes.utils.create_from_dict(
                self.k8s_client,
                network_policy
            )
            
            logger.info(f"Resources allocated for tenant {tenant_id}")
            return True
            
        except Exception as e:
            logger.error(f"Failed to allocate resources for tenant {tenant_id}: {e}")
            return False
    
    def monitor_tenant_usage(self, tenant_id: str) -> Dict[str, Any]:
        """Monitor resource usage for tenant"""
        try:
            # Get metrics from Kubernetes metrics server
            v1 = kubernetes.client.CoreV1Api()
            
            # Get pod metrics
            pods = v1.list_namespaced_pod(
                namespace=f'tenant-{tenant_id}'
            )
            
            total_cpu = 0
            total_memory = 0
            
            for pod in pods.items:
                if pod.status.phase == 'Running':
                    # Get resource usage (simplified)
                    for container in pod.spec.containers:
                        if container.resources.requests:
                            cpu_req = container.resources.requests.get('cpu', '0')
                            mem_req = container.resources.requests.get('memory', '0')
                            
                            # Convert to numeric values (simplified)
                            total_cpu += float(cpu_req.replace('m', '')) / 1000
                            total_memory += self._parse_memory(mem_req)
            
            # Get storage usage
            pvcs = v1.list_namespaced_persistent_volume_claim(
                namespace=f'tenant-{tenant_id}'
            )
            
            total_storage = sum(
                self._parse_storage(pvc.spec.resources.requests.get('storage', '0'))
                for pvc in pvcs.items
            )
            
            return {
                'tenant_id': tenant_id,
                'cpu_usage': total_cpu,
                'memory_usage': total_memory,
                'storage_usage': total_storage,
                'pod_count': len([p for p in pods.items if p.status.phase == 'Running']),
                'timestamp': datetime.now().isoformat()
            }
            
        except Exception as e:
            logger.error(f"Failed to get usage metrics for tenant {tenant_id}: {e}")
            return {}
    
    def _parse_memory(self, memory_str: str) -> float:
        """Parse memory string to MB"""
        if memory_str.endswith('Gi'):
            return float(memory_str[:-2]) * 1024
        elif memory_str.endswith('Mi'):
            return float(memory_str[:-2])
        elif memory_str.endswith('Ki'):
            return float(memory_str[:-2]) / 1024
        else:
            return float(memory_str) / (1024 * 1024)
    
    def _parse_storage(self, storage_str: str) -> float:
        """Parse storage string to GB"""
        if storage_str.endswith('Gi'):
            return float(storage_str[:-2])
        elif storage_str.endswith('Ti'):
            return float(storage_str[:-2]) * 1024
        elif storage_str.endswith('Mi'):
            return float(storage_str[:-2]) / 1024
        else:
            return float(storage_str) / (1024 * 1024 * 1024)
    
    def generate_tenant_report(self, tenant_id: str) -> Dict[str, Any]:
        """Generate comprehensive tenant report"""
        tenant = self.tenants.get(tenant_id)
        if not tenant:
            raise ValueError(f"Tenant {tenant_id} not found")
        
        usage_metrics = self.monitor_tenant_usage(tenant_id)
        resource_limits = self.get_resource_limits(tenant.tier)
        
        # Calculate utilization percentages
        cpu_utilization = (usage_metrics.get('cpu_usage', 0) / float(resource_limits.cpu_limit)) * 100
        memory_utilization = (usage_metrics.get('memory_usage', 0) / self._parse_memory(resource_limits.memory_limit)) * 100
        storage_utilization = (usage_metrics.get('storage_usage', 0) / self._parse_storage(resource_limits.storage_limit)) * 100
        
        return {
            'tenant_info': asdict(tenant),
            'resource_limits': asdict(resource_limits),
            'current_usage': usage_metrics,
            'utilization': {
                'cpu_percent': cpu_utilization,
                'memory_percent': memory_utilization,
                'storage_percent': storage_utilization
            },
            'recommendations': self._generate_recommendations(
                cpu_utilization, memory_utilization, storage_utilization
            ),
            'report_timestamp': datetime.now().isoformat()
        }
    
    def _generate_recommendations(self, cpu_util: float, memory_util: float, storage_util: float) -> List[str]:
        """Generate optimization recommendations"""
        recommendations = []
        
        if cpu_util > 80:
            recommendations.append("Consider upgrading to higher tier for more CPU resources")
        elif cpu_util < 20:
            recommendations.append("CPU resources are underutilized - consider downgrading tier")
        
        if memory_util > 80:
            recommendations.append("Memory usage is high - optimize applications or upgrade tier")
        elif memory_util < 20:
            recommendations.append("Memory resources are underutilized")
        
        if storage_util > 80:
            recommendations.append("Storage usage is high - implement cleanup policies or upgrade")
        
        if not recommendations:
            recommendations.append("Resource utilization is optimal")
        
        return recommendations

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Enterprise Platform Manager')
    parser.add_argument('--action', choices=['validate', 'allocate', 'monitor', 'report'], required=True)
    parser.add_argument('--tenant-id', required=True)
    parser.add_argument('--environment')
    parser.add_argument('--output-file', default='tenant-report.json')
    
    args = parser.parse_args()
    
    manager = EnterprisePlatformManager()
    
    try:
        if args.action == 'validate':
            if not args.environment:
                raise ValueError("Environment required for validation")
            
            result = manager.validate_tenant(args.tenant_id, args.environment)
            print(json.dumps(result, indent=2))
        
        elif args.action == 'allocate':
            tenant = manager.tenants.get(args.tenant_id)
            if not tenant:
                raise ValueError(f"Tenant {args.tenant_id} not found")
            
            limits = manager.get_resource_limits(tenant.tier)
            success = manager.allocate_resources(args.tenant_id, limits)
            print(f"Resource allocation {'successful' if success else 'failed'}")
        
        elif args.action == 'monitor':
            usage = manager.monitor_tenant_usage(args.tenant_id)
            print(json.dumps(usage, indent=2))
        
        elif args.action == 'report':
            report = manager.generate_tenant_report(args.tenant_id)
            
            with open(args.output_file, 'w') as f:
                json.dump(report, f, indent=2)
            
            print(f"Tenant report saved to {args.output_file}")
    
    except Exception as e:
        logger.error(f"Operation failed: {e}")
        exit(1)

if __name__ == "__main__":
    main()
```

---

## 5. Enterprise Governance and Compliance

### Governance Framework

```yaml
# .github/workflows/governance-compliance.yml
name: Enterprise Governance & Compliance

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  workflow_dispatch:
    inputs:
      compliance_framework:
        description: 'Compliance framework to check'
        required: true
        type: choice
        options:
          - 'all'
          - 'soc2'
          - 'pci-dss'
          - 'hipaa'
          - 'gdpr'

env:
  GOVERNANCE_REPO: 'company/governance-policies'
  COMPLIANCE_THRESHOLD: 95

jobs:
  policy-validation:
    runs-on: ubuntu-latest
    outputs:
      policy-violations: ${{ steps.validate.outputs.violations }}
      compliance-score: ${{ steps.validate.outputs.score }}
    steps:
      - name: Checkout governance policies
        uses: actions/checkout@v3
        with:
          repository: ${{ env.GOVERNANCE_REPO }}
          path: governance
      
      - name: Validate infrastructure policies
        id: validate
        run: |
          # Validate Terraform configurations against policies
          python scripts/policy-validator.py \
            --policies governance/policies \
            --terraform-configs infrastructure/ \
            --output-file policy-violations.json
      
      - name: Check code quality policies
        run: |
          # Validate code quality standards
          python scripts/code-quality-validator.py \
            --standards governance/code-standards \
            --source-code src/ \
            --output-file quality-violations.json
      
      - name: Validate security policies
        run: |
          # Check security policy compliance
          python scripts/security-policy-validator.py \
            --policies governance/security \
            --infrastructure infrastructure/ \
            --applications src/ \
            --output-file security-violations.json
  
  compliance-assessment:
    runs-on: ubuntu-latest
    needs: policy-validation
    if: ${{ github.event.inputs.compliance_framework != '' || github.event_name == 'schedule' }}
    steps:
      - name: SOC 2 Compliance Check
        if: ${{ github.event.inputs.compliance_framework == 'soc2' || github.event.inputs.compliance_framework == 'all' }}
        run: |
          # SOC 2 Type II compliance assessment
          python scripts/soc2-compliance.py \
            --audit-logs logs/ \
            --access-controls iam/ \
            --monitoring monitoring/ \
            --output-file soc2-compliance.json
      
      - name: PCI DSS Compliance Check
        if: ${{ github.event.inputs.compliance_framework == 'pci-dss' || github.event.inputs.compliance_framework == 'all' }}
        run: |
          # PCI DSS compliance assessment
          python scripts/pci-compliance.py \
            --network-configs network/ \
            --encryption-configs security/ \
            --access-logs logs/ \
            --output-file pci-compliance.json
      
      - name: HIPAA Compliance Check
        if: ${{ github.event.inputs.compliance_framework == 'hipaa' || github.event.inputs.compliance_framework == 'all' }}
        run: |
          # HIPAA compliance assessment
          python scripts/hipaa-compliance.py \
            --data-encryption security/ \
            --access-controls iam/ \
            --audit-trails logs/ \
            --output-file hipaa-compliance.json
      
      - name: GDPR Compliance Check
        if: ${{ github.event.inputs.compliance_framework == 'gdpr' || github.event.inputs.compliance_framework == 'all' }}
        run: |
          # GDPR compliance assessment
          python scripts/gdpr-compliance.py \
            --data-processing configs/ \
            --privacy-controls privacy/ \
            --consent-management consent/ \
            --output-file gdpr-compliance.json
  
  governance-reporting:
    runs-on: ubuntu-latest
    needs: [policy-validation, compliance-assessment]
    steps:
      - name: Generate governance dashboard
        run: |
          # Create comprehensive governance dashboard
          python scripts/governance-dashboard.py \
            --policy-violations policy-violations.json \
            --compliance-results compliance-*.json \
            --output-file governance-dashboard.json
      
      - name: Create compliance report
        run: |
          # Generate executive compliance report
          python scripts/compliance-report.py \
            --dashboard governance-dashboard.json \
            --template templates/compliance-report.html \
            --output-file compliance-report.html
      
      - name: Send governance alerts
        if: ${{ needs.policy-validation.outputs.compliance-score < env.COMPLIANCE_THRESHOLD }}
        run: |
          # Send alerts for compliance violations
          python scripts/governance-alerts.py \
            --violations policy-violations.json \
            --compliance-score "${{ needs.policy-validation.outputs.compliance-score }}" \
            --recipients compliance@company.com,ciso@company.com
```

### Enterprise Governance Manager

```python
#!/usr/bin/env python3
# scripts/governance-manager.py

import json
import yaml
import os
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
from datetime import datetime
import logging
from pathlib import Path
import re

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class PolicyViolation:
    policy_id: str
    policy_name: str
    resource_type: str
    resource_id: str
    violation_type: str
    severity: str
    description: str
    remediation: str
    compliance_framework: List[str]

@dataclass
class ComplianceControl:
    control_id: str
    control_name: str
    framework: str
    status: str
    evidence: List[str]
    gaps: List[str]
    remediation_plan: str
    owner: str
    due_date: str

class EnterpriseGovernanceManager:
    def __init__(self, policies_dir: str = 'governance/policies'):
        self.policies_dir = Path(policies_dir)
        self.policies = self._load_policies()
        self.compliance_frameworks = {
            'soc2': self._load_soc2_controls(),
            'pci-dss': self._load_pci_controls(),
            'hipaa': self._load_hipaa_controls(),
            'gdpr': self._load_gdpr_controls()
        }
    
    def _load_policies(self) -> Dict[str, Any]:
        """Load governance policies from files"""
        policies = {}
        
        if not self.policies_dir.exists():
            logger.warning(f"Policies directory {self.policies_dir} not found")
            return policies
        
        for policy_file in self.policies_dir.glob('*.yaml'):
            try:
                with open(policy_file, 'r') as f:
                    policy_data = yaml.safe_load(f)
                    policies[policy_file.stem] = policy_data
            except Exception as e:
                logger.error(f"Failed to load policy {policy_file}: {e}")
        
        return policies
    
    def _load_soc2_controls(self) -> List[ComplianceControl]:
        """Load SOC 2 compliance controls"""
        return [
            ComplianceControl(
                control_id='CC6.1',
                control_name='Logical and Physical Access Controls',
                framework='SOC2',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement MFA and access reviews',
                owner='security-team',
                due_date='2024-03-31'
            ),
            ComplianceControl(
                control_id='CC6.2',
                control_name='System Access Monitoring',
                framework='SOC2',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Deploy SIEM and log monitoring',
                owner='security-team',
                due_date='2024-03-31'
            ),
            ComplianceControl(
                control_id='CC7.1',
                control_name='System Monitoring',
                framework='SOC2',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement comprehensive monitoring',
                owner='ops-team',
                due_date='2024-03-31'
            )
        ]
    
    def _load_pci_controls(self) -> List[ComplianceControl]:
        """Load PCI DSS compliance controls"""
        return [
            ComplianceControl(
                control_id='PCI-1',
                control_name='Install and maintain firewall configuration',
                framework='PCI-DSS',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Configure network segmentation',
                owner='network-team',
                due_date='2024-03-31'
            ),
            ComplianceControl(
                control_id='PCI-3',
                control_name='Protect stored cardholder data',
                framework='PCI-DSS',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement data encryption',
                owner='security-team',
                due_date='2024-03-31'
            )
        ]
    
    def _load_hipaa_controls(self) -> List[ComplianceControl]:
        """Load HIPAA compliance controls"""
        return [
            ComplianceControl(
                control_id='HIPAA-164.308',
                control_name='Administrative Safeguards',
                framework='HIPAA',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement access management policies',
                owner='compliance-team',
                due_date='2024-03-31'
            ),
            ComplianceControl(
                control_id='HIPAA-164.312',
                control_name='Technical Safeguards',
                framework='HIPAA',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Deploy encryption and access controls',
                owner='security-team',
                due_date='2024-03-31'
            )
        ]
    
    def _load_gdpr_controls(self) -> List[ComplianceControl]:
        """Load GDPR compliance controls"""
        return [
            ComplianceControl(
                control_id='GDPR-25',
                control_name='Data Protection by Design',
                framework='GDPR',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement privacy by design principles',
                owner='privacy-team',
                due_date='2024-03-31'
            ),
            ComplianceControl(
                control_id='GDPR-32',
                control_name='Security of Processing',
                framework='GDPR',
                status='pending',
                evidence=[],
                gaps=[],
                remediation_plan='Implement data encryption and access controls',
                owner='security-team',
                due_date='2024-03-31'
            )
        ]
    
    def validate_terraform_policies(self, terraform_dir: str) -> List[PolicyViolation]:
        """Validate Terraform configurations against policies"""
        violations = []
        terraform_path = Path(terraform_dir)
        
        if not terraform_path.exists():
            logger.warning(f"Terraform directory {terraform_path} not found")
            return violations
        
        # Find all Terraform files
        tf_files = list(terraform_path.glob('**/*.tf'))
        
        for tf_file in tf_files:
            try:
                with open(tf_file, 'r') as f:
                    content = f.read()
                
                # Check for policy violations
                file_violations = self._check_terraform_policies(content, str(tf_file))
                violations.extend(file_violations)
            
            except Exception as e:
                logger.error(f"Failed to validate {tf_file}: {e}")
        
        return violations
    
    def _check_terraform_policies(self, content: str, file_path: str) -> List[PolicyViolation]:
        """Check Terraform content against policies"""
        violations = []
        
        # Check for unencrypted S3 buckets
        if 'resource "aws_s3_bucket"' in content:
            if 'server_side_encryption_configuration' not in content:
                violations.append(PolicyViolation(
                    policy_id='S3-001',
                    policy_name='S3 Bucket Encryption Required',
                    resource_type='aws_s3_bucket',
                    resource_id=file_path,
                    violation_type='missing_encryption',
                    severity='high',
                    description='S3 bucket must have server-side encryption enabled',
                    remediation='Add server_side_encryption_configuration block',
                    compliance_framework=['SOC2', 'PCI-DSS', 'HIPAA']
                ))
        
        # Check for public S3 buckets
        if 'aws_s3_bucket_public_access_block' not in content and 'aws_s3_bucket' in content:
            violations.append(PolicyViolation(
                policy_id='S3-002',
                policy_name='S3 Public Access Block Required',
                resource_type='aws_s3_bucket',
                resource_id=file_path,
                violation_type='missing_public_access_block',
                severity='critical',
                description='S3 bucket must have public access blocked',
                remediation='Add aws_s3_bucket_public_access_block resource',
                compliance_framework=['SOC2', 'PCI-DSS', 'GDPR']
            ))
        
        # Check for unencrypted RDS instances
        if 'resource "aws_db_instance"' in content:
            if 'storage_encrypted = true' not in content:
                violations.append(PolicyViolation(
                    policy_id='RDS-001',
                    policy_name='RDS Encryption Required',
                    resource_type='aws_db_instance',
                    resource_id=file_path,
                    violation_type='missing_encryption',
                    severity='high',
                    description='RDS instance must have storage encryption enabled',
                    remediation='Set storage_encrypted = true',
                    compliance_framework=['SOC2', 'PCI-DSS', 'HIPAA']
                ))
        
        # Check for EC2 instances without IMDSv2
        if 'resource "aws_instance"' in content:
            if 'metadata_options' not in content or 'http_tokens = "required"' not in content:
                violations.append(PolicyViolation(
                    policy_id='EC2-001',
                    policy_name='EC2 IMDSv2 Required',
                    resource_type='aws_instance',
                    resource_id=file_path,
                    violation_type='missing_imdsv2',
                    severity='medium',
                    description='EC2 instance must use IMDSv2',
                    remediation='Add metadata_options block with http_tokens = "required"',
                    compliance_framework=['SOC2']
                ))
        
        return violations
    
    def assess_compliance_framework(self, framework: str, evidence_dir: str) -> Dict[str, Any]:
        """Assess compliance for a specific framework"""
        if framework not in self.compliance_frameworks:
            raise ValueError(f"Unknown compliance framework: {framework}")
        
        controls = self.compliance_frameworks[framework].copy()
        evidence_path = Path(evidence_dir)
        
        # Assess each control
        for control in controls:
            control.evidence, control.gaps = self._assess_control(control, evidence_path)
            control.status = self._determine_control_status(control)
        
        # Calculate compliance score
        total_controls = len(controls)
        compliant_controls = sum(1 for c in controls if c.status == 'compliant')
        compliance_score = (compliant_controls / total_controls * 100) if total_controls > 0 else 0
        
        return {
            'framework': framework,
            'assessment_date': datetime.now().isoformat(),
            'compliance_score': compliance_score,
            'total_controls': total_controls,
            'compliant_controls': compliant_controls,
            'non_compliant_controls': total_controls - compliant_controls,
            'controls': [asdict(control) for control in controls],
            'recommendations': self._generate_compliance_recommendations(controls)
        }
    
    def _assess_control(self, control: ComplianceControl, evidence_path: Path) -> tuple:
        """Assess a specific compliance control"""
        evidence = []
        gaps = []
        
        # Look for evidence files
        control_evidence_dir = evidence_path / control.framework.lower() / control.control_id
        if control_evidence_dir.exists():
            evidence_files = list(control_evidence_dir.glob('*'))
            evidence = [str(f) for f in evidence_files]
        
        # Determine gaps based on control requirements
        if control.framework == 'SOC2':
            gaps = self._assess_soc2_control(control, evidence)
        elif control.framework == 'PCI-DSS':
            gaps = self._assess_pci_control(control, evidence)
        elif control.framework == 'HIPAA':
            gaps = self._assess_hipaa_control(control, evidence)
        elif control.framework == 'GDPR':
            gaps = self._assess_gdpr_control(control, evidence)
        
        return evidence, gaps
    
    def _assess_soc2_control(self, control: ComplianceControl, evidence: List[str]) -> List[str]:
        """Assess SOC 2 specific control"""
        gaps = []
        
        if control.control_id == 'CC6.1':
            if not any('access-policy' in e for e in evidence):
                gaps.append('Missing access control policy documentation')
            if not any('mfa-config' in e for e in evidence):
                gaps.append('Missing MFA configuration evidence')
        
        elif control.control_id == 'CC6.2':
            if not any('access-logs' in e for e in evidence):
                gaps.append('Missing access monitoring logs')
            if not any('siem-config' in e for e in evidence):
                gaps.append('Missing SIEM configuration')
        
        elif control.control_id == 'CC7.1':
            if not any('monitoring-config' in e for e in evidence):
                gaps.append('Missing system monitoring configuration')
            if not any('alerting-rules' in e for e in evidence):
                gaps.append('Missing alerting rules documentation')
        
        return gaps
    
    def _assess_pci_control(self, control: ComplianceControl, evidence: List[str]) -> List[str]:
        """Assess PCI DSS specific control"""
        gaps = []
        
        if control.control_id == 'PCI-1':
            if not any('firewall-config' in e for e in evidence):
                gaps.append('Missing firewall configuration documentation')
            if not any('network-diagram' in e for e in evidence):
                gaps.append('Missing network segmentation diagram')
        
        elif control.control_id == 'PCI-3':
            if not any('encryption-config' in e for e in evidence):
                gaps.append('Missing data encryption configuration')
            if not any('key-management' in e for e in evidence):
                gaps.append('Missing key management procedures')
        
        return gaps
    
    def _assess_hipaa_control(self, control: ComplianceControl, evidence: List[str]) -> List[str]:
        """Assess HIPAA specific control"""
        gaps = []
        
        if control.control_id == 'HIPAA-164.308':
            if not any('admin-safeguards' in e for e in evidence):
                gaps.append('Missing administrative safeguards documentation')
            if not any('workforce-training' in e for e in evidence):
                gaps.append('Missing workforce training records')
        
        elif control.control_id == 'HIPAA-164.312':
            if not any('technical-safeguards' in e for e in evidence):
                gaps.append('Missing technical safeguards implementation')
            if not any('audit-controls' in e for e in evidence):
                gaps.append('Missing audit controls configuration')
        
        return gaps
    
    def _assess_gdpr_control(self, control: ComplianceControl, evidence: List[str]) -> List[str]:
        """Assess GDPR specific control"""
        gaps = []
        
        if control.control_id == 'GDPR-25':
            if not any('privacy-design' in e for e in evidence):
                gaps.append('Missing privacy by design documentation')
            if not any('dpia' in e for e in evidence):
                gaps.append('Missing Data Protection Impact Assessment')
        
        elif control.control_id == 'GDPR-32':
            if not any('security-measures' in e for e in evidence):
                gaps.append('Missing security measures documentation')
            if not any('breach-procedures' in e for e in evidence):
                gaps.append('Missing data breach procedures')
        
        return gaps
    
    def _determine_control_status(self, control: ComplianceControl) -> str:
        """Determine the status of a compliance control"""
        if not control.gaps:
            return 'compliant'
        elif len(control.gaps) <= 1:
            return 'partially_compliant'
        else:
            return 'non_compliant'
    
    def _generate_compliance_recommendations(self, controls: List[ComplianceControl]) -> List[str]:
        """Generate compliance recommendations"""
        recommendations = []
        
        non_compliant = [c for c in controls if c.status == 'non_compliant']
        if non_compliant:
            recommendations.append(f"Address {len(non_compliant)} non-compliant controls immediately")
        
        partially_compliant = [c for c in controls if c.status == 'partially_compliant']
        if partially_compliant:
            recommendations.append(f"Complete remediation for {len(partially_compliant)} partially compliant controls")
        
        # Identify common gaps
        all_gaps = [gap for control in controls for gap in control.gaps]
        if 'Missing encryption' in str(all_gaps):
            recommendations.append('Implement comprehensive encryption strategy')
        
        if 'Missing monitoring' in str(all_gaps):
            recommendations.append('Deploy comprehensive monitoring and logging solution')
        
        if 'Missing documentation' in str(all_gaps):
            recommendations.append('Complete policy and procedure documentation')
        
        return recommendations
    
    def generate_governance_dashboard(self, violations: List[PolicyViolation], 
                                    compliance_results: Dict[str, Any]) -> Dict[str, Any]:
        """Generate comprehensive governance dashboard"""
        # Analyze violations
        total_violations = len(violations)
        critical_violations = len([v for v in violations if v.severity == 'critical'])
        high_violations = len([v for v in violations if v.severity == 'high'])
        
        # Group violations by type
        violations_by_type = {}
        for violation in violations:
            if violation.resource_type not in violations_by_type:
                violations_by_type[violation.resource_type] = []
            violations_by_type[violation.resource_type].append(violation)
        
        # Calculate overall compliance score
        framework_scores = []
        for framework, results in compliance_results.items():
            if isinstance(results, dict) and 'compliance_score' in results:
                framework_scores.append(results['compliance_score'])
        
        overall_compliance = sum(framework_scores) / len(framework_scores) if framework_scores else 0
        
        return {
            'dashboard_timestamp': datetime.now().isoformat(),
            'governance_summary': {
                'total_violations': total_violations,
                'critical_violations': critical_violations,
                'high_violations': high_violations,
                'overall_compliance_score': overall_compliance
            },
            'violations_by_type': {
                resource_type: {
                    'count': len(resource_violations),
                    'critical': len([v for v in resource_violations if v.severity == 'critical']),
                    'high': len([v for v in resource_violations if v.severity == 'high']),
                    'violations': [asdict(v) for v in resource_violations]
                }
                for resource_type, resource_violations in violations_by_type.items()
            },
            'compliance_by_framework': compliance_results,
            'top_violations': [asdict(v) for v in sorted(violations, key=lambda x: {'critical': 3, 'high': 2, 'medium': 1, 'low': 0}[x.severity], reverse=True)[:10]],
            'remediation_priorities': self._generate_remediation_priorities(violations, compliance_results)
        }
    
    def _generate_remediation_priorities(self, violations: List[PolicyViolation], 
                                       compliance_results: Dict[str, Any]) -> List[str]:
        """Generate prioritized remediation recommendations"""
        priorities = []
        
        # Critical violations first
        critical_violations = [v for v in violations if v.severity == 'critical']
        if critical_violations:
            priorities.append(f"URGENT: Address {len(critical_violations)} critical security violations")
        
        # High violations
        high_violations = [v for v in violations if v.severity == 'high']
        if high_violations:
            priorities.append(f"HIGH: Remediate {len(high_violations)} high-severity violations")
        
        # Compliance gaps
        for framework, results in compliance_results.items():
            if isinstance(results, dict) and results.get('compliance_score', 100) < 80:
                priorities.append(f"COMPLIANCE: Improve {framework.upper()} compliance score ({results['compliance_score']:.1f}%)")
        
        # Common patterns
        encryption_violations = [v for v in violations if 'encryption' in v.violation_type]
        if len(encryption_violations) > 3:
            priorities.append("PATTERN: Implement organization-wide encryption standards")
        
        access_violations = [v for v in violations if 'access' in v.violation_type]
        if len(access_violations) > 3:
            priorities.append("PATTERN: Review and strengthen access control policies")
        
        return priorities

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Enterprise Governance Manager')
    parser.add_argument('--action', choices=['validate', 'assess', 'dashboard'], required=True)
    parser.add_argument('--terraform-dir', default='infrastructure/')
    parser.add_argument('--evidence-dir', default='compliance-evidence/')
    parser.add_argument('--framework', choices=['soc2', 'pci-dss', 'hipaa', 'gdpr'])
    parser.add_argument('--output-file', default='governance-report.json')
    
    args = parser.parse_args()
    
    manager = EnterpriseGovernanceManager()
    
    try:
        if args.action == 'validate':
            violations = manager.validate_terraform_policies(args.terraform_dir)
            result = {
                'validation_timestamp': datetime.now().isoformat(),
                'total_violations': len(violations),
                'violations': [asdict(v) for v in violations]
            }
        
        elif args.action == 'assess':
            if not args.framework:
                raise ValueError("Framework required for assessment")
            result = manager.assess_compliance_framework(args.framework, args.evidence_dir)
        
        elif args.action == 'dashboard':
            violations = manager.validate_terraform_policies(args.terraform_dir)
            compliance_results = {}
            
            # Assess all frameworks
            for framework in ['soc2', 'pci-dss', 'hipaa', 'gdpr']:
                try:
                    compliance_results[framework] = manager.assess_compliance_framework(framework, args.evidence_dir)
                except Exception as e:
                    logger.warning(f"Failed to assess {framework}: {e}")
            
            result = manager.generate_governance_dashboard(violations, compliance_results)
        
        with open(args.output_file, 'w') as f:
            json.dump(result, f, indent=2, default=str)
        
        print(f"Governance analysis complete. Results saved to {args.output_file}")
        
        if args.action == 'dashboard':
            print(f"\nGovernance Summary:")
            print(f"Total violations: {result['governance_summary']['total_violations']}")
            print(f"Critical violations: {result['governance_summary']['critical_violations']}")
            print(f"Overall compliance: {result['governance_summary']['overall_compliance_score']:.1f}%")
    
    except Exception as e:
        logger.error(f"Governance analysis failed: {e}")
        exit(1)

if __name__ == "__main__":
    main()
```

---

## Hands-on Exercises

### Exercise 1: Multi-Tenant Platform Setup
**Objective**: Set up a multi-tenant CI/CD platform with proper isolation

**Tasks**:
1. Create tenant-specific GitHub Actions workflows
2. Implement resource isolation using namespaces
3. Set up tenant-specific monitoring and alerting
4. Configure cost allocation and tracking

**Deliverables**:
- Multi-tenant workflow configurations
- Namespace isolation setup
- Monitoring dashboard per tenant
- Cost allocation report

### Exercise 2: Blue-Green Deployment Implementation
**Objective**: Implement automated blue-green deployment with traffic management

**Tasks**:
1. Set up blue-green infrastructure using Terraform
2. Create automated deployment pipeline
3. Implement health checks and rollback mechanisms
4. Configure traffic routing and monitoring

**Deliverables**:
- Blue-green infrastructure code
- Automated deployment workflow
- Health check implementation
- Traffic management configuration

### Exercise 3: Disaster Recovery Testing
**Objective**: Implement and test comprehensive disaster recovery procedures

**Tasks**:
1. Create disaster recovery plans for critical services
2. Implement automated failover mechanisms
3. Set up cross-region replication
4. Conduct disaster recovery drills

**Deliverables**:
- Disaster recovery documentation
- Automated failover scripts
- Cross-region setup
- DR test results and improvements

### Exercise 4: Cost Optimization Implementation
**Objective**: Implement enterprise-wide cost optimization strategies

**Tasks**:
1. Set up cost monitoring and alerting
2. Implement automated resource cleanup
3. Create cost optimization recommendations
4. Establish FinOps governance processes

**Deliverables**:
- Cost monitoring dashboard
- Automated cleanup scripts
- Optimization recommendations report
- FinOps governance framework

### Exercise 5: Compliance Automation
**Objective**: Automate compliance checking and reporting

**Tasks**:
1. Implement policy-as-code for infrastructure
2. Set up automated compliance scanning
3. Create compliance reporting dashboard
4. Establish governance workflows

**Deliverables**:
- Policy-as-code implementation
- Compliance scanning automation
- Governance dashboard
- Compliance reports

---

## Troubleshooting Guide

### Multi-Tenant Issues

**Problem**: Tenant isolation failures
- **Symptoms**: Cross-tenant resource access, shared secrets
- **Diagnosis**: Check namespace configurations, RBAC policies
- **Solution**: Implement proper network policies, separate secret stores

**Problem**: Resource contention between tenants
- **Symptoms**: Performance degradation, resource limits exceeded
- **Diagnosis**: Monitor resource usage per tenant
- **Solution**: Implement resource quotas, auto-scaling policies

### Deployment Pattern Issues

**Problem**: Blue-green deployment failures
- **Symptoms**: Traffic not switching, health checks failing
- **Diagnosis**: Check load balancer configuration, health endpoints
- **Solution**: Verify health check endpoints, adjust timeout settings

**Problem**: Canary deployment stuck
- **Symptoms**: Traffic not progressing, metrics not updating
- **Diagnosis**: Check metrics collection, promotion criteria
- **Solution**: Verify metrics pipeline, adjust promotion thresholds

### Disaster Recovery Issues

**Problem**: Failover not triggering
- **Symptoms**: Services down but no automatic failover
- **Diagnosis**: Check health monitoring, failover triggers
- **Solution**: Verify monitoring thresholds, test failover mechanisms

**Problem**: Data inconsistency after recovery
- **Symptoms**: Data loss, inconsistent state
- **Diagnosis**: Check replication lag, backup integrity
- **Solution**: Implement proper data synchronization, verify backups

### Cost Optimization Issues

**Problem**: Cost alerts not firing
- **Symptoms**: Unexpected high costs, no notifications
- **Diagnosis**: Check alert thresholds, notification channels
- **Solution**: Adjust thresholds, verify notification setup

**Problem**: Resource cleanup not working
- **Symptoms**: Unused resources accumulating costs
- **Diagnosis**: Check cleanup scripts, resource tagging
- **Solution**: Verify cleanup logic, implement proper tagging

### Governance Issues

**Problem**: Policy violations not detected
- **Symptoms**: Non-compliant resources deployed
- **Diagnosis**: Check policy enforcement, scanning frequency
- **Solution**: Verify policy rules, increase scanning frequency

**Problem**: Compliance reports incomplete
- **Symptoms**: Missing data, outdated information
- **Diagnosis**: Check data collection, report generation
- **Solution**: Verify data sources, update collection scripts

---

## Summary

Module 20 covered advanced enterprise patterns for CI/CD at scale:

### Key Takeaways

1. **Enterprise Architecture**
   - Multi-tenant platform design
   - Scalable infrastructure patterns
   - Service mesh integration
   - Platform engineering principles

2. **Advanced Deployment Patterns**
   - Blue-green deployments
   - Canary releases with automated promotion
   - Feature flags integration
   - Progressive delivery strategies

3. **Disaster Recovery & Business Continuity**
   - Multi-region failover strategies
   - Automated disaster detection
   - Recovery plan execution
   - Business continuity reporting

4. **Cost Optimization & FinOps**
   - Enterprise cost analysis
   - Resource optimization strategies
   - Automated cost management
   - FinOps governance frameworks

5. **Enterprise Governance**
   - Policy-as-code implementation
   - Compliance automation
   - Multi-framework support
   - Governance dashboards

### Enterprise Readiness Checklist

- [ ] Multi-tenant platform implemented
- [ ] Advanced deployment patterns configured
- [ ] Disaster recovery plans tested
- [ ] Cost optimization automated
- [ ] Governance policies enforced
- [ ] Compliance reporting automated
- [ ] Monitoring and alerting comprehensive
- [ ] Documentation and runbooks complete

### Next Steps

1. **Platform Maturity Assessment**
   - Evaluate current enterprise readiness
   - Identify gaps and improvement areas
   - Create implementation roadmap

2. **Continuous Improvement**
   - Regular architecture reviews
   - Performance optimization cycles
   - Security and compliance updates
   - Cost optimization iterations

3. **Team Development**
   - Platform engineering training
   - DevOps culture advancement
   - Cross-functional collaboration
   - Knowledge sharing programs

Congratulations! You've completed the Enterprise Patterns module and are now equipped with advanced CI/CD patterns for large-scale enterprise environments.

---

## 2. Advanced Deployment Patterns

### Blue-Green Deployment with Traffic Management

```yaml
# .github/workflows/blue-green-enterprise.yml
name: Enterprise Blue-Green Deployment

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      traffic_percentage:
        description: 'Percentage of traffic to route to new version'
        required: false
        default: '10'
      rollback:
        description: 'Rollback to previous version'
        required: false
        default: 'false'

env:
  REGISTRY: ${{ vars.ENTERPRISE_REGISTRY }}
  SERVICE_NAME: ${{ vars.SERVICE_NAME }}
  NAMESPACE: ${{ vars.NAMESPACE }}

jobs:
  determine-deployment-slot:
    runs-on: ubuntu-latest
    outputs:
      current-slot: ${{ steps.slot.outputs.current }}
      target-slot: ${{ steps.slot.outputs.target }}
      previous-version: ${{ steps.slot.outputs.previous }}
    steps:
      - name: Determine deployment slot
        id: slot
        run: |
          # Get current active slot
          current=$(kubectl get service ${{ env.SERVICE_NAME }} \
            -n ${{ env.NAMESPACE }} \
            -o jsonpath='{.spec.selector.slot}')
          
          if [ "$current" = "blue" ]; then
            target="green"
          else
            target="blue"
          fi
          
          # Get previous version
          previous=$(kubectl get deployment ${{ env.SERVICE_NAME }}-$current \
            -n ${{ env.NAMESPACE }} \
            -o jsonpath='{.metadata.labels.version}' || echo "none")
          
          echo "current=$current" >> $GITHUB_OUTPUT
          echo "target=$target" >> $GITHUB_OUTPUT
          echo "previous=$previous" >> $GITHUB_OUTPUT
  
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build application
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }} .
          docker push ${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }}
      
      - name: Run comprehensive tests
        run: |
          # Run unit tests
          npm test
          
          # Run integration tests
          npm run test:integration
          
          # Run security tests
          npm run test:security
          
          # Run performance tests
          npm run test:performance
  
  deploy-to-target-slot:
    runs-on: ubuntu-latest
    needs: [determine-deployment-slot, build-and-test]
    if: ${{ github.event.inputs.rollback != 'true' }}
    steps:
      - name: Deploy to target slot
        run: |
          # Deploy to target slot
          kubectl set image deployment/${{ env.SERVICE_NAME }}-${{ needs.determine-deployment-slot.outputs.target-slot }} \
            app=${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }} \
            -n ${{ env.NAMESPACE }}
          
          # Wait for rollout to complete
          kubectl rollout status deployment/${{ env.SERVICE_NAME }}-${{ needs.determine-deployment-slot.outputs.target-slot }} \
            -n ${{ env.NAMESPACE }} \
            --timeout=600s
      
      - name: Health check target slot
        run: |
          # Wait for pods to be ready
          kubectl wait --for=condition=ready pod \
            -l app=${{ env.SERVICE_NAME }},slot=${{ needs.determine-deployment-slot.outputs.target-slot }} \
            -n ${{ env.NAMESPACE }} \
            --timeout=300s
          
          # Run health checks
          python scripts/health-check.py \
            --service ${{ env.SERVICE_NAME }} \
            --slot ${{ needs.determine-deployment-slot.outputs.target-slot }} \
            --namespace ${{ env.NAMESPACE }}
  
  canary-traffic-routing:
    runs-on: ubuntu-latest
    needs: [determine-deployment-slot, deploy-to-target-slot]
    if: ${{ github.event.inputs.rollback != 'true' }}
    steps:
      - name: Configure canary traffic
        run: |
          # Update Istio VirtualService for canary routing
          python scripts/configure-traffic.py \
            --service ${{ env.SERVICE_NAME }} \
            --current-slot ${{ needs.determine-deployment-slot.outputs.current-slot }} \
            --target-slot ${{ needs.determine-deployment-slot.outputs.target-slot }} \
            --traffic-percentage ${{ github.event.inputs.traffic_percentage || '10' }} \
            --namespace ${{ env.NAMESPACE }}
      
      - name: Monitor canary metrics
        run: |
          # Monitor error rates, latency, and other metrics
          python scripts/monitor-canary.py \
            --service ${{ env.SERVICE_NAME }} \
            --duration 300 \
            --error-threshold 1.0 \
            --latency-threshold 500
  
  automated-promotion:
    runs-on: ubuntu-latest
    needs: [determine-deployment-slot, canary-traffic-routing]
    if: ${{ github.event.inputs.rollback != 'true' }}
    steps:
      - name: Evaluate promotion criteria
        id: evaluate
        run: |
          # Check metrics and decide on promotion
          result=$(python scripts/evaluate-promotion.py \
            --service ${{ env.SERVICE_NAME }} \
            --target-slot ${{ needs.determine-deployment-slot.outputs.target-slot }})
          
          echo "promote=$result" >> $GITHUB_OUTPUT
      
      - name: Promote to production
        if: steps.evaluate.outputs.promote == 'true'
        run: |
          # Switch all traffic to new version
          python scripts/configure-traffic.py \
            --service ${{ env.SERVICE_NAME }} \
            --current-slot ${{ needs.determine-deployment-slot.outputs.current-slot }} \
            --target-slot ${{ needs.determine-deployment-slot.outputs.target-slot }} \
            --traffic-percentage 100 \
            --namespace ${{ env.NAMESPACE }}
          
          # Update service selector
          kubectl patch service ${{ env.SERVICE_NAME }} \
            -n ${{ env.NAMESPACE }} \
            -p '{"spec":{"selector":{"slot":"${{ needs.determine-deployment-slot.outputs.target-slot }}"}}}'
      
      - name: Cleanup old version
        if: steps.evaluate.outputs.promote == 'true'
        run: |
          # Scale down old version
          kubectl scale deployment ${{ env.SERVICE_NAME }}-${{ needs.determine-deployment-slot.outputs.current-slot }} \
            --replicas=0 \
            -n ${{ env.NAMESPACE }}
  
  rollback-deployment:
    runs-on: ubuntu-latest
    needs: [determine-deployment-slot]
    if: ${{ github.event.inputs.rollback == 'true' }}
    steps:
      - name: Rollback to previous version
        run: |
          # Switch traffic back to previous slot
          previous_slot=${{ needs.determine-deployment-slot.outputs.current-slot }}
          
          # Scale up previous version if needed
          kubectl scale deployment ${{ env.SERVICE_NAME }}-$previous_slot \
            --replicas=3 \
            -n ${{ env.NAMESPACE }}
          
          # Wait for rollout
          kubectl rollout status deployment/${{ env.SERVICE_NAME }}-$previous_slot \
            -n ${{ env.NAMESPACE }}
          
          # Switch traffic
          kubectl patch service ${{ env.SERVICE_NAME }} \
            -n ${{ env.NAMESPACE }} \
            -p '{"spec":{"selector":{"slot":"'$previous_slot'"}}}'
          
          echo "Rollback completed to slot: $previous_slot"
```

### Canary Deployment with Automated Promotion

```python
# scripts/canary-deployment-manager.py
import json
import time
import requests
import subprocess
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
import logging
from datetime import datetime, timedelta

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class CanaryMetrics:
    error_rate: float
    latency_p95: float
    latency_p99: float
    throughput: float
    cpu_usage: float
    memory_usage: float
    timestamp: datetime

@dataclass
class PromotionCriteria:
    max_error_rate: float = 1.0  # 1%
    max_latency_p95: float = 500  # 500ms
    max_latency_p99: float = 1000  # 1000ms
    min_throughput_ratio: float = 0.95  # 95% of baseline
    max_cpu_usage: float = 80.0  # 80%
    max_memory_usage: float = 80.0  # 80%
    observation_window: int = 300  # 5 minutes
    min_sample_size: int = 100  # minimum requests

class CanaryDeploymentManager:
    def __init__(self, service_name: str, namespace: str):
        self.service_name = service_name
        self.namespace = namespace
        self.prometheus_url = "http://prometheus.monitoring.svc.cluster.local:9090"
        self.grafana_url = "http://grafana.monitoring.svc.cluster.local:3000"
        
    def get_canary_metrics(self, canary_version: str, duration_minutes: int = 5) -> CanaryMetrics:
        """Get metrics for canary deployment"""
        end_time = datetime.now()
        start_time = end_time - timedelta(minutes=duration_minutes)
        
        # Query Prometheus for metrics
        queries = {
            'error_rate': f'rate(http_requests_total{{service="{self.service_name}",version="{canary_version}",status=~"5.."}}}[{duration_minutes}m]) / rate(http_requests_total{{service="{self.service_name}",version="{canary_version}"}}[{duration_minutes}m]) * 100',
            'latency_p95': f'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{{service="{self.service_name}",version="{canary_version}"}}[{duration_minutes}m])) * 1000',
            'latency_p99': f'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{{service="{self.service_name}",version="{canary_version}"}}[{duration_minutes}m])) * 1000',
            'throughput': f'rate(http_requests_total{{service="{self.service_name}",version="{canary_version}"}}[{duration_minutes}m])',
            'cpu_usage': f'avg(rate(container_cpu_usage_seconds_total{{pod=~"{self.service_name}-.*",container!="POD",container!=""}}[{duration_minutes}m])) * 100',
            'memory_usage': f'avg(container_memory_working_set_bytes{{pod=~"{self.service_name}-.*",container!="POD",container!=""}}) / avg(container_spec_memory_limit_bytes{{pod=~"{self.service_name}-.*",container!="POD",container!=""}}) * 100'
        }
        
        metrics = {}
        
        for metric_name, query in queries.items():
            try:
                response = requests.get(
                    f"{self.prometheus_url}/api/v1/query",
                    params={
                        'query': query,
                        'time': end_time.timestamp()
                    },
                    timeout=30
                )
                
                if response.status_code == 200:
                    data = response.json()
                    if data['data']['result']:
                        value = float(data['data']['result'][0]['value'][1])
                        metrics[metric_name] = value
                    else:
                        metrics[metric_name] = 0.0
                else:
                    logger.warning(f"Failed to get {metric_name}: {response.status_code}")
                    metrics[metric_name] = 0.0
                    
            except Exception as e:
                logger.error(f"Error getting {metric_name}: {e}")
                metrics[metric_name] = 0.0
        
        return CanaryMetrics(
            error_rate=metrics.get('error_rate', 0.0),
            latency_p95=metrics.get('latency_p95', 0.0),
            latency_p99=metrics.get('latency_p99', 0.0),
            throughput=metrics.get('throughput', 0.0),
            cpu_usage=metrics.get('cpu_usage', 0.0),
            memory_usage=metrics.get('memory_usage', 0.0),
            timestamp=datetime.now()
        )
    
    def get_baseline_metrics(self, stable_version: str, duration_minutes: int = 5) -> CanaryMetrics:
        """Get baseline metrics from stable version"""
        # Similar to get_canary_metrics but for stable version
        return self.get_canary_metrics(stable_version, duration_minutes)
    
    def evaluate_promotion(self, canary_metrics: CanaryMetrics, 
                          baseline_metrics: CanaryMetrics,
                          criteria: PromotionCriteria) -> Dict[str, Any]:
        """Evaluate if canary should be promoted"""
        checks = []
        
        # Error rate check
        error_rate_ok = canary_metrics.error_rate <= criteria.max_error_rate
        checks.append({
            'name': 'error_rate',
            'passed': error_rate_ok,
            'value': canary_metrics.error_rate,
            'threshold': criteria.max_error_rate,
            'message': f"Error rate: {canary_metrics.error_rate:.2f}% (max: {criteria.max_error_rate}%)"
        })
        
        # Latency checks
        latency_p95_ok = canary_metrics.latency_p95 <= criteria.max_latency_p95
        checks.append({
            'name': 'latency_p95',
            'passed': latency_p95_ok,
            'value': canary_metrics.latency_p95,
            'threshold': criteria.max_latency_p95,
            'message': f"P95 latency: {canary_metrics.latency_p95:.2f}ms (max: {criteria.max_latency_p95}ms)"
        })
        
        latency_p99_ok = canary_metrics.latency_p99 <= criteria.max_latency_p99
        checks.append({
            'name': 'latency_p99',
            'passed': latency_p99_ok,
            'value': canary_metrics.latency_p99,
            'threshold': criteria.max_latency_p99,
            'message': f"P99 latency: {canary_metrics.latency_p99:.2f}ms (max: {criteria.max_latency_p99}ms)"
        })
        
        # Throughput check (compared to baseline)
        throughput_ratio = canary_metrics.throughput / baseline_metrics.throughput if baseline_metrics.throughput > 0 else 1.0
        throughput_ok = throughput_ratio >= criteria.min_throughput_ratio
        checks.append({
            'name': 'throughput',
            'passed': throughput_ok,
            'value': throughput_ratio,
            'threshold': criteria.min_throughput_ratio,
            'message': f"Throughput ratio: {throughput_ratio:.2f} (min: {criteria.min_throughput_ratio})"
        })
        
        # Resource usage checks
        cpu_ok = canary_metrics.cpu_usage <= criteria.max_cpu_usage
        checks.append({
            'name': 'cpu_usage',
            'passed': cpu_ok,
            'value': canary_metrics.cpu_usage,
            'threshold': criteria.max_cpu_usage,
            'message': f"CPU usage: {canary_metrics.cpu_usage:.2f}% (max: {criteria.max_cpu_usage}%)"
        })
        
        memory_ok = canary_metrics.memory_usage <= criteria.max_memory_usage
        checks.append({
            'name': 'memory_usage',
            'passed': memory_ok,
            'value': canary_metrics.memory_usage,
            'threshold': criteria.max_memory_usage,
            'message': f"Memory usage: {canary_metrics.memory_usage:.2f}% (max: {criteria.max_memory_usage}%)"
        })
        
        # Overall decision
        all_passed = all(check['passed'] for check in checks)
        
        return {
            'promote': all_passed,
            'checks': checks,
            'canary_metrics': asdict(canary_metrics),
            'baseline_metrics': asdict(baseline_metrics),
            'evaluation_time': datetime.now().isoformat(),
            'summary': {
                'total_checks': len(checks),
                'passed_checks': sum(1 for check in checks if check['passed']),
                'failed_checks': sum(1 for check in checks if not check['passed'])
            }
        }
    
    def configure_traffic_split(self, canary_version: str, stable_version: str, 
                               canary_percentage: int) -> bool:
        """Configure traffic split between canary and stable versions"""
        try:
            # Create Istio VirtualService for traffic splitting
            virtual_service = {
                'apiVersion': 'networking.istio.io/v1beta1',
                'kind': 'VirtualService',
                'metadata': {
                    'name': f'{self.service_name}-canary',
                    'namespace': self.namespace
                },
                'spec': {
                    'hosts': [self.service_name],
                    'http': [{
                        'match': [{'headers': {'canary': {'exact': 'true'}}}],
                        'route': [{
                            'destination': {
                                'host': self.service_name,
                                'subset': 'canary'
                            }
                        }]
                    }, {
                        'route': [
                            {
                                'destination': {
                                    'host': self.service_name,
                                    'subset': 'canary'
                                },
                                'weight': canary_percentage
                            },
                            {
                                'destination': {
                                    'host': self.service_name,
                                    'subset': 'stable'
                                },
                                'weight': 100 - canary_percentage
                            }
                        ]
                    }]
                }
            }
            
            # Apply VirtualService
            with open('/tmp/virtual-service.yaml', 'w') as f:
                yaml.dump(virtual_service, f)
            
            result = subprocess.run([
                'kubectl', 'apply', '-f', '/tmp/virtual-service.yaml'
            ], capture_output=True, text=True)
            
            if result.returncode == 0:
                logger.info(f"Traffic split configured: {canary_percentage}% to canary")
                return True
            else:
                logger.error(f"Failed to configure traffic split: {result.stderr}")
                return False
                
        except Exception as e:
            logger.error(f"Error configuring traffic split: {e}")
            return False
    
    def promote_canary(self, canary_version: str) -> bool:
        """Promote canary to stable"""
        try:
            # Update deployment labels
            result = subprocess.run([
                'kubectl', 'patch', 'deployment', self.service_name,
                '-n', self.namespace,
                '-p', f'{{"metadata":{{"labels":{{"version":"{canary_version}"}}}}}}'
            ], capture_output=True, text=True)
            
            if result.returncode == 0:
                # Remove canary traffic splitting
                subprocess.run([
                    'kubectl', 'delete', 'virtualservice', f'{self.service_name}-canary',
                    '-n', self.namespace
                ], capture_output=True)
                
                logger.info(f"Canary {canary_version} promoted to stable")
                return True
            else:
                logger.error(f"Failed to promote canary: {result.stderr}")
                return False
                
        except Exception as e:
            logger.error(f"Error promoting canary: {e}")
            return False
    
    def rollback_canary(self, stable_version: str) -> bool:
        """Rollback canary deployment"""
        try:
            # Remove traffic splitting
            subprocess.run([
                'kubectl', 'delete', 'virtualservice', f'{self.service_name}-canary',
                '-n', self.namespace
            ], capture_output=True)
            
            # Scale down canary deployment
            result = subprocess.run([
                'kubectl', 'scale', 'deployment', f'{self.service_name}-canary',
                '--replicas=0',
                '-n', self.namespace
            ], capture_output=True, text=True)
            
            if result.returncode == 0:
                logger.info(f"Canary rolled back, stable version {stable_version} active")
                return True
            else:
                logger.error(f"Failed to rollback canary: {result.stderr}")
                return False
                
        except Exception as e:
            logger.error(f"Error rolling back canary: {e}")
            return False
    
    def run_automated_canary(self, canary_version: str, stable_version: str,
                            traffic_steps: List[int] = [10, 25, 50, 100],
                            step_duration: int = 300) -> Dict[str, Any]:
        """Run automated canary deployment with progressive traffic increase"""
        criteria = PromotionCriteria()
        results = []
        
        logger.info(f"Starting automated canary deployment for {canary_version}")
        
        try:
            for step, traffic_percentage in enumerate(traffic_steps):
                logger.info(f"Step {step + 1}: Routing {traffic_percentage}% traffic to canary")
                
                # Configure traffic split
                if not self.configure_traffic_split(canary_version, stable_version, traffic_percentage):
                    return {
                        'success': False,
                        'error': f'Failed to configure traffic split at step {step + 1}',
                        'results': results
                    }
                
                # Wait for traffic to stabilize
                time.sleep(60)
                
                # Monitor for step duration
                step_start = time.time()
                step_results = []
                
                while time.time() - step_start < step_duration:
                    # Get metrics
                    canary_metrics = self.get_canary_metrics(canary_version)
                    baseline_metrics = self.get_baseline_metrics(stable_version)
                    
                    # Evaluate
                    evaluation = self.evaluate_promotion(canary_metrics, baseline_metrics, criteria)
                    step_results.append(evaluation)
                    
                    # Check if we should abort
                    if not evaluation['promote'] and traffic_percentage > 10:
                        logger.warning("Canary metrics failing, initiating rollback")
                        self.rollback_canary(stable_version)
                        return {
                            'success': False,
                            'error': 'Canary metrics failed promotion criteria',
                            'results': results + step_results
                        }
                    
                    time.sleep(30)  # Check every 30 seconds
                
                results.extend(step_results)
                
                # If this is the final step and metrics are good, promote
                if traffic_percentage == 100:
                    final_evaluation = step_results[-1] if step_results else None
                    if final_evaluation and final_evaluation['promote']:
                        if self.promote_canary(canary_version):
                            logger.info("Canary successfully promoted to stable")
                            return {
                                'success': True,
                                'promoted': True,
                                'results': results
                            }
                        else:
                            return {
                                'success': False,
                                'error': 'Failed to promote canary',
                                'results': results
                            }
            
            return {
                'success': True,
                'promoted': False,
                'results': results
            }
            
        except Exception as e:
            logger.error(f"Automated canary deployment failed: {e}")
            self.rollback_canary(stable_version)
            return {
                'success': False,
                'error': str(e),
                'results': results
            }

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Canary Deployment Manager')
    parser.add_argument('--service', required=True, help='Service name')
    parser.add_argument('--namespace', required=True, help='Kubernetes namespace')
    parser.add_argument('--canary-version', required=True, help='Canary version')
    parser.add_argument('--stable-version', required=True, help='Stable version')
    parser.add_argument('--action', choices=['evaluate', 'promote', 'rollback', 'auto'], 
                       default='auto', help='Action to perform')
    parser.add_argument('--traffic-percentage', type=int, default=10, help='Traffic percentage for canary')
    parser.add_argument('--output-file', default='canary-results.json', help='Output file')
    
    args = parser.parse_args()
    
    manager = CanaryDeploymentManager(args.service, args.namespace)
    
    try:
        if args.action == 'evaluate':
            canary_metrics = manager.get_canary_metrics(args.canary_version)
            baseline_metrics = manager.get_baseline_metrics(args.stable_version)
            criteria = PromotionCriteria()
            
            result = manager.evaluate_promotion(canary_metrics, baseline_metrics, criteria)
            
            with open(args.output_file, 'w') as f:
                json.dump(result, f, indent=2)
            
            print(f"Evaluation result: {'PROMOTE' if result['promote'] else 'DO NOT PROMOTE'}")
            
        elif args.action == 'promote':
            success = manager.promote_canary(args.canary_version)
            print(f"Promotion {'successful' if success else 'failed'}")
            
        elif args.action == 'rollback':
            success = manager.rollback_canary(args.stable_version)
            print(f"Rollback {'successful' if success else 'failed'}")
            
        elif args.action == 'auto':
            result = manager.run_automated_canary(args.canary_version, args.stable_version)
            
            with open(args.output_file, 'w') as f:
                json.dump(result, f, indent=2)
            
            if result['success']:
                if result.get('promoted'):
                    print("Automated canary deployment completed successfully - PROMOTED")
                else:
                    print("Automated canary deployment completed - NOT PROMOTED")
            else:
                print(f"Automated canary deployment failed: {result.get('error')}")
                exit(1)
    
    except Exception as e:
        logger.error(f"Operation failed: {e}")
        exit(1)

if __name__ == "__main__":
    main()
```

---

## 3. Disaster Recovery and Business Continuity

### Multi-Region Disaster Recovery

```yaml
# .github/workflows/disaster-recovery.yml
name: Disaster Recovery Pipeline

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  workflow_dispatch:
    inputs:
      recovery_type:
        description: 'Type of recovery to perform'
        required: true
        type: choice
        options:
          - 'backup'
          - 'restore'
          - 'failover'
          - 'failback'
      target_region:
        description: 'Target region for recovery'
        required: false
        default: 'us-west-2'

env:
  PRIMARY_REGION: us-east-1
  SECONDARY_REGION: us-west-2
  BACKUP_BUCKET: enterprise-cicd-backups
  RTO_MINUTES: 30  # Recovery Time Objective
  RPO_HOURS: 4     # Recovery Point Objective

jobs:
  disaster-recovery-coordinator:
    runs-on: ubuntu-latest
    outputs:
      recovery-plan: ${{ steps.plan.outputs.plan }}
      estimated-rto: ${{ steps.plan.outputs.rto }}
      backup-status: ${{ steps.plan.outputs.backup_status }}
    steps:
      - name: Assess current state
        id: assess
        run: |
          # Check primary region health
          primary_health=$(python scripts/check-region-health.py --region ${{ env.PRIMARY_REGION }})
          secondary_health=$(python scripts/check-region-health.py --region ${{ env.SECONDARY_REGION }})
          
          echo "primary_health=$primary_health" >> $GITHUB_OUTPUT
          echo "secondary_health=$secondary_health" >> $GITHUB_OUTPUT
      
      - name: Create recovery plan
        id: plan
        run: |
          python scripts/create-recovery-plan.py \
            --recovery-type "${{ github.event.inputs.recovery_type || 'backup' }}" \
            --primary-region ${{ env.PRIMARY_REGION }} \
            --secondary-region ${{ env.SECONDARY_REGION }} \
            --primary-health "${{ steps.assess.outputs.primary_health }}" \
            --secondary-health "${{ steps.assess.outputs.secondary_health }}"
  
  backup-critical-data:
    runs-on: ubuntu-latest
    needs: disaster-recovery-coordinator
    if: ${{ contains(github.event.inputs.recovery_type, 'backup') || github.event_name == 'schedule' }}
    steps:
      - name: Backup application data
        run: |
          # Backup databases
          python scripts/backup-databases.py \
            --region ${{ env.PRIMARY_REGION }} \
            --backup-bucket ${{ env.BACKUP_BUCKET }} \
            --retention-days 30
      
      - name: Backup configuration
        run: |
          # Backup Kubernetes configurations
          kubectl get all --all-namespaces -o yaml > k8s-backup.yaml
          
          # Upload to S3
          aws s3 cp k8s-backup.yaml s3://${{ env.BACKUP_BUCKET }}/k8s/$(date +%Y%m%d)/
      
      - name: Backup secrets and certificates
        run: |
          # Backup secrets (encrypted)
          python scripts/backup-secrets.py \
            --backup-bucket ${{ env.BACKUP_BUCKET }} \
            --encryption-key ${{ secrets.BACKUP_ENCRYPTION_KEY }}
      
      - name: Verify backup integrity
        run: |
          # Verify all backups
          python scripts/verify-backups.py \
            --backup-bucket ${{ env.BACKUP_BUCKET }} \
            --date $(date +%Y%m%d)
  
  cross-region-replication:
    runs-on: ubuntu-latest
    needs: backup-critical-data
    steps:
      - name: Replicate to secondary region
        run: |
          # Replicate backups to secondary region
          aws s3 sync s3://${{ env.BACKUP_BUCKET }} \
            s3://${{ env.BACKUP_BUCKET }}-${{ env.SECONDARY_REGION }} \
            --source-region ${{ env.PRIMARY_REGION }} \
            --region ${{ env.SECONDARY_REGION }}
      
      - name: Update disaster recovery inventory
        run: |
          python scripts/update-dr-inventory.py \
            --primary-region ${{ env.PRIMARY_REGION }} \
            --secondary-region ${{ env.SECONDARY_REGION }} \
            --backup-date $(date +%Y%m%d)
  
  failover-execution:
    runs-on: ubuntu-latest
    needs: disaster-recovery-coordinator
    if: ${{ github.event.inputs.recovery_type == 'failover' }}
    steps:
      - name: Initiate failover
        run: |
          echo "🚨 INITIATING DISASTER RECOVERY FAILOVER 🚨"
          
          # Update DNS to point to secondary region
          python scripts/update-dns-failover.py \
            --primary-region ${{ env.PRIMARY_REGION }} \
            --secondary-region ${{ env.SECONDARY_REGION }}
      
      - name: Restore services in secondary region
        run: |
          # Restore from latest backup
          python scripts/restore-services.py \
            --region ${{ env.SECONDARY_REGION }} \
            --backup-bucket ${{ env.BACKUP_BUCKET }}-${{ env.SECONDARY_REGION }} \
            --restore-point latest
      
      - name: Validate failover
        run: |
          # Run health checks on secondary region
          python scripts/validate-failover.py \
            --region ${{ env.SECONDARY_REGION }} \
            --rto-minutes ${{ env.RTO_MINUTES }}
      
      - name: Notify stakeholders
        run: |
          # Send notifications about failover
          python scripts/send-dr-notifications.py \
            --event-type "failover_completed" \
            --region ${{ env.SECONDARY_REGION }}
  
  failback-execution:
    runs-on: ubuntu-latest
    needs: disaster-recovery-coordinator
    if: ${{ github.event.inputs.recovery_type == 'failback' }}
    steps:
      - name: Prepare primary region
        run: |
          # Ensure primary region is healthy
          python scripts/prepare-failback.py \
            --primary-region ${{ env.PRIMARY_REGION }} \
            --secondary-region ${{ env.SECONDARY_REGION }}
      
      - name: Sync data from secondary
        run: |
          # Sync any data changes from secondary back to primary
          python scripts/sync-failback-data.py \
            --source-region ${{ env.SECONDARY_REGION }} \
            --target-region ${{ env.PRIMARY_REGION }}
      
      - name: Execute failback
        run: |
          # Switch traffic back to primary region
          python scripts/execute-failback.py \
            --primary-region ${{ env.PRIMARY_REGION }} \
            --secondary-region ${{ env.SECONDARY_REGION }}
      
      - name: Validate failback
        run: |
          # Validate primary region is working correctly
          python scripts/validate-failback.py \
            --region ${{ env.PRIMARY_REGION }}
  
  dr-testing:
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'schedule' }}
    steps:
      - name: Run DR drill
        run: |
          # Perform non-disruptive DR testing
          python scripts/dr-drill.py \
            --test-type "non-disruptive" \
            --secondary-region ${{ env.SECONDARY_REGION }}
      
      - name: Generate DR report
        run: |
          # Generate comprehensive DR report
          python scripts/generate-dr-report.py \
            --output-file dr-report-$(date +%Y%m%d).json
      
      - name: Upload DR report
        uses: actions/upload-artifact@v3
        with:
          name: dr-report
          path: dr-report-*.json
```

### Business Continuity Management

```python
# scripts/business-continuity-manager.py
import json
import boto3
import time
import requests
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, asdict
import logging
from datetime import datetime, timedelta
from concurrent.futures import ThreadPoolExecutor, as_completed

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class ServiceHealth:
    service_name: str
    region: str
    status: str  # healthy, degraded, unhealthy
    response_time: float
    error_rate: float
    availability: float
    last_check: datetime

@dataclass
class RecoveryPlan:
    plan_id: str
    trigger_event: str
    recovery_steps: List[Dict[str, Any]]
    estimated_rto: int  # minutes
    estimated_rpo: int  # minutes
    dependencies: List[str]
    rollback_plan: List[Dict[str, Any]]

@dataclass
class DisasterEvent:
    event_id: str
    event_type: str
    severity: str  # low, medium, high, critical
    affected_services: List[str]
    affected_regions: List[str]
    start_time: datetime
    estimated_impact: str
    recovery_plan_id: Optional[str] = None

class BusinessContinuityManager:
    def __init__(self):
        self.aws_clients = {}
        self.recovery_plans = {}
        self.service_catalog = {}
        self.load_configurations()
    
    def load_configurations(self):
        """Load business continuity configurations"""
        try:
            # Load service catalog
            with open('config/service-catalog.json', 'r') as f:
                self.service_catalog = json.load(f)
            
            # Load recovery plans
            with open('config/recovery-plans.json', 'r') as f:
                plans_data = json.load(f)
                for plan_data in plans_data['plans']:
                    plan = RecoveryPlan(**plan_data)
                    self.recovery_plans[plan.plan_id] = plan
            
            logger.info(f"Loaded {len(self.service_catalog)} services and {len(self.recovery_plans)} recovery plans")
            
        except Exception as e:
            logger.error(f"Failed to load configurations: {e}")
            raise
    
    def get_aws_client(self, service: str, region: str):
        """Get AWS client for specific service and region"""
        key = f"{service}_{region}"
        if key not in self.aws_clients:
            self.aws_clients[key] = boto3.client(service, region_name=region)
        return self.aws_clients[key]
    
    def check_service_health(self, service_name: str, region: str) -> ServiceHealth:
        """Check health of a specific service"""
        service_config = self.service_catalog.get(service_name, {})
        health_endpoint = service_config.get('health_endpoint')
        
        if not health_endpoint:
            return ServiceHealth(
                service_name=service_name,
                region=region,
                status='unknown',
                response_time=0,
                error_rate=0,
                availability=0,
                last_check=datetime.now()
            )
        
        try:
            start_time = time.time()
            response = requests.get(
                health_endpoint.replace('{region}', region),
                timeout=30
            )
            response_time = (time.time() - start_time) * 1000
            
            if response.status_code == 200:
                health_data = response.json()
                status = 'healthy' if health_data.get('status') == 'ok' else 'degraded'
            else:
                status = 'unhealthy'
            
            # Get additional metrics from CloudWatch
            cloudwatch = self.get_aws_client('cloudwatch', region)
            
            # Get error rate
            error_rate_response = cloudwatch.get_metric_statistics(
                Namespace='AWS/ApplicationELB',
                MetricName='HTTPCode_Target_5XX_Count',
                Dimensions=[
                    {
                        'Name': 'LoadBalancer',
                        'Value': service_config.get('load_balancer_name', '')
                    }
                ],
                StartTime=datetime.now() - timedelta(minutes=5),
                EndTime=datetime.now(),
                Period=300,
                Statistics=['Sum']
            )
            
            error_count = sum(point['Sum'] for point in error_rate_response['Datapoints'])
            
            # Get total request count
            total_requests_response = cloudwatch.get_metric_statistics(
                Namespace='AWS/ApplicationELB',
                MetricName='RequestCount',
                Dimensions=[
                    {
                        'Name': 'LoadBalancer',
                        'Value': service_config.get('load_balancer_name', '')
                    }
                ],
                StartTime=datetime.now() - timedelta(minutes=5),
                EndTime=datetime.now(),
                Period=300,
                Statistics=['Sum']
            )
            
            total_requests = sum(point['Sum'] for point in total_requests_response['Datapoints'])
            error_rate = (error_count / total_requests * 100) if total_requests > 0 else 0
            
            # Calculate availability (simplified)
            availability = 100 - error_rate if error_rate < 100 else 0
            
            return ServiceHealth(
                service_name=service_name,
                region=region,
                status=status,
                response_time=response_time,
                error_rate=error_rate,
                availability=availability,
                last_check=datetime.now()
            )
            
        except Exception as e:
            logger.error(f"Health check failed for {service_name} in {region}: {e}")
            return ServiceHealth(
                service_name=service_name,
                region=region,
                status='unhealthy',
                response_time=0,
                error_rate=100,
                availability=0,
                last_check=datetime.now()
            )
    
    def check_all_services_health(self, regions: List[str]) -> Dict[str, List[ServiceHealth]]:
        """Check health of all services across regions"""
        results = {}
        
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = []
            
            for service_name in self.service_catalog.keys():
                for region in regions:
                    future = executor.submit(self.check_service_health, service_name, region)
                    futures.append((future, service_name, region))
            
            for future, service_name, region in futures:
                try:
                    health = future.result(timeout=60)
                    if service_name not in results:
                        results[service_name] = []
                    results[service_name].append(health)
                except Exception as e:
                    logger.error(f"Failed to check {service_name} in {region}: {e}")
        
        return results
    
    def detect_disaster_events(self, health_results: Dict[str, List[ServiceHealth]]) -> List[DisasterEvent]:
        """Detect potential disaster events from health data"""
        events = []
        
        for service_name, health_list in health_results.items():
            unhealthy_regions = [h.region for h in health_list if h.status == 'unhealthy']
            degraded_regions = [h.region for h in health_list if h.status == 'degraded']
            
            # Multi-region outage
            if len(unhealthy_regions) >= 2:
                event = DisasterEvent(
                    event_id=f"multi-region-outage-{service_name}-{int(time.time())}",
                    event_type="multi_region_outage",
                    severity="critical",
                    affected_services=[service_name],
                    affected_regions=unhealthy_regions,
                    start_time=datetime.now(),
                    estimated_impact="Service unavailable in multiple regions"
                )
                events.append(event)
            
            # Single region outage
            elif len(unhealthy_regions) == 1:
                event = DisasterEvent(
                    event_id=f"region-outage-{service_name}-{int(time.time())}",
                    event_type="region_outage",
                    severity="high",
                    affected_services=[service_name],
                    affected_regions=unhealthy_regions,
                    start_time=datetime.now(),
                    estimated_impact="Service unavailable in one region"
                )
                events.append(event)
            
            # Performance degradation
            elif len(degraded_regions) >= 1:
                avg_error_rate = sum(h.error_rate for h in health_list if h.region in degraded_regions) / len(degraded_regions)
                if avg_error_rate > 5:  # 5% error rate threshold
                    event = DisasterEvent(
                        event_id=f"performance-degradation-{service_name}-{int(time.time())}",
                        event_type="performance_degradation",
                        severity="medium",
                        affected_services=[service_name],
                        affected_regions=degraded_regions,
                        start_time=datetime.now(),
                        estimated_impact="Service experiencing performance issues"
                    )
                    events.append(event)
        
        return events
    
    def execute_recovery_plan(self, event: DisasterEvent) -> Dict[str, Any]:
        """Execute recovery plan for disaster event"""
        # Find appropriate recovery plan
        recovery_plan = None
        for plan in self.recovery_plans.values():
            if plan.trigger_event == event.event_type:
                recovery_plan = plan
                break
        
        if not recovery_plan:
            logger.error(f"No recovery plan found for event type: {event.event_type}")
            return {'success': False, 'error': 'No recovery plan found'}
        
        logger.info(f"Executing recovery plan {recovery_plan.plan_id} for event {event.event_id}")
        
        execution_results = []
        start_time = time.time()
        
        try:
            for step in recovery_plan.recovery_steps:
                step_start = time.time()
                logger.info(f"Executing step: {step['name']}")
                
                # Execute step based on type
                if step['type'] == 'dns_failover':
                    result = self._execute_dns_failover(step, event)
                elif step['type'] == 'service_restart':
                    result = self._execute_service_restart(step, event)
                elif step['type'] == 'traffic_reroute':
                    result = self._execute_traffic_reroute(step, event)
                elif step['type'] == 'scale_up':
                    result = self._execute_scale_up(step, event)
                elif step['type'] == 'notification':
                    result = self._execute_notification(step, event)
                else:
                    result = {'success': False, 'error': f'Unknown step type: {step["type"]}'}
                
                step_duration = time.time() - step_start
                execution_results.append({
                    'step': step['name'],
                    'success': result['success'],
                    'duration': step_duration,
                    'details': result
                })
                
                if not result['success'] and step.get('critical', False):
                    logger.error(f"Critical step failed: {step['name']}")
                    break
            
            total_duration = time.time() - start_time
            
            return {
                'success': True,
                'event_id': event.event_id,
                'recovery_plan_id': recovery_plan.plan_id,
                'execution_time': total_duration,
                'steps_executed': len(execution_results),
                'steps_successful': sum(1 for r in execution_results if r['success']),
                'results': execution_results
            }
            
        except Exception as e:
            logger.error(f"Recovery plan execution failed: {e}")
            return {
                'success': False,
                'error': str(e),
                'results': execution_results
            }
    
    def _execute_dns_failover(self, step: Dict[str, Any], event: DisasterEvent) -> Dict[str, Any]:
        """Execute DNS failover step"""
        try:
            route53 = self.get_aws_client('route53', 'us-east-1')
            
            # Update Route53 records to point to healthy region
            healthy_region = step['target_region']
            
            response = route53.change_resource_record_sets(
                HostedZoneId=step['hosted_zone_id'],
                ChangeBatch={
                    'Changes': [{
                        'Action': 'UPSERT',
                        'ResourceRecordSet': {
                            'Name': step['domain_name'],
                            'Type': 'A',
                            'SetIdentifier': 'failover',
                            'Failover': 'PRIMARY',
                            'TTL': 60,
                            'ResourceRecords': [{
                                'Value': step['target_ip']
                            }]
                        }
                    }]
                }
            )
            
            return {'success': True, 'change_id': response['ChangeInfo']['Id']}
            
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def _execute_service_restart(self, step: Dict[str, Any], event: DisasterEvent) -> Dict[str, Any]:
        """Execute service restart step"""
        try:
            # Restart Kubernetes deployments
            import subprocess
            
            for service in step['services']:
                result = subprocess.run([
                    'kubectl', 'rollout', 'restart', 'deployment', service,
                    '-n', step.get('namespace', 'default')
                ], capture_output=True, text=True)
                
                if result.returncode != 0:
                    return {'success': False, 'error': result.stderr}
            
            return {'success': True, 'restarted_services': step['services']}
            
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def _execute_traffic_reroute(self, step: Dict[str, Any], event: DisasterEvent) -> Dict[str, Any]:
        """Execute traffic rerouting step"""
        try:
            # Update load balancer configuration
            elbv2 = self.get_aws_client('elbv2', step['region'])
            
            # Modify target group to point to healthy instances
            response = elbv2.modify_target_group(
                TargetGroupArn=step['target_group_arn'],
                HealthCheckPath=step.get('health_check_path', '/health'),
                HealthCheckIntervalSeconds=step.get('health_check_interval', 30)
            )
            
            return {'success': True, 'target_group': step['target_group_arn']}
            
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def _execute_scale_up(self, step: Dict[str, Any], event: DisasterEvent) -> Dict[str, Any]:
        """Execute scale up step"""
        try:
            # Scale up services in healthy regions
            import subprocess
            
            for service in step['services']:
                result = subprocess.run([
                    'kubectl', 'scale', 'deployment', service,
                    f'--replicas={step["target_replicas"]}',
                    '-n', step.get('namespace', 'default')
                ], capture_output=True, text=True)
                
                if result.returncode != 0:
                    return {'success': False, 'error': result.stderr}
            
            return {'success': True, 'scaled_services': step['services']}
            
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def _execute_notification(self, step: Dict[str, Any], event: DisasterEvent) -> Dict[str, Any]:
        """Execute notification step"""
        try:
            sns = self.get_aws_client('sns', 'us-east-1')
            
            message = step['message_template'].format(
                event_id=event.event_id,
                event_type=event.event_type,
                severity=event.severity,
                affected_services=', '.join(event.affected_services),
                affected_regions=', '.join(event.affected_regions)
            )
            
            response = sns.publish(
                TopicArn=step['topic_arn'],
                Message=message,
                Subject=f"Disaster Recovery Alert: {event.severity.upper()}"
            )
            
            return {'success': True, 'message_id': response['MessageId']}
            
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    def generate_business_continuity_report(self, regions: List[str]) -> Dict[str, Any]:
        """Generate comprehensive business continuity report"""
        logger.info("Generating business continuity report")
        
        # Check all services health
        health_results = self.check_all_services_health(regions)
        
        # Detect any disaster events
        disaster_events = self.detect_disaster_events(health_results)
        
        # Calculate overall health metrics
        total_services = len(self.service_catalog)
        healthy_services = 0
        degraded_services = 0
        unhealthy_services = 0
        
        for service_name, health_list in health_results.items():
            service_status = 'healthy'
            for health in health_list:
                if health.status == 'unhealthy':
                    service_status = 'unhealthy'
                    break
                elif health.status == 'degraded':
                    service_status = 'degraded'
            
            if service_status == 'healthy':
                healthy_services += 1
            elif service_status == 'degraded':
                degraded_services += 1
            else:
                unhealthy_services += 1
        
        # Calculate RTO/RPO compliance
        rto_compliance = self._calculate_rto_compliance()
        rpo_compliance = self._calculate_rpo_compliance()
        
        return {
            'report_timestamp': datetime.now().isoformat(),
            'overall_health': {
                'total_services': total_services,
                'healthy_services': healthy_services,
                'degraded_services': degraded_services,
                'unhealthy_services': unhealthy_services,
                'health_percentage': (healthy_services / total_services * 100) if total_services > 0 else 0
            },
            'regional_health': self._summarize_regional_health(health_results),
            'disaster_events': [asdict(event) for event in disaster_events],
            'recovery_readiness': {
                'recovery_plans_count': len(self.recovery_plans),
                'rto_compliance': rto_compliance,
                'rpo_compliance': rpo_compliance
            },
            'recommendations': self._generate_bc_recommendations(health_results, disaster_events),
            'service_details': {
                service_name: [asdict(health) for health in health_list]
                for service_name, health_list in health_results.items()
            }
        }
    
    def _calculate_rto_compliance(self) -> Dict[str, Any]:
        """Calculate RTO compliance metrics"""
        # Simplified RTO compliance calculation
        total_plans = len(self.recovery_plans)
        compliant_plans = sum(1 for plan in self.recovery_plans.values() if plan.estimated_rto <= 30)
        
        return {
            'total_plans': total_plans,
            'compliant_plans': compliant_plans,
            'compliance_percentage': (compliant_plans / total_plans * 100) if total_plans > 0 else 0,
            'target_rto_minutes': 30
        }
    
    def _calculate_rpo_compliance(self) -> Dict[str, Any]:
        """Calculate RPO compliance metrics"""
        # Simplified RPO compliance calculation
        total_plans = len(self.recovery_plans)
        compliant_plans = sum(1 for plan in self.recovery_plans.values() if plan.estimated_rpo <= 240)  # 4 hours
        
        return {
            'total_plans': total_plans,
            'compliant_plans': compliant_plans,
            'compliance_percentage': (compliant_plans / total_plans * 100) if total_plans > 0 else 0,
            'target_rpo_minutes': 240
        }
    
    def _summarize_regional_health(self, health_results: Dict[str, List[ServiceHealth]]) -> Dict[str, Any]:
        """Summarize health by region"""
        regional_summary = {}
        
        # Get all regions
        all_regions = set()
        for health_list in health_results.values():
            for health in health_list:
                all_regions.add(health.region)
        
        for region in all_regions:
            region_services = []
            for service_name, health_list in health_results.items():
                for health in health_list:
                    if health.region == region:
                        region_services.append(health)
            
            healthy_count = sum(1 for h in region_services if h.status == 'healthy')
            degraded_count = sum(1 for h in region_services if h.status == 'degraded')
            unhealthy_count = sum(1 for h in region_services if h.status == 'unhealthy')
            
            avg_response_time = sum(h.response_time for h in region_services) / len(region_services) if region_services else 0
            avg_availability = sum(h.availability for h in region_services) / len(region_services) if region_services else 0
            
            regional_summary[region] = {
                'total_services': len(region_services),
                'healthy_services': healthy_count,
                'degraded_services': degraded_count,
                'unhealthy_services': unhealthy_count,
                'health_percentage': (healthy_count / len(region_services) * 100) if region_services else 0,
                'avg_response_time': avg_response_time,
                'avg_availability': avg_availability
            }
        
        return regional_summary
    
    def _generate_bc_recommendations(self, health_results: Dict[str, List[ServiceHealth]], 
                                   disaster_events: List[DisasterEvent]) -> List[str]:
        """Generate business continuity recommendations"""
        recommendations = []
        
        # Check for single points of failure
        for service_name, health_list in health_results.items():
            healthy_regions = [h.region for h in health_list if h.status == 'healthy']
            if len(healthy_regions) < 2:
                recommendations.append(f"Deploy {service_name} to additional regions for redundancy")
        
        # Check for high error rates
        for service_name, health_list in health_results.items():
            avg_error_rate = sum(h.error_rate for h in health_list) / len(health_list)
            if avg_error_rate > 1:
                recommendations.append(f"Investigate high error rate ({avg_error_rate:.2f}%) for {service_name}")
        
        # Check for slow response times
        for service_name, health_list in health_results.items():
            avg_response_time = sum(h.response_time for h in health_list) / len(health_list)
            if avg_response_time > 1000:  # 1 second
                recommendations.append(f"Optimize response time ({avg_response_time:.0f}ms) for {service_name}")
        
        # Check for active disaster events
        if disaster_events:
            recommendations.append(f"Address {len(disaster_events)} active disaster events")
        
        # Check recovery plan coverage
        services_without_plans = []
        for service_name in self.service_catalog.keys():
            has_plan = any(service_name in plan.dependencies for plan in self.recovery_plans.values())
            if not has_plan:
                services_without_plans.append(service_name)
        
        if services_without_plans:
            recommendations.append(f"Create recovery plans for: {', '.join(services_without_plans)}")
        
        if not recommendations:
            recommendations.append("All systems are operating within acceptable parameters")
        
        return recommendations

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Business Continuity Manager')
    parser.add_argument('--action', choices=['health-check', 'detect-events', 'execute-recovery', 'report'], 
                       required=True, help='Action to perform')
    parser.add_argument('--regions', nargs='+', default=['us-east-1', 'us-west-2'], help='Regions to check')
    parser.add_argument('--event-id', help='Event ID for recovery execution')
    parser.add_argument('--output-file', default='bc-report.json', help='Output file')
    
    args = parser.parse_args()
    
    manager = BusinessContinuityManager()
    
    try:
        if args.action == 'health-check':
            health_results = manager.check_all_services_health(args.regions)
            
            with open(args.output_file, 'w') as f:
                json.dump({
                    'timestamp': datetime.now().isoformat(),
                    'health_results': {
                        service_name: [asdict(health) for health in health_list]
                        for service_name, health_list in health_results.items()
                    }
                }, f, indent=2, default=str)
            
            print(f"Health check results saved to {args.output_file}")
        
        elif args.action == 'detect-events':
            health_results = manager.check_all_services_health(args.regions)
            events = manager.detect_disaster_events(health_results)
            
            with open(args.output_file, 'w') as f:
                json.dump({
                    'timestamp': datetime.now().isoformat(),
                    'events': [asdict(event) for event in events]
                }, f, indent=2, default=str)
            
            print(f"Detected {len(events)} disaster events")
            for event in events:
                print(f"  - {event.event_type} ({event.severity}): {event.estimated_impact}")
        
        elif args.action == 'report':
            report = manager.generate_business_continuity_report(args.regions)
            
            with open(args.output_file, 'w') as f:
                json.dump(report, f, indent=2, default=str)
            
            print(f"Business continuity report saved to {args.output_file}")
            print(f"Overall health: {report['overall_health']['health_percentage']:.1f}%")
            print(f"Active events: {len(report['disaster_events'])}")
    
    except Exception as e:
        logger.error(f"Operation failed: {e}")
        exit(1)

if __name__ == "__main__":
    main()
```

---

## 4. Cost Optimization and FinOps

### Enterprise Cost Management

```yaml
# .github/workflows/cost-optimization.yml
name: Enterprise Cost Optimization

on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly on Monday at 6 AM
  workflow_dispatch:
    inputs:
      optimization_type:
        description: 'Type of cost optimization'
        required: true
        type: choice
        options:
          - 'analysis'
          - 'recommendations'
          - 'automated-cleanup'
          - 'budget-alerts'

env:
  COST_THRESHOLD_PERCENT: 80
  BUDGET_ALERT_EMAIL: finops@company.com

jobs:
  cost-analysis:
    runs-on: ubuntu-latest
    outputs:
      total-cost: ${{ steps.analyze.outputs.total_cost }}
      cost-trend: ${{ steps.analyze.outputs.trend }}
      top-services: ${{ steps.analyze.outputs.top_services }}
    steps:
      - name: Analyze current costs
        id: analyze
        run: |
          # Analyze AWS costs across all accounts
          python scripts/cost-analyzer.py \
            --period 30 \
            --granularity DAILY \
            --output-file cost-analysis.json
      
      - name: Generate cost trends
        run: |
          # Generate cost trend analysis
          python scripts/cost-trends.py \
            --input-file cost-analysis.json \
            --output-file cost-trends.json
      
      - name: Upload cost data
        uses: actions/upload-artifact@v3
        with:
          name: cost-analysis
          path: |
            cost-analysis.json
            cost-trends.json
  
  resource-optimization:
    runs-on: ubuntu-latest
    needs: cost-analysis
    if: ${{ github.event.inputs.optimization_type == 'recommendations' || github.event_name == 'schedule' }}
    steps:
      - name: Identify optimization opportunities
        run: |
          # Find underutilized resources
          python scripts/resource-optimizer.py \
            --analyze-ec2 \
            --analyze-rds \
            --analyze-s3 \
            --analyze-lambda \
            --output-file optimization-recommendations.json
      
      - name: Calculate potential savings
        run: |
          # Calculate potential cost savings
          python scripts/calculate-savings.py \
            --recommendations optimization-recommendations.json \
            --current-costs cost-analysis.json \
            --output-file savings-report.json
      
      - name: Create optimization tickets
        run: |
          # Create Jira tickets for optimization opportunities
          python scripts/create-optimization-tickets.py \
            --recommendations optimization-recommendations.json \
            --savings-report savings-report.json
  
  automated-cleanup:
    runs-on: ubuntu-latest
    if: ${{ github.event.inputs.optimization_type == 'automated-cleanup' }}
    steps:
      - name: Clean up unused resources
        run: |
          # Automated cleanup of safe-to-delete resources
          python scripts/automated-cleanup.py \
            --dry-run false \
            --cleanup-snapshots \
            --cleanup-volumes \
            --cleanup-images \
            --retention-days 30
      
      - name: Generate cleanup report
        run: |
          # Generate report of cleaned up resources
          python scripts/cleanup-report.py \
            --output-file cleanup-report.json
  
  budget-monitoring:
    runs-on: ubuntu-latest
    needs: cost-analysis
    steps:
      - name: Check budget thresholds
        run: |
          # Check if costs exceed budget thresholds
          python scripts/budget-monitor.py \
            --current-costs "${{ needs.cost-analysis.outputs.total-cost }}" \
            --threshold-percent ${{ env.COST_THRESHOLD_PERCENT }} \
            --alert-email ${{ env.BUDGET_ALERT_EMAIL }}
      
      - name: Update budget forecasts
        run: |
          # Update budget forecasts based on current trends
          python scripts/budget-forecast.py \
            --cost-trends cost-trends.json \
            --output-file budget-forecast.json
  
  cost-reporting:
    runs-on: ubuntu-latest
    needs: [cost-analysis, resource-optimization, budget-monitoring]
    steps:
      - name: Generate executive dashboard
        run: |
          # Generate executive cost dashboard
          python scripts/executive-dashboard.py \
            --cost-analysis cost-analysis.json \
            --optimization-recommendations optimization-recommendations.json \
            --budget-forecast budget-forecast.json \
            --output-file executive-dashboard.json
      
      - name: Send cost reports
        run: |
          # Send cost reports to stakeholders
          python scripts/send-cost-reports.py \
            --dashboard executive-dashboard.json \
            --recipients finops@company.com,cto@company.com
```

### Cost Analyzer Script

```python
#!/usr/bin/env python3
# scripts/cost-analyzer.py

import boto3
import json
import argparse
from datetime import datetime, timedelta
from typing import Dict, List, Any
import logging
from dataclasses import dataclass, asdict
from collections import defaultdict

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@dataclass
class CostData:
    service: str
    account_id: str
    region: str
    cost: float
    currency: str
    date: str
    usage_type: str
    resource_id: str = None

@dataclass
class CostOptimization:
    resource_type: str
    resource_id: str
    current_cost: float
    potential_savings: float
    recommendation: str
    confidence: str
    implementation_effort: str
    risk_level: str

class EnterpriseFinOpsManager:
    def __init__(self):
        self.cost_explorer = boto3.client('ce')
        self.ec2 = boto3.client('ec2')
        self.rds = boto3.client('rds')
        self.s3 = boto3.client('s3')
        self.cloudwatch = boto3.client('cloudwatch')
        self.organizations = boto3.client('organizations')
        
        # Cost thresholds and rules
        self.cost_thresholds = {
            'ec2_cpu_utilization': 10,  # %
            'rds_cpu_utilization': 20,  # %
            's3_access_frequency': 30,  # days
            'ebs_unattached_days': 7,
            'snapshot_age_days': 90
        }
    
    def get_organization_accounts(self) -> List[str]:
        """Get all accounts in the organization"""
        try:
            response = self.organizations.list_accounts()
            return [account['Id'] for account in response['Accounts'] if account['Status'] == 'ACTIVE']
        except Exception as e:
            logger.warning(f"Could not fetch organization accounts: {e}")
            return [boto3.client('sts').get_caller_identity()['Account']]
    
    def analyze_costs(self, period_days: int = 30, granularity: str = 'DAILY') -> Dict[str, Any]:
        """Analyze costs across all accounts and services"""
        logger.info(f"Analyzing costs for the last {period_days} days")
        
        end_date = datetime.now().date()
        start_date = end_date - timedelta(days=period_days)
        
        # Get cost and usage data
        response = self.cost_explorer.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d')
            },
            Granularity=granularity,
            Metrics=['BlendedCost', 'UsageQuantity'],
            GroupBy=[
                {'Type': 'DIMENSION', 'Key': 'SERVICE'},
                {'Type': 'DIMENSION', 'Key': 'LINKED_ACCOUNT'},
                {'Type': 'DIMENSION', 'Key': 'REGION'}
            ]
        )
        
        cost_data = []
        total_cost = 0
        service_costs = defaultdict(float)
        account_costs = defaultdict(float)
        region_costs = defaultdict(float)
        
        for result in response['ResultsByTime']:
            date = result['TimePeriod']['Start']
            
            for group in result['Groups']:
                service = group['Keys'][0] if group['Keys'][0] else 'Unknown'
                account = group['Keys'][1] if group['Keys'][1] else 'Unknown'
                region = group['Keys'][2] if group['Keys'][2] else 'Unknown'
                
                cost = float(group['Metrics']['BlendedCost']['Amount'])
                currency = group['Metrics']['BlendedCost']['Unit']
                usage = float(group['Metrics']['UsageQuantity']['Amount'])
                
                cost_data.append(CostData(
                    service=service,
                    account_id=account,
                    region=region,
                    cost=cost,
                    currency=currency,
                    date=date,
                    usage_type='BlendedCost'
                ))
                
                total_cost += cost
                service_costs[service] += cost
                account_costs[account] += cost
                region_costs[region] += cost
        
        # Get top services, accounts, and regions
        top_services = sorted(service_costs.items(), key=lambda x: x[1], reverse=True)[:10]
        top_accounts = sorted(account_costs.items(), key=lambda x: x[1], reverse=True)[:10]
        top_regions = sorted(region_costs.items(), key=lambda x: x[1], reverse=True)[:10]
        
        # Calculate cost trends
        daily_costs = defaultdict(float)
        for data in cost_data:
            daily_costs[data.date] += data.cost
        
        sorted_dates = sorted(daily_costs.keys())
        if len(sorted_dates) >= 7:
            recent_avg = sum(daily_costs[date] for date in sorted_dates[-7:]) / 7
            previous_avg = sum(daily_costs[date] for date in sorted_dates[-14:-7]) / 7
            trend_percentage = ((recent_avg - previous_avg) / previous_avg * 100) if previous_avg > 0 else 0
        else:
            trend_percentage = 0
        
        return {
            'analysis_period': {
                'start_date': start_date.isoformat(),
                'end_date': end_date.isoformat(),
                'days': period_days
            },
            'total_cost': total_cost,
            'currency': 'USD',
            'cost_trend_percentage': trend_percentage,
            'top_services': top_services,
            'top_accounts': top_accounts,
            'top_regions': top_regions,
            'daily_costs': dict(daily_costs),
            'detailed_costs': [asdict(data) for data in cost_data]
        }
    
    def identify_ec2_optimization_opportunities(self) -> List[CostOptimization]:
        """Identify EC2 cost optimization opportunities"""
        logger.info("Analyzing EC2 instances for optimization opportunities")
        
        optimizations = []
        
        # Get all EC2 instances
        paginator = self.ec2.get_paginator('describe_instances')
        
        for page in paginator.paginate():
            for reservation in page['Reservations']:
                for instance in reservation['Instances']:
                    if instance['State']['Name'] not in ['running', 'stopped']:
                        continue
                    
                    instance_id = instance['InstanceId']
                    instance_type = instance['InstanceType']
                    
                    # Get CPU utilization
                    cpu_utilization = self._get_ec2_cpu_utilization(instance_id)
                    
                    # Check for underutilized instances
                    if cpu_utilization < self.cost_thresholds['ec2_cpu_utilization']:
                        # Estimate current cost
                        current_cost = self._estimate_ec2_cost(instance_type)
                        
                        # Suggest smaller instance type
                        suggested_type = self._suggest_smaller_instance_type(instance_type)
                        potential_savings = current_cost * 0.3  # Estimate 30% savings
                        
                        optimizations.append(CostOptimization(
                            resource_type='EC2',
                            resource_id=instance_id,
                            current_cost=current_cost,
                            potential_savings=potential_savings,
                            recommendation=f"Downsize from {instance_type} to {suggested_type} (CPU: {cpu_utilization:.1f}%)",
                            confidence='High' if cpu_utilization < 5 else 'Medium',
                            implementation_effort='Low',
                            risk_level='Low'
                        ))
                    
                    # Check for stopped instances
                    if instance['State']['Name'] == 'stopped':
                        current_cost = self._estimate_ec2_cost(instance_type)
                        
                        optimizations.append(CostOptimization(
                            resource_type='EC2',
                            resource_id=instance_id,
                            current_cost=current_cost,
                            potential_savings=current_cost,
                            recommendation=f"Terminate stopped instance {instance_type}",
                            confidence='High',
                            implementation_effort='Low',
                            risk_level='Medium'
                        ))
        
        return optimizations
    
    def identify_rds_optimization_opportunities(self) -> List[CostOptimization]:
        """Identify RDS cost optimization opportunities"""
        logger.info("Analyzing RDS instances for optimization opportunities")
        
        optimizations = []
        
        # Get all RDS instances
        paginator = self.rds.get_paginator('describe_db_instances')
        
        for page in paginator.paginate():
            for db_instance in page['DBInstances']:
                if db_instance['DBInstanceStatus'] != 'available':
                    continue
                
                db_identifier = db_instance['DBInstanceIdentifier']
                db_class = db_instance['DBInstanceClass']
                
                # Get CPU utilization
                cpu_utilization = self._get_rds_cpu_utilization(db_identifier)
                
                # Check for underutilized instances
                if cpu_utilization < self.cost_thresholds['rds_cpu_utilization']:
                    current_cost = self._estimate_rds_cost(db_class)
                    suggested_class = self._suggest_smaller_rds_class(db_class)
                    potential_savings = current_cost * 0.25  # Estimate 25% savings
                    
                    optimizations.append(CostOptimization(
                        resource_type='RDS',
                        resource_id=db_identifier,
                        current_cost=current_cost,
                        potential_savings=potential_savings,
                        recommendation=f"Downsize from {db_class} to {suggested_class} (CPU: {cpu_utilization:.1f}%)",
                        confidence='High' if cpu_utilization < 10 else 'Medium',
                        implementation_effort='Medium',
                        risk_level='Medium'
                    ))
        
        return optimizations
    
    def identify_s3_optimization_opportunities(self) -> List[CostOptimization]:
        """Identify S3 cost optimization opportunities"""
        logger.info("Analyzing S3 buckets for optimization opportunities")
        
        optimizations = []
        
        # Get all S3 buckets
        response = self.s3.list_buckets()
        
        for bucket in response['Buckets']:
            bucket_name = bucket['Name']
            
            try:
                # Get bucket size and object count
                bucket_size, object_count = self._get_s3_bucket_metrics(bucket_name)
                
                # Check for lifecycle policy
                has_lifecycle = self._check_s3_lifecycle_policy(bucket_name)
                
                if not has_lifecycle and bucket_size > 1000000000:  # 1GB
                    current_cost = bucket_size * 0.023 / 1000000000  # Estimate $0.023 per GB
                    potential_savings = current_cost * 0.4  # Estimate 40% savings with lifecycle
                    
                    optimizations.append(CostOptimization(
                        resource_type='S3',
                        resource_id=bucket_name,
                        current_cost=current_cost,
                        potential_savings=potential_savings,
                        recommendation=f"Implement lifecycle policy for {bucket_name} ({bucket_size/1000000000:.1f}GB)",
                        confidence='High',
                        implementation_effort='Low',
                        risk_level='Low'
                    ))
                
                # Check for infrequently accessed objects
                old_objects = self._get_s3_old_objects(bucket_name)
                if old_objects > 0:
                    estimated_cost = old_objects * 0.0125  # Estimate cost per object
                    potential_savings = estimated_cost * 0.6  # Savings with IA storage
                    
                    optimizations.append(CostOptimization(
                        resource_type='S3',
                        resource_id=bucket_name,
                        current_cost=estimated_cost,
                        potential_savings=potential_savings,
                        recommendation=f"Move {old_objects} old objects to IA storage class",
                        confidence='Medium',
                        implementation_effort='Low',
                        risk_level='Low'
                    ))
            
            except Exception as e:
                logger.warning(f"Could not analyze bucket {bucket_name}: {e}")
        
        return optimizations
    
    def identify_ebs_optimization_opportunities(self) -> List[CostOptimization]:
        """Identify EBS cost optimization opportunities"""
        logger.info("Analyzing EBS volumes for optimization opportunities")
        
        optimizations = []
        
        # Get all EBS volumes
        paginator = self.ec2.get_paginator('describe_volumes')
        
        for page in paginator.paginate():
            for volume in page['Volumes']:
                volume_id = volume['VolumeId']
                volume_type = volume['VolumeType']
                size = volume['Size']
                state = volume['State']
                
                # Check for unattached volumes
                if state == 'available':
                    current_cost = self._estimate_ebs_cost(volume_type, size)
                    
                    optimizations.append(CostOptimization(
                        resource_type='EBS',
                        resource_id=volume_id,
                        current_cost=current_cost,
                        potential_savings=current_cost,
                        recommendation=f"Delete unattached {volume_type} volume ({size}GB)",
                        confidence='High',
                        implementation_effort='Low',
                        risk_level='Medium'
                    ))
                
                # Check for oversized volumes
                elif state == 'in-use':
                    utilization = self._get_ebs_utilization(volume_id)
                    if utilization < 50:  # Less than 50% utilized
                        current_cost = self._estimate_ebs_cost(volume_type, size)
                        suggested_size = max(8, int(size * 0.7))  # Reduce by 30%
                        potential_savings = current_cost * 0.3
                        
                        optimizations.append(CostOptimization(
                            resource_type='EBS',
                            resource_id=volume_id,
                            current_cost=current_cost,
                            potential_savings=potential_savings,
                            recommendation=f"Resize volume from {size}GB to {suggested_size}GB (utilization: {utilization}%)",
                            confidence='Medium',
                            implementation_effort='Medium',
                            risk_level='Medium'
                        ))
        
        return optimizations
    
    def _get_ec2_cpu_utilization(self, instance_id: str) -> float:
        """Get average CPU utilization for EC2 instance"""
        try:
            end_time = datetime.utcnow()
            start_time = end_time - timedelta(days=7)
            
            response = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=start_time,
                EndTime=end_time,
                Period=3600,  # 1 hour
                Statistics=['Average']
            )
            
            if response['Datapoints']:
                return sum(dp['Average'] for dp in response['Datapoints']) / len(response['Datapoints'])
            return 0
        except Exception:
            return 50  # Default to moderate utilization if can't get metrics
    
    def _get_rds_cpu_utilization(self, db_identifier: str) -> float:
        """Get average CPU utilization for RDS instance"""
        try:
            end_time = datetime.utcnow()
            start_time = end_time - timedelta(days=7)
            
            response = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/RDS',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'DBInstanceIdentifier', 'Value': db_identifier}],
                StartTime=start_time,
                EndTime=end_time,
                Period=3600,
                Statistics=['Average']
            )
            
            if response['Datapoints']:
                return sum(dp['Average'] for dp in response['Datapoints']) / len(response['Datapoints'])
            return 0
        except Exception:
            return 50  # Default to moderate utilization
    
    def _estimate_ec2_cost(self, instance_type: str) -> float:
        """Estimate monthly cost for EC2 instance type"""
        # Simplified cost estimation (actual costs vary by region and usage)
        cost_map = {
            't2.micro': 8.5, 't2.small': 17, 't2.medium': 34,
            't3.micro': 7.6, 't3.small': 15.2, 't3.medium': 30.4,
            'm5.large': 70, 'm5.xlarge': 140, 'm5.2xlarge': 280,
            'c5.large': 62, 'c5.xlarge': 124, 'c5.2xlarge': 248
        }
        return cost_map.get(instance_type, 100)  # Default estimate
    
    def _estimate_rds_cost(self, db_class: str) -> float:
        """Estimate monthly cost for RDS instance class"""
        cost_map = {
            'db.t3.micro': 15, 'db.t3.small': 30, 'db.t3.medium': 60,
            'db.m5.large': 140, 'db.m5.xlarge': 280, 'db.m5.2xlarge': 560,
            'db.r5.large': 180, 'db.r5.xlarge': 360, 'db.r5.2xlarge': 720
        }
        return cost_map.get(db_class, 200)  # Default estimate
    
    def _estimate_ebs_cost(self, volume_type: str, size: int) -> float:
        """Estimate monthly cost for EBS volume"""
        cost_per_gb = {
            'gp2': 0.10, 'gp3': 0.08, 'io1': 0.125, 'io2': 0.125,
            'st1': 0.045, 'sc1': 0.025, 'standard': 0.05
        }
        return cost_per_gb.get(volume_type, 0.10) * size
    
    def _suggest_smaller_instance_type(self, current_type: str) -> str:
        """Suggest a smaller instance type"""
        downsize_map = {
            't3.medium': 't3.small', 't3.small': 't3.micro',
            'm5.xlarge': 'm5.large', 'm5.2xlarge': 'm5.xlarge',
            'c5.xlarge': 'c5.large', 'c5.2xlarge': 'c5.xlarge'
        }
        return downsize_map.get(current_type, current_type)
    
    def _suggest_smaller_rds_class(self, current_class: str) -> str:
        """Suggest a smaller RDS instance class"""
        downsize_map = {
            'db.m5.xlarge': 'db.m5.large', 'db.m5.2xlarge': 'db.m5.xlarge',
            'db.r5.xlarge': 'db.r5.large', 'db.r5.2xlarge': 'db.r5.xlarge',
            'db.t3.medium': 'db.t3.small', 'db.t3.small': 'db.t3.micro'
        }
        return downsize_map.get(current_class, current_class)
    
    def generate_cost_optimization_report(self) -> Dict[str, Any]:
        """Generate comprehensive cost optimization report"""
        logger.info("Generating cost optimization report")
        
        # Get all optimization opportunities
        ec2_optimizations = self.identify_ec2_optimization_opportunities()
        rds_optimizations = self.identify_rds_optimization_opportunities()
        s3_optimizations = self.identify_s3_optimization_opportunities()
        ebs_optimizations = self.identify_ebs_optimization_opportunities()
        
        all_optimizations = ec2_optimizations + rds_optimizations + s3_optimizations + ebs_optimizations
        
        # Calculate totals
        total_current_cost = sum(opt.current_cost for opt in all_optimizations)
        total_potential_savings = sum(opt.potential_savings for opt in all_optimizations)
        
        # Group by resource type
        by_resource_type = defaultdict(list)
        for opt in all_optimizations:
            by_resource_type[opt.resource_type].append(opt)
        
        # Group by confidence level
        by_confidence = defaultdict(list)
        for opt in all_optimizations:
            by_confidence[opt.confidence].append(opt)
        
        return {
            'report_timestamp': datetime.now().isoformat(),
            'summary': {
                'total_opportunities': len(all_optimizations),
                'total_current_cost': total_current_cost,
                'total_potential_savings': total_potential_savings,
                'savings_percentage': (total_potential_savings / total_current_cost * 100) if total_current_cost > 0 else 0
            },
            'by_resource_type': {
                resource_type: {
                    'count': len(optimizations),
                    'current_cost': sum(opt.current_cost for opt in optimizations),
                    'potential_savings': sum(opt.potential_savings for opt in optimizations),
                    'optimizations': [asdict(opt) for opt in optimizations]
                }
                for resource_type, optimizations in by_resource_type.items()
            },
            'by_confidence': {
                confidence: {
                    'count': len(optimizations),
                    'potential_savings': sum(opt.potential_savings for opt in optimizations)
                }
                for confidence, optimizations in by_confidence.items()
            },
            'top_opportunities': [asdict(opt) for opt in sorted(all_optimizations, key=lambda x: x.potential_savings, reverse=True)[:20]],
            'quick_wins': [asdict(opt) for opt in all_optimizations if opt.implementation_effort == 'Low' and opt.confidence == 'High'][:10]
        }

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='Enterprise FinOps Cost Analyzer')
    parser.add_argument('--action', choices=['analyze', 'optimize', 'report'], required=True)
    parser.add_argument('--period', type=int, default=30, help='Analysis period in days')
    parser.add_argument('--granularity', choices=['DAILY', 'MONTHLY'], default='DAILY')
    parser.add_argument('--output-file', default='cost-analysis.json')
    
    args = parser.parse_args()
    
    manager = EnterpriseFinOpsManager()
    
    try:
        if args.action == 'analyze':
            result = manager.analyze_costs(args.period, args.granularity)
        elif args.action == 'optimize':
            result = manager.generate_cost_optimization_report()
        elif args.action == 'report':
            cost_analysis = manager.analyze_costs(args.period, args.granularity)
            optimization_report = manager.generate_cost_optimization_report()
            result = {
                'cost_analysis': cost_analysis,
                'optimization_report': optimization_report
            }
        
        with open(args.output_file, 'w') as f:
            json.dump(result, f, indent=2, default=str)
        
        print(f"Analysis complete. Results saved to {args.output_file}")
        
        if args.action in ['optimize', 'report']:
            if 'optimization_report' in result:
                opt_report = result['optimization_report']
            else:
                opt_report = result
            
            print(f"\nOptimization Summary:")
            print(f"Total opportunities: {opt_report['summary']['total_opportunities']}")
            print(f"Potential monthly savings: ${opt_report['summary']['total_potential_savings']:.2f}")
            print(f"Savings percentage: {opt_report['summary']['savings_percentage']:.1f}%")
    
    except Exception as e:
        logger.error(f"Analysis failed: {e}")
        exit(1)

if __name__ == "__main__":
    main()
```