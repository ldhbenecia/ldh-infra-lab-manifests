# 2026-10-11 — 기존 백엔드 서버를 쿠버네티스로 이관하기

> **한 문장**: 로컬에서만 돌던 나인멘스모리스 게임 서버를 로컬 쿠버네티스(OrbStack) + ArgoCD로 옮기고, main 머지 한 번으로 테스트·빌드·배포까지 자동으로 돌게 만들었다. 하는 김에 msa 4개 서비스도 같은 구조로 바꿨는데, 그 과정에서 1년 묵은 결함 두 개가 드러났다.

---

## 0. 출발점

### 무엇을 하려 했나

> "백엔드에서 만들고 있는 게임 서버를 ArgoCD도 적용시키고 이 쿠버 인프라 안으로 가져오려고 함. 모노레포고 스프링부트 서버가 있음. 로컬에서만 띄워서 개발 중이고 실제로 서버에 띄우진 않았음"

### 목표 (내가 정한 것)

1. 클라우드에 안 띄우고 로컬에서 혼자 개발하면서 연습
2. OrbStack 쿠버네티스에 계속 띄워두고 **클라우드에 올린 것처럼** 생각
3. **main에 머지하면 쿠버네티스에서 자동으로 무중단 배포**
4. 실무처럼

### 학습 방식

- 한 단계씩, 결과를 보고 나서 다음 단계 ("한번에 다 알려주지 마셈")
- 회사 SRE 구조(NudgeIaC / NudgeManifest / GitHub Actions / ArgoCD)를 기준 아키텍처로 참고

---

## 1. 현황 파악

### 게임 레포를 읽어보니

| 항목 | 상태 |
|---|---|
| 구조 | Gradle 4모듈 (`morris-core` / `storage` / `support` / `api`) |
| Dockerfile | 이미 멀티스테이지 + 의존성 캐싱 + non-root + `MaxRAMPercentage` |
| 헬스체크 | liveness는 `livenessState`만, readiness는 `db`·`gameSession`까지 — 이미 제대로 분리 |
| 종료 | `server.shutdown: graceful`, 정산 30s + 스케줄러 20s 대기 |
| 설정 | 전부 환경변수. 기본값 없는 필수 변수 6개 |
| CI | `./gradlew build`(테스트 포함)까지만. **이미지를 안 만듦** |

레포 자체 ADR-2에 "EKS에 올리려면 먼저 고칠 것 7가지"가 있었는데, 다시 보니 **대부분 끝나 있었다.** 남은 건 이미지 레지스트리 관련 세 개뿐:

- GHCR 푸시 / 커밋 SHA 태그 / arm64 빌드

**이관이 막혀 있던 원인은 쿠버네티스 지식이 아니라 애플리케이션 결함이었고, 그걸 먼저 고쳐뒀기 때문에 지금 쉬웠다.**

### 로컬 레포가 GitHub보다 앞서 있었다

```
origin/main        c3ac5d0
feature/redis-ha   main보다 6커밋 앞 + 미커밋 작업
```

그 브랜치가 하던 일이 정확히 "판 상태를 Redis로 빼서 서버 여러 대로 띄우기"였다. → **Redis 작업이 끝나면 그때 이관하기로 결정.** 그동안 이론부터.

> **Q. 이미 백엔드 서버가 있을 때 현재 쿠버 인프라로 어떻게 이관하는지 연습해볼 기회 같음**

→ 이관 체크리스트 문서(`migration-existing-server-to-k8s.md`)를 먼저 썼다.

---

## 2. 이론 — 이관 체크리스트

### compose가 공짜로 깔아주던 것

| compose | 쿠버네티스 |
|---|---|
| 죽으면 같은 호스트에서 다시 뜸 | 다른 노드에서 뜸. 디스크가 안 따라옴 |
| 이름으로 상대를 찾음 | Service를 선언해야 이름이 생김 |
| `depends_on: service_healthy` | **기동 순서 보장 없음** → readinessProbe로 해결 |
| 오래 삶 | 자주 죽음 |
| 볼륨이 호스트에 남음 | PV/PVC 선언 필요 |

### 6축

이미지 / 설정 / 수명주기 / **상태** / 헬스 / 노출·의존성. **상태 축이 나머지를 압도한다** — 상태가 파드 메모리에 있으면 스케일아웃·롤링 업데이트·자동 복구를 못 쓴다.

> **Q. RDS는 보통 쿠버에서 어떻게 관리함? 쿠버에서 관리 안 하는 게 일반적인가?**

- 맞다. RDS는 **Terraform(클러스터 밖)**이 만들고, 쿠버네티스에는 **엔드포인트(ConfigMap) + 자격증명(Secret)** 두 줄만 들어온다
- 이유: ArgoCD는 쿠버네티스 API만 안다. RDS는 그 시야에 없다
- 회사 구조로 대입하면 `Nudge-IaC`가 RDS를 만들고 `NudgeManifest`는 엔드포인트 문자열만 안다
- ACK / Crossplane으로 RDS를 CRD로 선언할 수도 있지만, `prune: true`가 프로덕션 DB를 지울 수 있어서 보통 제외하거나 `deletionPolicy: Orphan`
- k8s + RDS 고유 함정: **파드 수 × 커넥션 풀 크기가 `max_connections`를 넘는다** → RDS Proxy, 또는 풀 크기를 역산
- **ExternalName Service**로 감싸면 앱은 환경과 무관하게 같은 이름(`morris-db`)만 본다

> **Q. 계획 공유 — 로컬 OrbStack에 계속 띄워두고, main 배포하면 자동 무중단 배포, 실무처럼**

"무중단"의 층을 나눠야 한다:

