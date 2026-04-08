# Kubernetes обзорно для QA

## Обзор

Kubernetes (K8s) — это платформа оркестрации контейнеров, которая автоматизирует развёртывание,
масштабирование и управление контейнеризированными приложениями. Если Docker — это способ
упаковать приложение в контейнер, то Kubernetes — это способ управлять сотнями таких контейнеров
в production. Для QA-инженера знание Kubernetes важно, потому что всё больше проектов деплоятся
именно в K8s, и QA должен уметь работать с тестовыми окружениями в кластере: просматривать логи,
подключаться к сервисам, понимать, почему тестовая среда упала, и как она устроена.

---

## Зачем нужен Kubernetes

### Проблемы без Kubernetes

При работе с десятками микросервисов в Docker возникают вопросы:
- Как автоматически перезапустить упавший контейнер?
- Как масштабировать сервис при росте нагрузки?
- Как обновлять приложение без downtime?
- Как балансировать трафик между инстансами?
- Как управлять конфигурацией и секретами?

### Что решает Kubernetes

1. **Self-healing** — автоматический перезапуск упавших контейнеров
2. **Auto-scaling** — масштабирование на основе метрик (CPU, memory)
3. **Rolling updates** — обновление без простоя
4. **Load balancing** — распределение трафика между инстансами
5. **Service discovery** — сервисы находят друг друга по имени
6. **Configuration management** — ConfigMaps и Secrets
7. **Storage orchestration** — управление хранилищем

---

## Архитектура Kubernetes

```
┌──────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│                                                              │
│  ┌─────────────────────────┐                                 │
│  │     Control Plane       │                                 │
│  │                         │                                 │
│  │  ┌───────────────────┐  │                                 │
│  │  │    API Server      │  │  ← Все команды идут сюда       │
│  │  ├───────────────────┤  │                                 │
│  │  │    Scheduler       │  │  ← Размещает Pod'ы на Node'ы   │
│  │  ├───────────────────┤  │                                 │
│  │  │ Controller Manager │  │  ← Следит за состоянием        │
│  │  ├───────────────────┤  │                                 │
│  │  │       etcd         │  │  ← Хранит состояние кластера   │
│  │  └───────────────────┘  │                                 │
│  └─────────────────────────┘                                 │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │          │
│  │             │  │             │  │             │          │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │          │
│  │ │  Pod A  │ │  │ │  Pod C  │ │  │ │  Pod E  │ │          │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │          │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │          │
│  │ │  Pod B  │ │  │ │  Pod D  │ │  │ │  Pod F  │ │          │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │          │
│  │             │  │             │  │             │          │
│  │  kubelet    │  │  kubelet    │  │  kubelet    │          │
│  │  kube-proxy │  │  kube-proxy │  │  kube-proxy │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

### Control Plane (управляющий уровень)

- **API Server** — центральная точка управления, принимает все команды (kubectl → API Server)
- **Scheduler** — решает, на какую Node разместить новый Pod
- **Controller Manager** — следит, чтобы текущее состояние совпадало с желаемым
- **etcd** — распределённое key-value хранилище состояния кластера

### Worker Node (рабочий узел)

- **kubelet** — агент на каждой Node, управляет Pod'ами
- **kube-proxy** — сетевой прокси, реализует Service networking
- **Container Runtime** — Docker или containerd, запускает контейнеры

---

## Core Concepts

### Pod

Pod — минимальная единица деплоя в Kubernetes. Содержит один или несколько контейнеров,
которые разделяют сеть и storage.

```yaml
# Пример: Pod с одним контейнером
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-app
      image: my-app:1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
