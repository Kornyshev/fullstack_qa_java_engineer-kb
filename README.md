# Fullstack QA Engineer (Java Stack) — Карта подготовки к интервью

## Мастер-документ для генерации конспектов

**Целевой уровень:** Fullstack QA Engineer, 1–2 года опыта (ручное + автоматизация)
**Целевая позиция:** QA Automation Engineer / Fullstack QA Engineer (Java stack)
**Язык конспектов:** Русский
**Названия технологий:** Английский
**Формат файлов:** Markdown (.md)
**Именование директорий и файлов:** Английский (snake_case)

---

## Правила для агента

1. Все тексты конспектов — на русском языке
2. Названия технологий, классов, аннотаций, команд — на английском
3. Примеры кода — с комментариями на русском
4. Каждый конспект — самодостаточный документ
5. Глубина — уровень 1–2 года опыта: уверенный junior+ / начинающий middle
6. В каждом конспекте: теория + примеры + типичные вопросы на интервью + практические задания
7. Перспектива — всегда с точки зрения тестировщика, а не разработчика

---

## Структура директорий

```
qa_interview_prep/
├── README.md                              # Этот файл — общая карта
│
├── 01_testing_theory/
│   ├── 01_fundamentals.md                 # Цели тестирования, QA vs QC vs Testing, верификация vs валидация
│   ├── 02_principles.md                   # 7 принципов ISTQB, парадокс пестицида, кластеризация дефектов
│   ├── 03_levels_and_types.md             # Уровни: unit → интеграционное → системное → приёмочное; пирамида тестирования
│   ├── 04_types_detailed.md               # Smoke, sanity, regression, re-test, exploratory, E2E, совместимость
│   ├── 05_test_design_techniques.md       # Классы эквивалентности, граничные значения, decision tables, state transition, pairwise
│   ├── 06_test_documentation.md           # Тест-план, тест-стратегия, тест-кейсы, чек-листы, баг-репорт
│   ├── 07_bug_reports.md                  # Severity vs Priority (с примерами комбинаций), жизненный цикл бага, best practices
│   ├── 08_metrics_and_coverage.md         # Тестовое покрытие, traceability matrix, метрики качества, DoD/DoR для QA
│   └── 09_non_functional_testing.md       # Производительность, безопасность, юзабилити, доступность, надёжность
│
├── 02_java_for_qa/
│   ├── 01_java_basics.md                  # Типы данных, переменные, операторы, условия, циклы, массивы
│   ├── 02_oop.md                          # Классы, наследование, полиморфизм, инкапсуляция, абстракция, SOLID
│   ├── 03_strings_and_wrappers.md         # String, StringBuilder, autoboxing, String pool
│   ├── 04_collections.md                  # List, Set, Map, Queue — когда что; HashMap internals, ArrayList vs LinkedList
│   ├── 05_stream_api.md                   # filter, map, flatMap, collect, reduce — с примерами для обработки тестовых данных
│   ├── 06_exceptions.md                   # Checked vs unchecked, try-with-resources, кастомные исключения
│   ├── 07_generics.md                     # Type erasure, wildcards, PECS — обзорно для QA
│   ├── 08_functional_interfaces.md        # Lambda, Predicate, Function, Consumer, Optional
│   ├── 09_annotations_and_reflection.md   # Встроенные аннотации, кастомные, Reflection — где используется в фреймворках
│   ├── 10_serialization_jackson.md        # Jackson ObjectMapper, сериализация/десериализация, @JsonProperty, @JsonIgnore
│   ├── 11_lombok.md                       # @Data, @Builder, @Slf4j, @RequiredArgsConstructor — для тестовых классов
│   └── 12_multithreading_overview.md      # Thread, Runnable, synchronized, volatile — обзорно для понимания параллельных тестов
│
├── 03_test_frameworks/
│   ├── 01_junit5.md                       # Аннотации, lifecycle, параметризация, @Nested, Extensions, параллельное выполнение
│   ├── 02_testng.md                       # Отличия от JUnit, @DataProvider, testng.xml, groups, listeners, soft assertions
│   ├── 03_assertj_hamcrest.md             # Fluent assertions, matchers, кастомные matchers, сравнение
│   ├── 04_mockito.md                      # @Mock, @InjectMocks, @Spy, when/thenReturn, verify, ArgumentCaptor
│   └── 05_test_data_management.md         # Faker, Builder pattern для тестовых объектов, Data-driven testing, Object Mother
│
├── 04_ui_automation/
│   ├── 01_selenium_webdriver.md           # Архитектура, локаторы (id, css, xpath), ожидания, Actions, alerts, iframes
│   ├── 02_selenide.md                     # Fluent API, коллекции, конфигурация, преимущества над чистым Selenium
│   ├── 03_playwright_java.md              # Browser contexts, локаторы (getByRole, getByText), auto-wait, network interception, tracing
│   ├── 04_page_object_model.md            # POM: зачем, структура, BasePage, компоненты, Page Factory, Fluent POM, Step Object
│   ├── 05_locator_strategies.md           # XPath оси, CSS-селекторы, data-testid, приоритеты выбора, антипаттерны
│   ├── 06_cross_browser_testing.md        # Selenoid, Selenium Grid, Docker-контейнеры для браузеров, cloud-сервисы
│   └── 07_mobile_testing_overview.md      # Appium обзорно, мобильный web vs нативные приложения
│
├── 05_api_automation/
│   ├── 01_rest_api_fundamentals.md        # HTTP-методы, статус-коды, заголовки, тело запроса, REST constraints
│   ├── 02_rest_assured.md                 # given/when/then, сериализация POJO, JSON Schema validation, specifications
│   ├── 03_api_auth_testing.md             # Basic, Bearer, OAuth2 — как тестировать аутентификацию в API
│   ├── 04_contract_testing.md             # Consumer-driven contracts, Pact, Spring Cloud Contract — обзорно
│   ├── 05_mocking_wiremock.md             # WireMock: stubbing, verification, scenarios; MockServer; зачем мокировать
│   └── 06_graphql_grpc_overview.md        # GraphQL и gRPC — обзорно с точки зрения тестирования
│
├── 06_framework_architecture/
│   ├── 01_framework_design_principles.md  # Зачем нужен фреймворк, слои, модульность, масштабируемость, поддерживаемость
│   ├── 02_project_structure.md            # Типичная структура: pages/, tests/, utils/, config/, data/; Maven/Gradle модули
│   ├── 03_config_management.md            # application.properties, .env, Owner library, профили (dev/staging/prod)
│   ├── 04_patterns_in_automation.md       # Factory, Builder, Strategy, Decorator, Chain of Responsibility — в контексте тестов
│   ├── 05_test_layers.md                  # UI-слой, API-слой, DB-слой — как комбинировать в одном фреймворке
│   ├── 06_parallel_execution.md           # Параллельный запуск: JUnit 5, TestNG, Selenoid, fork, thread safety
│   └── 07_framework_from_scratch.md       # Пошаговый гайд: как поднять фреймворк с нуля (от init до первого прогона)
│
├── 07_reporting/
│   ├── 01_allure_framework.md             # @Step, @Attachment, @Epic/@Feature/@Story, @Severity, Allure Report, trends
│   ├── 02_allure_integration.md           # Интеграция с JUnit 5, TestNG; скриншоты, логи, видео как attachments
│   ├── 03_allure_testops.md               # Allure TestOps — обзорно: TMS, запуск тестов, аналитика
│   ├── 04_report_portal.md                # ReportPortal — обзорно: ML-анализ падений, дашборды, интеграция
│   └── 05_other_reporting.md              # ExtentReports, pytest-html, Cucumber Reports — обзорно
│
├── 08_ci_cd_for_qa/
│   ├── 01_ci_cd_concepts.md               # CI/CD для QA: pipeline, stages, когда запускать тесты, артефакты
│   ├── 02_jenkins.md                      # Jenkins Pipeline: Jenkinsfile, stages, parallel тесты, Allure plugin, credentials
│   ├── 03_gitlab_ci.md                    # .gitlab-ci.yml: stages, runners, артефакты (Allure results), переменные
│   ├── 04_github_actions.md               # workflow.yml: matrix strategy, Allure отчёт, артефакты, кэширование
│   └── 05_test_infrastructure.md          # Selenoid, Docker Selenium Grid, TestContainers в CI, управление окружениями
│
├── 09_databases_for_qa/
│   ├── 01_sql_basics.md                   # SELECT, JOIN, WHERE, GROUP BY, INSERT, UPDATE, DELETE — для проверки данных
│   ├── 02_advanced_sql.md                 # Подзапросы, оконные функции, CTE — для сложных проверок
│   ├── 03_db_testing.md                   # Проверка бизнес-логики через БД, JDBC, прямые SQL-запросы в тестах
│   ├── 04_testcontainers.md               # PostgreSQL, Oracle, MongoDB в Docker для тестов
│   └── 05_transactions_acid.md            # ACID, уровни изоляции — понимание для тестирования
│
├── 10_architecture_for_qa/
│   ├── 01_client_server.md                # Frontend vs Backend, SPA vs MPA, как это влияет на тестирование
│   ├── 02_monolith_vs_microservices.md    # Архитектуры, как влияют на стратегию тестирования
│   ├── 03_rest_http_deep.md               # HTTP протокол глубже: методы, заголовки, cookies, кэширование, CORS
│   ├── 04_messaging_kafka.md              # Kafka: topics, partitions, consumer groups — как тестировать
│   ├── 05_docker_for_qa.md                # Docker, docker-compose, поднятие тестовых окружений, Dockerfile
│   ├── 06_kubernetes_overview.md          # K8s обзорно для QA: pods, services, как тестировать приложения в K8s
│   ├── 07_linux_basics.md                 # Базовые команды: ls, cd, grep, tail, ps, kill, chmod, ssh — для работы с серверами
│   └── 08_networking.md                   # OSI, TCP/IP, DNS, HTTP/HTTPS — обзорно для понимания
│
├── 11_testing_microservices/
│   ├── 01_strategies.md                   # Стратегии: unit + component + integration + E2E в микросервисах
│   ├── 02_kafka_testing.md                # Тестирование producer/consumer, Kafka TestContainers, проверка формата сообщений
│   ├── 03_spring_boot_test.md             # @SpringBootTest, @WebMvcTest, @DataJpaTest, test slices, @MockBean
│   └── 04_contract_testing_practice.md    # Практика контрактного тестирования в микросервисной архитектуре
│
├── 12_processes_for_qa/
│   ├── 01_sdlc_and_stlc.md               # SDLC и STLC: как тестирование встроено в жизненный цикл разработки
│   ├── 02_scrum_for_qa.md                 # Роль QA в Scrum: planning, grooming, daily, review, retro
│   ├── 03_team_roles.md                   # PM, BA, SA, Dev, QA, DevOps — взаимодействие, кто что делает
│   ├── 04_qa_daily_workflow.md            # Рабочий день QA: получение задачи, анализ, тестирование, автотесты, репортинг
│   ├── 05_release_process.md              # Релизный процесс: regression, smoke на staging, go/no-go, hotfix
│   ├── 06_test_management.md              # TMS: Jira + Zephyr, TestRail, Allure TestOps — управление тест-кейсами
│   ├── 07_task_example_manual.md          # Пример: ручное тестирование новой фичи от требований до sign-off
│   ├── 08_task_example_automation.md      # Пример: написание автотеста на новый эндпоинт от тикета до мержа
│   ├── 09_task_example_regression.md      # Пример: запуск регрессии, анализ падений, flaky-тесты, отчёт
│   └── 10_task_example_bug_investigation.md # Пример: расследование бага на проде (логи, БД, воспроизведение)
│
├── 13_self_presentation/
│   ├── 01_self_intro_template.md          # Шаблон самопрезентации QA: кто я, опыт, стек, что ищу
│   ├── 02_project_story_qa.md             # Шаблон рассказа о проекте QA: продукт, стек, покрытие, вызовы, результаты
│   ├── 03_behavioral_questions_qa.md      # STAR-метод для QA: конфликт с разработчиком, пропущенный баг, сложный тест
│   └── 04_questions_to_interviewer.md     # Что спрашивать: процессы QA, автоматизация, техдолг в тестах, отношение к качеству
│
├── 14_design_patterns/
│   ├── 01_patterns_overview.md            # GoF обзор: зачем тестировщику знать паттерны
│   ├── 02_creational.md                   # Singleton, Factory, Builder — с примерами в тестах (Builder для тестовых данных)
│   ├── 03_structural.md                   # Adapter, Decorator, Proxy, Facade — где встречаются в фреймворках
│   ├── 04_behavioral.md                   # Strategy, Observer, Template Method, Chain of Responsibility — в контексте тестов
│   ├── 05_solid_for_qa.md                 # SOLID с примерами из автоматизации тестирования
│   └── 06_clean_code_refactoring.md       # Code smells в тестах, DRY/KISS/YAGNI, рефакторинг тестов
│
├── 15_spring_for_qa/
│   ├── 01_spring_overview.md              # IoC/DI, ApplicationContext — что QA нужно знать о Spring
│   ├── 02_spring_boot_basics.md           # Spring Boot: стартеры, автоконфигурация, профили, application.properties
│   ├── 03_spring_architecture.md          # Controller → Service → Repository: понимание слоёв для тестирования
│   ├── 04_spring_annotations.md           # Основные аннотации: @Component, @Service, @RestController, @Autowired, @Transactional
│   └── 05_spring_testing.md               # @SpringBootTest, @WebMvcTest, @DataJpaTest, MockMvc — тестирование Spring-приложений
│
├── 16_git/
│   └── 01_git_workflow.md                 # Git для QA: основные команды, Git Flow, ветки для автотестов, resolve conflicts
│
├── 17_build_tools/
│   └── 01_maven_gradle.md                 # Maven: pom.xml, lifecycle, surefire plugin; Gradle: build.gradle, tasks; запуск тестов
│
├── 18_computer_science/
│   ├── 01_data_structures.md              # Массивы, списки, деревья, хеш-таблицы — обзорно, Big O
│   ├── 02_algorithms.md                   # Сортировки, поиск — базово, что могут спросить
│   └── 03_networking.md                   # OSI, TCP/IP, HTTP/HTTPS, DNS, WebSocket — для понимания архитектуры
│
├── 19_common_problems/
│   ├── 01_flaky_tests.md                  # Flaky-тесты: причины (timing, shared state, env), диагностика, стратегии исправления
│   ├── 02_environment_issues.md           # Проблемы окружений: нестабильные стенды, рассинхрон данных, VPN, доступы
│   ├── 03_test_maintenance.md             # Поддержка тестов: устаревшие локаторы, изменения API, рост сьюта, технический долг
│   ├── 04_debugging_failures.md           # Как анализировать падения: логи, скриншоты, видео, trace, Allure history
│   └── 05_common_automation_mistakes.md   # Типичные ошибки: хардкод, sleep вместо wait, тесты зависят друг от друга, нет изоляции
│
├── 20_performance_security/
│   ├── 01_performance_testing.md          # Load, stress, spike, soak; метрики (p50/p90/p99, throughput, error rate)
│   ├── 02_jmeter_basics.md               # JMeter: test plan, thread group, samplers, assertions, listeners
│   └── 03_security_testing_overview.md    # OWASP Top 10 обзорно, SQL Injection, XSS, CSRF — как тестировать
│
├── 21_ai_in_testing/
│   ├── 01_ai_tools_for_qa.md              # LLM для генерации тестов, prompt engineering для тест-кейсов и автотестов
│   └── 02_practical_ai_usage.md           # Bulk test generation, генерация тестовых данных, code review тестов с AI
│
└── 23_building_qa_from_scratch/
    ├── 01_starting_point.md               # С чего начать: первые шаги QA на новом проекте
    ├── 02_test_strategy.md                # Тестовая стратегия: что тестируем, как, зачем
    ├── 03_test_plan.md                    # Тест-план: детальное планирование тестирования
    ├── 04_testing_levels_deep.md          # Уровни тестирования: что, когда и почему
    ├── 05_manual_testing_process.md       # Ручное тестирование: процесс от требований до sign-off
    ├── 06_automation_strategy.md          # Стратегия автоматизации: что автоматизировать и когда
    ├── 07_api_testing_approach.md         # Подход к API-тестированию: покрытие и кейсы
    ├── 08_ui_testing_approach.md          # Подход к UI-тестированию: что автоматизировать в интерфейсе
    ├── 09_integration_testing_strategies.md # Стратегии интеграционного тестирования
    ├── 10_test_environments.md            # Тестовые окружения: настройка и управление
    ├── 11_test_coverage_and_metrics.md    # Тестовое покрытие и метрики качества
    ├── 12_release_process.md              # Релизный процесс: роль QA от feature freeze до production
    └── 13_growing_qa_maturity.md          # Рост зрелости тестирования: от хаоса к системе
```

