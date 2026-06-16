# Incidents & Decisions

운영하며 마주친 의사결정 기록.

---

## 1. 다른 네트워크 멀티클러스터 → 동일 네트워크로 범위 축소

### 문제
서로 다른 네트워크 대역의 클러스터를 East-West Gateway로 연결하려 했는데 **TLS Handshake가 계속 실패**.

### 원인
1. Istio가 클러스터 간 통신에 mTLS를 적용 — 양쪽이 같은 Root CA를 신뢰해야 함
2. 각 클러스터에 자동 생성된 Istio 자체 서명 인증서는 **클러스터별로 Root CA가 다름**
3. 상대 클러스터 Root CA를 신뢰하지 않으니 East-West Gateway 통과 시 TLS 단절

### 해결안 vs 결정
- **이상적 해결**: 공통 Root CA → 각 클러스터의 Intermediate CA (cert-manager + Vault, 또는 클라우드 매니지드 CA)
- **현실적 결정**: 학습 시간 안에 PKI 설계와 갱신 자동화까지 짤 수 없다고 판단 → **동일 네트워크 멀티클러스터로 범위 축소**하고 실패 시도를 회고로 명시

### 배운 점
멀티클러스터 가용성의 진짜 비용은 **PKI 운영 비용**. 인증서 만료 한 번에 클러스터 간 통신이 단절됨. 실무에서 클라우드 매니지드 CA가 표준인 이유를 이때 처음 체감.

---

## 2. 서킷브레이커 임계값 5/10s/30s/50%의 트레이드오프

각 값마다 *왜 그 숫자인가*를 매니페스트 옆에 적어두지 않으면 다음 사람이 이유 없이 바꾼다.

| 파라미터 | 값 | 다른 값으로 했을 때의 위험 |
|---|---|---|
| `consecutiveGatewayErrors` | 5 | 1~2: 일시적 네트워크 지연을 장애로 오인 (오탐 ↑) |
| `interval` | 10s | 1~2s: 회복 중인 서비스를 반복 격리 → 정상화 지연 |
| `baseEjectionTime` | 30s | 5s: 자동 복구 시간 부족, 격리-해제 진동 |
| `maxEjectionPercent` | 50% | 100: 전부 격리 시 서비스 완전 다운 |

### 실측 결과

```bash
# 적용 전 — 500 응답 누적
{"status":500,"error":"Internal Server Error","message":"status 502 reading Remote"}
{"status":500,...}
{"status":500,...}

# 적용 후 — 연속 5회 감지 → 장애 Pod 30초 격리 → 정상 Pod로 라우팅
{"name":"Pam Parry","photo":".../placeholder.png"}
{"name":"Pam Parry","photo":".../placeholder.png"}
```

---

## 3. Spinnaker → ArgoCD

### 문제
Halyard(Spinnaker 설치 관리 도구)가 8GB 학습 VM에서 OOM 반복.

### 원인
Spinnaker는 Deck/Gate/Orca/Clouddriver 등 10여 개 컴포넌트로 구성된 마이크로서비스. 권장 메모리 16GB+.

### 전환
ArgoCD로 갈아탐 — 단순히 가벼워서가 아니라 **GitOps라는 철학이 K8s 선언적 관리와 정렬**되기 때문.

| 항목 | Spinnaker | ArgoCD |
|---|---|---|
| 최소 리소스 | ~16GB | ~2GB |
| 배포 방식 | 명령형 파이프라인 | 선언적 GitOps |
| 롤백 | UI에서 파이프라인 재실행 | `git revert` |
| 단일 진실의 원천 | 파이프라인 상태 | Git |

### 배운 점
Git이 단일 진실의 원천이 되는 운영 모델로 바뀐 점이 도구 비교보다 본질적인 변화.