| 층 | replicas 1에서 |
|---|---|
| HTTP 요청 | ✅ maxSurge 1 → 새 파드 Ready 후 옛 파드 종료 |
| WebSocket 연결 | ❌ 끊김 → 재연결 필요 |
| 진행 중인 판 | ❌ 메모리에 있으면 사라짐 → **redis-ha가 이걸 푸는 작업** |

실무처럼 하려면 추가로 필요한 것: ArgoCD 웹훅, 배포 중 에러율 측정(k6), Argo Rollouts, **msa 이미지를 GHCR로**(로컬 이미지는 `docker prune` 한 번에 사라짐), 운영 숙제. 단일 노드로는 PDB·anti-affinity·노드 장애 복구가 재현 안 됨.

> **Q. 회사 SRE 구조 문서 공유 (NudgeIaC / NudgeManifest / CI / CD)**

비교해서 **세 개를 바꿨다**:

| | 회사 | 원래 계획 | 결정 |
|---|---|---|---|
| Secret | Sealed Secrets | `kubectl create secret` (Git 밖) | **Sealed Secrets로** |
| 태그 | metadata-action, `sha-{commit}` | 수동 `cut -c1-7` | **metadata-action으로** |
| 아키텍처 | arm64 단일 | amd64 + arm64 | **arm64 단일로** |

내 쪽이 더 나은 것: App of Apps (회사는 Application을 로컬에서 수동 apply), values 기반 태그 교체.

> **Q. api-gateway, user, order와 나인멘스모리스는 별개 프로젝트임**

| 분리됨 | 공유됨 (의도적) |
|---|---|
| 소스 레포·CI·이미지·**네임스페이스**·DB·Secret·values·호스트 | 매니페스트 레포(클러스터 단위), `http-service` 차트(워크로드 종류 단위), 클러스터 플랫폼 |

**아직 안 갈라진 것**: 전부 `project: default`. AppProject로 ArgoCD 수준에서 가르는 건 남은 과제.

---

## 3. 실행

Redis 머지 후 시작. 한 단계씩 결과를 확인하고 넘어갔다.

### 1단계 — 렌더 검증

**왜**: `apps/local` → `apps/msa`로 프로젝트 축을 만들고, 차트에 Secret·Ingress 지원을 추가했다. 기존 4개 서비스의 배포 결과가 바뀌면 안 된다.

```bash
git worktree add /tmp/manifests-before HEAD
# before / after 를 helm template 으로 각각 렌더 → diff
```

**결과**: 4개 전부 `차이 없음`.

### 2단계 — 커밋·push

**결과**: Application 전부 `Synced`, **파드 AGE 그대로** (`4d`, `5d`).

경로가 4개 다 바뀌었는데 파드가 안 죽은 이유 → **ArgoCD는 Git의 경로가 아니라 렌더된 결과를 클러스터와 비교한다.**

> **Q. secret, ingress 지원은 뭐하는 거임?**

- `existingSecret` → 렌더에 `envFrom: secretRef` 두 줄이 추가됨. 차트는 Secret을 **만들지 않고 이름만 받는다**. msa는 H2라 비밀값이 없어서 필요 없었음
- `ingress` → **Ingress는 같은 네임스페이스의 Service만 백엔드로 지목할 수 있다.** 게임 서버는 `morris` ns라 `platform/ingress`(default ns)에 한 줄 추가하는 게 불가능. 그래서 차트가 자기 ns에 렌더
- 둘 다 기본값이 꺼짐이라 msa 렌더엔 아무것도 안 나옴 → 1단계 "차이 없음"의 이유

### 3단계 — Sealed Secrets 컨트롤러

**왜**: `kubectl create secret`은 "Git만으로 재현 가능"을 깬다. 회사도 이 방식. `kubeseal --controller-name sealed-secrets --controller-namespace kube-system`이 그대로 동작하도록 이름을 맞춤.

> **Q. 이렇게 명령어로 하면 helm도 명령어로만 하게 되는 거 아닌가?**

| 명령 | 영향 | GitOps 위반 |
|---|---|---|
| `helm repo add` / `search` / `template` | 로컬 / 읽기 | ✗ — 오히려 검증 수단 |
| `helm install` / `upgrade` | 클러스터 변경 | ✓ |

설치는 ArgoCD가 한다. 그리고 진짜로 GitOps 밖에 있는 것: `helm install`로 깐 **kube-prometheus-stack**, ArgoCD·Istio·ingress-nginx·metrics-server 설치 자체 (부트스트랩 영역).

**문제**: `helm repo add` → `index.yaml 404`

- **내 실수**. `bitnami-labs.github.io`가 아니라 `bitnami.github.io`. 소스 레포가 `bitnami-labs/sealed-secrets`라 Pages도 같을 거라 추측했다
- 출력이 세 줄로 갈라져 헷갈렸던 이유: `repo add` 실패 → `repo update`는 등록돼 있던 prometheus-community만 갱신하고 "Update Complete" → `search`는 등록 안 된 레포라 결과 없음. **성공 메시지가 내가 하려던 일의 성공을 뜻하지 않는다**
- **Helm 차트 레포 = `index.yaml` 하나를 서비스하는 정적 파일 서버.** 404는 "차트 없음"이 아니라 "그 주소에 파일 없음"
- 로컬에서 먼저 `helm repo add`를 해본 덕에 ArgoCD에 넣기 전에 잡혔다. 그대로 갔으면 Application이 `ComparisonError`

차트 `2.20.0` (appVersion `0.40.0`) — 로컬 `kubeseal` CLI `0.40.0`과 맞춤.

> **Q. 파일 설명 좀 해주셈. 뭐가 무슨 문법이고 이건 ArgoCD 관련인 거임? 설명이 너무 빈약함**