---

## Детальное описание содержания каждой директории

### 01_testing_theory/

Фундамент профессии. Спрашивают на каждом интервью.

| Файл | Содержание | Ключевые вопросы на интервью |
|------|-----------|------|
| `01_fundamentals.md` | Зачем тестировать, что такое качество ПО, QA vs QC vs Testing, верификация vs валидация | «Чем QA отличается от QC?», «Что такое верификация?» |
| `02_principles.md` | 7 принципов ISTQB: исчерпывающее тестирование невозможно, кластеризация дефектов, парадокс пестицида, раннее тестирование, зависимость от контекста | «Перечислите принципы тестирования», «Что такое парадокс пестицида?» |
| `03_levels_and_types.md` | Пирамида тестирования (классическая, ice cream cone, trophy), unit → интеграционное → системное → приёмочное, кто за что отвечает | «Расскажите про пирамиду тестирования», «Чем системное отличается от интеграционного?» |
| `04_types_detailed.md` | Smoke, sanity, regression, re-test, E2E, exploratory, black/white/grey-box, позитивное/негативное | «Чем smoke отличается от sanity?», «Когда применять exploratory?» |
| `05_test_design_techniques.md` | Классы эквивалентности, граничные значения, decision tables, state transition, pairwise, error guessing — с примерами | «Какие техники тест-дизайна знаете?», «Примените BVA к полю возраста» |
| `06_test_documentation.md` | Тест-план (структура, кто пишет), тест-стратегия, тест-кейсы (поля, уровни детализации), чек-листы | «Что включает тест-план?», «Чем тест-план от стратегии отличается?» |
| `07_bug_reports.md` | Severity vs Priority (S1P3, S3P1 примеры), жизненный цикл бага, обязательные поля, best practices | «Приведите пример бага High Severity Low Priority» |
| `08_metrics_and_coverage.md` | Тестовое покрытие, traceability matrix, метрики (defect density, test pass rate, automation %), DoD/DoR | «Как измерить тестовое покрытие?» |
| `09_non_functional_testing.md` | Performance (load, stress, spike), безопасность (OWASP Top 10 обзорно), юзабилити, доступность | «Чем load-тест от stress-теста отличается?» |

