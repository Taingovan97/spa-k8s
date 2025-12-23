Dưới đây là đề xuất **chuẩn production** theo hướng bạn đang đi: **multi-repo microservices + 1 repo GitOps/Platform cho K8s**.

---

# 1) Cấu trúc chi tiết repo `spa-platform-k8s` (GitOps)

Mục tiêu: repo này là **single source of truth cho deploy** (dev/staging/prod), còn mỗi service repo chỉ build image.

```text
spa-platform-k8s/
├── README.md
├── docs/
│   ├── environments.md
│   ├── release-process.md
│   └── secrets.md
├── charts/
│   └── spa-platform/                     # Umbrella chart (deploy cả platform)
│       ├── Chart.yaml
│       ├── values.yaml                   # default values (safe)
│       ├── values.schema.json            # validate values (optional)
│       └── templates/                    # optional shared templates
│
├── apps/                                 # app-level Helm charts (mỗi service 1 chart)
│   ├── spa-api-gateway/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── configmap.yaml
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── ingress.yaml
│   │       ├── hpa.yaml                  # phase 3 (optional)
│   │       └── networkpolicy.yaml        # optional
│   └── spa-user-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _helpers.tpl
│           ├── configmap.yaml
│           ├── secret-db.yaml            # or ExternalSecret
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── hpa.yaml                  # optional
│           └── networkpolicy.yaml        # optional
│
├── envs/                                 # per-environment overlays
│   ├── dev/
│   │   ├── namespace.yaml
│   │   ├── values.spa-api-gateway.yaml
│   │   ├── values.spa-user-service.yaml
│   │   ├── ingress.yaml                  # nếu muốn tách ngoài Helm
│   │   └── kustomization.yaml            # optional (nếu dùng kustomize)
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
│
└── ops/
    ├── bootstrap/
    │   ├── install-ingress-nginx.md
    │   ├── install-cert-manager.md
    │   ├── install-external-secrets.md
    │   └── install-argocd.md             # nếu dùng Argo CD
    └── scripts/
        ├── bump-image-tag.sh
        └── deploy-dev.sh
```

### Tư duy dùng repo này

* Service repos: build & push image `your-registry/spa-user-service:<git-sha>`
* Platform repo: update values để trỏ đến image tag mới (GitOps)
* ArgoCD (hoặc pipeline) sync → K8s rollout

---

# 2) Nội dung `application-k8s.yml` chuẩn cho 2 service

Nguyên tắc:

* **Không hardcode** secret / URL env cụ thể
* Dùng env vars `${...}`
* Bật actuator probes để readiness/liveness

## 2.1 `spa-api-gateway` — `application-k8s.yml`

```yaml
server:
  port: 8080

spring:
  application:
    name: spa-api-gateway

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI}

  cloud:
    gateway:
      # tăng ổn định cho prod khi có nhiều hop
      httpclient:
        connect-timeout: 3000
        response-timeout: 10s

      default-filters:
        # bạn có thể add correlation-id sau
        - PreserveHostHeader

      routes:
        # Phase 1: giữ Path đơn giản, Phase 2 sẽ chuyển sang /api/v1/...
        - id: user-service
          uri: http://spa-user-service:8082
          predicates:
            - Path=/me/**

management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

> Nếu bạn muốn “bớt hardcode routes”, Phase 2 sẽ chuyển routes sang configmap/values hoặc dùng discovery locator.

---

## 2.2 `spa-user-service` — `application-k8s.yml`

```yaml
server:
  port: 8082

spring:
  application:
    name: spa-user-service

  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: none
    open-in-view: false
    properties:
      hibernate:
        format_sql: false

  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml

management:
  endpoint:
    health:
      probes:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus

logging:
  level:
    root: INFO
```

---

# 3) Dockerfile cho `spa-api-gateway` và `spa-user-service`

Mình đưa Dockerfile theo kiểu **2-stage build** (Maven build → runtime JRE), tối ưu image size và chuẩn K8s.

## 3.1 Dockerfile — `spa-api-gateway`

`spa-api-gateway/Dockerfile`

```dockerfile
# ===== Build stage =====
FROM maven:3.9.9-eclipse-temurin-21 AS build
WORKDIR /workspace

COPY pom.xml .
COPY src ./src

RUN mvn -q -DskipTests clean package

# ===== Runtime stage =====
FROM eclipse-temurin:21-jre
WORKDIR /app