| 줄 | 누구의 문법 | 뜻 |
|---|---|---|
| `kind: Application` | ArgoCD CRD | 쿠버네티스는 저장만, `argocd-application-controller`가 수렴 |
| `metadata.namespace: argocd` | — | **Application 자신이 사는 곳.** 배포 대상 아님 |
| `sync-wave: "-1"` | ArgoCD | 반드시 문자열. App of Apps에서 root가 Application들을 배포하는 순서 |
| `sources:` (복수) | ArgoCD 2.6+ | multi-source |
| `repoURL` + `chart` | — | **Helm 레포**로 해석. `targetRevision` = 차트 버전 |
| `repoURL` + `path` | — | **Git 디렉토리**로 해석. `targetRevision` = 브랜치/SHA |
| `helm.releaseName` | ArgoCD → helm | 기본은 Application 이름. 리소스 이름이 바뀌므로 덮어씀 |
| `$values/...` | multi-source | `ref: values`인 소스의 루트. **남의 차트에 내 values를 넘기는 장치** |
| `ref: values` (path 없음) | multi-source | 렌더 대상이 아니라 참조용 |
| `syncPolicy.automated` | ArgoCD | 없으면 수동 sync (회사 "대시보드에서 sync"가 이것일 수 있음) |
| `fullnameOverride` (values.yaml) | **차트가 정함** | Bitnami 계열 `_helpers.tpl` 관례. ArgoCD·쿠버네티스와 무관 |

> **Q. 우리 서비스는 직접 만들어서 path로 처리했는데 kubeseal은 외부 깃허브에 있어서 차트를 끌어온다는 거임? kubeseal은 암호화 관련?**

- 반만 맞음. **깃허브가 아니라 Helm 차트 레포**. `bitnami.github.io`는 GitHub Pages(정적 호스팅)라 ArgoCD는 git이 아니라 HTTP로 `.tgz`를 받는다
- 내 차트를 패키징 안 하는 이유: 내가 만들고 내가 쓰면 Git 커밋으로 버전 관리 끝. 남이 쓸 차트만 패키징해서 차트 레포/OCI에 올림
- 네 조각: `kubeseal`(CLI, **암호화**, 내 노트북) / 컨트롤러(**복호화**, 클러스터) / `SealedSecret` CRD / 키 쌍(컨트롤러가 기동 시 생성)
- 공개키로 암호화한 건 사설키 없이 못 푼다 → 공개 레포에 올려도 안전
- **SealedSecret은 ns + 이름에 묶인다** (`strict` scope). 다른 곳에 apply하면 복호화 거부

### 4단계 — 사설키 백업

사설키를 잃으면 Git의 모든 암호문이 영구히 복호화 불가. `~/sealed-secrets-key-backup.yaml` → 1Password. **Git에 암호문 + 1Password에 키 = 클러스터를 처음부터 재현 가능.**

### 5단계 — Secret 봉인

```bash
kubectl create secret generic ... --dry-run=client -o yaml | kubeseal ... > xxx-sealedsecret.yaml
```

**평문을 디스크에 안 쓴다.** `--dry-run=client`가 클러스터에 안 보내고 yaml만 표준출력으로.

| Secret | 키 |
|---|---|
| `morris-mysql-secret` | MYSQL_DATABASE / USER / PASSWORD / ROOT_PASSWORD |
| `morris-game-server-secret` | SPRING_DATASOURCE_PASSWORD / JWT_SECRET_KEY / KAKAO_* |

둘로 나눈 이유: 하나로 합치면 MySQL 컨테이너에 JWT 키가 주입됨. 필요 없는 곳에 비밀을 넣지 않는다. 비밀 아닌 접속 정보(URL, USERNAME)는 ConfigMap.

**문제**: zsh에서 `echo "⚠️ 평문 노출!"`의 `!`가 히스토리 확장을 건드려 `dquote>` 상태. 내가 준 명령에 이모지·느낌표를 넣은 게 원인.

**지적받음**: DB 비밀번호를 채팅에 붙여넣음. 실습용이라 위험은 없지만 이 단계가 정확히 그걸 안 흘리는 연습.

**확인**: 키 이름은 평문, 값만 암호문. `grep` 평문 검사 `exit=1`.

### 6단계 — SealedSecret 배포

- `SealedSecret` 2개 `SYNCED=True` + `Secret` 2개 `DATA=4`
- 컨트롤러 로그 `Unsealed successfully` ×2
- `MYSQL_DATABASE`를 꺼내 `morris` 출력 → **Git에는 암호문만, 클러스터에는 평문**
- 컨트롤러가 만든 Secret엔 ownerReference가 붙음 → SealedSecret을 지우면 Secret도 GC. **수명이 Git에 묶임**

### 7단계 — MySQL StatefulSet

**왜 StatefulSet**: 복제 때문이 아니라 **PVC 수명** 때문. Deployment + RWO PVC는 롤링 업데이트에서 교착(새 파드가 PVC를 못 잡고, 옛 파드는 새 파드가 Ready여야 죽음).

스토리지클래스 `local-path`:

| 컬럼 | 값 | 의미 |
|---|---|---|
| VOLUMEBINDINGMODE | `WaitForFirstConsumer` | 파드가 배치된 뒤에 그 노드에 볼륨 생성. PVC가 잠깐 Pending인 게 정상 |
| RECLAIMPOLICY | `Delete` | PVC 지우면 디스크도 삭제. 운영은 보통 `Retain` |

**문제**: `kubectl get pods,pvc,pv,sts,svc -w` → `you may only specify a single resource type`. **내 실수** — `-w`는 타입 하나만.

**관찰**:
- PVC `2m25s` / PV `2m17s` — PVC가 8초 먼저. `WaitForFirstConsumer`의 흔적
- PV 이름 `pvc-<uuid>` = 동적 프로비저닝
- PV는 ns가 없고 `CLAIM: morris/data-morris-mysql-0`으로 PVC를 가리킴
- 헤드리스 `CLUSTER-IP None`

**실험**: 테이블에 `42` 넣고 → 파드 삭제 → 다시 뜬 뒤 `42` 그대로. 파드 AGE `33s`, PVC AGE `3m46s`.

