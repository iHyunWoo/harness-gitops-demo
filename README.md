# harness-gitops-demo

Harness GitOps 실습용 레포. `hello-web` 하나를 dev / prod 두 환경에 Argo CD(Harness GitOps Agent)로 배포한다.

## 구조

```
apps/hello-web/
  base/            # Deployment + Service (http-echo)
  envs/dev/        # namespace hello-dev,  replicas 1
  envs/prod/       # namespace hello-prod, replicas 2
kind-cluster.yaml  # 로컬 클러스터(harness-gitops)
```

배포 = `envs/<env>/kustomization.yaml` 의 `greeting` 값(또는 이미지 태그)을 바꿔 커밋 → Agent 가 Sync.

## Harness 엔티티 매핑

| Harness Entity | 이 레포에서 |
|---|---|
| GitOps Agent | `harness-gitops` kind 클러스터에 설치된 agent `localagent` |
| GitOps Cluster | 같은 클러스터(in-cluster) |
| GitOps Repository | 이 레포 |
| Service | `hello_web` |
| Environment | `dev`, `prod` (둘 다 위 Cluster 에 연결) |
| GitOps Application | `hello-web-dev` → `apps/hello-web/envs/dev`, `hello-web-prod` → `apps/hello-web/envs/prod` |

## 확인

```bash
kubectl --context kind-harness-gitops -n hello-dev port-forward svc/hello-web 8080:80
curl localhost:8080   # hello from dev (v1)
```
