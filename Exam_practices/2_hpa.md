### Task

Create a new HorizontalPodAutoscaler (HPA) named apache-server in the
autoscale namespace. This HPA must target the existing Deployment called
apache-server in the autoscale namespace.

- Set the HPA to target for 50% CPU usage per pod.
- Configure hpa to have at min 1 Pod and no more than 4 Pods[max].
- Also, we have to set the downscale stabilization window to 30 seconds.

---

### Solution

1. HPA 생성: CPU 50%, min 1, max 4

- `k autoscale -n autoscale deploy apache-server --cpu=50% --min=1 --max=4`

2. 다운스케일 안정화 윈도우 설정: `spec` 하위에 `behavior` 추가

- `k edit hpa -n autoscale apache-server`

      spec:
        behavior:
          scaleDown:
            stabilizationWindowSeconds: 30

3. 확인

- `k get hpa -n autoscale`
- `k get hpa -n autoscale apache-server -o yaml | grep -A2 scaleDown`

---

### 참고

#### HPA (HorizontalPodAutoscaler)

Pod 부하를 기준으로 Deployment의 replicas를 자동 조절하는 컨트롤러.

- 평균 CPU 사용률이 목표(50%) 초과 → Pod 증가 (최대 4)
- 목표 미만 → Pod 감소 (최소 1)
- CPU 사용률은 Pod의 CPU **requests** 대비로 계산 → 대상 컨테이너에 `resources.requests.cpu`가 없으면 계산 불가
- 메트릭 수집에 metrics-server 필요

#### stabilizationWindowSeconds

스케일 결정 전 과거 추천값을 참고하는 기간.

- 부하가 잠깐 줄었다고 바로 축소하면 확장/축소가 반복됨(flapping) → 이를 방지
- `scaleDown` 기본값 300초. 이 문제는 30초로 단축
- `kubectl autoscale` 명령에 해당 옵션이 없음 → 생성 후 `edit` 또는 `patch`로 추가

patch 방식:

    k patch hpa -n autoscale apache-server -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":30}}}}'