```

**Ключевые моменты:**
- Pod — это эфемерная сущность (может быть пересоздан в любой момент)
- У Pod'а свой IP-адрес (не используйте его напрямую — он меняется)
- Pod обычно не создают вручную — используют Deployment

### Deployment

Deployment — управляет набором одинаковых Pod'ов (replicas). Обеспечивает rolling updates
и rollback.

```yaml
# Пример: Deployment с 3 репликами
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: testing
spec:
  replicas: 3                    # Количество инстансов приложения
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate          # Обновление без downtime
    rollingUpdate:
      maxSurge: 1                # Макс. дополнительных Pod'ов при обновлении
      maxUnavailable: 0          # Все текущие Pod'ы остаются доступными
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
          ports:
            - containerPort: 8080
          readinessProbe:        # Проба готовности (когда Pod готов принимать трафик)
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:         # Проба жизни (если не отвечает — перезапустить)
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

### Service

Service — стабильная точка доступа к набору Pod'ов. Pod'ы приходят и уходят, а Service
остаётся с одним и тем же DNS-именем и IP-адресом.

```yaml
# Пример: Service типа ClusterIP (доступен внутри кластера)
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: testing
spec:
  selector:
    app: my-app                  # Выбирает Pod'ы с этим label
  ports:
    - port: 80                   # Порт Service
      targetPort: 8080           # Порт контейнера
  type: ClusterIP                # Доступен только внутри кластера
```

**Типы Service:**
- **ClusterIP** (по умолчанию) — доступен только внутри кластера
- **NodePort** — доступен на порту каждой Node (30000-32767)
- **LoadBalancer** — создаёт внешний load balancer (в облаке)

### ConfigMap

ConfigMap — хранит конфигурационные данные в виде key-value. Позволяет менять конфигурацию
без пересборки образа.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: testing
data:
  application.properties: |
    spring.datasource.url=jdbc:postgresql://postgres-service:5432/testdb
    spring.jpa.hibernate.ddl-auto=update
    logging.level.root=INFO
  MAX_RETRY_ATTEMPTS: "3"
  FEATURE_FLAG_NEW_UI: "true"
```

### Secret

Secret — как ConfigMap, но для чувствительных данных (пароли, токены, сертификаты).
Данные хранятся в Base64 (не encryption!).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: testing
type: Opaque
data:
  # Значения в Base64: echo -n 'value' | base64
  username: cG9zdGdyZXM=          # postgres
  password: c2VjcmV0MTIz          # secret123
```

### Namespace

Namespace — виртуальный кластер внутри физического. Используется для изоляции окружений.

```
Kubernetes Cluster
├── namespace: production      ← Боевая среда
├── namespace: staging         ← Предпродуктивная среда
├── namespace: testing         ← Тестовое окружение QA
├── namespace: dev-team-1      ← Окружение разработчика 1
└── namespace: dev-team-2      ← Окружение разработчика 2
```

---

## Ingress

Ingress — управляет внешним HTTP/HTTPS-доступом к сервисам в кластере.
Работает как reverse proxy с правилами маршрутизации.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: testing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: testing.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
  tls:
    - hosts:
        - testing.example.com
      secretName: tls-secret
```

---

## kubectl: основные команды

```bash
# === Информация о кластере ===
kubectl cluster-info
kubectl get nodes

# === Работа с namespace ===
kubectl get namespaces
kubectl create namespace testing

# === Работа с Pod'ами ===
kubectl get pods -n testing                    # Список Pod'ов
kubectl get pods -n testing -o wide            # Расширенная информация (IP, Node)
kubectl describe pod my-app-xyz -n testing     # Детальная информация о Pod'е
kubectl logs my-app-xyz -n testing             # Логи Pod'а
kubectl logs my-app-xyz -n testing -f          # Follow логов
kubectl logs my-app-xyz -n testing --previous  # Логи предыдущего (упавшего) контейнера
kubectl exec -it my-app-xyz -n testing -- bash # Войти в контейнер

# === Работа с Deployment ===
kubectl get deployments -n testing
kubectl describe deployment my-app -n testing
kubectl scale deployment my-app --replicas=5 -n testing    # Масштабирование
kubectl rollout status deployment my-app -n testing        # Статус обновления
kubectl rollout history deployment my-app -n testing       # История обновлений
kubectl rollout undo deployment my-app -n testing          # Откат на предыдущую версию