### 8단계 — Redis StatefulSet

**판단**: 캐시면 Deployment로 충분. **판 상태의 원본**이라 영속화 필요 → StatefulSet + PVC + AOF everysec (compose와 동일).

**추가한 것 `--maxmemory 192mb`** (limit 256Mi보다 낮게):

| | 메모리 초과 시 |
|---|---|
| 없음 | cgroup이 SIGKILL → OOMKilled |
| 있음 | Redis가 먼저 쓰기 거절. 기본 `noeviction`이라 판 상태를 조용히 지우지 않음 |

**문제**: 실험 명령 블록을 통째로 붙여넣어서 Redis 컨테이너가 뜨기 전에 `set`이 실행됨 → `container not found` → 키가 안 들어감 → 재시작 후 nil. **영속화 실패가 아니라 실험 실패.** "1/1 되면"이라는 조건을 주석으로만 걸어둔 내 명령이 원인.

**재실험**: `OK` 확인 후 삭제 → `42`. AOF 구조 `base.rdb`(스냅샷) + `incr.aof`(증분) + `manifest`.

`everysec`인데 왜 안 잃었나 → `kubectl delete pod`는 **SIGTERM**. Redis가 AOF를 내리고 종료. 1초 유실은 OOMKilled·전원 차단처럼 SIGTERM 없이 죽을 때.

> **Q. 원래 보통 mysql, redis도 이렇게 관리함 실무에서?**

아니다. 손으로 쓴 StatefulSet은 학습용.

| 방식 | 주로 |
|---|---|
| 매니지드 (RDS, ElastiCache) | 프로덕션 대부분 |
| Operator (Percona, CloudNativePG, Redis Operator) | 온프레미스·직접 운영 |
| Helm 차트 | dev·스테이징 |
| 손으로 쓴 StatefulSet | 학습 |

없는 것 = Day 2: 백업, 복제·페일오버, 업그레이드, 모니터링, 노드 장애 시 데이터 이동.

**내 구성의 구멍**: readiness에 `db`, `redis`가 있는데 둘 다 파드 하나. 게임 서버를 2개로 올려도 **Redis가 재시작하면 두 파드가 동시에 NotReady → 전체 다운.** SPOF가 앱에서 Redis로 옮겨갔을 뿐.

### 9단계 — 게임 서버 이미지 → GHCR

브랜치 + PR (게임 레포 규칙: 구조 변경은 PR).

| 설정 | 이유 |
|---|---|
| `needs: build` | 테스트가 배포의 문지기 |
| `if: push && main` | PR마다 이미지를 밀지 않음 |
| `runs-on: ubuntu-24.04-arm` | arm64 네이티브. QEMU 회피 |
| `permissions: packages: write` | GITHUB_TOKEN 기본엔 GHCR 쓰기 없음 |
| `context: server` | 모노레포. `defaults.run.working-directory`는 `uses:`에 안 먹음 |

**PR**: `build` ✅ / `image` Skipped → 머지 → `sha-7c21528`, Public, OS/Arch 2 (arm64 + attestation).

덤: Dockerfile의 `HEALTHCHECK`는 쿠버네티스가 무시한다 (kubelet은 Pod spec의 probe만 봄).

### 10단계 — 게임 서버 배포

values 수정 (Redis 머지 반영):
- `SPRING_DATA_REDIS_HOST: morris-redis`
- `replicaCount: 1` 근거 갱신 — "메모리 상태라서"가 아니라 "Redis·MySQL이 단일이라"
- HPA 꺼둔 근거 갱신 — "부하 기준이 없어서. 기준 없이 켜면 MySQL 커넥션을 먼저 바닥냄"

**관찰**:
- `ContainerCreating` = 이미지 받는 중 (397MB, 14초). `kubectl describe pod`의 이벤트 순서 `Scheduled → Pulling → Pulled → Created → Started`
- `QoS Class: Burstable` (requests < limits)
- `Tolerations: not-ready / unreachable 300s` 자동 추가
- 로그: `with PID 1` (exec form → SIGTERM이 JVM에 직접), JSON 로그, Flyway `Empty Schema → V1 → V2` (0.4초), 52초 만에 Ready
- 게임 레포 숙제 두 개: Flyway가 MySQL 8.4를 공식 지원 목록에 안 갖고 있음 / Spring Data Redis가 JPA 리포지토리를 훑는 INFO 소음

**스모크 테스트**: `POST /api/v1/auth/visitors` → JWT. 이 요청 하나가 인그레스 → 앱 → Secret(서명키) → MySQL(행 생성) → Redis(발급 한도)를 다 지난다.

- `sub: "1"`, `exp - iat = 604800` (7일)
- `/actuator/health` 인그레스로 → **404** / port-forward로 → `{"status":"UP"}`. 막은 게 아니라 안 열어준 것

**발견 ① 닉네임 `?????6122`**
- 화면 문제인가 저장 문제인가 → `hex(nickname)` → `ECB0A8 EBB684 ...` (UTF-8 3바이트) → `차분한맷돌6122`
- 컨테이너 안 `mysql` CLI가 로케일이 없어 latin1로 출력한 것. **깨져 보이는 것과 깨진 것은 바이트로 가른다**

