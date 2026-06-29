# Deployment Flow

## Overview

```mermaid
flowchart LR
    A[fa:fa-code-branch Service Repo<br/>push to main] --> B[fa:fa-github GitHub Actions]
    B --> C[fa:fa-hammer Build and Test]
    C --> D[fa:fa-cube Docker Buildx<br/>linux/amd64]
    D --> E[fa:fa-docker Docker Hub<br/>cmh1448/finvibe-*:sha7]

    B --> F[fa:fa-book Checkout Manifest Repo]
    E --> G[fa:fa-pen-to-square Update image tag<br/>deployment.yaml]
    F --> G
    G --> H[fa:fa-upload Commit and Push<br/>Finvibe_Backend_Manifest]

    H --> I[fa:fa-arrows-rotate Argo CD]
    I --> J[fa:fa-cloud Kubernetes<br/>finvibe namespace]

    J --> K[fa:fa-box Deployment]
    J --> L[fa:fa-network-wired Service<br/>ClusterIP 80 -> 8080]
```

## Service Mapping

| Service repo | Docker image | Manifest path |
| --- | --- | --- |
| `Finvibe_Backend_Monolith` | `cmh1448/finvibe-monolith:sha7` | `backend/deployment.yaml` |
| `Finvibe_Backend_Batch` | `cmh1448/finvibe-backend-batch:sha7` | `batch/deployment.yaml` |
| `Finvibe-Websocket-Listener-main` | `cmh1448/finvibe-websocket-listener:sha7` | `websocket-listener/deployment.yaml` |
| `Finvibe_Profit_Worker` | `cmh1448/finvibe-profit-worker:sha7` | `profit-worker/deployment.yaml` |
| `Finvibe_Profit_Worker_Webflux` | `cmh1448/finvibe-profit-worker-webflux:sha7` | `profit-worker-webflux/deployment.yaml` |
| `Finvibe_Profit_Worker_Golang` | `cmh1448/finvibe-profit-worker-golang:sha7` | `profit-worker-golang/deployment.yaml` |

## Infra Managed By Argo CD

```mermaid
flowchart LR
    A[fa:fa-book Finvibe_Backend_Manifest] --> B[fa:fa-database Redis Application]
    A --> C[fa:fa-stream Kafka Application]

    B --> D[fa:fa-file-code redis/redis-cluster.yaml]
    C --> E[fa:fa-ship Bitnami Kafka Helm Chart]
    C --> F[fa:fa-file-code kafka/kafka-values-kraft.yaml]
    C --> G[fa:fa-folder kafka/manifests]

    D --> H[fa:fa-cloud finvibe namespace]
    E --> H
    F --> H
    G --> H
```

## Notes

- 배포는 서비스 레포의 `main` 브랜치 push에서 시작된다.
- 이미지 태그는 GitHub commit SHA 앞 7자리다.
- CI가 manifest 레포의 각 `deployment.yaml` 이미지 태그를 수정하고 push한다.
- Redis와 Kafka는 이 manifest 레포에 Argo CD Application manifest가 있다.
- 서비스 Deployment를 추적하는 Argo CD Application manifest는 로컬 파일에서 확인되지 않았다.
- Mermaid 렌더러가 Font Awesome을 지원하지 않으면 `fa:fa-*` 텍스트로 보일 수 있다.