### 02_java_for_qa/

Java с точки зрения тестировщика — что нужно знать для написания автотестов и прохождения интервью.

| Файл | Содержание |
|------|-----------|
| `01_java_basics.md` | Примитивы, обёртки, autoboxing, операторы, условия, циклы, массивы — базовые конструкции |
| `02_oop.md` | 4 принципа ООП, abstract vs interface, SOLID — каждый принцип с примером из автоматизации |
| `03_strings_and_wrappers.md` | String pool, иммутабельность, StringBuilder, обёртки примитивов |
| `04_collections.md` | List, Set, Map — когда что; HashMap internals (коллизии, rehashing); ArrayList vs LinkedList |
| `05_stream_api.md` | filter, map, flatMap, collect, reduce, groupingBy — примеры обработки тестовых данных |
| `06_exceptions.md` | Checked vs unchecked, try-with-resources, кастомные исключения в тестовом фреймворке |
| `07_generics.md` | Type erasure, wildcards — обзорно, достаточно для понимания фреймворков |
| `08_functional_interfaces.md` | Lambda, Predicate, Function, Consumer, Optional — использование в тестах |
| `09_annotations_and_reflection.md` | @Override, @Deprecated, кастомные аннотации; Reflection — где используется (JUnit, Spring, Allure) |
| `10_serialization_jackson.md` | Jackson ObjectMapper для API-тестов: сериализация POJO, @JsonProperty, @JsonIgnore |
| `11_lombok.md` | @Data, @Builder, @Slf4j — как упростить тестовый код |
| `12_multithreading_overview.md` | Thread, Runnable, synchronized, volatile — обзорно для понимания параллельных тестов и потокобезопасности |

