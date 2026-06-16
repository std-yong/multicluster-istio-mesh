# Remote Secret 교환

각 클러스터의 Istiod가 상대 클러스터의 서비스 엔드포인트를 *같은 mesh의 일부*로 인식하게 만드는 단계.

```bash
# Master → Slave
istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 \
  | kubectl apply -f - --context="${CTX_CLUSTER2}"

# Slave → Master
istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 \
  | kubectl apply -f - --context="${CTX_CLUSTER1}"
```

이 단계가 끝나면 Cluster1의 서비스가 Cluster2의 서비스를 마치 로컬처럼 호출 가능.

## 검증

```bash
istioctl proxy-status --context="${CTX_CLUSTER1}"
# 양쪽 클러스터 Envoy 모두 SYNCED 떠야 함
```
