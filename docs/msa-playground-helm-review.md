<aside>
💡

현재 캐시워크에서 일하면서 과거 연습한 내용을 거의 다 잊었다.

295일 전의 나를 리뷰한다.

</aside>

## 준비:

OrbStack 컨텍스트로 이동

```bash
kubectl config use-context orbstack
kubectl get pods -n default
```

!image.png

# msa-playground-helm 코드 리뷰 (295일 후)

> **작성 시점**: Phase 4(Helm) 학습 직후
**대상**: 취업 전 만든 `msa-playground-helm` 레포
**목적**: 파트장님이 조언한 top-down DFS — "그냥 됐던 것"을 "왜 되는지"로 바꾸기
> 

## 요약

취업 전 이론 없이 “단순 구축 경험용”을 목푤 만든 차트를, Helm을 공부하면서 다시 읽었다.

의외로 다양한 세팅과 구조적 결함을 찾을 수 있었고, 공통 차트 패턴을 사용하고 있었다.

### 레포지토리 구조

```bash
msa-playground-helm/
├── api-gateway-chart/
├── user-service-chart/
├── order-service-chart/
├── kafka-chart/
├── k8s-base-infra/      (service-rbac.yml 하나)
├── loki-stack/
└── README.md
```

서비스마다 차트를 1:1로 대응해서 생성하였다.

### 발견 1: 차트 3개가 사실상 동일

```bash
diff -r user-service-chart order-service-chart
```

```bash
Only in user-service-chart/templates: user-service-deployment.yml
Only in order-service-chart/templates: order-service-deployment.yml
...
```

diff가 "다르다"고 표시한 건 **파일명이 달라서 비교 자체가 안 된 것**이지 내용이 다른 게 아니었다.

**values.yaml의 실제 차이:**

| 항목 | user-service | order-service |
| --- | --- | --- |
| image.repository | msa-playground-user-service | msa-playground-order-service |
| image.tag | 8ea93d5 | 18513ca |
| service.name | user-service | order-service |
| service.port | 8081 | 8082 |
| configMap.name | user-service-config | order-service-config |
| DB_URL | `jdbc:h2:mem:userdb` | `jdbc:h2:mem:orderdb` |
| secret.name | user-service-secret | order-service-secret |

딱 이 값들이 다르다.

이미지, 이름, 포트, DB URL이 다르다.
MSA 구조로 다 다르기 때문이다.

이것들은 values 파일로 흡수될 차이들이다.
구조 (Deployment + Service + ConfigMap + Secret)은 동일하다.

### 발견 2: 이미 공통 차트를 사용하고 있었음

```bash
helm list -A
```

```bash
NAME            CHART
api-gateway     user-service-chart-0.1.0    ← !
order-service   user-service-chart-0.1.0    ← !
user-service    user-service-chart-0.1.0
monitoring      kube-prometheus-stack-79.2.1
```

세 릴리스가 전부 `user-service-chart` 하나로 배포되어 있었다.

레포지토리에 `order-service-chart`, `api-gateway-chart`는 레포에만 존재하고 실제로 사용하지 않았다.

대부분의 내용이 같고 해당 프로젝트는 전부 SpringBoot로 만들었기 떄문에 `spring-service` 같은 중립적 이름으로 하나의 차트를 사용하는게 맞다.

### 잘한 것

**HPA behavior 튜닝**

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0      # 즉시 확장
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300    # 5분 대기 후 축소
```

기본 값을 사용한 것이 아니라 “확장은 즉시, 축소는 신중히”를 명시적으로 설정했다.

**ServiceMonitor로 Prometheus 연동**

```yaml
kind: ServiceMonitor
metadata:
  labels:
    release: monitoring  # ✅ 중요: Prometheus가 찾을 수 있도록
spec:
  endpoints:
  - port: http
    path: /actuator/prometheus
```

kube-promethus-stack이 ServiceMonitor를 발견하는 메커니즘(release 라벨 매칭)을 이해하고 주석까지 남겼다.

**실측 기반 resources 조정**

```yaml
resources:
  requests:
    cpu: "400m"    # 200m -> 400m (기준을 높여서 민감도 낮춤)
  limits:
    cpu: "1000m"   # 500m -> 1000m (순간적인 버스트 허용)
    memory: "1Gi"  # (OOM 방지용 여유 공간)