# === Работа с Service ===
kubectl get services -n testing
kubectl describe service my-app-service -n testing

# === Port-forward (доступ к сервису из локальной машины) ===
kubectl port-forward service/my-app-service 8080:80 -n testing
# Теперь http://localhost:8080 → my-app-service внутри кластера

kubectl port-forward pod/my-app-xyz 8080:8080 -n testing
# Прямой доступ к конкретному Pod'у

# === ConfigMap и Secrets ===
kubectl get configmaps -n testing
kubectl get secrets -n testing
kubectl get secret db-credentials -n testing -o yaml

# === Отладка ===
kubectl get events -n testing --sort-by=.metadata.creationTimestamp
kubectl top pods -n testing                    # Потребление ресурсов
kubectl top nodes                              # Ресурсы Node'ов

# === Применение конфигурации из файла ===
kubectl apply -f deployment.yaml -n testing
kubectl delete -f deployment.yaml -n testing
```

---

## Как QA тестирует в Kubernetes

### Доступ к тестовой среде

**Вариант 1: Port-forward** (для разработки и отладки)
```bash
# Подключаемся к сервису в тестовом namespace
kubectl port-forward svc/my-app-service 8080:80 -n testing
# Запускаем тесты, направленные на http://localhost:8080
mvn test -Dbase.url=http://localhost:8080
```

**Вариант 2: Ingress** (для стабильной тестовой среды)
```
https://testing.example.com → Ingress → Service → Pod'ы
```
Тесты идут на стабильный URL, настроенный через Ingress.

**Вариант 3: NodePort** (для простых случаев)
```bash
# Сервис доступен на порту Node
curl http://<node-ip>:30080/api/health
```

### Анализ проблем в тестовой среде

Когда тестовая среда ведёт себя неожиданно, QA проверяет:

```bash
# 1. Все ли Pod'ы запущены?
kubectl get pods -n testing
# Ожидаем: все Pod'ы в статусе Running

# 2. Если Pod не Running — смотрим описание
kubectl describe pod <pod-name> -n testing
# Ищем Events: ошибки pull image, нехватка ресурсов, crashloop

# 3. Смотрим логи приложения
kubectl logs <pod-name> -n testing

# 4. Если Pod перезапускается — логи предыдущего контейнера
kubectl logs <pod-name> -n testing --previous

# 5. Проверяем конфигурацию (ConfigMap, Secrets)
kubectl get configmap app-config -n testing -o yaml