### 03_test_frameworks/

Инструменты для написания и запуска тестов.

| Файл | Содержание |
|------|-----------|
| `01_junit5.md` | Аннотации, lifecycle, @ParameterizedTest, @Nested, Extensions, параллельное выполнение |
| `02_testng.md` | Отличия от JUnit, @DataProvider, testng.xml, groups, listeners, soft assertions, dependsOnMethods |
| `03_assertj_hamcrest.md` | AssertJ fluent assertions, Hamcrest matchers, кастомные matchers, сравнение |
| `04_mockito.md` | @Mock, @InjectMocks, @Spy, when/thenReturn, verify, ArgumentCaptor, doThrow — для изоляции тестов |
| `05_test_data_management.md` | Faker, Builder pattern для тестовых объектов, Data-driven testing, Object Mother, Test Data Factory |

### 04_ui_automation/

Автоматизация UI: Selenium, Selenide, Playwright, паттерны.

| Файл | Содержание |
|------|-----------|
| `01_selenium_webdriver.md` | Архитектура, локаторы, ожидания (implicit/explicit/fluent), Actions, alerts, iframes, скриншоты |
| `02_selenide.md` | Отличия от Selenium, fluent API, коллекции ($$), конфигурация, Selenide + Allure |
| `03_playwright_java.md` | Browser contexts, pages, локаторы (getByRole/getByText/getByTestId), auto-wait, network interception |
| `04_page_object_model.md` | POM: зачем, BasePage, конкретные страницы, компоненты, Page Factory, Fluent POM, Step Object |
| `05_locator_strategies.md` | XPath оси (ancestor, following-sibling), CSS-селекторы, data-testid, приоритеты выбора локаторов |
| `06_cross_browser_testing.md` | Selenoid, Selenium Grid, Docker-контейнеры для браузеров, cloud-сервисы (BrowserStack, SauceLabs) |
| `07_mobile_testing_overview.md` | Appium обзорно, capabilities, мобильный web vs нативные приложения |