```

HPA가 너무 민감하게 반응하는 문제를 겪고 requests를 올린 흔적

## 지금이라면 고칠 것

**readnessProb 부재**

Deployment에 probe가 하나도 없다.

문제: Spring Boot는 부팅에 20초 정도 오래 걸린다. 컨테이너가 뜨자마자 Endpoints에 등록되어 트래픽이 들어간다.

롤링 업데이트마다 에러가 날 수 있고 Phase 3에서 재현한 FAILED 상황이 실제로 발생했을 것이다.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8081
  initialDelaySeconds: 20
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8081
  initialDelaySeconds: 60
```

Spring Actuator에 이미 존재하는 엔드포인트라 연결마 ㄴ하면 된다.

**Secret 템플릿에 이름 하드코딩 (버그)**

```yaml
kind: Secret
metadata:
  name: user-service-secret    # ← {{ .Values.secret.name }} 이어야 함
```

다른 필드는 모두 `{{ }}` 인데 여기만 고정 값으로 하드코딩 했다.

이 차트로 order-service를 배포하면 secret 이름이 user-service-secret으로 생성되기 때문이다.

실제로 배포한 세 릴리스가 같은 차트를 쓰고 있어 릴리스끼리 Secret을 덮어쓰는 충돌이 발생했을 수 있다.

**resources가 템플릿에 하드코딩**

```yaml
resources:
  requests:
    cpu: "400m"
```

서비스별로 다른 값을 줄 수 없다.
환경마다 다른 세팅을 위해 values로 분리 필요

```yaml
resources:
  {{- toYaml .Values.resources | nindent 10 }}
```

**차트 통합 필요**

`spring-service` 차트 하나 + `apps/{serice}/values.yaml` 구조로 재편

Secret이 Git에 평문

```yaml
secret:
  data:
    DB_USER: "sa"
    DB_PASSWORD: ""
```

H2 인메모리라 실제 위험은 없지만 패턴 자체가 위험하다.
실무라면 External Secrets Operator + AWS Secrets Manager로 분리해야 한다.

**graceful shutdown 미설정**

`terminationGracePeriodSeconds`, `preStop` hook이 없다.
처리 중인 요청을 위한 여유 시간이 없고, Endpoints 전파 지연으로 인한 502 가능성.

**`_helpers.tql` 을 정의만 하고 사용하지 않음**

`helm create`가 생성한 헬퍼(`fullname`, `labels`, `selectorLabels`)가 있으나 템플릿에서 호출하지 않는다.
대신 `{{ .Values.service.name }}`을 직접 사용 → 쿠버네티스 표준 라벨(`app.kubernetes.io/*`)을 따르지 않게 됨.

## 개선 후 목표 구조

```yaml
ldh-infra-lab-manifests/
├── charts/
│   └── spring-service/            ← 하나로 통합
│       ├── Chart.yaml
│       ├── values.yaml            (안전한 기본값)
│       └── templates/
│           ├── deployment.yaml    (+ probe, graceful shutdown)
│           ├── service.yaml
│           ├── configmap.yaml
│           ├── hpa.yaml           ({{- if }} 조건부)
│           └── servicemonitor.yaml ({{- if }} 조건부)
└── apps/
    └── local/
        ├── user-service/values.yaml
        ├── order-service/values.yaml
        └── api-gateway/values.yaml
```

## 러닝 포인트

**그때는 왜 몰랐나**
이론 없이 "일단 뜨게 하는 것"이 목표였다. 튜토리얼을 따라 `helm create`로 차트를 만들고 서비스마다 복제했고,
동작은 했기 때문에 구조적 문제를 인지할 계기가 없었다.

**무엇이 달라졌나**

- 차트를 나눌 기준이 **"앱 이름"이 아니라 "쿠버 리소스 구조"** 라는 것을 알게 됨
- probe가 헬스체크가 아니라 **무중단 배포의 전제조건**임을 실습으로 확인
- requests가 **스케줄링과 HPA 계산의 기준**이라는 것을 이해

**한 문장**

> 동작하는 것과 올바른 것은 다르다. 295일 전의 차트는 동작했지만,
readinessProbe 하나가 없어서 배포할 때마다 조용히 트래픽을 흘리고 있었다.
>
