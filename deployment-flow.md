# Finvibe Deployment Flow

## CI/CD flow

```mermaid
flowchart LR
    dev[Developer pushes to service repo main]
    pr[Pull request to main/develop]

    subgraph service_ci[GitHub Actions in each service repo]
        checkout[Checkout service repo]
        build[Build and test]
        tag[Create image tag from GITHUB_SHA first 7 chars]
        docker[Build linux/amd64 Docker image]
        push[Push image to Docker Hub]
        manifest_checkout[Checkout DEPth-FinVibe/Finvibe_Backend_Manifest]
        update_manifest[Update service deployment.yaml image tag]
        commit_manifest[Commit and push manifest repo]
    end

    subgraph registry[Docker Hub]
        images[cmh1448/finvibe-*:sha7]
    end

    subgraph gitops[Manifest repo]
        deployments[service/deployment.yaml]
        services[service/service.yaml]
        infra[redis and kafka Argo CD applications]
    end

    subgraph cluster[Kubernetes cluster: finvibe namespace]
        argocd[Argo CD]
        workloads[Deployments and Services]
        redis[Redis Cluster]
        kafka[Kafka KRaft]
    end

    pr --> checkout --> build
    build -. PR stops before deploy .-> pr_done[CI check result]

    dev --> checkout --> build --> tag --> docker --> push --> images
    push --> manifest_checkout --> update_manifest --> commit_manifest --> deployments
    deployments --> argocd
    services --> argocd
    infra --> argocd
    argocd --> workloads
    argocd --> redis
    argocd --> kafka
    workloads --> images
```

## Service image update map

```mermaid
flowchart TB
    monolith[Finvibe_Backend_Monolith CI] --> monolith_image[cmh1448/finvibe-monolith:sha7]
    monolith_image --> backend_manifest[manifest/backend/deployment.yaml]

    batch[Finvibe_Backend_Batch CI] --> batch_image[cmh1448/finvibe-backend-batch:sha7]
    batch_image --> batch_manifest[manifest/batch/deployment.yaml]

    websocket[Finvibe-Websocket-Listener-main CI] --> websocket_image[cmh1448/finvibe-websocket-listener:sha7]
    websocket_image --> websocket_manifest[manifest/websocket-listener/deployment.yaml]

    worker[Finvibe_Profit_Worker CI] --> worker_image[cmh1448/finvibe-profit-worker:sha7]
    worker_image --> worker_manifest[manifest/profit-worker/deployment.yaml]

    webflux[Finvibe_Profit_Worker_Webflux CI] --> webflux_image[cmh1448/finvibe-profit-worker-webflux:sha7]
    webflux_image --> webflux_manifest[manifest/profit-worker-webflux/deployment.yaml]

    golang[Finvibe_Profit_Worker_Golang CI] --> golang_image[cmh1448/finvibe-profit-worker-golang:sha7]
    golang_image --> golang_manifest[manifest/profit-worker-golang/deployment.yaml]

    gateway_manifest[manifest/gateway/deployment.yaml]
    gateway_note[Gateway image exists in manifest, but matching local CI repo was not found]
    gateway_note -.-> gateway_manifest
```

## Runtime topology from manifests

```mermaid
flowchart LR
    gateway[Gateway<br/>ClusterIP 80 -> 8080]
    backend[Backend Monolith<br/>ClusterIP 80 -> 8080]
    batch[Backend Batch<br/>ClusterIP 80 -> 8080]
    ws[WebSocket Listener<br/>ClusterIP 80 -> 8080]
    worker[Profit Worker<br/>ClusterIP 80 -> 8080]
    webflux[Profit Worker WebFlux<br/>ClusterIP 80 -> 8080]
    go_worker[Profit Worker Go<br/>ClusterIP 80 -> 8080]

    mysql[(MySQL via DB_URL)]
    redis[(Redis Cluster)]
    kafka[(Kafka)]
    secrets[Kubernetes Secrets<br/>finvibe-app-secrets<br/>finvibe-jwt-secret]

    gateway --> backend
    gateway --> ws

    backend --> mysql
    backend --> redis
    backend --> kafka
    backend --> secrets

    batch --> mysql
    batch --> redis
    batch --> kafka
    batch --> secrets

    worker --> mysql
    worker --> redis
    worker --> kafka
    worker --> secrets

    webflux --> redis
    webflux --> kafka
    webflux --> backend
    webflux --> secrets

    go_worker --> redis
    go_worker --> kafka
    go_worker --> secrets

    ws --> redis
    ws --> secrets
```

## Argo CD resources confirmed in this repo

```mermaid
flowchart TB
    manifest_repo[DEPth-FinVibe/Finvibe_Backend_Manifest]

    manifest_repo --> redis_app[argocd/redis-cluster-application.yaml]
    redis_app --> redis_path[path: redis]
    redis_path --> redis_ns[finvibe namespace]
    redis_app --> redis_policy[automated sync<br/>prune + selfHeal]

    manifest_repo --> kafka_app[argocd/kafka-application.yaml]
    kafka_app --> bitnami[Bitnami Kafka chart 32.4.3]
    kafka_app --> values[kafka/kafka-values-kraft.yaml]
    kafka_app --> kafka_extra[kafka/manifests]
    kafka_app --> kafka_ns[finvibe namespace]

    service_note[Service Deployment Argo CD Application manifests were not found locally]
    manifest_repo -. expected GitOps source .-> service_note
```

## Notes

- Service CI workflows deploy only on `push` to `main`.
- `pull_request` workflows for Batch, Profit Worker, Profit Worker WebFlux, and Profit Worker Go stop at build/test and do not push images or update manifests.
- Image tags are the first 7 characters of `GITHUB_SHA`.
- Application service manifests define `Deployment` plus `ClusterIP` `Service` resources with port `80` targeting container port `8080`.
- Redis and Kafka are explicitly represented as Argo CD applications in this repo. Service-level Argo CD `Application` manifests were not present in the checked files.
