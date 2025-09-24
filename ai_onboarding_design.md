## 7. Deployment Scripts & Configuration

### 7.1 Environment Setup Script

```bash
#!/bin/bash
# setup-environment.sh

set -e

echo "Setting up AI-Driven Onboarding Agent Environment..."

# Create necessary directories
mkdir -p {requirements-agent,validation-agent,orchestrator-agent,integration-agent,llm-service,knowledge-service}
mkdir -p {database/init,policies,monitoring,web-frontend,certificates}

# Generate SSL certificates
echo "Generating SSL certificates..."
openssl req -x509 -newkey rsa:4096 -nodes -out certificates/cert.pem -keyout certificates/key.pem -days 365 -subj "/CN=onboarding.local"

# Create environment file
cat > .env << EOF
# Database Configuration
POSTGRES_USER=onboarding
POSTGRES_PASSWORD=$(openssl rand -base64 32)
POSTGRES_DB=onboarding_db

# Redis Configuration
REDIS_PASSWORD=$(openssl rand -base64 32)

# API Keys
OPENAI_API_KEY=your_openai_api_key_here
CLAUDE_API_KEY=your_claude_api_key_here

# External System Configurations
SERVICENOW_URL=https://your-instance.servicenow.com
SERVICENOW_USERNAME=your_servicenow_user
SERVICENOW_PASSWORD=your_servicenow_password

JIRA_URL=https://your-instance.atlassian.net
JIRA_USERNAME=your_jira_user
JIRA_API_TOKEN=your_jira_token

CONFLUENCE_URL=https://your-instance.atlassian.net/wiki
CONFLUENCE_USERNAME=your_confluence_user
CONFLUENCE_API_TOKEN=your_confluence_token

# Security Keys
JWT_SECRET_KEY=$(openssl rand -base64 64)
ENCRYPTION_KEY=$(openssl rand -base64 32)

# Monitoring
GRAFANA_ADMIN_PASSWORD=$(openssl rand -base64 16)
EOF

echo "Environment setup complete!"
echo "Please update the API keys in .env file before running docker-compose up"
```

### 7.2 Database Initialization

```sql
-- database/init/01_init_schema.sql

-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Create enum types
CREATE TYPE lob_type AS ENUM (
    'retail_banking', 'corporate_banking', 'insurance', 
    'wealth_management', 'capital_markets'
);

CREATE TYPE application_type AS ENUM (
    'web_application', 'mobile_app', 'api_service', 
    'batch_process', 'microservice'
);

CREATE TYPE request_status AS ENUM (
    'draft', 'submitted', 'under_review', 'approved', 
    'rejected', 'deployed', 'completed'
);

CREATE TYPE task_status AS ENUM (
    'pending', 'in_progress', 'completed', 'blocked', 'cancelled'
);

-- Onboarding requests table
CREATE TABLE onboarding_requests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_id VARCHAR(50) UNIQUE NOT NULL,
    user_id VARCHAR(100) NOT NULL,
    lob lob_type NOT NULL,
    application_name VARCHAR(255) NOT NULL,
    application_type application_type NOT NULL,
    business_purpose TEXT NOT NULL,
    compliance_requirements JSONB DEFAULT '[]',
    technical_requirements JSONB DEFAULT '{}',
    sla_requirements JSONB DEFAULT '{}',
    integration_endpoints JSONB DEFAULT '[]',
    data_classification VARCHAR(50) DEFAULT 'internal',
    expected_users INTEGER DEFAULT 100,
    go_live_date TIMESTAMP,
    status request_status DEFAULT 'draft',
    risk_score DECIMAL(3,2) DEFAULT 0.0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by VARCHAR(100) NOT NULL,
    updated_by VARCHAR(100)
);

-- Workflow tasks table
CREATE TABLE workflow_tasks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_id VARCHAR(50) REFERENCES onboarding_requests(request_id),
    task_id VARCHAR(100) NOT NULL,
    task_name VARCHAR(255) NOT NULL,
    assigned_team VARCHAR(100) NOT NULL,
    status task_status DEFAULT 'pending',
    dependencies JSONB DEFAULT '[]',
    estimated_hours INTEGER DEFAULT 0,
    actual_hours INTEGER,
    start_date TIMESTAMP,
    completion_date TIMESTAMP,
    assignee VARCHAR(100),
    comments JSONB DEFAULT '[]',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Compliance checks table
CREATE TABLE compliance_checks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_id VARCHAR(50) REFERENCES onboarding_requests(request_id),
    framework VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    violations JSONB DEFAULT '[]',
    recommendations JSONB DEFAULT '[]',
    risk_score DECIMAL(3,2) DEFAULT 0.0,
    details JSONB DEFAULT '{}',
    checked_at TIMESTAMP DEFAULT NOW(),
    checked_by VARCHAR(100)
);

-- Audit trail table
CREATE TABLE audit_trail (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_id VARCHAR(50),
    user_id VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id VARCHAR(100),
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- Knowledge base table
CREATE TABLE knowledge_base (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id VARCHAR(100) UNIQUE NOT NULL,
    title VARCHAR(500) NOT NULL,
    content TEXT NOT NULL,
    document_type VARCHAR(50) NOT NULL,
    source_system VARCHAR(100),
    source_url TEXT,
    tags JSONB DEFAULT '[]',
    embedding VECTOR(1536), -- For OpenAI embeddings
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Create indexes
CREATE INDEX idx_onboarding_requests_status ON onboarding_requests(status);
CREATE INDEX idx_onboarding_requests_lob ON onboarding_requests(lob);
CREATE INDEX idx_onboarding_requests_created_at ON onboarding_requests(created_at);
CREATE INDEX idx_workflow_tasks_request_id ON workflow_tasks(request_id);
CREATE INDEX idx_workflow_tasks_status ON workflow_tasks(status);
CREATE INDEX idx_workflow_tasks_assigned_team ON workflow_tasks(assigned_team);
CREATE INDEX idx_compliance_checks_request_id ON compliance_checks(request_id);
CREATE INDEX idx_compliance_checks_framework ON compliance_checks(framework);
CREATE INDEX idx_audit_trail_request_id ON audit_trail(request_id);
CREATE INDEX idx_audit_trail_timestamp ON audit_trail(timestamp);
CREATE INDEX idx_knowledge_base_document_type ON knowledge_base(document_type);
CREATE INDEX idx_knowledge_base_tags ON knowledge_base USING GIN(tags);

-- Create trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$ language 'plpgsql';

CREATE TRIGGER update_onboarding_requests_updated_at 
    BEFORE UPDATE ON onboarding_requests 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_workflow_tasks_updated_at 
    BEFORE UPDATE ON workflow_tasks 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 7.3 Policy Definitions (OPA)

```rego
# policies/onboarding_policies.rego

package onboarding

# GDPR Compliance Policy
gdpr_compliant {
    input.data_classification != "public"
    input.consent_mechanism == true
    input.data_retention_policy.defined == true
    input.data_portability_support == true
    input.deletion_capability == true
    input.purpose_limitation == true
}

# HIPAA Compliance Policy
hipaa_compliant {
    input.data_classification == "phi"
    input.encryption_at_rest == true
    input.encryption_in_transit == true
    input.access_controls.mfa_required == true
    input.audit_logging == true
    input.business_associate_agreement == true
}

# Risk Scoring Rules
high_risk {
    input.data_classification == "confidential"
    input.external_integrations_count > 5
}

high_risk {
    input.expected_users > 10000
    input.financial_data_access == true
}

medium_risk {
    input.data_classification == "internal"
    input.compliance_requirements_count > 2
}

# Team Assignment Rules
security_review_required {
    input.data_classification in ["confidential", "restricted"]
    input.external_network_access == true
}

security_review_required {
    input.application_type == "api_service"
    input.authentication_type == "none"
}

compliance_review_required {
    count(input.compliance_requirements) > 0
}

finance_approval_required {
    input.estimated_cost > 50000
}

# SLA Calculation Rules
sla_hours = 168 {  # 1 week
    input.priority == "low"
}

sla_hours = 72 {   # 3 days
    input.priority == "medium"
}

sla_hours = 24 {   # 1 day
    input.priority == "high"
}

sla_hours = 4 {    # 4 hours
    input.priority == "critical"
}

