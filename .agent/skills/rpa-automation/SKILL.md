---
name: rpa-automation
description: RPA (Robotic Process Automation) principles for web automation with Playwright. Resilient navigation in legacy portals, state machines for multi-step workflows, adaptive selectors, error recovery, CAPTCHA handling, session management, and Human-in-the-Loop (HITL) patterns. Use when automating data entry in legacy web portals, building browser-based RPA bots, navigating systems with iframes/frames, implementing multi-step form filling workflows, or designing retry/recovery strategies for web automation. Use PROACTIVELY whenever the project involves browser automation beyond testing. Do NOT use for E2E testing (use webapp-testing), or for API integrations (use api-patterns).
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# RPA Automation

> **Web automation that survives real-world portals.**
> Use this skill for browser-based automation in legacy systems — NOT for testing.

---

## ⚠️ Core Principle

- RPA is NOT testing — it's production automation
- Legacy portals are hostile: frames, dynamic IDs, slow loads, session timeouts
- Every step can fail — design for recovery, not just happy path
- Human-in-the-Loop is a feature, not a limitation

---

## 1. RPA vs Testing — The Key Difference

```
Testing (webapp-testing):
├── Controlled environment
├── Known inputs/outputs
├── Fails = bug found ✓
├── Runs in CI/CD
└── Assertion-based

RPA (this skill):
├── Hostile environment (legacy portals)
├── Variable inputs from external data
├── Fails = must recover and retry
├── Runs in production
└── State machine-based
```

---

## 2. Resilient Selector Strategy

### Selector Priority (most to least stable)

```
1. data-testid / aria-label    (if available — rare in legacy)
2. Role-based selectors         (getByRole, getByLabel)
3. Text content                 (getByText — language dependent)
4. CSS with stable attributes   (name=, type=, class with semantic names)
5. XPath with structure         (last resort for legacy frames)
6. ❌ Position-based            (NEVER — breaks on any layout change)
```

### Adaptive Selectors Pattern

```python
async def find_element_resilient(page, strategies: list[dict]) -> ElementHandle:
    """Try multiple selector strategies in order of preference."""
    for strategy in strategies:
        try:
            selector = strategy["selector"]
            timeout = strategy.get("timeout", 5000)
            element = await page.wait_for_selector(selector, timeout=timeout)
            if element:
                return element
        except TimeoutError:
            continue
    raise ElementNotFoundError(
        f"No selector matched. Tried: {[s['selector'] for s in strategies]}"
    )

# Usage: ordered from most stable to least
await find_element_resilient(page, [
    {"selector": "[data-field='protocolo']"},
    {"selector": "input[name='txtProtocolo']"},
    {"selector": "//input[contains(@id, 'protocolo')]", "timeout": 3000},
])
```

### Legacy Portal Challenges

| Challenge | Strategy |
|-----------|----------|
| Dynamic IDs | Use partial attribute matching (`[id*='protocolo']`) |
| Nested iframes | `page.frame()` or `frame_locator()` with name/URL |
| JavaScript-rendered content | `wait_for_selector(state='visible')` |
| Inline JavaScript events | Use `evaluate()` to trigger events |
| Session timeouts | Detect expired sessions, re-authenticate |
| Slow page loads | Custom wait conditions, not hardcoded delays |

---

## 3. State Machine Architecture

### Why State Machines for RPA

```
Sequential scripts are fragile:
├── Step 3 fails → entire flow crashes
├── No way to resume from step 3
├── User can't intervene mid-flow
└── No visibility into progress

State machines are resilient:
├── Each step is an independent state
├── Failure → transition to error/retry state
├── Can resume from any state
├── HITL naturally fits as a "waiting" state
└── Progress is always known and loggable
```

### State Machine Pattern

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class AutomationState(Enum):
    IDLE = "idle"
    AUTHENTICATING = "authenticating"
    STEP_1_CADASTRO = "step_1_cadastro_profissional"
    STEP_2_DADOS = "step_2_dados_pessoais"
    STEP_3_ENDERECO = "step_3_endereco"
    STEP_4_TITULO = "step_4_titulo"
    WAITING_HITL = "waiting_human_intervention"
    ERROR_RECOVERY = "error_recovery"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class AutomationContext:
    state: AutomationState
    current_record: dict
    retry_count: int = 0
    max_retries: int = 3
    error_message: Optional[str] = None
    step_completed: list[str] = None

    def __post_init__(self):
        if self.step_completed is None:
            self.step_completed = []