# 6. Проверяем события в namespace
kubectl get events -n testing --sort-by=.metadata.creationTimestamp
```

### Типичные статусы Pod'ов и их значение

| Статус | Описание | Что делать QA |
|--------|---------|---------------|
| **Running** | Работает нормально | Всё ок |
| **Pending** | Ожидает ресурсов | Проверить nodes, ресурсы |
| **CrashLoopBackOff** | Падает при старте, K8s перезапускает | Смотреть `logs --previous` |
| **ImagePullBackOff** | Не может скачать Docker image | Проверить имя/тег образа, registry |
| **OOMKilled** | Контейнер превысил лимит памяти | Нужно увеличить limits |
| **Terminating** | Удаляется | Подождать или проверить finalizers |

---

## Мониторинг (обзорно)

### Prometheus

Prometheus — система мониторинга и сбора метрик. Собирает метрики с приложений по HTTP
(pull model).

**Основные метрики для QA:**
- `http_requests_total` — общее количество запросов
- `http_request_duration_seconds` — время ответа
- `jvm_memory_used_bytes` — потребление памяти JVM
- `container_cpu_usage_seconds_total` — использование CPU

### Grafana

Grafana — платформа визуализации метрик. Подключается к Prometheus и отображает
дашборды с графиками.

**Полезные дашборды для QA:**
- Application dashboard — RPS, latency, error rate
- JVM dashboard — heap memory, GC, threads
- Kubernetes dashboard — состояние Pod'ов, ресурсы Node'ов

**Зачем QA мониторинг:**
- Отслеживать состояние тестовой среды
- Замечать деградацию производительности после деплоя
- Находить утечки памяти и других ресурсов
- Подтверждать нефункциональные требования (SLA: latency < 200ms)

---

## Связь с тестированием

| Аспект K8s | Значение для QA |
|------------|----------------|
| Namespace | Изолированные тестовые среды |
| Deployment | Rolling update — тестировать поведение при обновлении |
| Service | Стабильный доступ к приложению в тестах |
| ConfigMap | Управление тестовой конфигурацией |
| Probes | Readiness/Liveness — влияют на доступность при тестировании |
| Port-forward | Локальный доступ к сервисам для отладки |
| kubectl logs | Анализ ошибок в тестовой среде |
| Monitoring | Мониторинг здоровья тестовой среды |

---

## Типичные ошибки

1. **Не проверяют readiness probe** — тесты начинаются до того, как приложение готово
2. **Используют IP Pod'а** — IP меняется при перезапуске, нужно использовать Service
3. **Не смотрят логи** — при падении теста не анализируют `kubectl logs`
4. **Не учитывают ресурсные лимиты** — тесты fail из-за OOMKilled
5. **Путают namespace** — тестируют не в том окружении
6. **Игнорируют events** — `kubectl get events` показывает причину проблем
7. **Не знают про `--previous`** — при CrashLoopBackOff не видят логи упавшего контейнера

---

## Вопросы на интервью

### 🟢 Базовый уровень
1. Что такое Kubernetes и зачем он нужен?
2. Что такое Pod?
3. В чём разница между Pod и Container?
4. Что такое Namespace?
5. Как посмотреть логи приложения в Kubernetes?

### 🟡 Средний уровень
6. Что такое Deployment и чем он отличается от Pod?
7. Для чего нужен Service? Какие типы Service вы знаете?
8. Что такое readiness probe и liveness probe?
9. Как получить доступ к сервису в K8s из локальной машины?
10. Что такое ConfigMap и Secret? В чём разница?
11. Как диагностировать CrashLoopBackOff?

### 🔴 Продвинутый уровень
12. Как организовать тестовые среды в Kubernetes (по namespace, по кластеру)?
13. Как тестировать rolling update — убедиться, что нет downtime?
14. Как настроить мониторинг тестовой среды с Prometheus и Grafana?
15. Как Ingress влияет на тестирование? Какие проблемы могут возникнуть?

---

## Практические задания

### Задание 1: Локальный кластер
Установите Minikube или Kind. Разверните простое приложение:
1. Создайте Deployment с 2 репликами
2. Создайте Service типа ClusterIP
3. Используйте `kubectl port-forward` для доступа
4. Проверьте, что приложение работает

### Задание 2: Диагностика
Намеренно создайте проблему (неправильный image name, нехватка ресурсов) и потренируйтесь
в диагностике:
1. `kubectl get pods` — увидеть проблему
2. `kubectl describe pod` — найти причину
3. `kubectl get events` — посмотреть события
4. Исправьте проблему и убедитесь, что Pod запустился

### Задание 3: Namespace для тестовых сред
1. Создайте 2 namespace: `testing` и `staging`
2. В каждом разверните одно и то же приложение с разными ConfigMap
3. Проверьте, что приложения изолированы и имеют разную конфигурацию

### Задание 4: Мониторинг
Установите Prometheus и Grafana в Minikube (через Helm chart).
Создайте дашборд, который показывает:
- Количество запущенных Pod'ов
- Потребление CPU и памяти
- Статус Pod'ов

---

## Дополнительные ресурсы

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Kubernetes The Hard Way (Kelsey Hightower)](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Minikube — локальный Kubernetes](https://minikube.sigs.k8s.io/)
- [Kind — Kubernetes в Docker](https://kind.sigs.k8s.io/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
