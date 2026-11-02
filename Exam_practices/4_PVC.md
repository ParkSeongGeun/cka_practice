### Task

A user accidentally deleted the MariaDB Deployment in the mariadb namespace, which was configured with persistent storage.

Your responsibility is re-establish the Deployment while ensuring data is preserved by reusing the available PersistentVolume.

- A PersistentVolume already exists and is retained for reuse (only one PV exists).
- Create a PersistentVolumeClaim (PVC) named mariadb in the mariadb namespace with the following spec:
- Access mode: ReadWriteOnce
- Storage: 250Mi
  - Edit the MariaDB Deploy file located at ~/mariadb-deploy.yaml
    to use the PVC created in the previous step.
  - Apply the updated Deployment file to the cluster.
  - Ensure the MariaDB Deployment is running and Stable

---

### Solution

1. PV 확인: STORAGECLASS 값 확인 (local-path)

- `k get pv`

2. PVC 생성

   k apply -f - <<EOF
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
   name: mariadb
   namespace: mariadb
   spec:
   accessModes: - ReadWriteOnce
   resources:
   requests:
   storage: 250Mi
   storageClassName: local-path
   EOF

3. 바인딩 확인: STATUS Bound, VOLUME mariadb-pv

- `k get pvc -n mariadb`

4. Deployment 파일 수정: `claimName: mariadb`

- `vi ~/mariadb-deploy.yaml`

5. 적용

- `k apply -f ~/mariadb-deploy.yaml`

6. 확인

- `k get deploy,pod -n mariadb`

---

### 참고

#### storageClassName 지정 이유

PVC는 storageClassName이 같은 PV와만 바인딩됨.

- 생략 시 default StorageClass가 자동 적용 → 기존 PV(local-path)와 불일치 → 기존 PV 대신 새 PV가 생기거나 Pending
- 기존 PV 재사용이 목적이므로 PV의 StorageClass와 동일하게 지정

#### Retain 정책

PVC가 삭제돼도 PV와 데이터 보존.

- PVC 삭제 후 PV 상태는 `Released` → 새 PVC와 바인딩 불가
- 재사용하려면 PV의 `claimRef` 제거 → `Available`로 전환 필요

      k patch pv mariadb-pv -p '{"spec":{"claimRef":null}}'

- 이 문제는 PV가 이미 `Available` 상태라 불필요

#### apply 경로

`-f`에는 실제 파일 경로 지정. 문제에 주어진 경로(`~/mariadb-deploy.yaml`) 그대로 사용.