**발견 ② `limit:visitor:127.0.0.6`** — 실제 버그
- `127.0.0.6` = Istio Envoy가 들어온 요청을 앱에 넘길 때 쓰는 출발지 주소 (Istio 교시 로그에서 본 것)
- 게임 서버엔 사이드카가 없는데? → **ingress-nginx에 사이드카가 있었다.** mTLS STRICT 실험 때 넣고 PERMISSIVE로 롤백했는데 사이드카만 남음
- nginx 로그 첫 필드 `127.0.0.6`으로 확정
- 영향: **모든 사용자가 게스트 발급 한도 카운터 하나를 공유.** 게임 레포 `application.yml`에 이미 경고로 적혀 있던 그 증상
- 주입 경로 확인: ns 라벨 `istio-injection=enabled`
- 수정: 라벨 제거 + `rollout restart` (주입은 파드 **생성 시** webhook이 하므로 재시작 필요)
- 결과: `192.168.194.1`, 새 Redis 키, `user.lab.local` 회귀 없음
- **실험 잔재가 다른 프로젝트의 버그를 만들었다.** ingress-nginx는 부트스트랩 영역이라 이 변경은 Git에 안 남음 — 나중에 Application으로 옮길 때 기억할 것

> **Q. 쿠버로 띄운 거 툴(TablePlus)에 연결하려는데 뭐 쳐야 할까**

`kubectl port-forward -n morris pod/morris-mysql-0 13306:3306`. Mac의 3306은 로컬 MySQL이 쓰니 13306. `root` 말고 `morris` 계정. 파드 이름을 그대로 박을 수 있는 것도 StatefulSet이라서.

> **Q. 디비는 계속 연결해둬야 하는 거임? 실무에서도 상용에 띄운 거 로컬에서 연결하려면 베스천 쓰긴 하는데**

정확히 같은 모양이다. 터널은 설정이 아니라 **프로세스**.

| | 실무 | 지금 |
|---|---|---|
| 경로 | 노트북 → 베스천 → RDS | 노트북 → API 서버 → kubelet → 파드 |
| 명령 | `ssh -L` | `kubectl port-forward` |
| 끄면 | 끊김 | 끊김 |

OrbStack은 Mac에서 `*.svc.cluster.local`로 바로 붙게 해주지만 EKS에선 안 되는 지름길.

### 11단계 — msa CI 정리

> **Q. user-service, order-service도 다 해주셈**

세 레포 상태:

| | 현황 |
|---|---|
| Dockerfile | `COPY . .` 통짜, root |
| CI 트리거 | push가 주석 처리됨 (수동만) |
| 레지스트리 | Docker Hub |
| 태그 갱신 대상 | 옛 `msa-playground-helm` 레포 (SSH 키) |

**전부 Spring Boot 3.5.5 확인** → product-service Dockerfile을 그대로 쓸 수 있음 (JarLauncher 경로가 3.2에서 바뀌므로 확인 필요했음).

**발견**: user·order는 `application-k8s.yml`이 `org.h2.Driver`를 쓰는데 `build.gradle`에 H2가 없다. product-service가 같은 이유로 크래시 났던 것. 지금 돌던 로컬 이미지는 **이 소스보다 예전 시점에 빌드된 것**이라 멀쩡해 보였을 뿐.

파일은 **product-service 것을 복사해서 이름·포트만** 바꿈 (손으로 다시 쓰면 오타). `diff`로 이름 외 동일 확인.

**빚 하나**: `apps/local` → `apps/msa`로 옮기면서 product-service CI의 `VALUES_PATH`가 깨진 상태. 로컬 경로를 못 찾아 헤맴 — **내 경로 추측이 두 번 틀림**(`find -maxdepth` 깊이도 하나 모자랐음). 결국 `playground-local/product-service`.

**토큰**:
- 기존 classic `manifests-write`는 `repo` 스코프 = **모든 레포 읽기·쓰기.** 범위 과다
- 새로 fine-grained: **`ldh-infra-lab-manifests` 하나, Contents Read and write**
- GitHub UI가 바뀌어서 Permissions를 검색해서 추가하는 방식이었음
- `gh secret set`으로 5개 레포에 (값이 셸 히스토리에 안 남음)
- 덤: Mac git 인증 토큰에 `delete_repo`, `admin:org`, `admin:enterprise`까지 있음 → 따로 축소 권장

> **Q. 왜 나인멘스모리스에는 이거 추가 안 함?**

그때까지 게임 서버 CI는 이미지 푸시까지만 해서 다른 레포에 쓸 일이 없었다(`GITHUB_TOKEN`으로 충분). 곧 자동 배포 잡을 붙일 거라 미리 넣음. 토큰 하나를 여러 레포가 공유하는 대가 = 새면 전부 교체. 레포가 많아지면 GitHub App.

**카나리 순서**: product-service 먼저 — 바뀐 게 토큰뿐이라 **변수 하나만** 시험 가능.

**문제**: product-service 빌드가 13분째. 의존성 해석만 5분 44초.
- amd64 러너에서 arm64를 **QEMU 에뮬레이션**으로 빌드 → JVM 전체가 에뮬레이션 위
- GHA 캐시는 7일 안 쓰면 지워짐 (마지막 빌드 8/30)

> **Q. 다른 것들도 이런가? 너무 느려서 필요 없는 건 걷어내고 개선하는 게 좋을 듯**

| 바꿈 | 이유 |
|---|---|
| `ubuntu-24.04-arm` 러너 | 네이티브 빌드 |
| `linux/arm64` 단일 | 노드가 전부 arm64. amd64 이미지는 아무도 안 받음 |
| yq 설치 제거 | 러너 기본 설치. `latest` 다운로드는 버전 고정이 아님 |

대가: 이제 x86 노드에선 안 돈다. `platforms` 한 줄로 되돌릴 수 있음.

> **Q. 13분 뚫어가는데 그냥 3개 다 개선 커밋하는 게 나을 듯**

동의. 단 그냥 push하면 **레이스**:

```
B(빠름) 끝 → 태그 = B
A(느림) 끝 → 태그 = A   ← 옛 커밋으로 되돌아감
```

→ `concurrency: cancel-in-progress` 추가. 진행 중이던 A는 옛 워크플로라 자동으로 안 끊기니 `gh run cancel`로 수동 취소.

**결과**: 4개 CI가 거의 동시에 매니페스트 레포에 push했는데 히스토리가 한 줄로 쌓임 — `update-manifests`의 `git pull --rebase`가 일함.