# Auto-approval Rules
auto_approve {
    input.risk_score < 0.3
    input.data_classification == "public"
    count(input.compliance_requirements) == 0
    input.estimated_cost < 10000
}
```

### 7.4 Monitoring Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'requirements-agent'
    static_configs:
      - targets: ['requirements-agent:8001']
    metrics_path: '/metrics'
    
  - job_name: 'validation-agent'
    static_configs:
      - targets: ['validation-agent:8002']
    metrics_path: '/metrics'
    
  - job_name: 'orchestrator-agent'
    static_configs:
      - targets: ['orchestrator-agent:8003']
    metrics_path: '/metrics'
    
  - job_name: 'integration-agent'
    static_configs:
      - targets: ['integration-agent:8004']
    metrics_path: '/metrics'
    
  - job_name: 'llm-service'
    static_configs:
      - targets: ['llm-service:8005']
    metrics_path: '/metrics'
    
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']
    
  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
    
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka:9092']

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

```yaml
# monitoring/alert_rules.yml
groups:
  - name: onboarding_alerts
    rules:
      - alert: HighRequestLatency
        expr: histogram_quantile(0.95, http_request_duration_seconds_bucket) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High request latency detected"
          description: "95th percentile latency is above 2 seconds"
      
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is above 10%"
      
      - alert: AgentDown
        expr: up{job=~".*-agent"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Agent service is down"
          description: "{{ $labels.job }} has been down for more than 1 minute"
      
      - alert: DatabaseConnectionIssue
        expr: up{job="postgres"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Database connection issue"
          description: "Cannot connect to PostgreSQL database"
```

## 8. Integration Connectors

### 8.1 ServiceNow Connector

```python
# integration-agent/connectors/servicenow_connector.py
import aiohttp
import json
import base64
from typing import Dict, List, Any
import asyncio

class ServiceNowConnector:
    def __init__(self, instance_url: str, username: str, password: str):
        self.instance_url = instance_url.rstrip('/')
        self.auth_header = base64.b64encode(f"{username}:{password}".encode()).decode()
        self.session = None
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession(
            headers={
                'Authorization': f'Basic {self.auth_header}',
                'Content-Type': 'application/json',
                'Accept': 'application/json'
            }
        )
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
    
    async def create_change_request(self, onboarding_data: Dict[str, Any]) -> Dict[str, Any]:
        """Create a change request in ServiceNow for the onboarding"""
        change_request_data = {
            'short_description': f"Application Onboarding: {onboarding_data['application_name']}",
            'description': self._build_change_description(onboarding_data),
            'category': 'Software',
            'type': 'Standard',
            'risk': self._calculate_servicenow_risk(onboarding_data.get('risk_score', 0)),
            'priority': self._map_priority(onboarding_data.get('priority', 'medium')),
            'requested_by': onboarding_data['user_id'],
            'business_justification': onboarding_data['business_purpose'],
            'implementation_plan': self._generate_implementation_plan(onboarding_data),
            'rollback_plan': 'Standard application rollback procedures apply',
            'test_plan': self._generate_test_plan(onboarding_data)
        }
        
        async with self.session.post(
            f"{self.instance_url}/api/now/table/change_request",
            json=change_request_data
        ) as response:
            if response.status == 201:
                result = await response.json()
                return {
                    'success': True,
                    'change_request_number': result['result']['number'],
                    'sys_id': result['result']['sys_id']
                }
            else:
                error_text = await response.text()
                return {
                    'success': False,
                    'error': f"Failed to create change request: {error_text}"
                }
    
    async def create_configuration_items(self, onboarding_data: Dict[str, Any]) -> List[Dict[str, Any]]:
        """Create configuration items for the application components"""
        ci_results = []
        
        # Main application CI
        app_ci = await self._create_application_ci(onboarding_data)
        ci_results.append(app_ci)
        
        # Database CI if required
        if onboarding_data.get('technical_requirements', {}).get('database_required'):
            db_ci = await self._create_database_ci(onboarding_data)
            ci_results.append(db_ci)
        
        # Integration CIs
        for integration in onboarding_data.get('integration_endpoints', []):
            integration_ci = await self._create_integration_ci(onboarding_data, integration)
            ci_results.append(integration_ci)
        
        return ci_results
    
    def _build_change_description(self, onboarding_data: Dict[str, Any]) -> str:
        """Build detailed change description"""
        description = f"""
        Application Onboarding Request
        
        Application: {onboarding_data['application_name']}
        Type: {onboarding_data['application_type']}
        Line of Business: {onboarding_data['lob']}
        
        Business Purpose: {onboarding_data['business_purpose']}
        
        Technical Requirements:
        {json.dumps(onboarding_data.get('technical_requirements', {}), indent=2)}
        
        Compliance Requirements: {', '.join(onboarding_data.get('compliance_requirements', []))}
        
        Expected Users: {onboarding_data.get('expected_users', 'TBD')}
        Data Classification: {onboarding_data.get('data_classification', 'TBD')}
        """
        return description.strip()
    
    def _calculate_servicenow_risk(self, risk_score: float) -> str:
        """Map risk score to ServiceNow risk levels"""
        if risk_score >= 0.7:
            return 'High'
        elif risk_score >= 0.4:
            return 'Medium'
        else:
            return 'Low'
    
    def _map_priority(self, priority: str) -> str:
        """Map internal priority to ServiceNow priority"""
        priority_mapping = {
            'critical': '1 - Critical',
            'high': '2 - High',
            'medium': '3 - Moderate',
            'low': '4 - Low'
        }
        return priority_mapping.get(priority, '3 - Moderate')
```

### 8.2 Jira Connector

```python
# integration-agent/connectors/jira_connector.py
import aiohttp
import json
from typing import Dict, List, Any

class JiraConnector:
    def __init__(self, jira_url: str, username: str, api_token: str):
        self.jira_url = jira_url.rstrip('/')
        self.auth = aiohttp.BasicAuth(username, api_token)
        self.session = None
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession(
            auth=self.auth,
            headers={
                'Content-Type': 'application/json',
                'Accept': 'application/json'
            }
        )
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
    
    async def create_epic(self, onboarding_data: Dict[str, Any]) -> Dict[str, Any]:
        """Create an epic for the onboarding project"""
        epic_data = {
            'fields': {
                'project': {'key': 'ONBOARD'},
                'summary': f"Onboard {onboarding_data['application_name']}",
                'description': self._build_epic_description(onboarding_data),
                'issuetype': {'name': 'Epic'},
                'priority': {'name': self._map_priority(onboarding_data.get('priority', 'medium'))},
                'labels': self._generate_labels(onboarding_data),
                'customfield_10011': f"onboarding-{onboarding_data['request_id']}"  # Epic Name
            }
        }
        
        async with self.session.post(
            f"{self.jira_url}/rest/api/3/issue",
            json=epic_data
        ) as response:
            if response.status == 201:
                result = await response.json()
                return {
                    'success': True,
                    'epic_key': result['key'],
                    'epic_id': result['id']
                }
            else:
                error_text = await response.text()
                return {
                    'success': False,
                    'error': f"Failed to create epic: {error_text}"
                }
    
    async def create_workflow_tasks(self, onboarding_data: Dict[str, Any], epic_key: str) -> List[Dict[str, Any]]:
        """Create Jira tasks for each workflow step"""
        tasks = []
        
        # Security review task
        if self._requires_security_review(onboarding_data):
            security_task = await self._create_task(
                epic_key,
                "Security Review",
                "Review security requirements and approve security architecture",
                "Security Team",
                onboarding_data
            )
            tasks.append(security_task)
        
        # Compliance review task
        if onboarding_data.get('compliance_requirements'):
            compliance_task = await self._create_task(
                epic_key,
                "Compliance Review",
                f"Review compliance requirements: {', '.join(onboarding_data['compliance_requirements'])}",
                "Compliance Team",
                onboarding_data
            )
            tasks.append(compliance_task)
        
        # Infrastructure setup task
        infra_task = await self._create_task(
            epic_key,
            "Infrastructure Setup",
            "Provision and configure required infrastructure",
            "Infrastructure Team",
            onboarding_data
        )
        tasks.append(infra_task)
        
        # Integration setup task
        if onboarding_data.get('integration_endpoints'):
            integration_task = await self._create_task(
                epic_key,
                "Integration Setup",
                f"Configure integrations: {', '.join(onboarding_data['integration_endpoints'])}",
                "Integration Team",
                onboarding_data
            )
            tasks.append(integration_task)
        
        return tasks
    
    async def _create_task(self, epic_key: str, summary: str, description: str, 
                          team: str, onboarding_data: Dict[str, Any]) -> Dict[str, Any]:
        """Create individual task in Jira"""
        task_data = {
            'fields': {
                'project': {'key': 'ONBOARD'},
                'summary': summary,
                'description': description,
                'issuetype': {'name': 'Task'},
                'priority': {'name': self._map_priority(onboarding_data.get('priority', 'medium'))},
                'labels': self._generate_labels(onboarding_data) + [team.lower().replace(' ', '_')],
                'customfield_10014': epic_key,  # Epic Link
                'assignee': await self._get_team_lead(team)
            }
        }
        
        async with self.session.post(
            f"{self.jira_url}/rest/api/3/issue",
            json=task_data
        ) as response:
            if response.status == 201:
                result = await response.json()
                return {
                    'success': True,
                    'task_key': result['key'],
                    'task_id': result['id'],
                    'team': team
                }
            else:
                error_text = await response.text()
                return {
                    'success': False,
                    'error': f"Failed to create task: {error_text}"
                }
    
    def _build_epic_description(self, onboarding_data: Dict[str, Any]) -> str:
        """Build detailed epic description"""
        return f"""
        h2. Application Onboarding Request
        
        *Application:* {onboarding_data['application_name']}
        *Type:* {onboarding_data['application_type']}
        *Line of Business:* {onboarding_data['lob']}
        *Request ID:* {onboarding_data['request_id']}
        
        h3. Business Purpose
        {onboarding_data['business_purpose']}
        
        h3. Technical Requirements
        {self._format_technical_requirements(onboarding_data.get('technical_requirements', {}))}
        
        h3. Compliance Requirements
        {', '.join(onboarding_data.get('compliance_requirements', ['None']))}
        
        h3. Expected Users
        {onboarding_data.get('expected_users', 'TBD')}
        
        h3. Data Classification
        {onboarding_data.get('data_classification', 'TBD')}
        """
    
    def _generate_labels(self, onboarding_data: Dict[str, Any]) -> List[str]:
        """Generate Jira labels based on onboarding data"""
        labels = [
            'onboarding',
            onboarding_data['lob'].replace('_', '-'),
            onboarding_data['application_type'].replace('_', '-'),
            onboarding_data.get('data_classification', 'unknown').replace('_', '-')
        ]
        
        # Add compliance labels
        for compliance in onboarding_data.get('compliance_requirements', []):
            labels.append(f"compliance-{compliance.lower()}")
        
        return labels
```

## 9. Frontend Implementation

### 9.1 React Application Structure

```typescript
// web-frontend/src/types/onboarding.ts
export interface OnboardingRequest {
  requestId: string;
  userId: string;
  lob: LOBType;
  applicationName: string;
  applicationType: ApplicationType;
  businessPurpose: string;
  complianceRequirements: ComplianceFramework[];
  technicalRequirements: TechnicalRequirements;
  slaRequirements: SLARequirements;
  integrationEndpoints: z.array(z.string()).optional(),
  goLiveDate: z.date().optional()
});

type OnboardingFormData = z.infer<typeof onboardingSchema>;

const { Step } = Steps;

export const OnboardingWizard: React.FC = () => {
  const [currentStep, setCurrentStep] = useState(0);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [requestId, setRequestId] = useState<string | null>(null);
  const [aiSuggestions, setAiSuggestions] = useState<any[]>([]);
  
  const methods = useForm<OnboardingFormData>({
    resolver: zodResolver(onboardingSchema),
    defaultValues: {
      technicalRequirements: {
        databaseRequired: false,
        externalApiAccess: false,
        highAvailability: false,
        loadBalancing: false
      },
      slaRequirements: {
        availability: 99.9,
        responseTime: 500,
        throughput: 100
      },
      complianceRequirements: [],
      integrationEndpoints: []
    }
  });

  // WebSocket connection for real-time AI assistance
  const { socket, isConnected } = useWebSocket('/ws/requirements');

  useEffect(() => {
    if (socket && isConnected) {
      socket.onmessage = (event) => {
        const data = JSON.parse(event.data);
        if (data.type === 'ai_suggestions') {
          setAiSuggestions(data.suggestions);
        } else if (data.type === 'validation_results') {
          // Handle real-time validation
          console.log('Validation results:', data);
        }
      };
    }
  }, [socket, isConnected]);

  const steps = [
    {
      title: 'Basic Information',
      content: <BasicInfoStep />,
      description: 'Application details and business purpose'
    },
    {
      title: 'Technical Requirements',
      content: <TechnicalRequirementsStep aiSuggestions={aiSuggestions} />,
      description: 'Infrastructure and technical specifications'
    },
    {
      title: 'Compliance & Security',
      content: <ComplianceStep />,
      description: 'Regulatory and security requirements'
    },
    {
      title: 'Review & Submit',
      content: <ReviewStep requestId={requestId} />,
      description: 'Final review and submission'
    }
  ];

  const next = async () => {
    const isStepValid = await methods.trigger();
    if (isStepValid) {
      // Send current form data to AI for suggestions
      if (socket && isConnected) {
        socket.send(JSON.stringify({
          type: 'form_update',
          step: currentStep,
          data: methods.getValues()
        }));
      }
      setCurrentStep(currentStep + 1);
    }
  };

  const prev = () => {
    setCurrentStep(currentStep - 1);
  };

  const onSubmit = async (data: OnboardingFormData) => {
    setIsSubmitting(true);
    try {
      const response = await OnboardingAPI.submitRequest(data);
      setRequestId(response.requestId);
      setCurrentStep(currentStep + 1);
    } catch (error) {
      console.error('Failed to submit request:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="onboarding-wizard">
      <Card title="Application Onboarding" className="wizard-card">
        <Steps current={currentStep} className="mb-6">
          {steps.map((step, index) => (
            <Step
              key={index}
              title={step.title}
              description={step.description}
              status={currentStep === index ? 'process' : currentStep > index ? 'finish' : 'wait'}
            />
          ))}
        </Steps>

        <FormProvider {...methods}>
          <form onSubmit={methods.handleSubmit(onSubmit)}>
            <div className="steps-content mb-6">
              {steps[currentStep]?.content}
            </div>

            <div className="steps-action flex justify-between">
              {currentStep > 0 && (
                <Button onClick={prev}>
                  Previous
                </Button>
              )}
              
              <div className="flex space-x-2">
                {currentStep < steps.length - 1 && (
                  <Button type="primary" onClick={next}>
                    Next
                  </Button>
                )}
                
                {currentStep === steps.length - 1 && (
                  <Button
                    type="primary"
                    htmlType="submit"
                    loading={isSubmitting}
                    disabled={!methods.formState.isValid}
                  >
                    Submit Request
                  </Button>
                )}
              </div>
            </div>
          </form>
        </FormProvider>
      </Card>
    </div>
  );
};
```

```tsx
// web-frontend/src/components/steps/BasicInfoStep.tsx
import React from 'react';
import { useFormContext, Controller } from 'react-hook-form';
import { Input, Select, TextArea, DatePicker, InputNumber, Card } from 'antd';
import { InfoCircleOutlined } from '@ant-design/icons';

const { Option } = Select;

export const BasicInfoStep: React.FC = () => {
  const { control, formState: { errors }, watch } = useFormContext();
  
  const selectedLOB = watch('lob');
  const selectedAppType = watch('applicationType');

  return (
    <div className="space-y-6">
      <Card title="Application Details" size="small">
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <Controller
            name="applicationName"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Application Name *
                </label>
                <Input
                  {...field}
                  placeholder="Enter application name"
                  status={errors.applicationName ? 'error' : ''}
                />
                {errors.applicationName && (
                  <span className="text-red-500 text-xs">
                    {errors.applicationName.message}
                  </span>
                )}
              </div>
            )}
          />

          <Controller
            name="applicationType"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Application Type *
                </label>
                <Select
                  {...field}
                  placeholder="Select application type"
                  className="w-full"
                  status={errors.applicationType ? 'error' : ''}
                >
                  <Option value="web_application">Web Application</Option>
                  <Option value="mobile_app">Mobile App</Option>
                  <Option value="api_service">API Service</Option>
                  <Option value="batch_process">Batch Process</Option>
                  <Option value="microservice">Microservice</Option>
                </Select>
                {errors.applicationType && (
                  <span className="text-red-500 text-xs">
                    {errors.applicationType.message}
                  </span>
                )}
              </div>
            )}
          />

          <Controller
            name="lob"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Line of Business *
                </label>
                <Select
                  {...field}
                  placeholder="Select LOB"
                  className="w-full"
                  status={errors.lob ? 'error' : ''}
                >
                  <Option value="retail_banking">Retail Banking</Option>
                  <Option value="corporate_banking">Corporate Banking</Option>
                  <Option value="insurance">Insurance</Option>
                  <Option value="wealth_management">Wealth Management</Option>
                  <Option value="capital_markets">Capital Markets</Option>
                </Select>
                {errors.lob && (
                  <span className="text-red-500 text-xs">
                    {errors.lob.message}
                  </span>
                )}
              </div>
            )}
          />

          <Controller
            name="dataClassification"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Data Classification *
                  <InfoCircleOutlined className="ml-1 text-gray-400" />
                </label>
                <Select
                  {...field}
                  placeholder="Select data classification"
                  className="w-full"
                  status={errors.dataClassification ? 'error' : ''}
                >
                  <Option value="public">Public</Option>
                  <Option value="internal">Internal</Option>
                  <Option value="confidential">Confidential</Option>
                  <Option value="restricted">Restricted</Option>
                </Select>
                {errors.dataClassification && (
                  <span className="text-red-500 text-xs">
                    {errors.dataClassification.message}
                  </span>
                )}
              </div>
            )}
          />

          <Controller
            name="expectedUsers"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Expected Users *
                </label>
                <InputNumber
                  {...field}
                  placeholder="Number of expected users"
                  className="w-full"
                  min={1}
                  status={errors.expectedUsers ? 'error' : ''}
                />
                {errors.expectedUsers && (
                  <span className="text-red-500 text-xs">
                    {errors.expectedUsers.message}
                  </span>
                )}
              </div>
            )}
          />

          <Controller
            name="goLiveDate"
            control={control}
            render={({ field }) => (
              <div>
                <label className="block text-sm font-medium mb-1">
                  Target Go-Live Date
                </label>
                <DatePicker
                  {...field}
                  placeholder="Select target date"
                  className="w-full"
                  format="YYYY-MM-DD"
                />
              </div>
            )}
          />
        </div>
      </Card>

      <Card title="Business Justification" size="small">
        <Controller
          name="businessPurpose"
          control={control}
          render={({ field }) => (
            <div>
              <label className="block text-sm font-medium mb-1">
                Business Purpose *
              </label>
              <TextArea
                {...field}
                placeholder="Describe the business purpose and value proposition of this application"
                rows={4}
                status={errors.businessPurpose ? 'error' : ''}
              />
              {errors.businessPurpose && (
                <span className="text-red-500 text-xs">
                  {errors.businessPurpose.message}
                </span>
              )}
            </div>
          )}
        />
      </Card>

      {/* AI-Powered Suggestions based on LOB and App Type */}
      {selectedLOB && selectedAppType && (
        <Card 
          title="AI Recommendations" 
          size="small"
          className="bg-blue-50 border-blue-200"
        >
          <div className="text-sm text-blue-700">
            <p className="mb-2">
              <strong>Based on your selection ({selectedLOB} - {selectedAppType}), we recommend:</strong>
            </p>
            <ul className="list-disc list-inside space-y-1">
              {selectedLOB === 'retail_banking' && (
                <>
                  <li>Consider PCI-DSS compliance for payment processing</li>
                  <li>High availability (99.9%+) typically required</li>
                  <li>Load balancing recommended for customer-facing apps</li>
                </>
              )}
              {selectedLOB === 'capital_markets' && (
                <>
                  <li>SOX compliance may be required for financial reporting</li>
                  <li>Ultra-low latency requirements for trading systems</li>
                  <li>Market data integrations commonly needed</li>
                </>
              )}
            </ul>
          </div>
        </Card>
      )}
    </div>
  );
};
```

## 10. Performance Optimization & Scaling

### 10.1 Caching Strategy

```python
# shared/cache_manager.py
import redis
import json
import pickle
from typing import Any, Optional, Union
from datetime import timedelta
import asyncio
import aioredis

