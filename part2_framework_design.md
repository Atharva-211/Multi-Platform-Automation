## Part 2: Test Framework Design

### 2.1 Framework Architecture

**Directory Structure:**
```
tests/
├── ui/
│   ├── pages/
│   │   ├── login_page.py
│   │   ├── dashboard_page.py
│   │   └── projects_page.py
│   ├── dashboard/
│   │   └── test_dashboard_functionality.py
│   └── auth/
│       └── test_authentication.py
├── api/
│   ├── clients/
│   │   └── project_client.py
│   └── test_projects_api.py
├── mobile/
│   └── test_mobile_workflows.py
├── data/
│   ├── tenants.yml
│   ├── users.json
│   └── test_projects.json
├── configs/
│   ├── local.toml
│   ├── staging.toml
│   └── production.toml
├── utils/
│   ├── auth_helpers.py
│   ├── data_factory.py
│   └── reporting.py
├── conftest.py
├── playwright.config.ts
└── pytest.ini
```

### 2.2 Configuration Management Strategy

**Hierarchical Configuration System:**
1. Base configuration in `configs/*.toml` files
2. Environment-specific overrides via CLI flags (`--env=staging`)
3. Runtime secrets via environment variables
4. CI/CD pipeline integration through Docker/Jenkins

**Sample Configuration:**
```toml
# configs/staging.toml
[environment]
base_url = "https://staging.workflowpro.com"
api_url = "https://api-staging.workflowpro.com"
timeout = 30000

[browsers]
desktop = ["chromium@latest", "firefox@latest-1", "webkit"]
mobile = ["ios@16.4", "android@13"]

[browserstack]
username = "${BS_USERNAME}"
access_key = "${BS_ACCESS_KEY}"
project = "WorkFlow Pro Staging"
```

### 2.3 Core Framework Components

**Page Object Implementation:**
```python
# ui/pages/base_page.py
class BasePage:
    def __init__(self, page):
        self.page = page
        self.timeout = 10000
    
    def wait_for_load(self):
        self.page.wait_for_load_state("domcontentloaded")
    
    def get_tenant_context(self):
        return self.page.locator("[data-tenant-id]").get_attribute("data-tenant-id")
```

**API Client Abstraction:**
```python
# api/clients/base_client.py
import requests
from tenacity import retry, stop_after_attempt, wait_exponential

class BaseAPIClient:
    def __init__(self, base_url, tenant_id, auth_token):
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {auth_token}",
            "X-Tenant-ID": tenant_id,
            "Content-Type": "application/json"
        }
    
    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
    def post(self, endpoint, data):
        response = requests.post(f"{self.base_url}{endpoint}", 
                               json=data, headers=self.headers, timeout=30)
        response.raise_for_status()
        return response.json()
```

### 2.4 Missing Requirements Analysis

**Critical Questions for Stakeholders:**

1. **Test Data Management**
   - Data retention policy for test tenants?
   - Database seeding/cleanup strategy?
   - PII handling in test environments?

2. **Reporting & Monitoring**
   - Required KPIs (pass rate, duration, flake rate)?
   - Integration with existing dashboards?
   - Alert mechanisms for test failures?

3. **Parallel Execution**
   - BrowserStack concurrency budget?
   - Test isolation requirements?
   - Resource allocation per test suite?

4. **CI/CD Integration**
   - Pipeline trigger conditions?
   - Test environment provisioning?
   - Artifact retention policies?

5. **Security & Compliance**
   - Access control for test environments?
   - Audit logging requirements?
   - GDPR/SOC2 considerations?
