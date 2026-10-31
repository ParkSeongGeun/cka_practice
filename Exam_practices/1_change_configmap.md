### Question

There is an existing deployment called nginx-static in the nginx-static
namespace. The deployment contains a ConfigMap named nginx-config that
supports TLSv1.3

Update the nginx-config ConfigMap to allow TLSv1.2 connections.
Re-create, restart, or scale resources as necessary.
By Using command to test the changes:

After verifying the changes, make the nginx-config ConfigMap immutable.

[candidate@cka0001]$ curl -k --tls-max 1.2 https://web.k8s.local:30007

---

### Solution

1. ConfigMap 수정: ssl_protocols TLSv1.2 TLSv1.3;

- `k edit cm -n nginx-static nginx-config`

2. 재시작

- `k rollout restart -n nginx-static deploy nginx-static`

3. 확인

- `k get -n nginx-static pod`

4. 테스트

- `curl -k --tls-max 1.2 https://web.k8s.local:30007`

5. immutable 설정

- `k patch cm -n nginx-static nginx-config -p '{"immutable":true}'`

---

### 참고

#### immutable

ConfigMap/Secret의 내용을 수정 불가로 고정하는 필드. `data`와 같은 레벨의 최상위 필드.

- `immutable: true` 설정 후 `data`, `binaryData` 수정 불가
- `false`로 되돌리기 불가 → 변경하려면 삭제 후 재생성
- 사용 이유
  - 실수나 무단 변경 방지
  - kubelet의 변경 감시(watch) 중단 → apiserver 부하 감소. ConfigMap이 많은 대규모 클러스터에서 효과

#### namespace 지정

`default`가 아닌 네임스페이스의 리소스는 매 명령에 `-n` 지정 필요.

- `-n` 생략 시 현재 context의 기본 namespace(보통 `default`)에서 조회 → `NotFound` 에러
- 이 문제는 `nginx-static` 네임스페이스 → 매 명령에 `-n nginx-static`

기본 namespace 변경 방법:

    k config set-context --current --namespace=nginx-static

- 이후 `-n` 없이 `nginx-static` 기준으로 동작
- 시험은 문제마다 namespace가 다름 → 되돌리지 않으면 다음 문제에서 엉뚱한 곳에 리소스 생성. 매 명령에 `-n` 명시하는 쪽이 안전
