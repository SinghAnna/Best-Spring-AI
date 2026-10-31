# Part 29: Spring AI — Production Deployment (Docker + K8s + Config)

> Production deployment ka core — **multi-stage Docker (build → runtime, JRE-alpine, non-root, layered jar) + Kubernetes (Deployment + Service + ConfigMap + Secret + HPA + PDB) + health probes (liveness + readiness + startup, `/actuator/health/{liveness,readiness}`) + graceful shutdown (`server.shutdown=graceful`) + config externalization (ConfigMap → property source, Secret → env) + stateless pods (shared state: Redis / Postgres / vector store) + probes must not call model** — resource limits (CPU + memory), rolling update (maxSurge 1, maxUnavailable 0), pre-deploy checklist. [2513][2515][2517][2524]

---

## 1. Production Mental Model

```text
Problem:
Dev = Single instance + hardcoded config
Prod = Multi-instance + external config + resilience

Solution:
1. Docker (containerize)
2. Kubernetes (orchestrate)
3. Config (externalize)
4. Health (probes)
5. Scaling (HPA)
6. Resilience (PDB + rolling)
```

### Dev vs Prod

| Aspect | Dev | Prod |
|---|---|---|
| Instances | 1 | N (HPA) |
| Config | Hardcoded | External (ConfigMap / Secret) |
| State | In-memory | Shared (Redis / DB) |
| Health | None | Probes (liveness + readiness) |
| Deploy | Manual | Rolling (zero downtime) |

Dev vs prod — instances + config + state + health + deploy. [2513][2515][2524]

### Production Checklist

```text
1. Multi-stage Docker (small + secure)
2. Non-root user
3. Health probes (liveness + readiness + startup)
4. Graceful shutdown
5. Config externalized (ConfigMap + Secret)
6. Stateless pods (shared state)
7. Resource limits (CPU + memory)
8. HPA (auto-scale)
9. PDB (disruption budget)
10. Observability (metrics + traces)
```

Production checklist — 10 items (Docker + K8s + config + health + scale + resilience). [2515][2517][2524]

---

## 2. Docker (Multi-Stage Build)

```text
Multi-stage = Build (JDK) → Runtime (JRE)
= Small + secure image
```

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage — build (JDK) → runtime (JRE-alpine, non-root). [2515][2518][2519]

### Layered Jar (Faster Rebuild)

```text
Layered jar = Dependencies separate from app
= Dependency layer cached (rebuild fast)
```

Layered jar — dependencies separate (cache). [2519]

### Docker Best Practices

```text
1. Multi-stage (small image)
2. Non-root user (security)
3. JRE-alpine (small base)
4. Layered jar (fast rebuild)
5. HEALTHCHECK (container health)
6. No secrets in image
```

Docker best practices — multi-stage + non-root + JRE-alpine + layered + health + no secrets. [2515][2519]

---

## 3. Kubernetes (Deployment + Service)

```text
K8s = Orchestration (pods + services + config)
```

### K8s Resources

| Resource | Purpose |
|---|---|
| Deployment | Pod template + replicas + strategy |
| Service | Stable network endpoint |
| ConfigMap | Non-secret config |
| Secret | Sensitive config (API keys) |
| HPA | Auto-scale (CPU / memory) |
| PDB | Disruption budget (rolling) |

K8s resources — Deployment + Service + ConfigMap + Secret + HPA + PDB. [2513][2517]

### Deployment (Rolling Update)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-ai-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod during update
      maxUnavailable: 0  # 0 pods down
  template:
    spec:
      containers:
      - name: app
        image: myregistry/spring-ai-app:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-secrets
              key: openai-api-key
```

Deployment — replicas 3 + rolling (maxSurge 1, maxUnavailable 0) + Secret. [2515][2517]

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-ai-service
spec:
  selector:
    app: spring-ai-app
  ports:
  - port: 80
    targetPort: 8080
```

Service — stable endpoint (selector → pods). [2513][2517]

---

## 4. Health Probes (Liveness + Readiness + Startup)

```text
Probes = K8s health checks
= Liveness (restart?) + Readiness (traffic?) + Startup (slow start?)
```

### 3 Probes

| Probe | Question | Action if Fail |
|---|---|---|
| Liveness | App running? | Restart pod |
| Readiness | Can handle traffic? | Stop sending traffic |
| Startup | Slow start? | Delay other probes |

3 probes — liveness (restart) + readiness (traffic) + startup (delay). [2513][2515]

### Probe Endpoints

```text
/actuator/health/liveness  = Liveness
/actuator/health/readiness = Readiness
/actuator/health           = Overall (DB + disk)
```

Probe endpoints — liveness + readiness + overall. [2513][2515]

### Probe Config (K8s)

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Probe config — liveness (30s) + readiness (20s) + startup (10s). [2513][2515]

### Probe Rule (Critical)

```text
Probe = Own dependencies (DB + cache)
NOT model (LLM call)

Why:
Model blip = Restart storm (bad)
```

Probe rule — own deps (not model), restart storm prevent. [2524]

---

## 5. Config Externalization

```text
Config = External (ConfigMap + Secret)
= No hardcoded values
```