# Copy jar
COPY --from=build /workspace/target/*.jar app.jar

# Non-root (optional but recommended)
RUN useradd -r -u 10001 appuser && chown -R appuser:appuser /app
USER 10001

EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

## 3.2 Dockerfile — `spa-user-service`

`spa-user-service/Dockerfile`

```dockerfile
# ===== Build stage =====
FROM maven:3.9.9-eclipse-temurin-21 AS build
WORKDIR /workspace

COPY pom.xml .
COPY src ./src

RUN mvn -q -DskipTests clean package

# ===== Runtime stage =====
FROM eclipse-temurin:21-jre
WORKDIR /app

COPY --from=build /workspace/target/*.jar app.jar

RUN useradd -r -u 10001 appuser && chown -R appuser:appuser /app
USER 10001

EXPOSE 8082
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

> Tip: nếu bạn dùng Spring Boot buildpacks (`mvn spring-boot:build-image`) thì không cần Dockerfile, nhưng Dockerfile giúp bạn kiểm soát tốt hơn.

---

# 4) Pipeline hoạt động như thế nào? (pattern để nhân rộng cho mọi service)

## Mục tiêu pipeline

* Mỗi service repo chỉ làm:

  1. test
  2. build jar
  3. build image
  4. push image
  5. **update platform repo values** (GitOps)

Platform repo sẽ:

* ArgoCD sync (khuyến nghị) hoặc pipeline “helm upgrade”

---

## 4.1 Pipeline cho mỗi service repo (GitHub Actions mẫu)

### a) Build & push image (tag = git sha)

`.github/workflows/build-push.yml`

```yaml
name: Build & Push

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"

      - name: Build jar
        run: mvn -DskipTests clean package

      - name: Login registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ secrets.REGISTRY_HOST }} -u "${{ secrets.REGISTRY_USERNAME }}" --password-stdin

      - name: Build image
        run: |
          IMAGE=${{ secrets.REGISTRY_HOST }}/spa-api-gateway:${{ github.sha }}
          docker build -t $IMAGE .
          docker push $IMAGE
```

### b) Update platform repo values (GitOps commit)

Thêm step cuối:

```yaml
      - name: Update platform repo (bump image tag)
        uses: actions/checkout@v4
        with:
          repository: your-org/spa-platform-k8s
          token: ${{ secrets.PLATFORM_REPO_TOKEN }}
          path: platform

      - name: Bump image tag in platform repo
        run: |
          cd platform
          yq -i '.image.tag = "${{ github.sha }}"' envs/dev/values.spa-api-gateway.yaml
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git add envs/dev/values.spa-api-gateway.yaml
          git commit -m "spa-api-gateway: bump image to ${{ github.sha }}"
          git push
```

> `yq` cần install; hoặc dùng `sed`/`python` thay thế.

---

## 4.2 Deploy lên K8s (2 cách)

### Cách A (khuyến nghị production): ArgoCD (GitOps)

* ArgoCD watch repo `spa-platform-k8s`
* Mỗi commit bump tag → ArgoCD sync → rollout

Ưu điểm:

* audit trail rõ
* rollback dễ
* tách quyền deploy khỏi service repo

### Cách B (đơn giản ban đầu): pipeline chạy `helm upgrade`

Trong platform repo:

```bash
helm upgrade --install spa-api-gateway apps/spa-api-gateway -n spa -f envs/dev/values.spa-api-gateway.yaml
helm upgrade --install spa-user-service apps/spa-user-service -n spa -f envs/dev/values.spa-user-service.yaml
```

---

# 5) Cách deploy chuẩn để làm tương tự cho các service còn lại

## Pattern cho mỗi service mới

1. Tạo repo service mới từ template:

   * `application.yml` + `application-k8s.yml`
   * Dockerfile
   * Actuator probes
2. Trong `spa-platform-k8s/apps/<service>`:

   * Helm chart templates giống user-service
3. Trong `spa-platform-k8s/envs/dev`:

   * `values.<service>.yaml` chứa:

     * image repo/tag
     * env vars / secrets refs
     * replicas/resources
4. CI service repo:

   * build/push image
   * bump tag platform repo
5. CD:

   * ArgoCD sync hoặc helm upgrade

---

# 6) Ví dụ values theo env (platform repo)

`spa-platform-k8s/envs/dev/values.spa-api-gateway.yaml`

```yaml
replicaCount: 2
image:
  repository: your-registry/spa-api-gateway
  tag: "REPLACE_BY_CI"
service:
  port: 8080
keycloak:
  issuerUri: "http://keycloak.dev.example.com/realms/spa-booking"
ingress:
  enabled: true
  className: nginx
  host: api.dev.spa.local
```

`spa-platform-k8s/envs/dev/values.spa-user-service.yaml`

```yaml
replicaCount: 2
image:
  repository: your-registry/spa-user-service
  tag: "REPLACE_BY_CI"
service:
  port: 8082
db:
  url: "jdbc:postgresql://postgresql.spa.svc.cluster.local:5432/user_db"
  username: "user"
  password: "user"
```

---

## Chốt lại (để bạn ra quyết định nhanh)

* **Service repo**: code + `application-k8s.yml` generic + Dockerfile + CI build/push
* **Platform repo**: Helm/manifests + values theo env + GitOps deploy
* **Deploy**: ArgoCD watch platform repo là sạch nhất

Nếu bạn muốn, mình có thể:

* tạo luôn **full Helm chart template** chuẩn (bao gồm NetworkPolicy chỉ cho gateway gọi user-service)
* viết **ArgoCD Application YAML** để bạn “apply 1 lần” là GitOps chạy luôn