### 05_api_automation/

REST API автоматизация — вторая по важности после UI.

| Файл | Содержание |
|------|-----------|
| `01_rest_api_fundamentals.md` | HTTP-методы, статус-коды, заголовки, тело, REST constraints, idempotency |
| `02_rest_assured.md` | given/when/then, POJO-сериализация, JSON Schema validation, request/response specifications, логирование |
| `03_api_auth_testing.md` | Basic Auth, Bearer Token, OAuth2 flows — как тестировать каждый тип |
| `04_contract_testing.md` | Consumer-driven contracts, Pact, Spring Cloud Contract — зачем в микросервисах |
| `05_mocking_wiremock.md` | WireMock: stubbing, verification, scenarios; зачем мокировать внешние сервисы |
| `06_graphql_grpc_overview.md` | GraphQL: queries/mutations, тестирование; gRPC: protobuf, тестирование — обзорно |

### 06_framework_architecture/

Как проектировать тестовый фреймворк — критически важно для middle+.

| Файл | Содержание |
|------|-----------|
| `01_framework_design_principles.md` | Зачем фреймворк, слоистая архитектура, модульность, переиспользование, расширяемость |
| `02_project_structure.md` | Типичная структура: pages/, tests/, api/, utils/, config/, data/; Maven/Gradle модули |
| `03_config_management.md` | application.properties, .env файлы, Owner library, профили для разных окружений |
| `04_patterns_in_automation.md` | Factory (WebDriver), Builder (тестовые данные), Strategy (разные браузеры), Decorator (логирование шагов) |
| `05_test_layers.md` | UI-слой + API-слой + DB-слой: как комбинировать; когда тест идёт через API, а проверяет через UI |
| `06_parallel_execution.md` | Параллельный запуск: JUnit 5 (maven-surefire-plugin), TestNG (testng.xml), Selenoid, thread safety |
| `07_framework_from_scratch.md` | Пошаговый гайд: от init проекта до первого зелёного теста с Allure-отчётом |

