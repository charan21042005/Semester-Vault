# 🧪 UNIT III — PART 3: TDD AND BDD (TEST-DRIVEN AND BEHAVIOR-DRIVEN DEVELOPMENT)

> **Covered in this document:** why testing is central to Agile and CI/CD → types of tests and the **test pyramid** → what a unit test is (anatomy: Arrange-Act-Assert) → **TDD** in depth (the Red-Green-Refactor cycle, the three laws, a complete worked example in Python with pytest, fixtures, parametrization, exceptions, mocking and stubbing, code coverage, testing a Flask app) → TDD benefits, myths, limits, code smells and refactoring → **BDD** in depth (origin, Given-When-Then, Gherkin syntax, the Three Amigos, scenario outlines, Cucumber/behave/pytest-bdd, step definitions, living documentation) → ATDD → TDD vs BDD vs ATDD comparison → how tests plug into CI/CD → common mistakes → exam answer template → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 2 scaled the **people**. Part 3 protects the **code**. TDD and BDD are the engineering practices that make fast, frequent, safe releases possible. Without them, CI/CD (Parts 4-5) just ships bugs faster.

---

# 1️⃣ WHY TESTING MATTERS IN AGILE AND CI/CD 🛡️

```
Agile: change code every day
CI/CD: release many times a day
→ you need a fast, automatic way to know "did I break anything?"
→ AUTOMATED TESTS are that safety net
```

| Without automated tests | With automated tests |
|---|---|
| Fear of changing code | Confident refactoring |
| Manual regression takes days | Regression in minutes |
| Bugs found in production | Bugs found in the pipeline |
| "It works on my machine" | Same tests run everywhere (CI) |

**Cost of a bug rises the later it is found** (roughly: requirements → design → code → test → production, each step costs more). "Shift left" = test earlier.

---

# 2️⃣ TYPES OF TESTS AND THE TEST PYRAMID ⭐⭐

## 2.1 Types

| Type | What it checks | Speed | Example |
|---|---|---|---|
| **Unit test** | One function/class in isolation | ms | `Cart.total()` returns the right number |
| **Integration test** | Several components together (code + DB, service + service) | seconds | API saves an order to PostgreSQL |
| **Contract test** | A service meets the agreed API contract with consumers | seconds | Pact tests |
| **End-to-end (E2E) / UI test** | Whole system through the real interface | slow (minutes) | Selenium/Cypress clicks through checkout |
| **Acceptance test** | Feature meets business criteria | varies | BDD scenarios |
| **Smoke test** | Is the deployed app basically alive? | seconds | `GET /health` returns 200 |
| **Regression test** | Old features still work after changes | varies | Whole suite on every commit |
| **Performance/load test** | Speed and capacity under load | minutes | JMeter, k6, Locust |
| **Security test (SAST/DAST)** | Vulnerabilities | varies | Unit V |
| **Exploratory (manual)** | Human creativity finds surprises | — | Tester explores new feature |

## 2.2 The Test Pyramid ⭐⭐⭐ (draw this)

```
              /\
             /UI\          ← FEW: slow, brittle, expensive (E2E)
            /────\
           /Service\       ← SOME: integration/API/contract tests
          /──────────\
         /   UNIT     \    ← MANY: fast, cheap, reliable (70%+ of tests)
        /──────────────\
```

Anti-pattern: **ice-cream cone** (mostly manual and UI tests, few unit tests): slow feedback, flaky pipelines.

Rule: **push tests as low in the pyramid as possible**; use higher levels only for what lower levels can't cover.

---

# 3️⃣ ANATOMY OF A GOOD UNIT TEST ✅

## 3.1 Arrange-Act-Assert (AAA) / Given-When-Then

```python
def test_total_with_two_items():
    # ARRANGE: set up
    cart = Cart()
    cart.add("book", 500, 2)
    # ACT: do the thing
    result = cart.total()
    # ASSERT: check the outcome
    assert result == 1000
```

## 3.2 Qualities: the **F.I.R.S.T.** principles