class CacheManager:
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.redis_client = None
    
    async def connect(self):
        """Initialize Redis connection"""
        self.redis_client = await aioredis.from_url(self.redis_url)
    
    async def disconnect(self):
        """Close Redis connection"""
        if self.redis_client:
            await self.redis_client.close()
    
    async def get(self, key: str, default: Any = None) -> Any:
        """Get value from cache"""
        try:
            value = await self.redis_client.get(key)
            if value is None:
                return default
            return pickle.loads(value)
        except Exception as e:
            print(f"Cache get error: {e}")
            return default
    
    async def set(self, key: str, value: Any, expiry: Optional[timedelta] = None) -> bool:
        """Set value in cache with optional expiry"""
        try:
            serialized_value = pickle.dumps(value)
            if expiry:
                await self.redis_client.setex(key, int(expiry.total_seconds()), serialized_value)
            else:
                await self.redis_client.set(key, serialized_value)
            return True
        except Exception as e:
            print(f"Cache set error: {e}")
            return False
    
    async def delete(self, key: str) -> bool:
        """Delete key from cache"""
        try:
            await self.redis_client.delete(key)
            return True
        except Exception as e:
            print(f"Cache delete error: {e}")
            return False
    
    async def get_or_set(self, key: str, func, expiry: Optional[timedelta] = None) -> Any:
        """Get from cache or execute function and cache result"""
        cached_value = await self.get(key)
        if cached_value is not None:
            return cached_value
        
        # Execute function to get value
        if asyncio.iscoroutinefunction(func):
            value = await func()
        else:
            value = func()
        
        # Cache the result
        await self.set(key, value, expiry)
        return value

