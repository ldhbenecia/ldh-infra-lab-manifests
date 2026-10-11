# ldh-infra-lab-manifests

로컬 쿠버네티스(OrbStack) 실습 클러스터의 **GitOps 매니페스트 레포**.
ArgoCD가 이 레포의 `main`을 감시하며 클러스터를 이 상태로 맞춘다. **이 레포에 커밋하는 것이 곧 배포다.**

---

## 무엇이 돌고 있나

| 네임스페이스 | 프로젝트 | 구성 |
|---|---|---|
| `default` | msa 플레이그라운드 | user / order / product-service, api-gateway (Spring Cloud) |
| `morris` | 나인멘스모리스 게임 서버 | game-server + MySQL + Redis |
| `kube-system` | 플랫폼 | sealed-secrets 컨트롤러 |

두 프로젝트는 서로 별개다. 네임스페이스가 갈려 있고, api-gateway의 쿠버네티스 디스커버리 Role이 `default`에만 걸려 있어 게임 서버가 게이트웨이에 노출되지 않는다.

## 배포 흐름

```
앱 레포 main 머지
  → GitHub Actions: 테스트 → arm64 이미지 빌드 → GHCR 푸시
  → 같은 CI가 이 레포의 apps/<프로젝트>/<앱>/values.yaml 의 image.tag 를 커밋
  → ArgoCD (최대 3분 폴링)가 변경 감지
  → kubelet이 GHCR에서 이미지를 받아 롤링 업데이트
```

ArgoCD는 레지스트리를 보지 않는다. 새 이미지가 있다는 사실은 **태그 문자열이 바뀐 커밋**으로만 전달된다.

| 앱 레포 | 이미지 | 이 레포에서 바뀌는 파일 |
|---|---|---|
| `msa-playground-user-service` | `ghcr.io/ldhbenecia/user-service` | `apps/msa/user-service/values.yaml` |
| `msa-playground-order-service` | `ghcr.io/ldhbenecia/order-service` | `apps/msa/order-service/values.yaml` |
| `msa-playground-product-service` | `ghcr.io/ldhbenecia/product-service` | `apps/msa/product-service/values.yaml` |
| `msa-playground-api-gateway` | `ghcr.io/ldhbenecia/api-gateway` | `apps/msa/api-gateway/values.yaml` |
| `nine-mens-morris` | `ghcr.io/ldhbenecia/nine-mens-morris` | `apps/morris/game-server/values.yaml` |

CI는 fine-grained 토큰 `MANIFESTS_TOKEN`(이 레포 하나, Contents 쓰기만)으로 커밋한다.

---

## 디렉토리

```
argocd/
├── root-app.yaml          App of Apps 루트. 이것 하나만 수동 apply
└── apps/                  Application 정의. 파일 추가 = 앱 추가
charts/
└── http-service/          직접 만든 공통 차트 (HTTP 포트를 받는 장기 실행 서비스)
apps/
├── msa/<앱>/              앱별 배포 입력
├── msa/secrets/           msa 공유 SealedSecret
├── morris/<앱>/
└── morris/secrets/
platform/
├── ingress/               *.lab.local 인그레스 (default, argocd ns)
├── rbac/                  msa ServiceAccount + Role
└── sealed-secrets/        외부 차트에 넘기는 values
docs/                      학습 기록
```

**`apps/<프로젝트>/<앱>/`의 규칙**: 그 앱의 배포 입력이다. 차트를 쓰면 `values.yaml` 하나, 차트 없이 직접 쓰면 매니페스트 파일들.

| 앱 | 방식 | 이유 |
|---|---|---|
| msa 4개, game-server | `http-service` 차트 + `values.yaml` | 구조가 같은 HTTP 서비스 |
| morris mysql, redis | StatefulSet 직접 작성 | 파드와 디스크가 1:1. Deployment 전제인 차트와 구조가 다름 |
| `*/secrets` | SealedSecret 직접 | 차트가 아니라 봉인된 Secret |

---

## Helm 차트란 — 이 레포 기준으로

차트는 **"값을 넣으면 쿠버네티스 yaml이 나오는 템플릿 묶음"**이다. 같은 모양의 Deployment·Service를 서비스마다 복사해 쓰지 않으려고 쓴다.

```
charts/http-service/
├── Chart.yaml           차트의 이름과 버전. "이 디렉토리는 차트다"라는 표시
├── values.yaml          기본값. 앱이 아무것도 안 주면 이 값이 쓰임
├── profiles/
│   └── spring.yaml      (이 레포 관례) Spring 공통값. Helm 기능이 아니라 그냥 values 파일
└── templates/           Go 템플릿. {{ .Values.xxx }} 자리에 값이 들어가 yaml 이 됨
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml       values 의 config: 가 있을 때만
    ├── ingress.yaml         ingress.enabled 일 때만
    ├── hpa.yaml             autoscaling.enabled 일 때만
    └── servicemonitor.yaml  serviceMonitor.enabled 일 때만
```

**차트 디렉토리 안에는 앱별 정보가 없다.** 앱별 값은 전부 `apps/` 아래에 있고, Application이 둘을 합친다.

```yaml
# argocd/apps/user-service.yaml
source:
  path: charts/http-service              # 어떤 차트로
  helm:
    valueFiles:
      - profiles/spring.yaml             # 어떤 언어 기본값을 깔고
      - ../../apps/msa/user-service/values.yaml   # 어떤 앱 값으로 덮어써서
```

### values가 합쳐지는 순서

뒤에 오는 것이 앞을 덮어쓴다. 맵(`config:` 등)은 키 단위로 병합된다.

