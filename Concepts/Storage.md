## 스토리지 구성 방법

1. **워커 노드를 저장공간으로 사용**: 노드 디스크(hostPath 등)
2. **기존 스토리지 솔루션 연결**: 외부 NFS, Ceph, MinIO 등
3. **클라우드 스토리지 서비스 사용**
4. **볼륨 솔루션 전용 노드 구성**: Longhorn, NFS, MinIO 등을 클러스터 안에 설치

## 스토리지 유형별 유즈 케이스

| 유형   | 용도                 | 배포 방식                                           | 접근 모드                       |
| ------ | -------------------- | --------------------------------------------------- | ------------------------------- |
| Block  | DB (빠른 성능)       | StatefulSet, Pod마다 별도 볼륨                      | ReadWriteOnce, ReadWriteOncePod |
| File   | 여러 Pod 데이터 공유 | Deployment, 하나의 볼륨 공유                        | ReadOnlyMany, ReadWriteMany     |
| Object | 데이터 확장          | Deployment, 앱이 Object 서버에 직접 연결 (POST/GET) | 해당 없음 (PVC 미사용)          |

## File Storage 리소스 생성 방법

**공통**: Pod가 `volumes.persistentVolumeClaim`으로 PVC 참조 → `volumeMounts`로 컨테이너 경로(`/var/lib/share`)에 마운트

**1. 직접 생성 (Static)**

- 순서: NFS 준비 → PV 생성 → PVC 생성 → Pod 연결
- PVC가 요청한 accessModes, 용량을 만족하는 PV와 바인딩
- 특정 PV 지정 방법: PVC의 `volumeName` 또는 `selector`(PV label 매칭)

**2. 동적 프로비저닝 (Dynamic)**

- 순서: StorageClass 생성 → PVC 생성 → PV 자동 생성 → Pod 연결
- PVC에 `storageClassName`만 지정하면 provisioner가 PV를 자동 생성
- StorageClass는 볼륨 솔루션별로 1~2개 정도 생성 (볼륨마다 만들지 않음)

**StorageClass 주요 필드**

- `provisioner`: 볼륨 생성 주체 (예: `k8s-sigs.io/nfs`)
- `reclaimPolicy`: PVC 삭제 시 PV 처리 (Retain / Delete)
- `volumeBindingMode`: `WaitForFirstConsumer`면 Pod 생성 시점에 바인딩
- `allowVolumeExpansion`: 볼륨 확장 허용 여부
- `parameters`: 솔루션별 설정 (NFS server, path)
