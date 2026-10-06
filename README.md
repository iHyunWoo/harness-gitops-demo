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
| GitOps Agent | `localagent` (kind 클러스터 `harness-gitops`, ns `harness-gitops`) |
| GitOps Cluster | dev → `incluster`(`https://kubernetes.default.svc`), prod → `prod_cluster`(`https://kubernetes.default.svc.cluster.local`, 같은 클러스터를 다른 주소로 등록) |
| GitOps Repository | `hello_repo` (이 레포, 익명 HTTPS) |
| ApplicationSet | `argocd/hello-web-appset.yaml` → `hello-web-dev`, `hello-web-prod` 생성 (라벨 `harness.io/serviceRef`·`envRef`) |
| Service | `hello_web` — Artifact `hashicorp/http-echo`(Docker Hub, tag 실행 시 입력) · Release Repo `envs/<+env.name>/values.yaml` · appsetConfigs `hello-web` |
| Environment | `dev`(PreProduction) → `incluster`, `prod`(Production) → `prod_cluster` |
| Pipeline | `hello_web_gitops_deploy`: Update Release Repo(greeting, `image.tag`=`<+artifacts.primary.tag>`) → Merge PR → GitOps Sync(`hello-web-<+env.name>`) → Verify(dev 만) / 실패 시 Revert PR → Merge |
| Connector | `github_demo`(쓰기, Secret `github_token`), `dockerhub`(`https://registry.hub.docker.com/v2/` — `index.docker.io` 는 태그 조회 실패), `prometheus_demo`, `loki_demo` |
| Delegate | `kind-delegate` |

실행: 파이프라인 입력 = Environment(dev/prod) · 이미지 태그 · greeting.

## 확인

```bash
kubectl --context kind-harness-gitops -n hello-dev port-forward svc/hello-web 8080:80
curl localhost:8080   # hello from dev (v5 via Harness pipeline)
```

## 셋업 메모

- Harness: Org `default` / Project `gitops_demo`, Argo 프로젝트 매핑 `default-gitops-demo` → Application 의 `spec.project` 는 이 이름이어야 한다.
- Application 라벨 `harness.io/serviceRef=hello_web`, `harness.io/envRef=<env>` 가 Service·Environment 연결 고리.
- Agent 기본 매니페스트는 repo-server / application-controller 가 각각 메모리 3Gi 를 요청해서(init container 포함) Docker 메모리 3.5GB 에선 Pending 된다. 로컬에선 `kubectl set resources` / `patch` 로 요청을 256Mi 수준으로 낮춰서 띄움.
- Fetch Linked Apps 는 ApplicationSet 의 자식 앱을 **환경 구분 없이 전부** 가져오고, GitOps Sync 는 실행 환경과 다른 앱을 `Application does not correspond to the environment(s)` 로 실패 처리한다. ApplicationSet 하나가 dev·prod 를 함께 만드는 구조라 Fetch Linked Apps 를 빼고 Sync 대상을 `hello-web-<+env.name>` 으로 지정했다. (Deployment Repo 매니페스트는 2026-10-04 지원 종료 → Service `appsetConfigs` 사용)
- 인스턴스(Service 대시보드)는 Service+Environment 로 실행된 PR 파이프라인 기준으로 잡힌다. Artifact 를 정의해야 카드에 `http-echo:<tag>` 버전이 표시된다.
- GitHub 커넥터 `github_demo` 는 프로젝트 시크릿 `github_token` 을 쓴다.

## 모니터링 (Harness Monitored Service)

클러스터 `monitoring` 네임스페이스에 Prometheus · Loki · Promtail (helm), `loadgen` 에 dev 로 2초마다 요청하는 트래픽 생성기.

| Harness | 값 |
|---|---|
| Connector | `prometheus_demo` → `http://prometheus-server.monitoring/`, `loki_demo`(Custom Health) → `http://loki.monitoring:3100/` (둘 다 Delegate 경유) |
| Monitored Service | `hello_web_dev` (= Service `hello_web` × Environment `dev`) |
| Health Source | Prometheus: cpu / memory / restarts (pod 별), Loki: `{namespace="hello-dev", container="hello-web"}` |

Harness → Project `gitops-demo` → Monitored Services(또는 Service Reliability) → `hello_web_dev` 에서 확인.

## 일반 CD(push) 비교용

| Harness | 값 |
|---|---|
| Connector | `k8s_local` (K8sCluster, Delegate 권한 상속, selector `kind-delegate`) |
| Infrastructure Definition | `dev`/`prod` Environment 각각 `local_kind` → ns `hello-cd-dev` / `hello-cd-prod` |
| Service | `hello_web_cd` (GitOps 아님) — HelmChart `apps/hello-web/chart` + values `apps/hello-web/cd/values.yaml`(Harness 표현식), Artifact 동일 |
| Pipeline | `hello_web_cd_deploy` — K8s Rolling Deploy / 실패 시 Rolling Rollback (입력: Environment, 이미지 태그) |

같은 Environment 에 **GitOps Cluster 링크(GitOps 용)** 와 **Infrastructure Definition(일반 CD 용)** 이 함께 붙는다.