### 12단계 — 게임 서버 자동 배포 잡

| 추가 | 이유 |
|---|---|
| `concurrency` | 같은 레이스 |
| 이미지 잡 `outputs.tag` | 태그를 **한 곳에서만** 만듦. 이미지 태그와 매니페스트 태그가 어긋날 수 없는 구조 |
| `manifests` 잡 | 매니페스트 레포에 태그 커밋. `needs: image`라 PR에선 같이 건너뜀 |

> **Q. 지금 로컬 쿠버의 ArgoCD가 GHCR에서 pull해오는 거지?**

반만 맞음. **ArgoCD는 GHCR에 접속하지 않는다.**

| 누가 | 어디서 | 무엇을 |
|---|---|---|
| ArgoCD | GitHub | yaml (이미지 이름 **문자열**) |
| kubelet | GHCR | 이미지 레이어 |

그래서 태그를 틀리게 적으면 ArgoCD는 `Synced`인데 파드는 `ImagePullBackOff` — Sync와 Health가 따로 노는 이유.

**화살표가 전부 Mac에서 밖으로** 나간다. pull 모델이라 공인 IP 없는 로컬 클러스터에서도 GitOps가 성립한다. 같은 이유로 **웹훅은 안 된다**(GitHub → Mac 방향).

**머지 결과**: `6467137 ci(morris): game-server image → sha-63edec2` → 파드 `sha-63edec2` Ready. **3번 목표 완성 — 머지 이후 사람 손 0.**

### 13단계 — msa 롤아웃 실패와 복구

**발단**: "된다고 가정하고 다음". `kubectl get deploy`의 IMAGE 컬럼이 전부 ghcr로 바뀐 걸 보고 **내가 "user·order가 떴다, H2가 맞았다"고 말했다.** 그건 Deployment **스펙**이었지 실제 파드가 아니었다. **내 실수 — 확인 안 된 걸 확인된 것처럼 말함.**

**드러난 경위**: 로컬 이미지를 지우려 했더니 docker가 거부 — `container is using its referenced image`.
- 컨테이너 이름 `k8s_app_user-service-845d7dc555-...` → **옛 파드가 아직 돌고 있었다**
- user·gateway의 새 파드는 `CrashLoopBackOff`
- 롤링 업데이트가 중간에 멈춘 상태. **readinessProbe 덕에 옛 파드가 계속 트래픽을 받아 장애 0**
- `docker rmi -f`로 밀었으면 옛 파드 재시작 시 이미지가 없어 **진짜 장애**

**원인**: `Could not resolve placeholder 'token.secret'`
- JWT 키가 `application-local.yml`에만 있고 k8s 프로파일엔 없음
- local 값은 공개 레포에 커밋된 튜토리얼 키 → 그대로 못 씀
- **"로컬 이미지는 지금 소스로 재현 가능한가를 아무도 검증 안 한다"가 H2에 이어 두 번째로 터짐**

**전수조사**: 코드의 `@Value` 플레이스홀더 vs k8s 프로파일 키

| 서비스 | 빠진 것 |
|---|---|
| user-service | `token.secret`, `token.expiration_time` |
| api-gateway | `token.secret` |

**키 형식 확인**: 둘 다 `secretKey.getBytes()` (base64 디코딩 없음), HS256 → 32바이트 이상. user는 기본 문자셋, gateway는 UTF-8로 바이트를 뽑아서 **ASCII 영숫자로만** 생성.

**user와 gateway가 같은 Secret을 봐야 한다** — 서명과 검증이 같은 키여야 해서. 다르면 둘 다 뜨는데 로그인 후 전부 401.

> **Q. 각 레포에서 수정 안 하고 여기서(매니페스트) 하는 이유는?**

- **값**은 매니페스트에 있어야 한다: 앱 레포는 공개 / 같은 이미지·다른 환경 / 재빌드 불필요 / 회사도 이렇게
- 그런데 이 방식만으로는 **요구사항이 숨는다.** `application-k8s.yml`을 읽어도 `TOKEN_SECRET`이 필요한지 알 수 없다
- 이상적인 분담: **앱 레포 = 선언(필요한 환경변수 목록), 매니페스트 = 값**
- 게임 서버 `application.yml`이 이미 그렇게 돼 있다(`${JWT_SECRET_KEY}` 기본값 없이 + 주석). 그래서 필수 변수를 바로 뽑을 수 있었다

> **Q. 제대로 하셈. 장애 복구 우선 말고**

둘 다 함:

| 레포 | 내용 |
|---|---|
| user-service | `token.secret: ${TOKEN_SECRET}` (기본값 없음), `expiration_time: ${TOKEN_EXPIRATION_TIME:86400000}` |
| api-gateway | `token.secret: ${TOKEN_SECRET}` (기본값 없음) |
| 매니페스트 | SealedSecret `msa-jwt-secret` 하나 + 두 서비스에 `existingSecret` |

**비밀 키에 기본값을 일부러 안 둔다** — 기본값이 있으면 Secret이 빠져도 조용히 뜨고, 두 서비스가 다른 키로 뜨면 찾기 어려운 401 장애가 된다. **빠지면 기동에서 죽는 게 좋은 실패.**

**순서**: 값 먼저(A: 매니페스트) → 값을 요구하는 코드 나중(B: 앱 레포).

**문제**: A 직후 user-service가 또 크래시 — `token.expiration_time`.
- **내 실수.** 만료 시간 기본값을 앱 레포로 옮기면서 매니페스트에서 값을 **코드보다 먼저** 뺐다. 기본값을 가진 코드는 아직 배포 전이었다
- 동시에 좋은 증거: **`token.secret` 에러가 사라졌다** = 환경변수 `TOKEN_SECRET`이 `${token.secret}`으로 해석된다는 증명
- B 커밋 → 새 이미지 `f36958f`로 해결