class StateMachine:
    def __init__(self, context: AutomationContext):
        self.context = context
        self.transitions = {
            AutomationState.IDLE: self._start,
            AutomationState.AUTHENTICATING: self._authenticate,
            AutomationState.STEP_1_CADASTRO: self._cadastro_profissional,
            AutomationState.STEP_2_DADOS: self._dados_pessoais,
            AutomationState.STEP_3_ENDERECO: self._endereco,
            AutomationState.STEP_4_TITULO: self._titulo,
            AutomationState.ERROR_RECOVERY: self._handle_error,
            AutomationState.WAITING_HITL: self._wait_human,
        }

    async def run(self):
        while self.context.state not in (
            AutomationState.COMPLETED,
            AutomationState.FAILED,
        ):
            handler = self.transitions.get(self.context.state)
            if not handler:
                raise ValueError(f"No handler for state: {self.context.state}")

            next_state = await handler()
            self._log_transition(self.context.state, next_state)
            self.context.state = next_state

    def _log_transition(self, from_state, to_state):
        # Log to automation_logs table
        ...
```

### State Transition Diagram

```
IDLE → AUTHENTICATING → STEP_1 → STEP_2 → STEP_3 → STEP_4 → COMPLETED
                           │         │         │         │
                           └────>────┴────>────┴────>────┤
                                                         │
                                                   ERROR_RECOVERY
                                                         │
                                                  ┌──────┤
                                                  │      │
                                            WAITING_HITL │
                                                  │      │
                                                  └──>RETRY or FAILED
```

---

## 4. Error Recovery & Retry

### Retry Strategy

```python
import asyncio
from typing import Callable

async def retry_with_backoff(
    action: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_errors: tuple = (TimeoutError, ConnectionError),
) -> any:
    """Retry with exponential backoff for transient failures."""
    for attempt in range(max_retries + 1):
        try:
            return await action()
        except retryable_errors as e:
            if attempt == max_retries:
                raise
            delay = min(base_delay * (2 ** attempt), max_delay)
            await asyncio.sleep(delay)
```

### Error Categories

| Category | Strategy | Example |
|----------|----------|---------|
| **Transient** | Retry with backoff | Network timeout, slow page load |
| **Validation** | HITL intervention | "Professional already registered" |
| **Session** | Re-authenticate | Session expired, cookie invalid |
| **Structural** | Adaptive selector | Page layout changed |
| **Fatal** | Abort + log | Portal down, blocked IP |

### Detection Patterns

```python
async def detect_error_type(page) -> str:
    """Analyze current page state to categorize error."""
    # Check for validation error messages
    validation_error = await page.query_selector(".msg-erro, .validation-error")
    if validation_error:
        text = await validation_error.inner_text()
        if "já cadastrado" in text.lower():
            return "DUPLICATE_RECORD"
        return "VALIDATION_ERROR"

    # Check for session expiry
    if "/login" in page.url or await page.query_selector("#loginForm"):
        return "SESSION_EXPIRED"

    # Check for system error pages
    error_page = await page.query_selector(".error-500, .system-error")
    if error_page:
        return "SYSTEM_ERROR"

    return "UNKNOWN"
```

---

## 5. Human-in-the-Loop (HITL)

### HITL Architecture

```
Automation Flow:
Step N → Error/Low Confidence → PAUSE
                                  │
                                  ├── Notify user (visual + audio)
                                  ├── Show context (what failed, why)
                                  ├── Present options (retry, skip, manual fix)
                                  │
                                  ▼
                              User decides
                                  │
                                  ├── "Continue" → Resume from Step N
                                  ├── "Skip" → Move to Step N+1
                                  └── "Abort" → Stop workflow
```

### HITL Implementation Pattern

```python
@dataclass
class HITLRequest:
    workflow_id: str
    step: str
    error_type: str
    message: str
    context: dict  # Data being processed
    screenshot_path: Optional[str] = None
    options: list[str] = None

    def __post_init__(self):
        if self.options is None:
            self.options = ["continue", "skip", "abort"]

async def request_human_intervention(
    request: HITLRequest,
    supabase_client,
    notification_callback: Callable,
) -> str:
    """Pause automation and wait for human decision."""
    # 1. Save state to database
    await supabase_client.table("automation_logs").insert({
        "workflow_id": request.workflow_id,
        "step": request.step,
        "status": "waiting_hitl",
        "message": request.message,
        "context": request.context,
    }).execute()

    # 2. Trigger notification (audio + visual)
    await notification_callback(
        type="hitl_required",
        message=request.message,
    )

    # 3. Wait for user response (via Realtime or polling)
    response = await wait_for_user_response(request.workflow_id)
    return response  # "continue" | "skip" | "abort"
