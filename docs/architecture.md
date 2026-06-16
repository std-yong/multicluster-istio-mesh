# Architecture

## 멀티클러스터 토폴로지

같은 mesh ID(`mesh1`)를 공유하는 **Primary-Remote** 구조.

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Cluster 1 (Primary)   │         │   Cluster 2 (Remote)    │
│   ┌─────────────┐       │         │       ┌─────────────┐   │
│   │ Istiod      │ ◀──── Remote Secret ───▶│ Istiod      │   │
│   └──────┬──────┘       │         │       └──────┬──────┘   │
│          │ sidecar inj. │         │ sidecar inj. │          │
│   ┌──────▼──────┐       │         │       ┌──────▼──────┐   │
│   │ App + Envoy │ ◀───── mTLS ───────────▶│ App + Envoy │   │
│   └─────────────┘       │         │       └─────────────┘   │
└─────────────────────────┘         └─────────────────────────┘
                같은 mesh ID = mesh1
```

- **Remote Secret**: 각 Istiod가 상대 클러스터 API Server에 접근할 수 있는 자격증명 교환
- **mTLS**: 클러스터 간 트래픽도 자동 암호화 (같은 Root CA 신뢰가 전제)
- **East-West Gateway**: 다른 네트워크 대역일 때 필요 — 본 프로젝트에서는 동일 네트워크로 범위 축소

## CI/CD 흐름

```
PR merge → Jenkins (Build + Docker Hub push)
        → Manifest repo image tag 갱신
        → ArgoCD가 Git 변경 감지
        → 양쪽 클러스터에 자동 sync
        → Envoy sidecar 주입된 Pod로 롤링
```

## 모니터링 대시보드 분리

| 대시보드 | 관점 | 보는 사람 |
|---|---|---|
| Mesh Dashboard | 사용자 요청이 정상 처리되는가 | 서비스 담당자 |
| Control Plane Dashboard | Istiod·Envoy 자체가 정상인가 | 인프라 담당자 |

두 관점을 섞으면 "서비스 장애"와 "Istio 자체 장애"를 같은 알람으로 받게 되어 대응 경로가 모호해진다.