> **Q. env 파일들도 쿠버로 관리하면, Vault 이런 거 말고 kubeseal로 하는 거임?**

둘 다 쓴다. 차이는 **비밀의 원본이 어디 있나.**

| | 원본 | 키 교체 | 여러 클러스터 | 주로 |
|---|---|---|---|---|
| Sealed Secrets | Git | 재봉인 → 커밋 | 클러스터마다 재봉인 | 소규모 (회사 현재) |
| ESO + Secrets Manager | 외부 | 값만 바꾸면 자동 동기화 | 같은 원본 참조 | AWS 쓰는 대부분 |
| Vault 직접 주입 | 외부 | 동적 비밀 가능 | 동일 | 대규모·규제 |

**k8s Secret 자체는 base64일 뿐.** Sealed Secrets가 지키는 건 Git 구간. 클러스터 안은 RBAC + etcd 저장 암호화(EKS는 KMS)가 지킨다.

`.env`는 "비밀인가"로 쪼갠다 — 비밀 아님 → ConfigMap, 비밀 → Secret.

**최종 확인 (파드 기준)**:

```
api-gateway      0240761   true ×2
order-service    ec0b9d0   true
product-service  aff5ee3   true
user-service     f36958f   true
game-server      sha-63edec2  true
```

`msa/` 이미지를 쓰는 파드 0개. **다섯 번 실패하는 동안 `user.lab.local`은 한 번도 안 끊겼다.**

---

## 4. 내 실수 목록

정직하게 남긴다. 다음에 같은 걸 피하려고.

| # | 실수 | 결과 | 교훈 |
|---|---|---|---|
| 1 | Sealed Secrets 차트 URL을 소스 레포 이름으로 추측 | 404 | 외부 주소는 추측하지 말고 README에서 확인 |
| 2 | `kubectl get a,b,c -w` | 거부됨 | `-w`는 타입 하나 |
| 3 | 명령에 이모지·`!` 넣음 | zsh `dquote>` | zsh에서 `!`는 히스토리 확장 |
| 4 | "1/1 되면"을 주석으로만 걸어둔 명령 블록 | Redis 실험이 컨테이너 뜨기 전에 실행 | 조건이 있는 명령은 블록을 끊어서 준다 |
| 5 | product-service 로컬 경로 추측, `find` 깊이 부족 | 두 번 헛걸음 | 찾고 나서 간다 |
| 6 | **Deployment 스펙을 보고 파드가 떴다고 말함** | 크래시를 놓침 | **원하는 상태 ≠ 실제 상태. 파드를 본다** |
| 7 | **값을 코드보다 먼저 제거** | user-service 재크래시 | 값 깔기 → 요구하는 코드 → 그다음 중복 제거 |
| 8 | 디바이스 VM에서 `git fetch`/`pull` | 네 레포에 lock 파일 잔재 | 네 레포의 git 네트워크 명령은 네가 친다 |

---

## 5. 지금 상태

```
                       Mac (OrbStack, 단일 노드 arm64)
┌─────────────────────────────────────────────────────────────────────┐
│ ingress-nginx  (사이드카 제거됨)                                       │
│   ├─ user.lab.local / order.lab.local ──▶ default ns                 │
│   ├─ argocd.lab.local                 ──▶ argocd ns                  │
│   └─ morris.lab.local  /api /ws 만    ──▶ morris ns                  │
│                                                                     │
│ default ns (istio-injection, mTLS PERMISSIVE)                       │
│   user / order / product / api-gateway   ← 전부 ghcr, CI 자동 갱신   │
│   msa-jwt-secret  ◀── SealedSecret                                  │
│                                                                     │
│ morris ns (사이드카 없음)                                             │
│   morris-game-server  sha-63edec2        ← CI 자동 갱신              │
│   morris-mysql-0  StatefulSet + PVC 2Gi                             │
│   morris-redis-0  StatefulSet + PVC 1Gi + AOF                       │
│   morris-*-secret ×2  ◀── SealedSecret                              │
│                                                                     │
│ kube-system                                                          │
│   sealed-secrets 컨트롤러 2.20.0  ◀── multi-source Application        │
└─────────────────────────────────────────────────────────────────────┘
```

### 배포 흐름 (전 서비스 공통)

```
main 머지 → build(테스트) → image(arm64, GHCR) → manifests(태그 커밋)
          → ArgoCD (폴링 3분) → kubelet pull → 롤링 업데이트
```

### 매니페스트 레포 구조

```
ldh-infra-lab-manifests/
├── argocd/apps/
│   ├── platform-ingress / platform-rbac / platform-sealed-secrets
│   ├── user-service / order-service / product-service / api-gateway / msa-secrets
│   └── morris-secrets / morris-mysql / morris-redis / morris-game-server
├── charts/http-service/        0.3.0 — existingSecret, ingress 추가
├── apps/
│   ├── msa/{user,order,product}-service, api-gateway, secrets/
│   └── morris/game-server, mysql/, redis/, secrets/
└── platform/ingress, rbac, sealed-secrets
```

### sync-wave

| wave | Application |
|---|---|
| -1 | platform-sealed-secrets |
| 0 | morris-secrets (+ 지정 안 한 것들) |
| 1 | morris-mysql, morris-redis |
| 2 | morris-game-server |

---

## 6. 남은 것