# Cache decorators for common patterns
def cache_result(expiry: timedelta = timedelta(hours=1), key_prefix: str = ""):
    """Decorator to cache function results"""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            cache_key = f"{key_prefix}{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            cache_manager = CacheManager("redis://redis:6379")
            await cache_manager.connect()
            
            try:
                result = await cache_manager.get_or_set(
                    cache_key,
                    lambda: func(*args, **kwargs) if not asyncio.iscoroutinefunction(func) else func(*args, **kwargs),
                    expiry
                )
                return result
            finally:
                await cache_manager.disconnect()
        
        return wrapper
    return decorator
```

### 10.2 Load Balancing Configuration

```yaml
# load-balancer/nginx.conf
upstream requirements_agent {
    least_conn;
    server requirements-agent-1:8001 weight=1 max_fails=3 fail_timeout=30s;
    server requirements-agent-2:8001 weight=1 max_fails=3 fail_timeout=30s;
    server requirements-agent-3:8001 weight=1 max_fails=3 fail_timeout=30s;
}

upstream validation_agent {
    least_conn;
    server validation-agent-1:8002 weight=1 max_fails=3 fail_timeout=30s;
    server validation-agent-2:8002 weight=1 max_fails=3 fail_timeout=30s;
    server validation-agent-3:8002 weight=1 max_fails=3 fail_timeout=30s;
}

upstream orchestrator_agent {
    least_conn;
    server orchestrator-agent-1:8003 weight=1 max_fails=3 fail_timeout=30s;
    server orchestrator-agent-2:8003 weight=1 max_fails=3 fail_timeout=30s;
}