| Letter | Meaning |
|---|---|
| **F**ast | Runs in milliseconds |
| **I**ndependent | No order dependence between tests |
| **R**epeatable | Same result every time, any environment |
| **S**elf-validating | Pass/fail automatically (no manual reading) |
| **T**imely | Written at the right time (ideally before the code: TDD) |

Good test names tell a story: `test_apply_coupon_rejects_unknown_code`.

---

# 4️⃣ TDD: TEST-DRIVEN DEVELOPMENT ⭐⭐⭐

## 4.1 Definition

> **TDD** is a development practice where you **write a failing automated test first**, then write the **minimum code** to make it pass, then **refactor**, in tiny repeated cycles. Part of Extreme Programming; popularized by **Kent Beck**.

Mind-shift: tests are not "checking after", they **drive the design**.

## 4.2 The Red-Green-Refactor cycle ⭐⭐⭐

```
        ┌─────────────────────────────────────────────┐
        │                                             │
   1. 🔴 RED  ──────────▶  2. 🟢 GREEN  ──────────▶  3. 🔵 REFACTOR
   Write a small test      Write the simplest code     Clean up code AND tests
   that FAILS              that makes it PASS          (tests stay green)
        ▲                                             │
        └─────────────────── repeat ◀─────────────────┘
```

| Phase | Action | Rule |
|---|---|---|
| 🔴 **Red** | Write one small test for behavior that doesn't exist yet; run it; **see it fail** (for the right reason) | Never write production code without a failing test |
| 🟢 **Green** | Write the **minimum** code to pass (even "cheating" with a constant is fine at first) | Don't add extra features |
| 🔵 **Refactor** | Remove duplication, improve names, restructure | Run tests after every change; behavior must not change |

## 4.3 Uncle Bob's Three Laws of TDD

```
1. You may not write production code until you have written a failing unit test.
2. You may not write more of a unit test than is sufficient to fail (a compile error is a failure).
3. You may not write more production code than is sufficient to pass the currently failing test.
```

## 4.4 COMPLETE WORKED EXAMPLE: A SHOPPING CART (PYTHON + PYTEST) 🛒

### Setup

```bash
mkdir tdd-demo && cd tdd-demo
python3 -m venv venv && source venv/bin/activate      # Windows: venv\Scripts\activate
pip install pytest pytest-cov
```

Files: `cart.py` (production), `test_cart.py` (tests).

### Cycle 1: an empty cart totals zero

🔴 **RED**: `test_cart.py`

```python
from cart import Cart

def test_new_cart_total_is_zero():
    assert Cart().total() == 0
```

Run: `pytest -q` → **fails** (`ModuleNotFoundError: No module named 'cart'`). Good: it fails.

🟢 **GREEN**: simplest code: `cart.py`

```python
class Cart:
    def total(self):
        return 0
```

`pytest -q` → `1 passed`.

### Cycle 2: adding items changes the total

🔴 Add test:

```python
def test_add_item_updates_total():
    cart = Cart()
    cart.add("book", 500, 2)
    assert cart.total() == 1000
```

Fails (`AttributeError: 'Cart' object has no attribute 'add'`).

🟢 Make it pass:

```python
class Cart:
    def __init__(self):
        self._items = []

    def add(self, name, price, qty=1):
        self._items.append((name, price, qty))

    def total(self):
        return sum(price * qty for _, price, qty in self._items)
```

Both tests pass ✅.

### Cycle 3: invalid input is rejected

🔴 Add tests:

```python
import pytest

def test_add_rejects_zero_or_negative_quantity():
    cart = Cart()
    with pytest.raises(ValueError):
        cart.add("book", 500, 0)

def test_add_rejects_negative_price():
    cart = Cart()
    with pytest.raises(ValueError):
        cart.add("book", -1, 1)
```

🟢 Update `add`:

```python
    def add(self, name, price, qty=1):
        if qty <= 0:
            raise ValueError("quantity must be positive")
        if price < 0:
            raise ValueError("price cannot be negative")
        self._items.append((name, price, qty))
```

### Cycle 4: coupons

🔴 Tests:

```python
def test_coupon_save10_gives_ten_percent_off():
    cart = Cart()
    cart.add("book", 500, 2)          # 1000
    cart.apply_coupon("SAVE10")
    assert cart.total() == pytest.approx(900.0)

def test_unknown_coupon_is_rejected():
    cart = Cart()
    with pytest.raises(ValueError):
        cart.apply_coupon("FREEMONEY")
```

🟢 Code:

```python
class Cart:
    COUPONS = {"SAVE10": 0.10}

    def __init__(self):
        self._items = []
        self._discount = 0.0

    def add(self, name, price, qty=1):
        if qty <= 0:
            raise ValueError("quantity must be positive")
        if price < 0:
            raise ValueError("price cannot be negative")
        self._items.append((name, price, qty))

    def apply_coupon(self, code):
        if code not in self.COUPONS:
            raise ValueError("invalid coupon")
        self._discount = self.COUPONS[code]

    def total(self):
        subtotal = sum(price * qty for _, price, qty in self._items)
        return round(subtotal * (1 - self._discount), 2)
```

### 🔵 REFACTOR (tests protect you)

Tests repeat `Cart()` setup; extract a **fixture**:

```python
import pytest
from cart import Cart

@pytest.fixture
def cart():
    return Cart()

def test_new_cart_total_is_zero(cart):
    assert cart.total() == 0

def test_add_item_updates_total(cart):
    cart.add("book", 500, 2)
    assert cart.total() == 1000
```

Run again → still green ✅. That is the whole point: **refactor fearlessly**.

### Parametrized tests (many cases, one test)

```python
@pytest.mark.parametrize("qty, price, expected", [
    (1, 100, 100),
    (3, 100, 300),
    (2, 49.5, 99.0),
])
def test_total_for_various_inputs(cart, qty, price, expected):
    cart.add("item", price, qty)
    assert cart.total() == pytest.approx(expected)
```

### Run and measure coverage

```bash
pytest -v                                  # verbose
pytest --cov=cart --cov-report=term-missing   # which lines are not tested?
```

Sample output:

```
Name      Stmts   Miss  Cover   Missing
cart.py      17      0   100%
```

⚠️ **Coverage is a guide, not a goal.** 100% coverage does not mean no bugs (tests may assert nothing meaningful). Healthy targets are often 70-90% with meaningful assertions on critical logic.

## 4.5 Stubs, mocks and fakes (test doubles) ⭐

When code depends on slow/external things (payment gateway, database, email), replace them with **test doubles** so unit tests stay fast and deterministic.

| Double | Purpose |
|---|---|
| **Dummy** | Placeholder object, never used |
| **Stub** | Returns canned answers |
| **Fake** | Working lightweight implementation (in-memory DB) |
| **Mock** | Records calls so you can assert **how** it was used |
| **Spy** | Real object that also records calls |

Example: checkout uses a payment gateway.

`checkout.py`:

```python
def checkout(cart, gateway):
    amount = cart.total()
    if amount <= 0:
        raise ValueError("cart is empty")
    status = gateway.charge(amount)
    return "PAID" if status == "OK" else "FAILED"
```

`test_checkout.py`:

```python
from unittest.mock import Mock
import pytest
from cart import Cart
from checkout import checkout

def test_checkout_charges_gateway_and_returns_paid():
    cart = Cart()
    cart.add("book", 500, 2)

    gateway = Mock()
    gateway.charge.return_value = "OK"          # stub the response

    result = checkout(cart, gateway)

    assert result == "PAID"
    gateway.charge.assert_called_once_with(1000)  # mock verification

def test_checkout_returns_failed_when_gateway_declines():
    cart = Cart()
    cart.add("book", 500)
    gateway = Mock()
    gateway.charge.return_value = "DECLINED"
    assert checkout(cart, gateway) == "FAILED"

def test_checkout_empty_cart_never_calls_gateway():
    gateway = Mock()
    with pytest.raises(ValueError):
        checkout(Cart(), gateway)
    gateway.charge.assert_not_called()
```

No real money and no network: the test is instant and repeatable.