```

### HITL UI Considerations

| Element | Purpose |
|---------|---------|
| Error message display | Clear explanation of what happened |
| Document preview | Side-by-side view for context |
| Action buttons | Continue / Skip / Abort |
| Audio notification | Distinct sounds per error type |
| History log | Show previous steps and results |

---

## 6. Session Management

### Session Lifecycle

```python
class SessionManager:
    """Manage browser sessions for legacy portals."""

    def __init__(self, browser, credentials: dict):
        self.browser = browser
        self.credentials = credentials
        self.context = None
        self.page = None
        self._last_activity = None

    async def ensure_session(self):
        """Check if session is active, re-authenticate if needed."""
        if not self.page or self.page.is_closed():
            await self._create_session()
            return

        if await self._is_session_expired():
            await self._re_authenticate()

        self._last_activity = time.time()

    async def _is_session_expired(self) -> bool:
        """Detect session expiration via URL or page content."""
        try:
            # Check if redirected to login
            if "/login" in self.page.url:
                return True
            # Check for session timeout message
            timeout_msg = await self.page.query_selector(
                ".session-timeout, .expired"
            )
            return timeout_msg is not None
        except Exception:
            return True

    async def _create_session(self):
        """Create new browser context with persistent cookies."""
        self.context = await self.browser.new_context(
            viewport={"width": 1280, "height": 720},
            user_agent="Mozilla/5.0 ...",  # Mimic real browser
        )
        self.page = await self.context.new_page()

    async def save_cookies(self, path: str):
        """Persist cookies for session reuse."""
        cookies = await self.context.cookies()
        with open(path, "w") as f:
            json.dump(cookies, f)

    async def load_cookies(self, path: str):
        """Restore cookies from previous session."""
        with open(path, "r") as f:
            cookies = json.load(f)
        await self.context.add_cookies(cookies)
```

---

## 7. Form Filling Patterns

### Smart Form Filling

```python
async def fill_form_field(page, field_config: dict, value: str):
    """Fill a form field with type-aware handling."""
    field_type = field_config.get("type", "text")
    selector = field_config["selector"]

    match field_type:
        case "text" | "email" | "number":
            await page.fill(selector, "")  # Clear first
            await page.fill(selector, value)
        case "select":
            await page.select_option(selector, value)
        case "date":
            # Some legacy portals need manual date input
            await page.evaluate(
                f"document.querySelector('{selector}').value = '{value}'"
            )
            await page.dispatch_event(selector, "change")
        case "checkbox":
            checked = await page.is_checked(selector)
            if checked != bool(value):
                await page.click(selector)
        case "radio":
            await page.click(f"{selector}[value='{value}']")

    # Wait for any AJAX validation
    await page.wait_for_timeout(300)
```

### Handling Legacy Frames

```python
async def navigate_to_frame(page, frame_identifiers: list[str]):
    """Navigate through nested frames in legacy portals."""
    current = page
    for identifier in frame_identifiers:
        frame = current.frame_locator(identifier)
        if not frame:
            # Try by name
            frame = current.frame(name=identifier)
        if not frame:
            # Try by URL pattern
            for f in current.frames:
                if identifier in f.url:
                    frame = f
                    break
        if not frame:
            raise FrameNotFoundError(f"Frame not found: {identifier}")
        current = frame
    return current
```

---

## 8. Logging & Monitoring

### Structured Logging

```python
import structlog

logger = structlog.get_logger()

async def log_step(
    workflow_id: str,
    action: str,
    step: int,
    status: str,
    message: str = "",
    supabase_client=None,
):
    """Log automation step to both local logger and database."""
    log_entry = {
        "workflow_id": workflow_id,
        "action_type": action,  # "e2doc" | "sic"
        "step": step,
        "status": status,       # "started" | "completed" | "error" | "waiting_hitl"
        "message": message,
        "timestamp": datetime.utcnow().isoformat(),
    }

    logger.info("automation_step", **log_entry)

    if supabase_client:
        await supabase_client.table("automation_logs").insert(log_entry).execute()
```

---

## 9. Decision Checklist

Before building RPA automation:

- [ ] **Mapped all portal pages and forms?** (Manual walkthrough first)
- [ ] **Identified frame/iframe structure?**
- [ ] **Defined state machine with all transitions?**
- [ ] **Designed error categories and recovery strategies?**
- [ ] **Planned HITL triggers and UI?**
- [ ] **Implemented session management?**
- [ ] **Created structured logging?**
- [ ] **Tested with edge cases?** (Duplicate records, timeouts, etc.)
- [ ] **Defined retry limits?** (Prevent infinite loops)
- [ ] **Added monitoring/alerting?**

---

## 10. Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Use hardcoded `sleep()` waits | Use `wait_for_selector()` with conditions |
| Write linear scripts | Implement state machines |
| Ignore session timeouts | Check and refresh sessions proactively |
| Crash on first error | Categorize errors and recover |
| Skip logging | Log every state transition |
| Fill forms without clearing | Clear fields before filling |
| Use position-based selectors | Use semantic/attribute selectors |
| Run at maximum speed | Add realistic delays to avoid detection |
| Store screenshots forever | Implement retention policies |

---

> **Remember:** RPA automation is a production system, not a test suite. Design for the portal that crashes, the session that expires, and the user who needs to intervene. Every step should be logged, every error should be recoverable, and every transition should be visible.
