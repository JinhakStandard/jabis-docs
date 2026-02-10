# jabis-helm

> 전체 Helm 배포 관리 (K8S)

---

## 기술 스택

- Helm 3.9.3
- common-helm 1.1.0 의존성
- Harbor (harbor-apply.jinhaksa.com)
- Vault Agent
- yq (YAML 처리)
- K3S

## 프로젝트 구조

```
jabis-helm/
├── Chart.yaml              # v1.0.198-PROD
├── Chart.lock
├── charts/
│   └── common-helm-1.0.4.tgz
├── templates/
│   └── NOTES.txt
├── prod-applications-values.yaml   # 서비스 설정
├── prod-ingress-values.yaml        # Ingress 라우팅
├── prod-pvc-values.yaml            # PVC (암호화)
└── bitbucket-pipelines.yml
```

## 주요 명령어

| 명령어 | 설명 |
|--------|------|
| `helm dependency update` | 의존성 chart 업데이트 |
| `helm lint` | chart 문법 검증 |
| `helm package .` | chart 패키징 |
| `helm install/upgrade` | K8S 배포 |
| `helm push` | Harbor에 chart 업로드 |

## 포트

N/A (여러 서비스 배포)

## 환경변수 (Dockerfile)

| 변수 | 값 | 설명 |
|------|-----|------|
| `NAMESPACE` | jabis | K8S 네임스페이스 |
| `ENVIRONMENT` | PROD/ALPHA/DEV | 배포 환경 |
| `HARBOR_DOMAIN` | harbor-apply.jinhaksa.com | Harbor 레지스트리 |
| `DOCKER_USERNAME` | - | Harbor 인증 |
| `DOCKER_PASSWORD` | - | Harbor 인증 |

## Dockerfile

없음 (Helm chart)

## Bitbucket Pipeline

- Pattern A
- 자동 트리거 없음 (다른 서비스의 CI/CD에서 업데이트)

## Skills

없음

## 프로젝트 고유 규칙

- 전체 JABIS 서비스의 중앙 배포 허브 역할
- common-helm chart 의존성으로 표준화된 K8S 패턴 적용
- `prod-applications-values.yaml`에 12+ 서비스 설정 포함
- `prod-ingress-values.yaml`에 도메인 라우팅 설정
- 다른 서비스 CI/CD에서 `yq`로 image tag 업데이트 후 push
- ArgoCD가 Harbor에서 pull하여 K8S에 적용
- **VALUES FILES 순서 중요**: `values.yaml` -> `values-{env}.yaml` (이 순서 반드시 준수)
