# Claude Code Agent — Follow-Up Prompt: Building QA from Scratch on a Project

## Context

You have already generated the main QA knowledge base (directories 01–22). Now you need to create an additional major section that covers a critically important topic: **how to build and manage testing on a project from zero** — as if you were the first (or only) QA engineer joining a new team, or a QA Lead setting up testing processes for a new product.

This is a strategic, high-level topic that goes beyond individual test techniques. It covers: test strategy, test planning, choosing what to test first, building automation incrementally, integrating different testing levels, establishing processes, and growing testing maturity over time.

## Task

### Step 1: Determine Placement

Analyze the existing directory structure and decide where to place this content. The recommended approach is to create a new top-level directory **`23_building_qa_from_scratch/`** — this is a standalone strategic topic that doesn't fit neatly into any existing directory. However, if you see a better placement after analyzing the project structure, you may adjust.

### Step 2: Create the Directory and Files

Create the following structure with full, detailed content in each file. Every file should follow the same formatting rules as the rest of the knowledge base (Russian text, English tech terms, code examples with Russian comments, interview questions section, practical exercises).

## Proposed Structure

```
23_building_qa_from_scratch/
│
├── 01_starting_point.md
│   # С чего начать: первые шаги QA на новом проекте
│   
│   Cover:
│   - What to do on day 1 / week 1 as the first QA on a project
│   - Discovery phase: understanding the product, architecture, team, existing processes
│   - Key questions to ask: What is the product? Who are the users? What's the tech stack?
│     What are the biggest risks? Is there any existing testing? What are the release cycles?
│   - Mapping the system: drawing a high-level architecture diagram, identifying components,
│     understanding data flows, identifying integration points
│   - Identifying testing scope: what CAN be tested vs what SHOULD be tested first
│   - Quick wins: what can you test TODAY to start providing value immediately
│   - Communicating with the team: setting expectations, explaining QA role, building trust
│   - Common mistakes: trying to automate everything on day 1, not understanding the product
│     before writing tests, not talking to developers
│
├── 02_test_strategy.md
│   # Тестовая стратегия: что тестируем, как, зачем
│   
│   Cover:
│   - What is a test strategy vs test plan (and why both matter)
│   - Test strategy structure: scope, objectives, approach, resources, schedule, risks
│   - Risk-based testing: how to prioritize what to test based on business risk and likelihood
│   - Choosing testing levels for YOUR project:
│     * When you need extensive unit testing (complex business logic)
│     * When integration testing is critical (microservices, external APIs)
│     * When E2E testing is essential (user-facing workflows, payment flows)
│     * When you can skip certain levels (and the trade-offs)
│   - Testing quadrants (Brian Marick's model): technology-facing vs business-facing,
│     supporting development vs critiquing the product
│   - Entry and exit criteria: when testing starts, when it's "done enough"
│   - Test strategy template with example for a typical web application
│   - How to present and get buy-in from the team and management
│   - Interview angle: "How would you build a test strategy for a new project?"
│
├── 03_test_plan.md
│   # Тест-план: детальное планирование тестирования
│   
│   Cover:
│   - Test plan structure (IEEE 829 inspired but practical):
│     * Scope and objectives
│     * Features to be tested / not tested (with reasoning)
│     * Testing approach per feature/component
│     * Test environment requirements
│     * Test data requirements
│     * Schedule and milestones
│     * Roles and responsibilities
│     * Risk assessment and mitigation
│     * Deliverables
│   - Test plan for a single feature vs test plan for a release vs master test plan
│   - Practical example: test plan for "Add payment processing to e-commerce app"
│   - How test plan evolves during the project (it's a living document)
│   - Estimation: how to estimate testing effort (test points, historical data, expert judgment)
│   - Template with fillable sections
│
├── 04_testing_levels_deep.md
│   # Уровни тестирования: что, когда и почему
│   
│   Cover in depth:
│   - Unit testing:
│     * What QA should know about unit tests (even if devs write them)
│     * When QA writes unit tests (component testing, service layer testing)
│     * Code coverage metrics: line, branch, mutation — what they mean, what's "good enough"
│   - Integration testing:
│     * Strategies: top-down, bottom-up, sandwich/hybrid, big bang
│     * Top-down: start from UI/API layer, mock lower layers, gradually replace mocks
│     * Bottom-up: start from DB/service layer, build up to API, then UI
│     * Big bang: integrate everything at once (risky, but sometimes necessary)
│     * Sandwich: combine top-down and bottom-up, meet in the middle
│     * When to use each strategy (with decision matrix)
│     * Integration testing in microservices: testing service-to-service communication
│     * TestContainers for realistic integration tests
│   - System testing:
│     * Full system testing vs E2E testing — is there a difference?
│     * Testing the whole application as a black box
│   - Acceptance testing:
│     * UAT: who does it, how QA supports it
│     * Alpha/beta testing
│   - The testing pyramid revisited:
│     * Classic pyramid: many unit → fewer integration → few E2E
│     * Ice cream cone anti-pattern: too many E2E, too few unit
│     * Testing trophy (Kent C. Dodds): focus on integration
│     * Practical pyramid for YOUR project — how to decide the right balance
│   - Interview questions: "How do you decide the balance between unit, integration, and E2E tests?"
│
├── 05_manual_testing_process.md
│   # Ручное тестирование: процесс от требований до sign-off
│   
│   Cover:
│   - Requirements analysis: what QA looks for in requirements
│     * Ambiguity, missing scenarios, edge cases, testability
│     * How to review requirements and provide feedback BEFORE development starts
│     * Shift-left: catching defects in requirements saves 10x vs finding them in production
│   - Test design: from requirements to test cases
│     * Applying test design techniques (EP, BVA, decision tables) to real requirements
│     * Positive scenarios: happy path, all valid combinations
│     * Negative scenarios: invalid input, missing fields, boundary violations
│     * Edge cases: empty strings, very long strings, special characters, null, zero, max values
│     * Cross-browser / cross-device considerations
│   - Test execution: how to test effectively
│     * Structured testing vs exploratory testing — when to use each
│     * Session-based exploratory testing
│     * Regression testing: what to include, how often
│     * Smoke testing: minimal set to verify deployment
│     * Sanity testing: focused verification of specific fix/feature
│   - Bug reporting and tracking
│     * Writing effective bug reports (Steps to Reproduce that developers love)
│     * Severity vs Priority decision framework
│     * Bug lifecycle: New → Assigned → In Progress → Fixed → Verified → Closed (or Reopened)
│     * When to reopen vs create new bug
│   - Sign-off: when is testing "done"?
│     * Exit criteria: all critical/high bugs fixed, test pass rate > X%, no open blockers
│     * Risk-based sign-off: accepting known issues with documented risk
│     * Go/No-Go meeting: QA's role in release decisions
│
├── 06_automation_strategy.md
│   # Стратегия автоматизации: что автоматизировать и когда
│   
│   Cover:
│   - The automation decision: what to automate, what to keep manual
│     * Criteria: frequency of execution, stability of feature, complexity, ROI
│     * Automation candidates: regression, smoke, data-driven, cross-browser
│     * Keep manual: exploratory, usability, visual, one-time checks
│     * The 80/20 rule: automate the 20% of tests that cover 80% of risk
│   - When to START automating on a new project
│     * Not on day 1 — first understand the product and manual testing
│     * Typical timeline: week 1-2 manual → week 3-4 first automation → month 2+ framework
│     * Starting with API tests (more stable, faster ROI) before UI tests
│   - Automation ROI: how to calculate and present to management
│     * Cost of automation: framework setup + test creation + maintenance
│     * Cost of manual: time × frequency × number of testers
│     * Break-even point: when automation starts saving money
│     * Non-monetary benefits: speed, consistency, confidence, CI/CD enablement
│   - Automation pyramid applied to YOUR project
│     * Layer 1: API tests (70%) — fast, stable, cover business logic
│     * Layer 2: UI tests (20%) — critical user flows only
│     * Layer 3: Manual + Exploratory (10%) — creativity, edge cases, UX
│   - Growing automation maturity:
│     * Stage 1: Basic scripts — individual tests, no framework
│     * Stage 2: Framework — POM, configuration, utilities, reporting
│     * Stage 3: CI/CD integration — tests run automatically on every commit/merge
│     * Stage 4: Full pipeline — smoke on PR, regression on merge, scheduled nightly runs
│     * Stage 5: Advanced — parallel execution, multi-environment, monitoring, self-healing
│   - Selling automation to the team: how to get developers and management on board
│   - Interview angle: "How would you decide what to automate on a new project?"
│
├── 07_api_testing_approach.md
│   # Подход к API-тестированию: покрытие и кейсы
│   
│   Cover:
│   - API testing checklist per endpoint:
│     * Positive cases: valid request → expected response (status, body, headers)
│     * Negative cases: missing required fields, invalid types, empty body, wrong method
│     * Boundary cases: max length strings, zero, negative numbers, very large payloads
│     * Authentication: valid token, expired token, no token, wrong role
│     * Authorization: access to own resources, access to others' resources (IDOR)
│     * Pagination: first page, last page, page beyond total, invalid page/size
│     * Filtering/sorting: valid filters, invalid filters, SQL injection attempts
│     * Error handling: proper error codes, meaningful error messages, no stack traces exposed
│   - API coverage strategy:
│     * What "good coverage" looks like: every endpoint × positive + 3-5 negative scenarios
│     * CRUD coverage matrix: Create → Read → Update → Delete for each resource
│     * Integration scenarios: create via API → verify via DB, create via UI → verify via API
│   - Response validation levels:
│     * Level 1: Status code only (200, 201, 404, etc.)
│     * Level 2: Status + body structure (JSON Schema validation)
│     * Level 3: Status + body + specific field values
│     * Level 4: Status + body + headers + response time
│   - Contract testing: when and why
│     * Consumer-driven contracts: frontend team defines what they expect
│     * Provider verification: backend verifies it meets the contract
│     * When contracts matter most: separate frontend/backend teams, microservices
│   - Common API bugs to look for:
│     * Wrong status codes (200 on creation instead of 201, 200 on error instead of 4xx)
│     * Missing validation (accepts null for required fields)
│     * Inconsistent error format (sometimes JSON, sometimes plain text)
│     * Leaking sensitive data in responses (passwords, tokens, internal IDs)
│     * Missing pagination (returns all 10,000 records at once)
│     * Race conditions on concurrent requests
│
├── 08_ui_testing_approach.md
│   # Подход к UI-тестированию: что автоматизировать в интерфейсе
│   
│   Cover:
│   - What to automate in UI (and what NOT to):
│     * Automate: critical user flows (login, registration, checkout, core business actions)
│     * Automate: smoke tests (basic navigation, page loads, key elements present)
│     * Don't automate: complex visual layouts, pixel-perfect design, dynamic content
│     * Don't automate: features that change every sprint (until stabilized)
│   - UI test coverage strategy:
│     * Critical path coverage: 5-10 most important user journeys
│     * Page-level smoke: each page loads without errors, key elements visible
│     * Form validation: required fields, format validation, error messages
│     * Cross-browser: Chrome + Firefox minimum, add Safari/Edge based on user analytics
│   - UI test design principles:
│     * Test through API when possible, verify through UI (create data via API, check UI)
│     * Independent tests: each test sets up its own data, doesn't depend on others
│     * Stable locators: data-testid > CSS > XPath; avoid fragile locators
│     * Smart waits: explicit waits for specific conditions, never Thread.sleep
│   - Dealing with flaky UI tests:
│     * Root causes: timing issues, shared state, environment instability, poor locators
│     * Prevention: proper waits, test isolation, retry strategy
│     * Quarantine: move flaky tests to separate suite, fix or delete within 2 weeks
│   - Visual regression testing (overview):
│     * Screenshot comparison: Playwright, Percy, Applitools
│     * When it's useful: design-heavy applications, component libraries
│
├── 09_integration_testing_strategies.md
│   # Стратегии интеграционного тестирования
│   
│   Cover in detail:
│   - Top-down integration:
│     * Start from the top layer (UI or API controller)
│     * Mock lower layers (services, repositories, external APIs)
│     * Gradually replace mocks with real implementations
│     * Pros: early validation of user flows, identifies high-level issues
│     * Cons: lower layers tested later, mocks may hide integration issues
│     * When to use: well-defined API, unclear lower layers
│   - Bottom-up integration:
│     * Start from the lowest layer (database, external services)
│     * Build test drivers for higher layers
│     * Gradually integrate upward
│     * Pros: thorough testing of core logic, no mocks for data layer
│     * Cons: UI/API tested last, late discovery of integration issues at higher levels
│     * When to use: complex business logic in service/data layer
│   - Sandwich (hybrid) integration:
│     * Combine top-down and bottom-up, meet in the middle
│     * Test top layers with mocks AND bottom layers with drivers simultaneously
│     * Pros: balanced, parallel development of tests
│     * Cons: more complex test setup, requires coordination
│   - Big bang integration:
│     * Integrate all components at once, test as a whole
│     * Pros: simple to set up, realistic testing
│     * Cons: hard to isolate failures, late defect discovery, debugging nightmare
│     * When to use: small systems, time pressure, well-tested individual components
│   - Integration testing in microservices:
│     * Service-level integration: test one service with its dependencies (DB, cache)
│     * Cross-service integration: test communication between services
│     * Contract testing as lightweight integration validation
│     * TestContainers for realistic service dependencies
│   - Comparison table: all strategies with pros/cons/when to use
│   - Practical example: choosing integration strategy for a 3-service system
│     (API Gateway → Order Service → Payment Service → PostgreSQL)
│
├── 10_test_environments.md
│   # Тестовые окружения: настройка и управление
│   
│   Cover:
│   - Environment types: local, dev, QA/staging, pre-prod, production
│   - What each environment is for:
│     * Local: developer + QA running tests on their machine
│     * Dev: latest code, unstable, for quick integration checks
│     * QA/Staging: stable for manual and automated testing, mirrors production
│     * Pre-prod: final validation before release, production-like data
│     * Production: monitoring, smoke tests, canary testing
│   - Environment management for QA:
│     * How to request/configure environments
│     * Test data management: seeding, cleanup, isolation
│     * Docker-compose for local test environments
│     * Environment-specific configuration in tests
│   - Common environment problems and solutions:
│     * Stale data, environment down, version mismatch, shared state between testers
│   - Interview: "How do you manage test environments on your project?"
│
├── 11_test_coverage_and_metrics.md
│   # Тестовое покрытие и метрики качества
│   
│   Cover:
│   - Test coverage types:
│     * Requirements coverage: % of requirements with at least one test case
│     * Code coverage: line, branch, condition (what it means, what's "good enough")
│     * Risk coverage: % of high-risk areas tested
│     * API coverage: endpoints × methods × scenarios
│   - Traceability matrix: requirements → test cases → bugs
│   - Quality metrics QA should track:
│     * Defect density (bugs per feature/module)
│     * Test pass rate (% passing in latest run)
│     * Automation coverage (% automated vs total test cases)
│     * Defect leakage (bugs found in production vs total bugs)
│     * Test execution time (regression suite duration)
│     * Flaky test rate (% of unreliable tests)
│   - What is "good enough" coverage — the pragmatic approach
│   - How to present metrics to management (dashboards, trends, not just numbers)
│   - What NOT to measure (vanity metrics, coverage for coverage's sake)
│
├── 12_release_process.md
│   # Релизный процесс: роль QA от feature freeze до production
│   
│   Cover:
│   - Release process stages:
│     * Feature freeze: no new features, only bug fixes
│     * Regression testing: full regression suite (manual + automated)
│     * Smoke testing on staging: critical paths after deployment
│     * Go/No-Go meeting: QA presents quality status, risks, open issues
│     * Production deployment: smoke tests, monitoring
│     * Post-release: monitoring for issues, hotfix process
│   - Hotfix process:
│     * Critical bug in production → emergency fix → targeted testing → deploy
│     * What to test: the fix itself + related functionality (no full regression)
│   - Release notes from QA perspective:
│     * Known issues, workarounds, areas of risk
│   - Rollback criteria: when to roll back a release
│   - Canary releases and feature flags: how they change QA's role
│   - Continuous deployment: how QA adapts when there's no "release day"
│
└── 13_growing_qa_maturity.md
    # Рост зрелости тестирования: от хаоса к системе
    
    Cover:
    - QA maturity levels:
      * Level 0: No testing (or ad-hoc only)
      * Level 1: Basic manual testing, some test cases
      * Level 2: Structured testing, test plans, bug tracking
      * Level 3: Automation started, CI/CD integration
      * Level 4: Comprehensive automation, metrics, reporting, shift-left
      * Level 5: Quality engineering culture, testing embedded in development
    - How to move between levels:
      * Level 0→1: Hire first QA, start documenting bugs, create basic test cases
      * Level 1→2: Introduce test plans, structured regression, prioritized testing
      * Level 2→3: Start API automation, integrate with CI, establish smoke suite
      * Level 3→4: Expand automation (UI), add reporting, track metrics, parallel runs
      * Level 4→5: Contract testing, shift-left, developers write tests, quality gates
    - Timeline: realistic expectations (each level = 3-6 months)
    - How to identify current maturity level on a project
    - Making the case for improvement: speaking management's language (risk, cost, speed)
    - Interview: "You join a project with no testing. What do you do in the first 3 months?"
```