### 07_reporting/

Репортинг и интеграция с TMS.

| Файл | Содержание |
|------|-----------|
| `01_allure_framework.md` | Аннотации (@Step, @Attachment, @Epic/@Feature/@Story, @Severity), Allure Report, timeline, categories |
| `02_allure_integration.md` | Интеграция с JUnit 5 и TestNG, скриншоты на падении, логи, видео, browser logs |
| `03_allure_testops.md` | Allure TestOps: TMS, ручные + автоматизированные тесты, запуск тестов, аналитика |
| `04_report_portal.md` | ReportPortal: ML-анализ падений, дашборды, интеграция с CI/CD |
| `05_other_reporting.md` | ExtentReports, Cucumber Reports — обзорно |

### 08_ci_cd_for_qa/

CI/CD с точки зрения тестировщика.

| Файл | Содержание |
|------|-----------|
| `01_ci_cd_concepts.md` | Pipeline, stages, когда запускать какие тесты (smoke на PR, regression на merge to main) |
| `02_jenkins.md` | Jenkinsfile, stages (build → test → report), параллельные тесты, Allure plugin, параметризованные сборки |
| `03_gitlab_ci.md` | .gitlab-ci.yml, stages, runners, артефакты с Allure results, переменные окружения |
| `04_github_actions.md` | workflow.yml, matrix strategy для кросс-браузерного запуска, артефакты Allure |
| `05_test_infrastructure.md` | Selenoid, Docker Selenium Grid, TestContainers в CI, управление тестовыми окружениями |

### 09_databases_for_qa/

SQL и работа с БД в контексте тестирования.

| Файл | Содержание |
|------|-----------|
| `01_sql_basics.md` | SELECT, JOIN, WHERE, GROUP BY, INSERT, UPDATE, DELETE — для проверки данных в тестах |
| `02_advanced_sql.md` | Подзапросы, оконные функции, CTE — для сложных проверок и отчётов |
| `03_db_testing.md` | Проверка бизнес-логики через БД, JDBC в Java-тестах, когда API-ответа недостаточно |
| `04_testcontainers.md` | TestContainers: PostgreSQL, MongoDB в Docker для тестов — setup, cleanup, best practices |
| `05_transactions_acid.md` | ACID, уровни изоляции — понимание для тестирования транзакционной логики |

### 10_architecture_for_qa/

Архитектура приложений — что тестировщик должен понимать.

