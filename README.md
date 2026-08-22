# ldh-infra-lab-manifests

인프라 학습용 배포 매니페스트 레포.

## 구조
- `charts/` — 재사용 가능한 Helm 차트 (템플릿)
- `apps/` — 환경별 values (dev, prod)

## 왜 코드 레포와 분리하나
앱 코드와 배포 설정이 같은 레포에 있으면, CI가 이미지 태그를 커밋할 때
다시 CI가 트리거되는 루프가 생긴다. 또한 ArgoCD는 "배포 상태"만 추적하길
원하는데 애플리케이션 코드 커밋이 섞이면 노이즈가 된다.

## 관련 레포
- `ldh-infra-lab-services` — 애플리케이션 소스 (예정)
- `ldh-infra-lab-cluster` — Terraform (예정)