## 4.6 Testing a Flask app (links to Unit I's Docker app) 🐍

`app.py`:

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return "<h1>Hello CI/CD</h1>"

@app.route("/health")
def health():
    return jsonify(status="ok")

@app.route("/add/<int:a>/<int:b>")
def add(a, b):
    return jsonify(result=a + b)
```

`test_app.py`:

```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c

def test_home_page(client):
    resp = client.get("/")
    assert resp.status_code == 200
    assert b"Hello CI/CD" in resp.data

def test_health_endpoint(client):
    resp = client.get("/health")
    assert resp.get_json() == {"status": "ok"}

def test_add_endpoint(client):
    assert client.get("/add/2/3").get_json()["result"] == 5
```

This is exactly the kind of test your Jenkins pipeline will run in Practical 6.

## 4.7 TDD in other languages (same idea)

| Language | Test framework | Mock library |
|---|---|---|
| Python | pytest, unittest | unittest.mock, pytest-mock |
| Java | JUnit 5, TestNG | Mockito |
| JavaScript/TypeScript | Jest, Mocha, Vitest | jest.fn, Sinon |
| C# | xUnit, NUnit, MSTest | Moq |
| Go | `testing`, testify | gomock |
| Ruby | RSpec, Minitest | built-in doubles |

Java JUnit 5 sketch:

```java
class CartTest {
    @Test
    void newCartTotalIsZero() {
        assertEquals(0, new Cart().total());
    }
}
```

## 4.8 Benefits, myths and limits of TDD ⚖️

| Benefits | Limitations / honest criticism |
|---|---|
| Better, simpler, more modular design (testable code) | Learning curve; feels slow at first |
| High-quality regression suite as a by-product | Hard for UI, legacy code without seams, exploratory work |
| Fast feedback; fewer bugs and less debugging | Bad tests (brittle, over-mocked) create maintenance burden |
| Living documentation of behavior | Doesn't replace integration/E2E/exploratory testing |
| Confidence to refactor | Needs discipline and team buy-in |
| Smaller, focused commits | Studies on productivity/quality show mixed results; benefits depend on skill |

**Myths:** "TDD = tests after coding" (no: before); "100% coverage = quality" (no); "TDD replaces QA" (no: complements).

## 4.9 Refactoring and code smells 🔵

Refactoring = changing internal structure **without changing behavior**.

| Smell | Refactoring |
|---|---|
| Duplicated code | Extract method/function |
| Long method | Split into smaller functions |
| Magic numbers (`0.10`) | Named constants (`COUPONS`) |
| Large class | Extract class |
| Poor names | Rename |
| Deep nesting | Guard clauses/early return |

Rule: **refactor only when tests are green; run tests after each small step.**

---

# 5️⃣ BDD: BEHAVIOR-DRIVEN DEVELOPMENT ⭐⭐⭐

## 5.1 Why BDD? The communication gap

TDD tests speak to **programmers** (`assert total == 900`). Business people can't read them. Misunderstood requirements remain a top reason projects fail.

> **BDD** (introduced by **Dan North**, ~2003) extends TDD by describing software **behavior in plain, shared language** (business-readable examples) so developers, testers and business stakeholders collaborate and agree on what to build. Tests = **executable specifications**.

```
TDD asks:  "Does the code work correctly?"         (developer's view)
BDD asks:  "Does the system behave as the business expects?" (shared view)
```

## 5.2 The BDD process

```
1. DISCOVERY    → Three Amigos conversation (Product Owner/BA + Developer + Tester) explore examples/rules
2. FORMULATION  → Write examples as Given-When-Then scenarios in Gherkin (.feature files)
3. AUTOMATION   → Developers wire scenarios to code via step definitions; scenarios fail first (Red)
4. IMPLEMENT    → Build behavior with TDD underneath until scenarios pass (Green), then refactor
5. LIVING DOCS  → Feature files + results double as always-current documentation
```

**Three Amigos:** business (what/why), development (how), testing (what could go wrong). Short meetings before development begin.

## 5.3 Given-When-Then

```
GIVEN  a context/precondition       (the starting state)
WHEN   an action/event occurs       (what the user/system does)
THEN   an observable outcome        (what should happen)
AND / BUT  extend any of the above
```

## 5.4 Gherkin syntax ⭐

**Gherkin** is a simple, structured, plain-text language (files end with `.feature`) used by Cucumber, behave, SpecFlow, Behat, pytest-bdd.

Keywords:

| Keyword | Purpose |
|---|---|
| `Feature:` | Name and description of the capability |
| `Background:` | Steps run before every scenario in the file |
| `Scenario:` | One concrete example |
| `Scenario Outline:` + `Examples:` | Same scenario with a table of data |
| `Given / When / Then / And / But` | Steps |
| `@tag` | Labels to select scenarios (`@smoke`, `@wip`) |
| `#` | Comment |

`features/cart.feature`:

```gherkin
Feature: Shopping cart discounts
  As a shopper
  I want to apply coupon codes
  So that I pay less for my books

  Background:
    Given an empty cart

  @smoke
  Scenario: Cart total without discount
    When I add "book" priced 500 with quantity 2
    Then the total should be 1000

  Scenario: Valid coupon gives 10 percent off
    Given I add "book" priced 500 with quantity 2
    When I apply the coupon "SAVE10"
    Then the total should be 900

  Scenario: Unknown coupon is rejected
    Given I add "book" priced 500 with quantity 1
    When I apply the coupon "FREEMONEY"
    Then I should see the error "invalid coupon"

  Scenario Outline: Totals for several quantities
    When I add "pen" priced <price> with quantity <qty>
    Then the total should be <total>

    Examples:
      | price | qty | total |
      | 10    | 1   | 10    |
      | 10    | 5   | 50    |
      | 25    | 4   | 100   |
```

Good scenario rules: business language (not UI clicks like "click button #submit"), one behavior per scenario, concrete examples, declarative not imperative.

## 5.5 Step definitions with **behave** (Python Cucumber-style) 🥒

```bash
pip install behave
```

Folder structure:

```
tdd-demo/
├── cart.py
└── features/
    ├── cart.feature
    └── steps/
        └── cart_steps.py
```

`features/steps/cart_steps.py`:

```python
from behave import given, when, then
from cart import Cart

@given("an empty cart")
def step_empty_cart(context):
    context.cart = Cart()
    context.error = None

@given('I add "{name}" priced {price:g} with quantity {qty:d}')
@when('I add "{name}" priced {price:g} with quantity {qty:d}')
def step_add_item(context, name, price, qty):
    context.cart.add(name, price, qty)

@when('I apply the coupon "{code}"')
def step_apply_coupon(context, code):
    try:
        context.cart.apply_coupon(code)
    except ValueError as e:
        context.error = str(e)

@then("the total should be {expected:g}")
def step_check_total(context, expected):
    assert context.cart.total() == expected, f"expected {expected}, got {context.cart.total()}"

@then('I should see the error "{message}"')
def step_check_error(context, message):
    assert context.error == message
```

Run:

```bash
behave                         # run all features
behave --tags=@smoke           # only smoke scenarios
behave --format=pretty         # readable output
behave --junit                 # JUnit XML reports (for Jenkins!)
```

Output (shape):

```
Feature: Shopping cart discounts
  Scenario: Valid coupon gives 10 percent off
    Given an empty cart ... passed
    Given I add "book" priced 500 with quantity 2 ... passed
    When I apply the coupon "SAVE10" ... passed
    Then the total should be 900 ... passed
4 scenarios passed, 0 failed
```

Missing steps? Behave prints **snippets** to copy: that's the **Red** stage of BDD.

## 5.6 Other BDD tools

| Tool | Language |
|---|---|
| **Cucumber** | Java, JavaScript, Ruby, and more (the original) |
| **behave**, **pytest-bdd** | Python |
| **SpecFlow / Reqnroll** | .NET |
| **Behat** | PHP |
| **Godog** | Go |
| **Robot Framework** | Keyword-driven acceptance testing |
| **Selenium/Playwright/Cypress** | Drive browser steps in UI scenarios |

## 5.7 ATDD (Acceptance Test-Driven Development)

> Teams write **acceptance tests** with the customer **before** implementation; the feature is "done" when the acceptance tests pass. BDD is a flavor of ATDD that standardizes on Given-When-Then and ubiquitous language.

## 5.8 Benefits and pitfalls of BDD ⚖️

| Benefits | Pitfalls |
|---|---|
| Shared understanding; fewer misunderstood requirements | Becomes overhead if business doesn't participate (just "Cucumber as a test tool") |
| Living documentation always in sync with code | Too many UI-level scenarios = slow, brittle |
| Tests readable by non-programmers | Poorly written (imperative/technical) scenarios |
| Focus on user value | Requires collaboration culture and facilitation |
| Fits Definition of Done and acceptance criteria | Maintenance of step libraries |

---

# 6️⃣ TDD vs BDD vs ATDD ⭐⭐⭐ (WRITE THIS TABLE)

| Aspect | **TDD** | **BDD** | **ATDD** |
|---|---|---|---|
| Focus | Correctness of code units | System behavior from user's view | Meeting acceptance criteria |
| Primary audience | Developers | Developers + testers + business | Customer + team |
| Language | Programming language (asserts) | **Gherkin** (Given-When-Then), plain English | Any acceptance-test format |
| Test level | Mostly unit | Acceptance/feature (with unit tests beneath) | Acceptance |
| Written by | Developers | Three Amigos collaboratively | Team + customer |
| Tools | pytest, JUnit, Jest | Cucumber, behave, SpecFlow | FitNesse, Robot, Cucumber |
| Starts from | A small piece of behavior in code | A user story/example | Acceptance criteria |
| Output | Regression unit tests, better design | Living documentation + acceptance tests | Verified acceptance |
| Cycle | Red-Green-Refactor | Discover-Formulate-Automate (+TDD inside) | Discuss-Distill-Develop-Demo |

```
They stack together:
  BDD scenario (outer loop: "does the feature behave right?")
     └── TDD unit tests (inner loop: "does each piece work?")
```

**Double loop:** write failing acceptance scenario (outer Red) → TDD cycles until pieces exist → scenario Green → refactor.

---

# 7️⃣ HOW TESTS PLUG INTO CI/CD 🔄

```
git push ──▶ Jenkins pipeline
                │  1. Checkout
                │  2. Install deps
                │  3. Lint / static analysis
                │  4. Unit tests (pytest --junitxml=reports/unit.xml)         ← TDD output
                │  5. Acceptance tests (behave --junit)                       ← BDD output
                │  6. Coverage gate (fail if < threshold)
                │  7. Build Docker image → integration/smoke tests
                │  8. Publish test reports (Jenkins junit step) → PASS ⇒ deploy, FAIL ⇒ stop and notify
```

Rules: **pipeline must fail fast** and **block merges** when tests fail; flaky tests must be fixed or quarantined; run the quickest tests first.

Report generation commands:

```bash
pytest --junitxml=reports/pytest.xml --cov=. --cov-report=xml:reports/coverage.xml
behave --junit --junit-directory reports/behave
```

---

# 8️⃣ COMMON MISTAKES 🚨

| Mistake | Fix |
|---|---|
| Writing tests after code and calling it TDD | Write the failing test first |
| Testing implementation details (private methods) | Test behavior via public interface |
| Over-mocking everything | Mock only external boundaries |
| Giant tests with many asserts | One behavior per test |
| Tests depending on each other/order | Independent setup via fixtures |
| Flaky tests (random failures: time, network, order) | Control time/randomness, isolate, fix root cause |
| Ignoring failing tests ("it's just flaky") | Treat red build as stop-the-line |
| Scenarios written as UI click scripts | Declarative business language |
| No business involvement in BDD | Hold Three Amigos sessions |
| Chasing 100% coverage | Cover critical behavior meaningfully |
| Slow test suite | Pyramid: more unit, fewer UI tests; parallelize |

---

# 9️⃣ EXAM ANSWER TEMPLATE ✍️

**Q. Explain TDD and BDD with examples and compare them.**

```
1. Intro: both are Agile engineering practices that make tests drive development.
2. TDD: Red-Green-Refactor diagram; three steps explained; tiny example (test_total → fail → Cart.total → refactor).
3. Benefits of TDD (design, regression suite, confidence).
4. BDD: Dan North; Given-When-Then; Gherkin feature example (5-6 lines); Three Amigos; Cucumber/behave; living documentation.
5. Comparison table (focus, audience, language, tools, level, output) – 6-8 rows.
6. How they combine (double loop) and fit into CI/CD (tests run in pipeline).
7. Conclusion: TDD builds the thing right; BDD ensures we build the right thing.
```

---

# 🔟 VIVA QUESTIONS (PART 3) 🎯

**Q1. What is TDD?** Writing a failing test first, then minimum code to pass, then refactoring, in short cycles.

**Q2. Explain Red-Green-Refactor.** Red: failing test; Green: simplest code to pass; Refactor: clean up while tests stay green.

**Q3. What is a unit test?** Automated test of the smallest testable piece (function/class) in isolation.

**Q4. What is the test pyramid?** Many unit tests, fewer integration/service tests, very few UI/E2E tests for speed and reliability.

**Q5. Mock vs stub?** Stub returns canned data; mock additionally verifies interactions (calls).

**Q6. What is code coverage? Is 100% enough?** Percentage of code executed by tests; not enough because it doesn't prove assertions are meaningful.

**Q7. What is BDD?** Collaborative practice describing behavior in plain-language Given-When-Then examples that become executable tests.

**Q8. What is Gherkin?** Structured plain-text language with Feature, Scenario, Given/When/Then, used by Cucumber/behave.

**Q9. Who are the Three Amigos?** Business (PO/BA), developer, tester discussing examples before development.

**Q10. TDD vs BDD?** TDD is developer-focused, unit-level, in code; BDD is team-wide, behavior-level, in Gherkin; BDD often uses TDD underneath.

**Q11. What is ATDD?** Defining acceptance tests with the customer before coding; done when they pass.

**Q12. What is a Scenario Outline?** A scenario template run with multiple rows from an Examples table.

**Q13. What does FIRST stand for?** Fast, Independent, Repeatable, Self-validating, Timely.

**Q14. What is refactoring?** Improving internal structure without changing external behavior, protected by tests.

**Q15. Why are tests essential for CI/CD?** They provide automatic, fast feedback so every commit can be verified and safely deployed.

---

# 🧠 PART 3 MASTER SHEET

```
TEST TYPES: unit • integration • contract • E2E/UI • acceptance • smoke • regression • performance • security
PYRAMID: many unit (fast) → some integration/API → few UI/E2E ; avoid ice-cream cone
UNIT TEST: Arrange-Act-Assert ; FIRST (Fast, Independent, Repeatable, Self-validating, Timely)
TDD: 🔴 write failing test → 🟢 minimum code → 🔵 refactor ; Three Laws ; Kent Beck/XP
Tools: pytest (fixtures, parametrize, raises), unittest.mock (Mock, return_value, assert_called_once_with), pytest-cov
Doubles: dummy, stub, fake, mock, spy
BDD: Dan North ; Three Amigos ; Given-When-Then ; Gherkin (.feature: Feature, Background, Scenario, Outline+Examples, @tags)
Tools: Cucumber, behave, pytest-bdd, SpecFlow ; living documentation ; ATDD
TDD=right way (code) | BDD=right thing (behavior) ; double loop: BDD outer + TDD inner
CI/CD: tests run on every push → junit/coverage reports → fail fast → block deploy
```

---

## 🫡 PART 3 STATUS

You can now practice TDD with Red-Green-Refactor, write mocks and fixtures in pytest, write Gherkin scenarios and step definitions with behave, and explain how tests feed a CI/CD pipeline.

**STOP here.** Next up:

> **PART 4 — CI/CD Concepts (continuous integration, delivery vs deployment, pipelines, branching strategies, deployment strategies, feature flags, DORA metrics) 🔄**
