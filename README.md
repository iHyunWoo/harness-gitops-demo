# harness-gitops-demo

Harness GitOps 실습용 레포. `hello-web` 하나를 dev / prod 두 환경에 Argo CD(Harness GitOps Agent)로 배포한다.

## 구조

```
apps/hello-web/
  chart/                # Helm chart (Deployment + Service, http-echo)
  envs/dev/values.yaml  # namespace hello-dev,  replicas 1  ← Release Repo 파일
  envs/prod/values.yaml # namespace hello-prod, replicas 2
kind-cluster.yaml  # 로컬 클러스터(harness-gitops)
```

배포 = `envs/<env>/values.yaml` 의 `greeting` 을 바꿔 커밋 → Agent 가 Sync.
직접 커밋하거나, Harness PR 파이프라인 `hello_web_gitops_deploy` 로 PR→머지→Sync 를 돌린다.

## Harness 엔티티 매핑

| Harness Entity | 이 레포에서 |
|---|---|
| GitOps Agent | `harness-gitops` kind 클러스터에 설치된 agent `localagent` |
| GitOps Cluster | 같은 클러스터(in-cluster) |
| GitOps Repository | 이 레포 |
| Service | `hello_web` |
| Environment | `dev`, `prod` (둘 다 위 Cluster 에 연결) |
| GitOps Application | `hello-web-dev` / `hello-web-prod` → `apps/hello-web/chart` + `envs/<env>/values.yaml` |
| Release Repo Manifest | Service 의 `apps/hello-web/envs/<env>/values.yaml` |
| Pipeline | `hello_web_gitops_deploy` (Update Release Repo → Merge PR → GitOps Sync) |
| Delegate | `kind-delegate` (같은 클러스터) |

## 확인

```bash
kubectl --context kind-harness-gitops -n hello-dev port-forward svc/hello-web 8080:80
curl localhost:8080   # hello from dev (v5 via Harness pipeline)
```

## 셋업 메모

- Harness: Org `default` / Project `gitops_demo`, Argo 프로젝트 매핑 `default-gitops-demo` → Application 의 `spec.project` 는 이 이름이어야 한다.
- Application 라벨 `harness.io/serviceRef=hello_web`, `harness.io/envRef=<env>` 가 Service·Environment 연결 고리.
- Agent 기본 매니페스트는 repo-server / application-controller 가 각각 메모리 3Gi 를 요청해서(init container 포함) Docker 메모리 3.5GB 에선 Pending 된다. 로컬에선 `kubectl set resources` / `patch` 로 요청을 256Mi 수준으로 낮춰서 띄움.
- PR 파이프라인: 실행 시 `environmentRef`(dev/prod) 와 `greeting` 을 입력 → Delegate 가 `envs/<env>/values.yaml` 수정 브랜치·PR 생성 → 머지 → `hello-web-<env>` Sync.
- Fetch Linked Apps 단계는 Service 의 Deployment Repo 매니페스트가 **ApplicationSet YAML** 이어야 동작한다. 이 데모는 Application 을 직접 만들었으므로 빼고, GitOps Sync 에 앱 이름을 직접 지정했다.
- GitHub 커넥터 `github_demo` 는 프로젝트 시크릿 `github_token` 을 쓴다.

## 모니터링 (Harness Monitored Service)

클러스터 `monitoring` 네임스페이스에 Prometheus · Loki · Promtail (helm), `loadgen` 에 dev 로 2초마다 요청하는 트래픽 생성기.

| Harness | 값 |
|---|---|
| Connector | `prometheus_demo` → `http://prometheus-server.monitoring/`, `loki_demo`(Custom Health) → `http://loki.monitoring:3100/` (둘 다 Delegate 경유) |
| Monitored Service | `hello_web_dev` (= Service `hello_web` × Environment `dev`) |
| Health Source | Prometheus: cpu / memory / restarts (pod 별), Loki: `{namespace="hello-dev", container="hello-web"}` |

Harness → Project `gitops-demo` → Monitored Services(또는 Service Reliability) → `hello_web_dev` 에서 확인.