### Step 3: Cross-Reference

After creating all files in this directory, add a reference to it in the main `README.md` by updating the directory listing and status table. This directory should appear as directory 23 in the table.

### Step 4: Link from Existing Content

Where appropriate, add cross-references from existing files to this new directory:
- In `01_testing_theory/06_test_documentation.md` — link to test strategy and test plan files
- In `06_framework_architecture/01_framework_design_principles.md` — link to automation strategy
- In `12_processes_for_qa/05_release_process.md` — link to release process file
- In `12_processes_for_qa/01_sdlc_and_stlc.md` — link to testing levels and integration strategies

## Content Rules

Same as the main knowledge base:
- All text in Russian, tech terms in English
- QA perspective always
- Code examples with Russian comments
- Each file: 150-300+ lines minimum
- Sections: Обзор → Content → Связь с тестированием → Типичные ошибки → Вопросы на интервью → Практические задания → Ресурсы

## Key Quality Requirements for This Directory

This directory is **strategically critical** — it covers the topic that separates a test executor from a test leader. Interview questions like "How would you set up testing on a new project?" or "What's your automation strategy?" are extremely common for middle+ positions. The content must be:

1. **Practical, not academic** — real examples from real projects, not textbook definitions
2. **Opinionated where appropriate** — "In most cases, start with API automation before UI" is more useful than "it depends"
3. **Progressive** — show how testing evolves over time, not a final state
4. **Honest about trade-offs** — every decision has a cost, acknowledge it
5. **Rich in examples** — each strategy should have a concrete example applied to a realistic project

## Begin

Create the directory and generate all 13 files with full, detailed content.