1. `charts/http-service/values.yaml` — 언어 중립 기본값 (`/health`, 128Mi)
2. `charts/http-service/profiles/spring.yaml` — Spring 공통 (actuator 경로, 512Mi, `SPRING_PROFILES_ACTIVE`)
3. `apps/<프로젝트>/<앱>/values.yaml` — 앱 개별 (이미지, 포트, 환경변수)

### 이 레포의 차트는 두 종류

| | 위치 | Application 필드 | ArgoCD가 가져오는 방법 |
|---|---|---|---|
| **직접 만든 차트** | 이 레포 `charts/http-service` | `path:` | Git clone |
| **남이 만든 차트** | `https://bitnami.github.io/sealed-secrets` | `chart:` + `targetRevision:`(차트 버전) | HTTP로 `index.yaml` → `.tgz` |

남의 차트에는 이 레포의 values 파일이 없으므로 multi-source로 넘긴다 (`argocd/apps/platform-sealed-secrets.yaml`의 `$values` / `ref: values`).

### 렌더 결과를 직접 보기

ArgoCD가 하는 일을 로컬에서 그대로 할 수 있다. **차트나 values를 고치면 커밋 전에 이걸로 diff를 본다.**

```bash
helm template user-service charts/http-service \
  -f charts/http-service/profiles/spring.yaml \
  -f apps/msa/user-service/values.yaml
```

### 차트 명명 원칙

차트는 언어가 아니라 **워크로드 종류**로 나눈다. Spring이든 Node든 HTTP 서비스면 `http-service`.

- `http-service` — HTTP 포트를 받는 장기 실행 서비스
- `worker` — 큐 컨슈머, Service 없음 (예정)
- `cronjob` — 배치 (예정)

---

## Secret

평문을 Git에 두지 않는다. **Sealed Secrets**로 봉인한 암호문만 커밋한다.

```bash
kubectl create secret generic <이름> -n <ns> \
  --from-literal=KEY=value \
  --dry-run=client -o yaml \
| kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
    --format=yaml > apps/<프로젝트>/secrets/<이름>-sealedsecret.yaml
```

- `--dry-run=client` 파이프라 평문이 디스크에 안 남는다
- 암호문은 **ns + 이름에 묶인다.** 다른 곳에 apply하면 복호화가 거부된다
- 비밀이 아닌 값(URL, 사용자명, 만료 시간)은 Secret이 아니라 values의 `config:`로

| Secret | ns | 쓰는 곳 |
|---|---|---|
| `msa-jwt-secret` | default | user-service(서명), api-gateway(검증) — **반드시 같은 키** |
| `morris-mysql-secret` | morris | MySQL |
| `morris-game-server-secret` | morris | game-server (DB 비밀번호, JWT, 카카오) |

**값은 이 레포, 선언은 앱 레포.** 앱의 `application-k8s.yml`은 `${TOKEN_SECRET}`처럼 필요한 환경변수를 기본값 없이 선언하고, 이 레포가 그 값을 공급한다.

**컨트롤러 사설키는 반드시 백업한다.** 잃으면 이 레포의 모든 SealedSecret이 복호화 불가능해진다.

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml
```

---

## 동기화 순서 (sync-wave)

root가 `argocd/apps/`의 Application들을 이 순서로 올린다. 낮은 수가 먼저.

| wave | Application | 이유 |
|---|---|---|
| -1 | platform-sealed-secrets | 컨트롤러가 없으면 SealedSecret이 안 풀림 |
| 0 | morris-secrets, (지정 없는 것 전부) | |
| 1 | morris-mysql, morris-redis | game-server readiness의 전제 |
| 2 | morris-game-server | |

---

## 새 앱을 추가할 때

1. 앱 레포 `application-k8s.yml`에 필요한 환경변수를 선언 (비밀은 기본값 없이)
2. 비밀값이 있으면 `apps/<프로젝트>/secrets/`에 SealedSecret
3. `apps/<프로젝트>/<앱>/values.yaml`
4. `argocd/apps/<앱>.yaml` — 차트 경로와 valueFiles
5. `helm template`으로 렌더 확인
6. 앱 레포 CI에 `MANIFESTS_TOKEN` 시크릿 + 태그 갱신 잡
7. GHCR 패키지가 Public인지 확인 (Private이면 kubelet이 못 받음)

**배포 확인은 Deployment가 아니라 파드로 한다.** `kubectl get deploy`의 이미지는 원하는 상태일 뿐이다.

---

## Git 밖에서 설치한 것 (부트스트랩)

클러스터를 처음부터 다시 세울 때 이 레포보다 먼저 손으로 설치해야 하는 것들.

| | 비고 |
|---|---|
| ArgoCD v3.5.3 | `kubectl apply --server-side --force-conflicts` 필수 (CRD 크기). 설치 URL은 버전 고정 |
| ingress-nginx | helm install. `ingress-nginx` ns에 `istio-injection` 라벨을 붙이지 않는다 (클라이언트 IP가 127.0.0.6으로 바뀜) |
| Istio (demo, mTLS PERMISSIVE) | `default` ns에 사이드카 주입 중 |
| metrics-server | `--kubelet-insecure-tls` |
| kube-prometheus-stack | helm install, 릴리스 이름 `monitoring` |
| `/etc/hosts` | `user.lab.local`, `order.lab.local`, `argocd.lab.local`, `morris.lab.local` → 127.0.0.1 |
| sealed-secrets 사설키 | 백업본을 먼저 apply한 뒤 컨트롤러 재시작 |

---

## 기록

| 문서 | 내용 |
|---|---|
| `docs/msa-playground-helm-review.md` | 예전 msa 차트 4개 리뷰 |
| `docs/2026-10-11-morris-migration.md` | 게임 서버 이관, Sealed Secrets, msa CI 전환, 그 과정에서 터진 것들 |

Helm 입문 실습(simple-app, web-service)은 태그 `phase4-helm-practice` 참고.
