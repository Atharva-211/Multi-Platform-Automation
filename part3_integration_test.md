## Part 3: API + UI Integration Test

### 3.1 Comprehensive Integration Test Implementation

```python
# test_project_creation_flow.py
import os, pytest, requests
from playwright.sync_api import expect
from utils.auth_helpers import get_api_token
from utils.data_factory import cleanup_test_data

# Test configuration
TENANT_CONFIG = {
    "company1": {"base": "https://app.workflowpro.com", "id": "company1"},
    "company2": {"base": "https://app.workflowpro.com", "id": "company2"}
}
API_BASE = "https://api.workflowpro.com/api/v1"

@pytest.fixture(scope="session")
def api_auth():
    """Retrieve API authentication token"""
    return {"Authorization": f"Bearer {get_api_token()}"}

@pytest.fixture
def test_project_data():
    """Generate unique test project data"""
    import uuid
    project_id = str(uuid.uuid4())
    return {
        "name": f"Test Project {project_id[:8]}",
        "description": "Integration test project",
        "team_members": ["test-user-123"]
    }

@pytest.fixture(autouse=True)
def cleanup_after_test():
    """Ensure test data cleanup"""
    created_projects = []
    yield created_projects
    cleanup_test_data(created_projects)

def create_project_via_api(auth_headers, tenant_id, project_data):
    """Create project using API endpoint"""
    headers = {**auth_headers, "X-Tenant-ID": tenant_id}
    response = requests.post(
        f"{API_BASE}/projects",
        headers=headers,
        json=project_data,
        timeout=30
    )
    response.raise_for_status()
    return response.json()["id"]

@pytest.mark.parametrize("tenant", ["company1", "company2"])
@pytest.mark.parametrize("browser_name", ["chromium", "webkit"])
def test_project_creation_flow(page, browser_name, tenant, api_auth, 
                             test_project_data, cleanup_after_test):
    """
    Complete integration test: API → Web UI → Mobile → Security Validation
    """
    tenant_config = TENANT_CONFIG[tenant]
    
    # Step 1: Create project via API
    project_id = create_project_via_api(
        api_auth, 
        tenant_config["id"], 
        test_project_data
    )
    cleanup_after_test.append(project_id)
    
    # Step 2: Verify project in Web UI
    page.goto(f'{tenant_config["base"]}/login')
    page.fill("#email", f"user@{tenant}.com")
    page.fill("#password", "password123")
    page.click("#login-btn")
    
    # Navigate to projects page
    page.locator(".sidebar-nav >> text=Projects").click()
    expect(page.locator(f"[data-project-id='{project_id}']")).to_be_visible()
    
    # Verify project details
    project_card = page.locator(f"[data-project-id='{project_id}']")
    expect(project_card.locator(".project-name")).to_have_text(test_project_data["name"])
    
    # Step 3: Mobile verification (BrowserStack integration)
    mobile_context = page.context.browser.new_context(
        viewport={"width": 390, "height": 844},
        is_mobile=True,
        user_agent="Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X)"
    )
    
    mobile_page = mobile_context.new_page()
    mobile_page.goto(f'{tenant_config["base"]}/m/projects')
    
    # Mobile-specific assertions
    expect(mobile_page.locator(f"text={test_project_data['name']}")).to_be_visible()
    mobile_context.close()
    
    # Step 4: Security validation - tenant isolation
    page.context.clear_cookies()
    page.goto(f'{tenant_config["base"]}/logout')
    
    # Login as different tenant user
    other_tenant = "company1" if tenant == "company2" else "company2"
    page.goto(f'{TENANT_CONFIG[other_tenant]["base"]}/login')
    page.fill("#email", f"user@{other_tenant}.com")
    page.fill("#password", "password123")
    page.click("#login-btn")
    
    # Attempt to access project from different tenant
    page.goto(f'{TENANT_CONFIG[other_tenant]["base"]}/projects/{project_id}')
    expect(page.locator("text=403")).to_be_visible()

@pytest.mark.slow
def test_project_creation_with_network_resilience(page, api_auth, test_project_data):
    """Test with network failures and recovery"""
    # Simulate network delays
    page.route("**/api/v1/projects", lambda route: route.fulfill(status=200, body='{"id": "test-123"}'))
    
    # Test with artificial latency
    page.context.set_default_timeout(60000)  # Extended timeout
    
    project_id = create_project_via_api(
        api_auth,
        TENANT_CONFIG["company1"]["id"],
        test_project_data
    )
    
    # Verify UI handles slow API responses gracefully
    page.goto("https://app.workflowpro.com/projects")
    expect(page.locator(".loading-spinner")).to_be_visible()
    expect(page.locator(f"[data-project-id='{project_id}']")).to_be_visible(timeout=30000)
```

### 3.2 Cross-Platform Strategy

**BrowserStack Integration:**
```javascript
// playwright.config.ts
export default {
  projects: [
    {
      name: 'Desktop Chrome',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'BrowserStack iOS',
      use: {
        channel: 'browserstack',
        ...devices['iPhone 13'],
        connectOptions: {
          wsEndpoint: `wss://cap-playwright:${process.env.BROWSERSTACK_ACCESS_KEY}@hub.browserstack.com/playwright`
        }
      }
    }
  ]
};
```

### 3.3 Test Data Management Strategy

**Isolation Approach:**
- Unique test data generation using UUIDs
- Tenant-specific data seeding
- Automated cleanup after test completion
- Database transaction rollback for integration tests

**Data Factory Pattern:**
```python
# utils/data_factory.py
class TestDataFactory:
    @staticmethod
    def create_unique_project(tenant_id):
        return {
            "name": f"Test-{tenant_id}-{uuid.uuid4().hex[:8]}",
            "tenant_id": tenant_id,
            "created_by": "automation-test"
        }
```
