# ldh-infra-lab-manifests

로컬 쿠버네티스(OrbStack) 학습용 GitOps 매니페스트 레포.
ArgoCD가 이 레포를 감시하며 클러스터 상태를 동기화한다.

## 구조

| 경로 | 역할 | 관리 |
|---|---|---|
| `argocd/root-app.yaml` | App of Apps 루트. 이것만 수동 apply | 수동 1회 |
| `argocd/apps/` | Application 정의. 파일 추가 = 앱 추가 | root가 자동 생성 |
| `charts/http-service/` | HTTP 서비스 공통 차트 (언어 무관) | — |
| `charts/http-service/profiles/` | 언어별 기본값 (spring.yaml) | — |
| `apps/local/<앱>/values.yaml` | 앱별 설정값 | 각 Application |
| `platform/ingress/` | Ingress 규칙 (`*.lab.local`) | platform-ingress |
| `platform/rbac/` | msa ServiceAccount + Role | platform-rbac |
| `docs/` | 학습 기록 | — |

## values 적용 순서
뒤에 오는 것이 앞을 덮어쓴다. 맵(`config` 등)은 키 단위로 병합된다.

1. `charts/http-service/values.yaml` — 언어 중립 기본값
2. `charts/http-service/profiles/<언어>.yaml` — 언어 공통
3. `apps/local/<앱>/values.yaml` — 앱 개별

## 차트 명명 원칙
차트는 언어가 아니라 워크로드 종류로 나눈다.
- `http-service` — HTTP 포트를 받는 장기 실행 서비스
- `worker` — 큐 컨슈머 (예정)
- `cronjob` — 배치 (예정)

## Git 밖에서 설치한 것
- ArgoCD v3.5.3 — `kubectl apply --server-side --force-conflicts` 필수
- Istio (demo, mTLS PERMISSIVE), ingress-nginx, metrics-server, kube-prometheus-stack

## 이전 실습
Helm 입문 실습(simple-app, web-service)은 태그 `phase4-helm-practice` 참고.
