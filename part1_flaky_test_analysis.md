## Part 1: Debugging Flaky Test Code

### 1.1 Identified Flakiness Issues

| Issue | Root Cause | Impact |
|-------|------------|--------|
| URL assertion fires before redirect | No navigation wait mechanism | Intermittent failures in CI |
| `.welcome-message` selector occasionally absent | Element loads via XHR after dashboard mount | Race condition on slow networks |
| Hard-coded credentials bypass 2FA | 2FA randomly enforced for privileged users | Environment-specific behavior |
| Project card iteration without wait | No retry between card rendering | Timing-dependent failures |
| Inconsistent browser/viewport settings | No explicit configuration | Layout-dependent test breaks |
| No error recovery mechanisms | Hard failures without retry logic | Poor test reliability |

### 1.2 Root Cause Analysis

**Why CI/CD ≠ Local Environment:**

1. **Resource Constraints**: Shared CI executors throttle CPU/network, extending XHR completion times beyond Playwright's implicit waits
2. **Browser Matrix Differences**: CI may execute on WebKit/Safari while local uses Chromium, causing selector/CSS inconsistencies  
3. **Viewport Variations**: Mobile emulation in CI causes responsive layout shifts that hide elements
4. **Security Configuration**: 2FA toggles often enabled only in non-development environments

### 1.3 Technical Solutions

**Enhanced Configuration & Fixtures:**
```python
# conftest.py – Shared test fixtures
import os, pytest
from playwright.sync_api import sync_playwright, expect

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        headless = os.getenv("HEADLESS", "true") == "true"
        browser = p.chromium.launch(headless=headless)
        yield browser
        browser.close()

@pytest.fixture
def page(browser):
    context = browser.new_context(
        viewport={"width": 1280, "height": 800},
        locale="en-US"
    )
    page = context.new_page()
    yield page
    context.close()
```

**Improved Test Implementation:**
```python
# test_login.py – Reliable login tests
import pytest, os
from playwright.sync_api import expect

BASE_URL = os.getenv("BASE_URL", "https://app.workflowpro.com")

def do_login(page, email, password):
    page.goto(f"{BASE_URL}/login", wait_until="domcontentloaded")
    page.locator("#email").fill(email)
    page.locator("#password").fill(password)
    page.click("#login-btn")
    
    # Handle optional 2FA
    if page.locator("text=Verify Code").is_visible(timeout=2000):
        code = os.getenv("TOTP_CODE", "000000")
        page.locator("#totp").fill(code)
        page.click("button:text('Verify')")

@pytest.mark.flaky(reruns=2)
def test_user_login(page):
    do_login(page, "admin@company1.com", "password123")
    expect(page).to_have_url(f"{BASE_URL}/dashboard")
    expect(page.locator(".welcome-message")).to_be_visible()

def test_multi_tenant_access(page):
    do_login(page, "user@company2.com", "password123")
    cards = page.locator(".project-card")
    expect(cards).to_have_count_greater_than(0)
    
    cards.filter(has_text="Company2").first.wait_for(state="visible")
    for card in cards.all():
        assert "Company2" in card.text_content()
```

**Technical Reasoning:**
- `wait_until="domcontentloaded"` ensures page structure is ready before interaction
- Playwright's "web-first" assertions provide automatic retry logic
- Conditional 2FA handling makes tests environment-agnostic
- Fixed viewport prevents responsive layout issues
- pytest-rerun marks legitimate flakes while trending metrics