| Файл | Содержание |
|------|-----------|
| `01_client_server.md` | Frontend vs Backend, как запрос идёт от клиента до БД и обратно |
| `02_monolith_vs_microservices.md` | Как архитектура влияет на стратегию тестирования; проблемы E2E в микросервисах |
| `03_rest_http_deep.md` | HTTP глубже: методы, заголовки, cookies, кэширование (ETag, Cache-Control), CORS |
| `04_messaging_kafka.md` | Kafka для QA: topics, partitions, consumer groups, как тестировать producer/consumer |
| `05_docker_for_qa.md` | Docker, docker-compose для тестовых окружений, Dockerfile, поднятие стенда одной командой |
| `06_kubernetes_overview.md` | K8s обзорно: pods, services, deployments — как это влияет на тестирование |
| `07_linux_basics.md` | ls, cd, grep, tail -f, ps, kill, chmod, ssh, scp — для работы с серверами и логами |
| `08_networking.md` | OSI, TCP/IP, DNS, HTTP/HTTPS, TLS — базовое понимание |

### 11_testing_microservices/

Отдельный блок — тестирование микросервисной архитектуры.

| Файл | Содержание |
|------|-----------|
| `01_strategies.md` | Стратегии: unit + component + integration + E2E, тестирование отдельного сервиса vs цепочки |
| `02_kafka_testing.md` | Тестирование producer/consumer, Kafka TestContainers, проверка формата сообщений (Avro, JSON) |
| `03_spring_boot_test.md` | @SpringBootTest, @WebMvcTest, @DataJpaTest, test slices, @MockBean — для QA |
| `04_contract_testing_practice.md` | Практика: Pact consumer test → provider verification → CI интеграция |

### 12_processes_for_qa/

Процессы с точки зрения QA — от получения задачи до релиза.

| Файл | Содержание |
|------|-----------|
| `01_sdlc_and_stlc.md` | Жизненный цикл разработки и тестирования, как QA встроен в каждый этап |
| `02_scrum_for_qa.md` | QA в Scrum: когда участвовать, что делать на каждой церемонии |
| `03_team_roles.md` | Кто есть кто: PM, BA, SA, Dev, QA, DevOps — как QA взаимодействует с каждым |
| `04_qa_daily_workflow.md` | Типичный день QA: утро (daily), тестирование фичей, написание автотестов, анализ падений, отчёт |
| `05_release_process.md` | Регрессия, smoke на staging, go/no-go meeting, hotfix, rollback |
| `06_test_management.md` | TMS: Jira + Zephyr, TestRail, Allure TestOps — управление тест-кейсами, метрики |
| `07_task_example_manual.md` | **Пример:** Тестирование новой фичи вручную: требования → тест-кейсы → выполнение → баг-репорты → retesting → sign-off |
| `08_task_example_automation.md` | **Пример:** Автоматизация: тикет → анализ → Page Object → тест → Allure → PR → ревью → мерж |
| `09_task_example_regression.md` | **Пример:** Запуск регрессии: триггер в CI → анализ отчёта → flaky-тесты → новые падения → отчёт команде |
| `10_task_example_bug_investigation.md` | **Пример:** Баг на проде: алерт → логи (ELK/Kibana) → БД → воспроизведение → баг-репорт → hotfix |

### 13_self_presentation/

Подготовка к рассказу о себе на интервью QA.

| Файл | Содержание |
|------|-----------|
| `01_self_intro_template.md` | Шаблон: кто я → опыт в ручном и автоматизации → стек → что ищу. 2–3 минуты |
| `02_project_story_qa.md` | Рассказ о проекте QA: продукт, архитектура, стек автоматизации, покрытие, процессы, вызовы |
| `03_behavioral_questions_qa.md` | STAR для QA: пропущенный баг, конфликт с разработчиком, сложный flaky-тест, предложение улучшения |
| `04_questions_to_interviewer.md` | Что спрашивать: процессы QA, автоматизация, соотношение ручного/авто, техдолг, отношение к качеству |

### 14_design_patterns/

Паттерны проектирования — с точки зрения автоматизатора.

| Файл | Содержание |
|------|-----------|
| `01_patterns_overview.md` | Зачем тестировщику знать паттерны, три категории GoF |
| `02_creational.md` | Singleton (WebDriver), Factory (создание драйверов), Builder (тестовые данные) |
| `03_structural.md` | Adapter, Decorator (логирование шагов), Proxy, Facade (упрощение API фреймворка) |
| `04_behavioral.md` | Strategy (разные браузеры/окружения), Template Method (базовый тест), Chain of Responsibility |
| `05_solid_for_qa.md` | Каждый принцип SOLID с примером из автоматизации тестирования |
| `06_clean_code_refactoring.md` | Code smells в тестах (God test, Copy-paste tests), DRY/KISS, рефакторинг тестового кода |