### ConfigMap (Non-Secret)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-ai-config
data:
  SERVER_PORT: "8080"
  SPRING_AI_OPENAI_MODEL: "gpt-4o-mini"
  SPRING_AI_RETRY_MAX_ATTEMPTS: "3"
  LOGGING_LEVEL_ROOT: "INFO"
```

ConfigMap — non-secret (port + model + retry + log). [2513][2515]

### Secret (Sensitive)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-secrets
type: Opaque
stringData:
  openai-api-key: sk-...
```

Secret — sensitive (API key). [2515][2516]

### Config Mount (Volume)

```yaml
volumes:
- name: config-volume
  configMap:
    name: spring-ai-config
containers:
- name: app
  volumeMounts:
  - name: config-volume
    mountPath: /config
    readOnly: true
```

Config mount — volume (readOnly). [2513]

### Secrets Manager (Prod)

```text
Prod = AWS Parameter Store / Secrets Manager
= Spring Cloud AWS (auto-inject)
```

Secrets manager — AWS Parameter Store / Secrets Manager (auto-inject). [2516]

---

## 6. Stateless Pods (Shared State)

```text
Stateless = No in-memory state
= Any pod serves any request
= Rolling update safe
```

### Stateless Rule

```text
Stateless = Conversation memory + cache + budget + vector
= Shared stores (Redis / Postgres / vector store)
= Any pod → Any request
```

Stateless — shared stores (Redis + Postgres + vector). [2524]

### Stateless Benefits

```text
1. Horizontal scale (HPA)
2. Rolling update (zero downtime)
3. Pod restart (no data loss)
4. Multi-region (easy)
```

Stateless benefits — scale + rolling + restart + multi-region. [2524]

---

## 7. Full Working Example

### Dockerfile

```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### application.yml

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      model: ${SPRING_AI_OPENAI_MODEL:gpt-4o-mini}

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-ai-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: spring-ai-app
  template:
    metadata:
      labels:
        app: spring-ai-app
    spec:
      containers:
      - name: app
        image: myregistry/spring-ai-app:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: ai-secrets
              key: openai-api-key
        - name: SPRING_AI_OPENAI_MODEL
          valueFrom:
            configMapKeyRef:
              name: spring-ai-config
              key: SPRING_AI_OPENAI_MODEL
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 5
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
```

### k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-ai-service
spec:
  selector:
    app: spring-ai-app
  ports:
  - port: 80
    targetPort: 8080
```

### k8s/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-ai-config
data:
  SPRING_AI_OPENAI_MODEL: "gpt-4o-mini"
  SPRING_AI_RETRY_MAX_ATTEMPTS: "3"
```

### k8s/secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ai-secrets
type: Opaque
stringData:
  openai-api-key: sk-...
```

---

## 8. Run + Deploy

```bash
# 1. Build image
docker build -t myregistry/spring-ai-app:1.0.0 .

# 2. Push
docker push myregistry/spring-ai-app:1.0.0

# 3. Deploy (K8s)
kubectl apply -f k8s/

# 4. Verify
kubectl get pods -l app=spring-ai-app
kubectl rollout status deployment/spring-ai-app

# 5. Test
kubectl port-forward svc/spring-ai-service 8080:80
curl http://localhost:8080/actuator/health
```

---

## 9. Common Gotchas

### Gotcha 1 — Secret in Image

```text
Problem:
API key in Dockerfile / image

Fix:
Secret (K8s) / env var
```

### Gotcha 2 — Probe Calls Model

```text
Problem:
Model blip = restart storm

Fix:
Probe own deps (DB + cache)
```

### Gotcha 3 — Stateful Pod

```text
Problem:
In-memory state (rolling fail)

Fix:
Shared stores (Redis / DB)
```

### Gotcha 4 — No Resource Limits

```text
Problem:
Pod eats cluster resources

Fix:
requests + limits (CPU + memory)
```

---

## 10. Interview Questions

| Question | Answer |
|---|---|
| Multi-stage Docker? | Build (JDK) → runtime (JRE). |
| K8s probes? | Liveness + readiness + startup. |
| ConfigMap vs Secret? | Non-secret vs sensitive. |
| Stateless pods? | Shared state (Redis / DB). |
| Rolling update? | maxSurge 1 + maxUnavailable 0. |
| Graceful shutdown? | `server.shutdown=graceful`. |

### Scenario-Based

| Scenario | Solution |
|---|---|
| Small image | Multi-stage + JRE-alpine |
| Zero downtime | Rolling update + readiness |
| API key | Secret (K8s) / Secrets Manager |
| Auto-scale | HPA (CPU / memory) |
| Restart storm | Probe own deps (not model) |

---

## 11. Key Takeaways

```text
1. Production = Docker + K8s + config + health + scale.

2. Multi-stage Docker = small + secure.

3. Non-root user = security.

4. Probes = liveness + readiness + startup.

5. Probe = own deps (not model).

6. ConfigMap = non-secret, Secret = sensitive.

7. Stateless = shared state (Redis / DB).

8. Rolling update = maxSurge 1 + maxUnavailable 0.

9. Resource limits = CPU + memory.

10. Graceful shutdown = `server.shutdown=graceful`.
```

---

## 12. Next Part Preview

**Part 30: Capstone Project (Full Production AI App)**

Isme hum cover karenge:

```text
- Capstone deep dive
- Full production AI app
- All concepts combined
- RAG + agents + streaming + security + observability
- Deployment + testing
```