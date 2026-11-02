### Task

- Create a new StorageClass named low-latency that uses the existing provisioner rancher.io/local-path.
- Set the VolumeBindingMode to WaitForFirstConsumer.
  (Mandatory or the score will be reduced.)
- Make the newly created StorageClass (low-latency) the default StorageClass in the cluster.
- Do NOT modify any existing Deployments or PersistentVolumeClaims. (If modified, the score will be reduced)

---

### Solution

1. StorageClass 생성

   kubectl apply -f - <<EOF
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
   name: low-latency
   provisioner: rancher.io/local-path
   volumeBindingMode: WaitForFirstConsumer
   EOF

2. 기존 default 확인

- `k get sc`

3. low-latency를 default로 지정

- `k patch sc low-latency -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`

4. 기존 default 해제

- `k patch sc <기존-sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'`

5. 확인

- `k get sc` → low-latency만 `(default)` 표시

---

### 참고

#### default StorageClass

`storageClassName`을 지정하지 않은 PVC에 자동 적용되는 StorageClass.

- annotation `storageclass.kubernetes.io/is-default-class: "true"`로 지정
- default가 여러 개면 가장 최근 생성된 것이 적용됨 → 의도와 다를 수 있으니 기존 default는 해제
- 이미 생성된 PVC에는 영향 없음 → Deployment, PVC 수정 금지 조건과 충돌 없음

#### volumeBindingMode

- `Immediate` (기본값): PVC 생성 즉시 PV 바인딩
- `WaitForFirstConsumer`: PVC를 사용하는 Pod가 스케줄될 때 바인딩
  - local-path처럼 노드 로컬 디스크를 쓰는 경우 필수. Pod가 배치될 노드에 볼륨이 생겨야 하기 때문
  - Pod 생성 전까지 PVC는 `Pending` 상태가 정상

#### StorageClass는 cluster 범위 리소스

namespace 지정 불필요.