### 15_spring_for_qa/

Spring Framework — что QA нужно понимать.

| Файл | Содержание |
|------|-----------|
| `01_spring_overview.md` | IoC/DI — зачем, как Spring управляет объектами |
| `02_spring_boot_basics.md` | Стартеры, автоконфигурация, профили, properties — что QA видит в проекте |
| `03_spring_architecture.md` | Controller → Service → Repository — что тестировать на каждом уровне |
| `04_spring_annotations.md` | @Component, @Service, @RestController, @Autowired, @Transactional — понимание кода разработчика |
| `05_spring_testing.md` | @SpringBootTest, @WebMvcTest, @DataJpaTest, MockMvc — инструменты тестирования Spring |

### 16_git/ и 17_build_tools/

| Файл | Содержание |
|------|-----------|
| `01_git_workflow.md` | Git для QA: ветки для автотестов, Git Flow, merge vs rebase, resolve conflicts, .gitignore |
| `01_maven_gradle.md` | Maven pom.xml, Gradle build.gradle, surefire/failsafe plugins, запуск тестов, зависимости |

### 18_computer_science/

| Файл | Содержание |
|------|-----------|
| `01_data_structures.md` | Массивы, списки, хеш-таблицы, деревья — обзорно, Big O |
| `02_algorithms.md` | Сортировки, поиск — базово |
| `03_networking.md` | OSI, TCP/IP, HTTP/HTTPS, DNS — для понимания архитектуры |

### 19_common_problems/

Типичные проблемы в тестировании — практический опыт.

| Файл | Содержание |
|------|-----------|
| `01_flaky_tests.md` | Причины (timing, shared state, env, данные), диагностика, retry-стратегии, quarantine |
| `02_environment_issues.md` | Нестабильные стенды, рассинхрон данных, VPN, доступы, нехватка ресурсов |
| `03_test_maintenance.md` | Устаревшие локаторы, изменения API, рост сьюта, технический долг, когда удалять тесты |
| `04_debugging_failures.md` | Анализ падений: логи, скриншоты, видео, Allure trace, сравнение с историей |
| `05_common_automation_mistakes.md` | Хардкод, Thread.sleep, зависимости между тестами, нет изоляции данных, тесты через UI вместо API |

### 20_performance_security/

| Файл | Содержание |
|------|-----------|
| `01_performance_testing.md` | Типы, метрики (p50/p90/p99, throughput), bottleneck analysis |
| `02_jmeter_basics.md` | Test plan, thread group, samplers, assertions, listeners — базовый сценарий |
| `03_security_testing_overview.md` | OWASP Top 10: SQL Injection, XSS, CSRF, IDOR — как тестировать обзорно |

### 21_ai_in_testing/

| Файл | Содержание |
|------|-----------|
| `01_ai_tools_for_qa.md` | LLM для генерации тестов, prompt engineering для тест-кейсов |
| `02_practical_ai_usage.md` | Bulk test generation, тестовые данные с AI, code review тестов, ограничения |

---

## Статистика

| Директория | Файлов | Статус |
|-----------|--------|--------|
| 01_testing_theory | 9 | ✅ Готово |
| 02_java_for_qa | 12 | ✅ Готово |
| 03_test_frameworks | 5 | ✅ Готово |
| 04_ui_automation | 7 | ✅ Готово |
| 05_api_automation | 6 | ✅ Готово |
| 06_framework_architecture | 7 | ✅ Готово |
| 07_reporting | 5 | ✅ Готово |
| 08_ci_cd_for_qa | 5 | ✅ Готово |
| 09_databases_for_qa | 5 | ✅ Готово |
| 10_architecture_for_qa | 8 | ✅ Готово |
| 11_testing_microservices | 4 | ✅ Готово |
| 12_processes_for_qa | 10 | ✅ Готово |
| 13_self_presentation | 4 | ✅ Готово |
| 14_design_patterns | 6 | ✅ Готово |
| 15_spring_for_qa | 5 | ✅ Готово |
| 16_git | 1 | ✅ Готово |
| 17_build_tools | 1 | ✅ Готово |
| 18_computer_science | 3 | ✅ Готово |
| 19_common_problems | 5 | ✅ Готово |
| 20_performance_security | 3 | ✅ Готово |
| 21_ai_in_testing | 2 | ✅ Готово |
| 22_practice_guide | 30 | ✅ Готово |
| 23_building_qa_from_scratch | 13 | ✅ Готово |
| **Итого** | **155 файлов** | ✅ |