| | 항목 | 크기 |
|---|---|---|
| 1 | user·order의 `${DB_USER}` / `${DB_PASSWORD}`가 리터럴로 들어갔을 가능성 확인 | 작음 |
| 2 | AppProject — `platform` / `msa` / `morris` | 중간 |
| 3 | **게임 서버 진짜 무중단 실험** — replicas 2, 판 진행 중 배포, Redis 죽이기 | 중간 |
| 4 | msa 태그 형식 `sha-` 통일 (metadata-action) | 작음 |
| 5 | ingress-nginx·kube-prometheus-stack을 Application으로 (부트스트랩 목록 줄이기) | 중간 |
| 6 | Mac git 인증 토큰 권한 축소 | 작음 |
| 7 | 게임 레포 숙제 — Flyway 버전 / Redis 리포지토리 스캔 끄기 | 작음 |
| 8 | ArgoCD 웹훅 (cloudflared 터널) 또는 폴링 주기 단축 | 중간 |
| 9 | 옛 시크릿 정리 — `DOCKERHUB_*`, `HELM_DEPLOY_KEY` | 작음 |

---

## 7. 화이트보드 질문

1. compose의 `depends_on: service_healthy`를 쿠버네티스에서는 무엇이 대신하나. 왜 그게 더 나은가
2. 경로가 4개 다 바뀐 커밋을 push했는데 파드가 하나도 재시작되지 않았다. ArgoCD는 무엇과 무엇을 비교하는가
3. ArgoCD는 GHCR에 접속하는가. 새 이미지가 파드가 되기까지의 화살표를 그리고, 각 화살표의 주체를 적어라
4. 로컬 클러스터에서 GitOps는 되는데 웹훅은 안 되는 이유를 화살표 방향으로 설명하라
5. Helm 차트 레포의 정체는 무엇인가. `index.yaml` 404는 무엇을 뜻하나
6. multi-source Application에서 `$values`와 `ref`는 각각 무엇인가. 왜 필요한가
7. Sealed Secrets의 키 쌍은 누가 언제 만들고, 사설키를 잃으면 무슨 일이 생기나
8. SealedSecret을 다른 네임스페이스에 apply하면 왜 복호화가 거부되나
9. 복제본 1개 DB에 StatefulSet을 쓰는 이유를 PVC 수명과 RWO 교착으로 설명하라
10. `WaitForFirstConsumer`가 없으면 멀티 노드에서 무슨 일이 생기나
11. Redis `maxmemory`를 컨테이너 limit보다 낮게 잡는 이유. `noeviction`이 원본 저장소에 맞는 이유
12. 게임 서버를 2개로 올려도 무중단이 아닌 이유 (readiness 그룹 + 단일 Redis)
13. `limit:visitor:127.0.0.6`이 어디서 왔는지 nginx → 사이드카 → 앱까지 추적하라
14. Istio 사이드카 주입 라벨을 지워도 왜 재시작해야 빠지나
15. `kubectl get deploy`의 IMAGE가 바뀌었는데 서비스는 옛 이미지로 돌고 있었다. 무엇이 무엇을 막았나
16. 새 파드가 크래시하는 동안 서비스가 안 끊긴 이유. `docker rmi -f`를 했다면 어떻게 됐나
17. 비밀 값에 기본값을 두면 안 되는 이유를 user/gateway 서명·검증 구조로 설명하라
18. "값은 매니페스트, 선언은 앱 레포" — 각각 어디에 무엇이 있어야 하나
19. CI 두 개가 같은 레포에 동시에 커밋할 때, 그리고 같은 앱의 옛 빌드가 늦게 끝날 때 각각 무엇이 막아주나
20. Sealed Secrets / ESO / Vault 직접 주입은 비밀의 원본이 각각 어디에 있나

---

## 8. ADR 한 줄

- **[이관]** 이관이 막혀 있던 원인은 쿠버네티스 지식이 아니라 애플리케이션 결함이었다. 앱을 먼저 고쳐둔 덕에 실제 이관은 매니페스트 작성과 이미지 레지스트리뿐이었다.
- **[이미지]** 로컬에서만 빌드한 이미지는 "지금 소스로 재현 가능한가"를 아무도 검증하지 않는다. 레지스트리 + CI로 옮기는 순간 1년 묵은 결함 두 개(H2 누락, JWT 키 누락)가 드러났다.
- **[상태]** Deployment 스펙(원하는 상태)과 파드(실제 상태)는 다르다. 배포 확인은 파드로 한다.
- **[무중단]** 다섯 번의 롤아웃 실패 동안 장애가 없었던 건 readinessProbe 때문이다. 무중단 배포의 전제조건은 파이프라인이 아니라 프로브다.
- **[Secret]** 비밀 값은 매니페스트(SealedSecret), 그 값이 필요하다는 선언은 앱 레포 설정에. 비밀에는 기본값을 두지 않는다 — 빠지면 기동에서 죽어야 한다.
- **[Secret]** 값 교체 순서: 값을 깔고 → 그 값을 요구하는 코드를 올리고 → 중복을 지운다. 역순이면 그 사이가 장애다.
- **[CI]** 같은 레포에 여러 CI가 동시에 커밋하면 `pull --rebase`, 같은 앱의 옛 빌드가 늦게 끝나면 `concurrency: cancel-in-progress`. 둘은 다른 문제다.
- **[CI]** 태그는 이미지 잡에서 한 번만 만들고 출력으로 넘긴다. 두 곳에서 계산하면 언젠가 어긋난다.
- **[CI]** 노드가 전부 arm64면 amd64 이미지는 빌드 시간만 늘린다. arm 러너에서 네이티브로.
- **[토큰]** CI가 다른 레포에 쓰는 토큰은 그 레포 하나, 필요한 권한 하나로. classic `repo` 스코프는 계정 전체다.
- **[GitOps]** pull 모델이라 공인 IP 없는 로컬 클러스터에서도 GitOps가 된다. 같은 이유로 웹훅은 안 된다.
- **[실험 잔재]** 롤백한 실험의 잔재(ingress-nginx 사이드카)가 다른 프로젝트의 버그(IP 한도 공유)를 만들었다. 롤백은 설정만이 아니라 부산물까지.
- **[관찰]** "깨져 보인다"와 "깨졌다"는 바이트로 가른다.