upstream integration_agent {
    least_conn;
    server integration-agent-1:8004 weight=1 max_fails=3 fail_timeout=30s;
    server integration-agent-2:8004 weight=1 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name onboarding.company.com;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=ws:10m rate=5r/s;
    
    # Security headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;
    
    # API Routes
    location /api/v1/requirements {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://requirements_agent;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    location /api/v1/validation {
        limit_req zone=api burst=15 nodelay;
        proxy_pass http://validation_agent;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    location /api/v1/orchestration {
        limit_req zone=api burst=10 nodelay;
        proxy_pass http://orchestrator_agent;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    location /api/v1/integration {
        limit_req zone=api burst=15 nodelay;
        proxy_pass http://integration_agent;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # WebSocket connections
    location /ws/ {
        limit_req zone=ws burst=10 nodelay;
        proxy_pass http://requirements_agent;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### 10.3 Horizontal Scaling Configuration

```yaml
# docker-compose.scale.yml
version: '3.8'

services:
  # Scaled Requirements Agents
  requirements-agent-1:
    extends:
      file: docker-compose.yml
      service: requirements-agent
    container_name: requirements-agent-1
    ports:
      - "8011:8001"

  requirements-agent-2:
    extends:
      file: docker-compose.yml
      service: requirements-agent
    container_name: requirements-agent-2
    ports:
      - "8012:8001"

  requirements-agent-3:
    extends:
      file: docker-compose.yml
      service: requirements-agent
    container_name: requirements-agent-3
    ports:
      - "8013:8001"

  # Scaled Validation Agents
  validation-agent-1:
    extends:
      file: docker-compose.yml
      service: validation-agent
    container_name: validation-agent-1
    ports:
      - "8021:8002"

  validation-agent-2:
    extends:
      file: docker-compose.yml
      service: validation-agent
    container_name: validation-agent-2
    ports:
      - "8022:8002"

  validation-agent-3:
    extends:
      file: docker-compose.yml
      service: validation-agent
    container_name: validation-agent-3
    ports:
      - "8023:8002"

  # Load Balancer
  nginx-lb:
    image: nginx:alpine
    container_name: onboarding-lb
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./load-balancer/nginx.conf:/etc/nginx/nginx.conf
      - ./certificates:/etc/nginx/certs
    depends_on:
      - requirements-agent-1
      - requirements-agent-2
      - requirements-agent-3
      - validation-agent-1
      - validation-agent-2
      - validation-agent-3
    networks:
      - onboarding-network
    restart: unless-stopped

  # Database Read Replicas
  postgres-replica-1:
    image: postgres:15
    container_name: postgres-replica-1
    environment:
      POSTGRES_DB: onboarding_db
      POSTGRES_USER: onboarding
      POSTGRES_PASSWORD: password
      POSTGRES_MASTER_SERVICE: postgres
    volumes:
      - postgres_replica_1_data:/var/lib/postgresql/data
    networks:
      - onboarding-network
    restart: unless-stopped

  # Redis Cluster
  redis-cluster:
    image: redis:7-alpine
    container_name: redis-cluster
    ports:
      - "7000-7005:7000-7005"
    command: >
      sh -c "
      redis-server --port 7000 --cluster-enabled yes --cluster-config-file nodes-7000.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7000.aof --dbfilename dump-7000.rdb --logfile redis-7000.log --daemonize no &
      redis-server --port 7001 --cluster-enabled yes --cluster-config-file nodes-7001.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7001.aof --dbfilename dump-7001.rdb --logfile redis-7001.log --daemonize no &
      redis-server --port 7002 --cluster-enabled yes --cluster-config-file nodes-7002.conf --cluster-node-timeout 5000 --appendonly yes --appendfilename appendonly-7002.aof --dbfilename dump-7002.rdb --logfile redis-7002.log --daemonize no &
      wait
      "
    volumes:
      - redis_cluster_data:/data
    networks:
      - onboarding-network
    restart: unless-stopped

volumes:
  postgres_replica_1_data:
  redis_cluster_data:
```

## 11. Comprehensive Testing Strategy

### 11.1 Unit Testing Framework

```python
# tests/test_requirements_agent.py
import pytest
import asyncio
from unittest.mock import Mock, AsyncMock, patch
from requirements_agent.main import RequirementsGatheringAgent

@pytest.fixture
def mock_redis():
    with patch('redis.Redis.from_url') as mock:
        yield mock

@pytest.fixture
def mock_openai():
    with patch('langchain.llms.OpenAI') as mock:
        yield mock

@pytest.fixture
def requirements_agent(mock_redis, mock_openai):
    return RequirementsGatheringAgent()

@pytest.mark.asyncio
async def test_process_onboarding_request_success(requirements_agent):
    """Test successful onboarding request processing"""
    request_data = {
        "request_id": "test-001",
        "user_id": "user123",
        "application_name": "Test App",
        "application_type": "web_application",
        "lob": "retail_banking",
        "business_purpose": "Test application for customer onboarding"
    }
    
    with patch.object(requirements_agent, 'store_request', new=AsyncMock()) as mock_store:
        with patch.object(requirements_agent, 'generate_questionnaire', new=AsyncMock()) as mock_generate:
            mock_generate.return_value = {"fields": [{"name": "test", "type": "text"}]}
            
            result = await requirements_agent.process_onboarding_request(request_data)
            
            assert result["request_id"] == "test-001"
            assert result["status"] == "pending_user_input"
            assert "form_schema" in result
            mock_store.assert_called_once_with("test-001", request_data)

@pytest.mark.asyncio
async def test_dynamic_form_generation(requirements_agent):
    """Test AI-powered dynamic form generation"""
    requirements = "Mobile banking application with biometric authentication"
    
    with patch.object(requirements_agent.llm, '__call__') as mock_llm:
        mock_llm.return_value = '{"security_requirements": ["biometric", "encryption"]}'
        
        tool = requirements_agent.create_dynamic_form_tool()
        result = tool.func(requirements)
        
        assert "security_requirements" in result
        mock_llm.assert_called_once()

@pytest.mark.asyncio 
async def test_validation_error_handling(requirements_agent):
    """Test error handling in validation scenarios"""
    invalid_request = {
        "request_id": "",  # Invalid empty request_id
        "user_id": "user123"
    }
    
    with pytest.raises(Exception):
        await requirements_agent.process_onboarding_request(invalid_request)

class TestComplianceChecker:
    @pytest.fixture
    def compliance_checker(self):
        from validation_agent.main import ComplianceChecker
        return ComplianceChecker()
    
    @pytest.mark.asyncio
    async def test_gdpr_compliance_check(self, compliance_checker):
        """Test GDPR compliance validation"""
        request_data = {
            "data_processing_purpose": "Customer onboarding",
            "consent_mechanism": True,
            "data_retention_policy": {"defined": True, "period": "7 years"},
            "data_portability_support": True,
            "deletion_capability": True
        }
        
        result = await compliance_checker.check_gdpr_compliance(request_data)
        
        assert result["framework"] == "GDPR"
        assert result["status"] == "compliant"
        assert len(result["violations"]) == 0
    
    @pytest.mark.asyncio
    async def test_risk_score_calculation(self, compliance_checker):
        """Test risk score calculation logic"""
        high_risk_request = {
            "data_classification": "confidential",
            "external_integrations_count": 10,
            "financial_data_access": True,
            "expected_users": 50000
        }
        
        with patch.object(compliance_checker, 'calculate_risk_score') as mock_calc:
            mock_calc.return_value = 0.85
            
            result = await compliance_checker.validate_request({
                "request_id": "test-001",
                "required_compliance": ["GDPR", "PCI_DSS"],
                **high_risk_request
            })
            
            assert result["risk_score"] >= 0.7  # High risk threshold

@pytest.mark.integration
class TestWorkflowIntegration:
    """Integration tests for workflow orchestration"""
    
    @pytest.mark.asyncio
    async def test_end_to_end_workflow(self):
        """Test complete workflow from request to task creation"""
        # This would test the entire flow through all agents
        request_data = {
            "request_id": "integration-test-001",
            "user_id": "user123",
            "application_name": "Integration Test App",
            "application_type": "web_application",
            "lob": "retail_banking",
            "business_purpose": "Test end-to-end workflow",
            "compliance_requirements": ["GDPR", "PCI_DSS"],
            "data_classification": "confidential"
        }
        
        # Test would involve:
        # 1. Submit to requirements agent
        # 2. Validate through compliance agent  
        # 3. Route through orchestrator
        # 4. Verify task creation in integration systems
        
        # Mock external system responses
        with patch('httpx.AsyncClient.post') as mock_post:
            mock_post.return_value.status_code = 201
            mock_post.return_value.json.return_value = {"id": "task-123"}
            
            # Execute workflow
            # ... test implementation
            
            assert True  # Placeholder for actual assertions
```

### 11.2 Performance Testing

```python
# tests/performance/load_test.py
import asyncio
import aiohttp
import time
import statistics
from concurrent.futures import ThreadPoolExecutor
import json

class LoadTester:
    def __init__(self, base_url: str, max_concurrent: int = 100):
        self.base_url = base_url
        self.max_concurrent = max_concurrent
        self.results = []
    
    async def make_request(self, session: aiohttp.ClientSession, endpoint: str, data: dict = None):
        """Make a single HTTP request and measure response time"""
        start_time = time.time()
        try:
            if data:
                async with session.post(f"{self.base_url}{endpoint}", json=data) as response:
                    await response.text()
                    status = response.status
            else:
                async with session.get(f"{self.base_url}{endpoint}") as response:
                    await response.text()
                    status = response.status
            
            end_time = time.time()
            return {
                'duration': end_time - start_time,
                'status': status,
                'success': 200 <= status < 400
            }
        except Exception as e:
            end_time = time.time()
            return {
                'duration': end_time - start_time,
                'status': 0,
                'success': False,
                'error': str(e)
            }
    
    async def run_load_test(self, endpoint: str, num_requests: int, request_data: dict = None):
        """Run load test against specified endpoint"""
        semaphore = asyncio.Semaphore(self.max_concurrent)
        
        async def bounded_request(session):
            async with semaphore:
                return await self.make_request(session, endpoint, request_data)
        
        async with aiohttp.ClientSession() as session:
            tasks = [bounded_request(session) for _ in range(num_requests)]
            results = await asyncio.gather(*tasks)
            
        self.results = results
        return self.analyze_results()
    
    def analyze_results(self):
        """Analyze load test results"""
        successful_requests = [r for r in self.results if r['success']]
        failed_requests = [r for r in self.results if not r['success']]
        
        durations = [r['duration'] for r in successful_requests]
        
        if not durations:
            return {
                'success_rate': 0,
                'total_requests': len(self.results),
                'failed_requests': len(failed_requests),
                'error': 'No successful requests'
            }
        
        return {
            'total_requests': len(self.results),
            'successful_requests': len(successful_requests),
            'failed_requests': len(failed_requests),
            'success_rate': len(successful_requests) / len(self.results) * 100,
            'avg_response_time': statistics.mean(durations),
            'median_response_time': statistics.median(durations),
            'p95_response_time': self.percentile(durations, 95),
            'p99_response_time': self.percentile(durations, 99),
            'min_response_time': min(durations),
            'max_response_time': max(durations),
            'requests_per_second': len(successful_requests) / sum(durations) if durations else 0
        }
    
    @staticmethod
    def percentile(data, percentile):
        """Calculate percentile of data"""
        sorted_data = sorted(data)
        index = int(len(sorted_data) * percentile / 100)
        return sorted_data[min(index, len(sorted_data) - 1)]

# Performance test scenarios
async def test_requirements_agent_performance():
    """Test requirements gathering agent under load"""
    tester = LoadTester("http://localhost:8001", max_concurrent=50)
    
    test_data = {
        "request_id": f"perf-test-{int(time.time())}",
        "user_id": "perf-user",
        "application_name": "Performance Test App",
        "application_type": "web_application",
        "lob": "retail_banking",
        "business_purpose": "Performance testing application"
    }
    
    results = await tester.run_load_test("/api/v1/start-onboarding", 1000, test_data)
    
    print("Requirements Agent Performance Results:")
    print(f"Success Rate: {results['success_rate']:.2f}%")
    print(f"Average Response Time: {results['avg_response_time']:.3f}s")
    print(f"95th Percentile: {results['p95_response_time']:.3f}s")
    print(f"Requests/Second: {results['requests_per_second']:.2f}")
    
    # Performance assertions
    assert results['success_rate'] > 95, f"Success rate too low: {results['success_rate']}"
    assert results['p95_response_time'] < 2.0, f"95th percentile too high: {results['p95_response_time']}"

if __name__ == "__main__":
    asyncio.run(test_requirements_agent_performance())
```

## 12. Disaster Recovery & Business Continuity

### 12.1 Backup Strategy

```bash
#!/bin/bash
# scripts/backup.sh

set -e

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
POSTGRES_CONTAINER="onboarding-postgres"
REDIS_CONTAINER="onboarding-redis"
ELASTICSEARCH_CONTAINER="onboarding-elasticsearch"

echo "Starting backup process at $(date)"

# Create backup directory
mkdir -p ${BACKUP_DIR}/${DATE}

# PostgreSQL Backup
echo "Backing up PostgreSQL database..."
docker exec ${POSTGRES_CONTAINER} pg_dump -U onboarding -d onboarding_db > ${BACKUP_DIR}/${DATE}/postgres_backup.sql
gzip ${BACKUP_DIR}/${DATE}/postgres_backup.sql

# Redis Backup
echo "Backing up Redis data..."
docker exec ${REDIS_CONTAINER} redis-cli BGSAVE
sleep 5  # Wait for background save to complete
docker cp ${REDIS_CONTAINER}:/data/dump.rdb ${BACKUP_DIR}/${DATE}/redis_backup.rdb

# Elasticsearch Backup
echo "Backing up Elasticsearch indices..."
docker exec ${ELASTICSEARCH_CONTAINER} \
    curl -X PUT "localhost:9200/_snapshot/backup_repo/${DATE}?wait_for_completion=true" \
    -H 'Content-Type: application/json' \
    -d '{"indices": "*","ignore_unavailable": true,"include_global_state": false}'

# Application configuration backup
echo "Backing up application configurations..."
tar -czf ${BACKUP_DIR}/${DATE}/config_backup.tar.gz \
    docker-compose.yml \
    .env \
    policies/ \
    monitoring/ \
    certificates/

# Knowledge base backup (if using local storage)
if [ -d "knowledge-service/data" ]; then
    echo "Backing up knowledge base..."
    tar -czf ${BACKUP_DIR}/${DATE}/knowledge_backup.tar.gz knowledge-service/data/
fi

# Upload to cloud storage (example with AWS S3)
if command -v aws &> /dev/null; then
    echo "Uploading backup to S3..."
    aws s3 sync ${BACKUP_DIR}/${DATE} s3://onboarding-backups/${DATE}/ --delete
fi

# Cleanup old backups (keep last 30 days)
find ${BACKUP_DIR} -type d -mtime +30 -exec rm -rf {} +

echo "Backup completed successfully at $(date)"
```

### 12.2 Disaster Recovery Procedures

```yaml
# disaster-recovery/docker-compose.dr.yml
version: '3.8'

services:
  # Disaster Recovery PostgreSQL
  postgres-dr:
    image: postgres:15
    container_name: postgres-dr
    environment:
      POSTGRES_DB: onboarding_db
      POSTGRES_USER: onboarding
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_dr_data:/var/lib/postgresql/data
      - ./disaster-recovery/restore:/docker-entrypoint-initdb.d
    networks:
      - onboarding-dr-network
    command: >
      postgres
      -c wal_level=replica
      -c max_wal_senders=3
      -c checkpoint_completion_target=0.9
      -c wal_compression=on
      -c shared_preload_libraries=pg_stat_statements
    restart: unless-stopped

  # Disaster Recovery Redis
  redis-dr:
    image: redis:7-alpine
    container_name: redis-dr
    volumes:
      - redis_dr_data:/data
      - ./disaster-recovery/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    networks:
      - onboarding-dr-network
    restart: unless-stopped

  # Application services in DR mode
  requirements-agent-dr:
    build: ./requirements-agent
    container_name: requirements-agent-dr
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres-dr:5432/onboarding_db
      - REDIS_URL=redis://redis-dr:6379
      - DR_MODE=true
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - postgres-dr
      - redis-dr
    networks:
      - onboarding-dr-network
    restart: unless-stopped

volumes:
  postgres_dr_data:
  redis_dr_data:

networks:
  onboarding-dr-network:
    driver: bridge
```

```bash
#!/bin/bash
# disaster-recovery/restore.sh

set -e

BACKUP_DATE=$1
BACKUP_DIR="/backups/${BACKUP_DATE}"

if [ -z "$BACKUP_DATE" ]; then
    echo "Usage: $0 <backup_date>"
    echo "Example: $0 20241201_120000"
    exit 1
fi

if [ ! -d "$BACKUP_DIR" ]; then
    echo "Backup directory ${BACKUP_DIR} not found"
    exit 1
fi

echo "Starting disaster recovery restore from ${BACKUP_DATE}"

# Stop current services
echo "Stopping current services..."
docker-compose down

# Restore PostgreSQL
echo "Restoring PostgreSQL database..."
docker-compose -f disaster-recovery/docker-compose.dr.yml up -d postgres-dr
sleep 10  # Wait for PostgreSQL to start

gunzip -c ${BACKUP_DIR}/postgres_backup.sql.gz | \
docker exec -i postgres-dr psql -U onboarding -d onboarding_db

# Restore Redis
echo "Restoring Redis data..."
docker cp ${BACKUP_DIR}/redis_backup.rdb postgres-dr:/data/dump.rdb
docker-compose -f disaster-recovery/docker-compose.dr.yml restart redis-dr

# Restore configurations
echo "Restoring configurations..."
if [ -f "${BACKUP_DIR}/config_backup.tar.gz" ]; then
    tar -xzf ${BACKUP_DIR}/config_backup.tar.gz
fi

# Restore knowledge base
if [ -f "${BACKUP_DIR}/knowledge_backup.tar.gz" ]; then
    echo "Restoring knowledge base..."
    tar -xzf ${BACKUP_DIR}/knowledge_backup.tar.gz
fi

# Start disaster recovery services
echo "Starting disaster recovery services..."
docker-compose -f disaster-recovery/docker-compose.dr.yml up -d

# Health checks
echo "Performing health checks..."
sleep 30

# Check database connectivity
if docker exec postgres-dr pg_isready -U onboarding; then
    echo "✓ PostgreSQL is healthy"
else
    echo "✗ PostgreSQL health check failed"
    exit 1
fi

# Check Redis connectivity
if docker exec redis-dr redis-cli ping | grep -q PONG; then
    echo "✓ Redis is healthy"
else
    echo "✗ Redis health check failed"
    exit 1
fi

# Check application services
if curl -f http://localhost:8001/health > /dev/null 2>&1; then
    echo "✓ Requirements agent is healthy"
else
    echo "✗ Requirements agent health check failed"
fi

echo "Disaster recovery completed successfully!"
echo "Services are running in DR mode"
echo "Remember to:"
echo "1. Update DNS to point to DR environment"
echo "2. Notify stakeholders of the recovery"
echo "3. Monitor system performance"
```

### 12.3 High Availability Configuration

```yaml
# ha/docker-compose.ha.yml
version: '3.8'

services:
  # PostgreSQL Primary-Replica Setup
  postgres-primary:
    image: postgres:15
    container_name: postgres-primary
    environment:
      POSTGRES_DB: onboarding_db
      POSTGRES_USER: onboarding
      POSTGRES_PASSWORD: password
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: replicator_pass
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./ha/postgresql-primary.conf:/etc/postgresql/postgresql.conf
      - ./ha/pg_hba.conf:/etc/postgresql/pg_hba.conf
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  postgres-replica:
    image: postgres:15
    container_name: postgres-replica
    environment:
      PGUSER: replicator
      POSTGRES_PASSWORD: replicator_pass
      POSTGRES_PRIMARY_HOST: postgres-primary
      POSTGRES_PRIMARY_PORT: 5432
      POSTGRES_DB: onboarding_db
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
    command: >
      bash -c "
      until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replica_slot --host=postgres-primary --port=5432 -U replicator
      do
        echo 'Waiting for primary to connect...'
        sleep 1s
      done
      echo 'Backup done, starting replica...'
      postgres
      "
    depends_on:
      - postgres-primary
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  # Redis Cluster for HA
  redis-node-1:
    image: redis:7-alpine
    container_name: redis-node-1
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7000
    ports:
      - "7000:7000"
    volumes:
      - redis_node_1_data:/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  redis-node-2:
    image: redis:7-alpine
    container_name: redis-node-2
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7001
    ports:
      - "7001:7001"
    volumes:
      - redis_node_2_data:/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  redis-node-3:
    image: redis:7-alpine
    container_name: redis-node-3
    command: >
      redis-server
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --appendonly yes
      --port 7002
    ports:
      - "7002:7002"
    volumes:
      - redis_node_3_data:/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  # Elasticsearch Cluster
  elasticsearch-master:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: es-master
    environment:
      - node.name=es-master
      - cluster.name=onboarding-cluster
      - discovery.seed_hosts=es-data1,es-data2
      - cluster.initial_master_nodes=es-master
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - node.roles=master
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_master_data:/usr/share/elasticsearch/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  elasticsearch-data1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: es-data1
    environment:
      - node.name=es-data1
      - cluster.name=onboarding-cluster
      - discovery.seed_hosts=es-master,es-data2
      - cluster.initial_master_nodes=es-master
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - node.roles=data,ingest
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data1_data:/usr/share/elasticsearch/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  elasticsearch-data2:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: es-data2
    environment:
      - node.name=es-data2
      - cluster.name=onboarding-cluster
      - discovery.seed_hosts=es-master,es-data1
      - cluster.initial_master_nodes=es-master
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - node.roles=data,ingest
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data2_data:/usr/share/elasticsearch/data
    networks:
      - onboarding-ha-network
    restart: unless-stopped

  # Application Services with Health Checks
  requirements-agent-ha:
    build: ./requirements-agent
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres-primary:5432/onboarding_db
      - DATABASE_REPLICA_URL=postgresql://onboarding:password@postgres-replica:5432/onboarding_db
      - REDIS_CLUSTER_URLS=redis://redis-node-1:7000,redis://redis-node-2:7001,redis://redis-node-3:7002
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - onboarding-ha-network

volumes:
  postgres_primary_data:
  postgres_replica_data:
  redis_node_1_data:
  redis_node_2_data:
  redis_node_3_data:
  es_master_data:
  es_data1_data:
  es_data2_data:

networks:
  onboarding-ha-network:
    driver: overlay
    attachable: true
```

## 13. Conclusion & Next Steps

This comprehensive end-to-end design provides a robust, scalable AI-driven onboarding agent system that addresses enterprise requirements across multiple dimensions:

### **Key Strengths of the Architecture:**

1. **Microservices-Based Design**: Each AI agent is independently deployable and scalable
2. **Comprehensive Docker Integration**: Full containerization with proper orchestration
3. **Real-time Processing**: WebSocket connections for interactive AI assistance
4. **Enterprise Integration**: Native connectors for ServiceNow, Jira, Confluence, and other systems
5. **Compliance-First Approach**: Built-in validation for GDPR, HIPAA, SOX, and PCI-DSS
6. **High Availability & DR**: Robust backup, recovery, and failover mechanisms
7. **Performance Optimization**: Caching, load balancing, and horizontal scaling capabilities

### **Implementation Roadmap:**

**Phase 1 (Weeks 1-4): Core Infrastructure**
- Set up Docker containers and basic services
- Implement PostgreSQL schema and Redis caching
- Deploy basic AI agents (Requirements & Validation)

**Phase 2 (Weeks 5-8): AI Agent Development**
- Complete all four AI agents with LLM integration
- Implement workflow orchestration with Temporal
- Add basic web interface

**Phase 3 (Weeks 9-12): Enterprise Integration**
- Develop ServiceNow and Jira connectors
- Implement knowledge base integration
- Add comprehensive monitoring and alerting

**Phase 4 (Weeks 13-16): Security & Compliance**
- Complete RBAC implementation
- Add comprehensive audit logging
- Implement data encryption and security controls

**Phase 5 (Weeks 17-20): Performance & Scaling**
- Implement load balancing and horizontal scaling
- Add performance monitoring and optimization
- Complete disaster recovery procedures

### **Success Metrics:**
- **Onboarding Speed**: 50-70% reduction in time-to-production
- **Compliance Coverage**: 100% automated validation for required frameworks
- **User Adoption**: >80% of new applications using the automated system
- **System Availability**: 99.9% uptime with <2 second response times
- **Cost Reduction**: 40-60% decrease in manual processing costs

This design provides TCS and its customers with a competitive advantage through intelligent automation, ensuring faster, more compliant, and more consistent application onboarding across all lines of business.dpoints: string[];
  dataClassification: DataClassification;
  expectedUsers: number;
  goLiveDate?: Date;
  status: RequestStatus;
  riskScore: number;
  createdAt: Date;
  updatedAt: Date;
}

export enum LOBType {
  RETAIL_BANKING = 'retail_banking',
  CORPORATE_BANKING = 'corporate_banking',
  INSURANCE = 'insurance',
  WEALTH_MANAGEMENT = 'wealth_management',
  CAPITAL_MARKETS = 'capital_markets'
}

export enum ApplicationType {
  WEB_APPLICATION = 'web_application',
  MOBILE_APP = 'mobile_app',
  API_SERVICE = 'api_service',
  BATCH_PROCESS = 'batch_process',
  MICROSERVICE = 'microservice'
}

export enum RequestStatus {
  DRAFT = 'draft',
  SUBMITTED = 'submitted',
  UNDER_REVIEW = 'under_review',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  DEPLOYED = 'deployed',
  COMPLETED = 'completed'
}
```

```tsx
// web-frontend/src/components/OnboardingWizard.tsx
import React, { useState, useEffect } from 'react';
import { useForm, FormProvider } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Steps, Button, Card, Alert } from 'antd';
import { BasicInfoStep } from './steps/BasicInfoStep';
import { TechnicalRequirementsStep } from './steps/TechnicalRequirementsStep';
import { ComplianceStep } from './steps/ComplianceStep';
import { ReviewStep } from './steps/ReviewStep';
import { useWebSocket } from '../hooks/useWebSocket';
import { OnboardingAPI } from '../services/api';

const onboardingSchema = z.object({
  applicationName: z.string().min(1, 'Application name is required'),
  applicationType: z.enum(['web_application', 'mobile_app', 'api_service', 'batch_process', 'microservice']),
  lob: z.enum(['retail_banking', 'corporate_banking', 'insurance', 'wealth_management', 'capital_markets']),
  businessPurpose: z.string().min(10, 'Business purpose must be at least 10 characters'),
  dataClassification: z.enum(['public', 'internal', 'confidential', 'restricted']),
  expectedUsers: z.number().min(1),
  complianceRequirements: z.array(z.string()).optional(),
  technicalRequirements: z.object({
    databaseRequired: z.boolean(),
    databaseType: z.string().optional(),
    externalApiAccess: z.boolean(),
    highAvailability: z.boolean(),
    loadBalancing: z.boolean(),
    estimatedStorage: z.number().optional(),
    estimatedBandwidth: z.number().optional()
  }),
  slaRequirements: z.object({
    availability: z.number().min(90).max(100),
    responseTime: z.number().min(100),
    throughput: z.number().min(1)
  }),
  integrationEn# AI-Driven Onboarding Agent - End-to-End Detailed Design

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PRESENTATION LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  Web Portal    │  Mobile App   │  API Gateway   │  Webhook Endpoints │  Slack Bot │
│  (React SPA)   │  (React Native)│  (Kong/Nginx) │  (FastAPI)         │  (Bot SDK) │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        AI AGENT ORCHESTRATION LAYER                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ │
│ │ Requirements    │ │ Validation      │ │ Workflow        │ │ Integration     │ │
│ │ Gathering       │ │ & Compliance    │ │ Orchestrator    │ │ Manager         │ │
│ │ Agent           │ │ Agent           │ │ Agent           │ │ Agent           │ │
│ └─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          CORE SERVICES LAYER                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│ │ LLM Service │ │ Knowledge   │ │ Policy      │ │ Notification│ │ Audit &     │ │
│ │ (OpenAI/    │ │ Base        │ │ Engine      │ │ Service     │ │ Logging     │ │
│ │ Claude API) │ │ (LlamaIndex)│ │ (REGO/OPA)  │ │ (SMTP/Slack)│ │ Service     │ │
│ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           DATA & INTEGRATION LAYER                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│ │PostgreSQL│ │ Redis    │ │Elasticsearch│ Kafka    │ │MinIO     │ │ Vector   │ │
│ │Database  │ │ Cache    │ │ Search   │ │ Message  │ │ Object   │ │ Database │ │
│ │          │ │          │ │ Engine   │ │ Queue    │ │ Storage  │ │(Pinecone)│ │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL INTEGRATIONS                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│ ServiceNow │ Jira │ Confluence │ Active Directory │ MuleSoft │ CI/CD Pipelines │
│ Connector  │ API  │ API        │ LDAP            │ ESB      │ (Jenkins/GitLab)│
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 2. AI Agent Architecture Details

### 2.1 Requirements Gathering Agent

**Purpose**: Intelligent collection and validation of onboarding requirements

**Components**:
- **Conversation Manager**: Natural language processing for user interactions
- **Context Analyzer**: Understands LOB-specific requirements
- **Dynamic Form Generator**: Creates adaptive questionnaires
- **Validation Engine**: Real-time input validation

**Docker Container**: `onboarding-requirements-agent`

```yaml
# requirements-agent/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8001
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

**Key Technologies**:
- FastAPI for REST endpoints
- LangChain for LLM orchestration
- Pydantic for data validation
- asyncio for concurrent processing

### 2.2 Validation & Compliance Agent

**Purpose**: Automated compliance checking and risk assessment

**Components**:
- **Policy Rule Engine**: REGO-based policy evaluation
- **Regulatory Checker**: GDPR, HIPAA, SOX, PCI-DSS validation
- **Risk Scorer**: ML-based risk assessment
- **Conflict Detector**: Identifies conflicting requirements

**Docker Container**: `onboarding-validation-agent`

### 2.3 Workflow Orchestrator Agent

**Purpose**: Intelligent routing and task coordination across LOBs

**Components**:
- **Dependency Graph Builder**: Creates task dependency maps
- **Team Router**: AI-powered team assignment
- **SLA Monitor**: Tracks and escalates delays
- **Parallel Executor**: Manages concurrent workflows

**Docker Container**: `onboarding-orchestrator-agent`

### 2.4 Integration Manager Agent

**Purpose**: Seamless integration with enterprise systems

**Components**:
- **Connector Factory**: Dynamic connector instantiation
- **Data Transformer**: Format conversion and mapping
- **API Gateway**: Unified interface for external systems
- **Error Handler**: Robust error recovery mechanisms

**Docker Container**: `onboarding-integration-agent`

## 3. Docker Compose Architecture

```yaml
# docker-compose.yml
version: '3.8'

services:
  # API Gateway
  api-gateway:
    image: kong:3.4
    container_name: onboarding-gateway
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
      - "8444:8444"
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: postgres
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
    depends_on:
      - postgres
    networks:
      - onboarding-network

  # Requirements Gathering Agent
  requirements-agent:
    build: ./requirements-agent
    container_name: requirements-agent
    ports:
      - "8001:8001"
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres:5432/onboarding_db
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      - postgres
      - redis
      - kafka
    volumes:
      - ./requirements-agent/config:/app/config
    networks:
      - onboarding-network
    restart: unless-stopped

  # Validation & Compliance Agent
  validation-agent:
    build: ./validation-agent
    container_name: validation-agent
    ports:
      - "8002:8002"
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres:5432/onboarding_db
      - REDIS_URL=redis://redis:6379
      - OPA_SERVER_URL=http://opa:8181
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      - postgres
      - redis
      - opa
      - kafka
    volumes:
      - ./validation-agent/policies:/app/policies
    networks:
      - onboarding-network
    restart: unless-stopped

  # Workflow Orchestrator Agent
  orchestrator-agent:
    build: ./orchestrator-agent
    container_name: orchestrator-agent
    ports:
      - "8003:8003"
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres:5432/onboarding_db
      - REDIS_URL=redis://redis:6379
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - TEMPORAL_SERVER_URL=temporal:7233
    depends_on:
      - postgres
      - redis
      - kafka
      - temporal
    networks:
      - onboarding-network
    restart: unless-stopped

  # Integration Manager Agent
  integration-agent:
    build: ./integration-agent
    container_name: integration-agent
    ports:
      - "8004:8004"
    environment:
      - DATABASE_URL=postgresql://onboarding:password@postgres:5432/onboarding_db
      - REDIS_URL=redis://redis:6379
      - KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - SERVICENOW_URL=${SERVICENOW_URL}
      - SERVICENOW_USERNAME=${SERVICENOW_USERNAME}
      - SERVICENOW_PASSWORD=${SERVICENOW_PASSWORD}
      - JIRA_URL=${JIRA_URL}
      - JIRA_USERNAME=${JIRA_USERNAME}
      - JIRA_API_TOKEN=${JIRA_API_TOKEN}
    depends_on:
      - postgres
      - redis
      - kafka
    networks:
      - onboarding-network
    restart: unless-stopped

  # LLM Service
  llm-service:
    build: ./llm-service
    container_name: llm-service
    ports:
      - "8005:8005"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - CLAUDE_API_KEY=${CLAUDE_API_KEY}
      - REDIS_URL=redis://redis:6379
      - VECTOR_DB_URL=http://pinecone:8080
    depends_on:
      - redis
    networks:
      - onboarding-network
    restart: unless-stopped

  # Knowledge Base Service (LlamaIndex)
  knowledge-service:
    build: ./knowledge-service
    container_name: knowledge-service
    ports:
      - "8006:8006"
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - VECTOR_DB_URL=http://pinecone:8080
      - CONFLUENCE_URL=${CONFLUENCE_URL}
      - CONFLUENCE_USERNAME=${CONFLUENCE_USERNAME}
      - CONFLUENCE_API_TOKEN=${CONFLUENCE_API_TOKEN}
    depends_on:
      - elasticsearch
    volumes:
      - ./knowledge-service/data:/app/data
    networks:
      - onboarding-network
    restart: unless-stopped

  # Policy Engine (Open Policy Agent)
  opa:
    image: openpolicyagent/opa:latest
    container_name: opa-server
    ports:
      - "8181:8181"
    command:
      - "run"
      - "--server"
      - "/policies"
    volumes:
      - ./policies:/policies
    networks:
      - onboarding-network
    restart: unless-stopped

  # Workflow Engine (Temporal)
  temporal:
    image: temporalio/temporal:1.20.0
    container_name: temporal-server
    ports:
      - "7233:7233"
      - "8080:8080"
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=postgres
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development.yaml
    depends_on:
      - postgres
    networks:
      - onboarding-network
    restart: unless-stopped

  # PostgreSQL Database
  postgres:
    image: postgres:15
    container_name: onboarding-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: onboarding_db
      POSTGRES_USER: onboarding
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d
    networks:
      - onboarding-network
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: onboarding-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - onboarding-network
    restart: unless-stopped

  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: onboarding-elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - onboarding-network
    restart: unless-stopped

  # Apache Kafka
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    container_name: onboarding-kafka
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,PLAINTEXT_HOST://0.0.0.0:29092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: true
    depends_on:
      - zookeeper
    networks:
      - onboarding-network
    restart: unless-stopped

  # Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    container_name: onboarding-zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_logs:/var/lib/zookeeper/log
    networks:
      - onboarding-network
    restart: unless-stopped

  # MinIO Object Storage
  minio:
    image: minio/minio:latest
    container_name: onboarding-minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    networks:
      - onboarding-network
    restart: unless-stopped

  # Web Frontend
  web-frontend:
    build: ./web-frontend
    container_name: onboarding-frontend
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8000
      - REACT_APP_WS_URL=ws://localhost:8000
    depends_on:
      - api-gateway
    networks:
      - onboarding-network
    restart: unless-stopped

  # Monitoring & Observability
  prometheus:
    image: prom/prometheus:latest
    container_name: onboarding-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - onboarding-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: onboarding-grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      - prometheus
    networks:
      - onboarding-network
    restart: unless-stopped

networks:
  onboarding-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  elasticsearch_data:
  zookeeper_data:
  zookeeper_logs:
  minio_data:
  prometheus_data:
  grafana_data:
```

## 4. Agent Implementation Details

### 4.1 Requirements Gathering Agent Code Structure

```python
# requirements-agent/main.py
from fastapi import FastAPI, WebSocket, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from langchain.agents import initialize_agent, Tool
from langchain.llms import OpenAI
from langchain.memory import ConversationBufferMemory
from langchain.tools import BaseTool
import asyncio
import redis
import json

app = FastAPI(title="Requirements Gathering Agent", version="1.0.0")

class RequirementsGatheringAgent:
    def __init__(self):
        self.llm = OpenAI(temperature=0.1)
        self.memory = ConversationBufferMemory()
        self.redis_client = redis.Redis.from_url("redis://redis:6379")
        
        # Initialize tools
        self.tools = [
            self.create_dynamic_form_tool(),
            self.create_validation_tool(),
            self.create_context_analysis_tool(),
        ]
        
        self.agent = initialize_agent(
            tools=self.tools,
            llm=self.llm,
            agent="conversational-react-description",
            memory=self.memory,
            verbose=True
        )
    
    def create_dynamic_form_tool(self) -> Tool:
        def generate_form(requirements: str) -> str:
            # AI-powered dynamic form generation
            prompt = f"""
            Based on the following requirements, generate a JSON form schema:
            Requirements: {requirements}
            
            Generate a form that captures:
            1. Application type and purpose
            2. Technical requirements (database, integrations, etc.)
            3. Compliance requirements (GDPR, HIPAA, etc.)
            4. SLA expectations
            5. Security requirements
            """
            return self.llm(prompt)
        
        return Tool(
            name="DynamicFormGenerator",
            description="Generates adaptive forms based on requirements",
            func=generate_form
        )
    
    async def process_onboarding_request(self, request_data: dict) -> dict:
        """Main entry point for processing onboarding requests"""
        try:
            # Store initial request
            request_id = request_data.get("request_id")
            await self.store_request(request_id, request_data)
            
            # Generate dynamic questionnaire
            form_schema = await self.generate_questionnaire(request_data)
            
            # Return form to user
            return {
                "request_id": request_id,
                "status": "pending_user_input",
                "form_schema": form_schema,
                "next_steps": "Please complete the generated form"
            }
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/v1/start-onboarding")
async def start_onboarding(request_data: dict):
    agent = RequirementsGatheringAgent()
    return await agent.process_onboarding_request(request_data)

@app.websocket("/ws/requirements")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    agent = RequirementsGatheringAgent()
    
    while True:
        try:
            data = await websocket.receive_json()
            response = await agent.process_message(data)
            await websocket.send_json(response)
        except Exception as e:
            await websocket.send_json({"error": str(e)})
            break
```

### 4.2 Validation & Compliance Agent

```python
# validation-agent/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import asyncio
import httpx
import json
from typing import List, Dict

app = FastAPI(title="Validation & Compliance Agent", version="1.0.0")

class ComplianceChecker:
    def __init__(self):
        self.opa_url = "http://opa:8181"
        self.compliance_frameworks = {
            "GDPR": self.check_gdpr_compliance,
            "HIPAA": self.check_hipaa_compliance,
            "SOX": self.check_sox_compliance,
            "PCI_DSS": self.check_pci_compliance
        }
    
    async def validate_request(self, request_data: dict) -> dict:
        """Comprehensive validation of onboarding request"""
        validation_results = {
            "request_id": request_data.get("request_id"),
            "overall_status": "pending",
            "compliance_checks": {},
            "risk_score": 0,
            "violations": [],
            "recommendations": []
        }
        
        # Run all compliance checks concurrently
        tasks = []
        for framework, checker in self.compliance_frameworks.items():
            if framework in request_data.get("required_compliance", []):
                tasks.append(self.run_compliance_check(framework, checker, request_data))
        
        compliance_results = await asyncio.gather(*tasks)
        
        # Aggregate results
        for framework, result in compliance_results:
            validation_results["compliance_checks"][framework] = result
            
        # Calculate risk score
        validation_results["risk_score"] = self.calculate_risk_score(validation_results)
        
        # Determine overall status
        validation_results["overall_status"] = self.determine_status(validation_results)
        
        return validation_results
    
    async def check_gdpr_compliance(self, request_data: dict) -> dict:
        """GDPR compliance validation"""
        checks = {
            "data_processing_purpose": self.has_valid_purpose(request_data),
            "consent_mechanism": self.has_consent_mechanism(request_data),
            "data_retention_policy": self.has_retention_policy(request_data),
            "data_portability": self.supports_data_portability(request_data),
            "right_to_deletion": self.supports_deletion(request_data)
        }
        
        return {
            "framework": "GDPR",
            "status": "compliant" if all(checks.values()) else "non_compliant",
            "checks": checks,
            "violations": [k for k, v in checks.items() if not v]
        }

@app.post("/api/v1/validate")
async def validate_onboarding_request(request_data: dict):
    checker = ComplianceChecker()
    return await checker.validate_request(request_data)
```

### 4.3 Workflow Orchestrator Agent

```python
# orchestrator-agent/main.py
from fastapi import FastAPI
from temporal.client import Client
from temporal.worker import Worker
import asyncio
from dataclasses import dataclass
from typing import List, Dict

app = FastAPI(title="Workflow Orchestrator Agent", version="1.0.0")

@dataclass
class OnboardingWorkflow:
    request_id: str
    lob: str
    dependencies: List[str]
    assigned_teams: List[str]
    sla_hours: int
    priority: str

class WorkflowOrchestrator:
    def __init__(self):
        self.temporal_client = Client.connect("temporal:7233")
        self.team_assignments = {
            "security": ["security_review", "vulnerability_scan"],
            "compliance": ["gdpr_check", "audit_preparation"],
            "it_ops": ["infrastructure_setup", "monitoring_config"],
            "finance": ["cost_approval", "billing_setup"]
        }
    
    async def orchestrate_onboarding(self, request_data: dict) -> dict:
        """Main orchestration logic"""
        workflow_id = f"onboarding_{request_data['request_id']}"
        
        # Create workflow definition
        workflow_def = await self.create_workflow_definition(request_data)
        
        # Start Temporal workflow
        workflow_handle = await self.temporal_client.start_workflow(
            workflow_def.workflow_type,
            workflow_def,
            id=workflow_id,
            task_queue="onboarding_queue"
        )
        
        return {
            "workflow_id": workflow_id,
            "status": "started",
            "estimated_completion": workflow_def.estimated_completion,
            "assigned_teams": workflow_def.assigned_teams
        }
    
    async def create_workflow_definition(self, request_data: dict) -> OnboardingWorkflow:
        """AI-powered workflow creation"""
        # Analyze request to determine dependencies and team assignments
        lob = request_data.get("line_of_business")
        app_type = request_data.get("application_type")
        compliance_requirements = request_data.get("compliance_requirements", [])
        
        # AI-driven team assignment
        assigned_teams = await self.determine_team_assignments(
            lob, app_type, compliance_requirements
        )
        
        # Calculate dependencies
        dependencies = await self.analyze_dependencies(request_data)
        
        return OnboardingWorkflow(
            request_id=request_data["request_id"],
            lob=lob,
            dependencies=dependencies,
            assigned_teams=assigned_teams,
            sla_hours=self.calculate_sla(request_data),
            priority=self.determine_priority(request_data)
        )

@app.post("/api/v1/orchestrate")
async def orchestrate_workflow(request_data: dict):
    orchestrator = WorkflowOrchestrator()
    return await orchestrator.orchestrate_onboarding(request_data)
```

## 5. Data Models & Schemas

```python
# shared/models.py
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from datetime import datetime
from enum import Enum

class LOBType(str, Enum):
    RETAIL_BANKING = "retail_banking"
    CORPORATE_BANKING = "corporate_banking"
    INSURANCE = "insurance"
    WEALTH_MANAGEMENT = "wealth_management"
    CAPITAL_MARKETS = "capital_markets"

class ApplicationType(str, Enum):
    WEB_APPLICATION = "web_application"
    MOBILE_APP = "mobile_app"
    API_SERVICE = "api_service"
    BATCH_PROCESS = "batch_process"
    MICROSERVICE = "microservice"

class ComplianceFramework(str, Enum):
    GDPR = "gdpr"
    HIPAA = "hipaa"
    SOX = "sox"
    PCI_DSS = "pci_dss"
    BASEL_III = "basel_iii"

class OnboardingRequest(BaseModel):
    request_id: str = Field(..., description="Unique request identifier")
    user_id: str = Field(..., description="Requesting user identifier")
    lob: LOBType = Field(..., description="Line of business")
    application_name: str = Field(..., description="Application name")
    application_type: ApplicationType = Field(..., description="Type of application")
    business_purpose: str = Field(..., description="Business justification")
    compliance_requirements: List[ComplianceFramework] = Field(default=[], description="Required compliance frameworks")
    technical_requirements: Dict[str, Any] = Field(default={}, description="Technical specifications")
    sla_requirements: Dict[str, Any] = Field(default={}, description="SLA expectations")
    integration_endpoints: List[str] = Field(default=[], description="Required integrations")
    data_classification: str = Field(default="internal", description="Data sensitivity level")
    expected_users: int = Field(default=100, description="Expected number of users")
    go_live_date: Optional[datetime] = Field(None, description="Target go-live date")
    
    class Config:
        use_enum_values = True

class WorkflowTask(BaseModel):
    task_id: str
    task_name: str
    assigned_team: str
    status: str = "pending"
    dependencies: List[str] = []
    estimated_hours: int = 0
    actual_hours: Optional[int] = None
    start_date: Optional[datetime] = None
    completion_date: Optional[datetime] = None
    assignee: Optional[str] = None
    comments: List[str] = []

class ValidationResult(BaseModel):
    framework: str
    status: str
    violations: List[str] = []
    recommendations: List[str] = []
    risk_score: float = 0.0
    details: Dict[str, Any] = {}
```

## 6. Security & Authentication

```yaml
# security/auth-config.yml
authentication:
  providers:
    - name: "active_directory"
      type: "ldap"
      config:
        server: "ldap://ad.company.com:389"
        base_dn: "dc=company,dc=com"
        user_dn_template: "cn={username},ou=users,dc=company,dc=com"
    
    - name: "okta_saml"
      type: "saml2"
      config:
        entity_id: "https://onboarding.company.com"
        sso_url: "https://company.okta.com/app/saml2/onboarding"
        certificate_path: "/certs/okta.crt"

authorization:
  rbac:
    roles:
      - name: "onboarding_requester"
        permissions:
          - "create_onboarding_request"
          - "view_own_requests"
          - "update_own_requests"
      
      - name: "security_reviewer"
        permissions:
          - "view_all_requests"
          - "approve_security_requirements"
          - "flag_security_violations"
          - "access_security_templates"
      
      - name: "compliance_officer"
        permissions:
          - "view_all_requests"
          - "approve_compliance_requirements"
          - "access_regulatory_frameworks"
          - "generate_compliance_reports"
      
      - name: "it_ops_lead"
        permissions:
          - "view_technical_requests"
          - "approve_infrastructure_requirements"
          - "access_deployment_pipelines"
          - "manage_system_integrations"
      
      - name: "system_admin"
        permissions:
          - "*"  # All permissions

  policies:
    - name: "data_classification_access"
      rule: "user.clearance_level >= request.data_classification_level"
    
    - name: "lob_access_control"
      rule: "user.lob in request.authorized_lobs OR user.role == 'system_admin'"

encryption:
  at_rest:
    algorithm: "AES-256-GCM"
    key_rotation_days: 90
  
  in_transit:
    tls_version: "1.3"
    cipher_suites:
      - "TLS_AES_256_GCM_SHA384"
      - "TLS_CHACHA20_POLY1305_SHA256"

audit:
  events:
    - "user_authentication"
    - "permission_granted"
    - "permission_denied"
    - "data_access"
    - "configuration_change"
    - "workflow_state_change"
  
  retention_days: 2555  # 7 years for compliance
  storage: "elasticsearch"
    