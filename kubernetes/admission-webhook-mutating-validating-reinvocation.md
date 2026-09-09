# Kubernetes Admission Webhook: Mutating→Validating 순서와 reinvocationPolicy 재호출 메커니즘

> **Primary source:** Kubernetes 공식 문서 — Reference / Access Authn Authz / Extensible Admission Controllers ("Dynamic Admission Control")
> **Secondary:** 없음
> **Date:** 2026-09-09
> **Status:** draft
> 블로그: https://velog.io/@jungseonw00/admission-webhook-reinvocation-idempotency

## 왜 봤나

- admission webhook은 "요청을 가로채 검사/수정하는 훅" 정도로 뭉뚱그려 이해하기 쉽지만, 실제로는 **mutating과 validating이 완전히 분리된 두 단계**로 실행되고, 그 사이에 스키마 검증이 끼며, mutating 쪽은 한 번으로 끝나지 않고 **재호출(reinvocation)** 될 수 있다는 점이 잘 드러나지 않는다.
- 특히 "mutating webhook이 파드에 사이드카 컨테이너를 추가하면, 그 컨테이너에 대한 기본값 설정(예: imagePullPolicy)은 누가 채워주는가?"라는 질문에 답하려면 재호출 정책의 존재 이유(GitHub issue 64333)까지 봐야 한다.

## 핵심 한 문장

> Mutating admission은 "모든 mutating plugin(내장+webhook)이 더 이상 객체를 바꾸지 않을 때까지" 수렴적으로 반복 실행된 뒤에야 확정되고, validating admission은 그 확정된 최종 객체만을 대상으로 정확히 한 번 실행된다.

## 내부 동작

### 1. 전체 admission 파이프라인 순서

```
요청 도달
  → 인증(Authentication) → 인가(Authorization)
  → [Mutating 단계]
       내장 mutating admission plugin들 (순서 고정)
       ↕ (webhook이 객체를 바꾸면 내장 plugin 재실행)
       MutatingWebhookConfiguration들의 webhook (이름순/설정순, reinvocation 가능)
  → 객체 스키마(OpenAPI) 검증
  → [Validating 단계]
       내장 validating admission plugin들
       ValidatingWebhookConfiguration들의 webhook (재호출 없음, 정확히 1회)
  → etcd에 persist
```

문서에 명시된 대로 mutating webhook이 먼저 호출되어 들어오는 객체를 바꿀 수 있고, **모든 객체 수정이 끝난 뒤에, 그리고 API 서버가 그 객체를 검증한 뒤에** validating webhook이 호출된다. 즉 validating webhook은 "최종 상태를 보장받고 싶다면 반드시 validating을 써야 한다"는 문서의 권고가 성립하는 이유가 바로 이 순서 자체다 — mutating 단계에서는 뒤에 실행되는 다른 mutating webhook이 방금 만든 필드를 또 바꿀 수 있기 때문에, mutating webhook 스스로는 "내가 마지막으로 본 상태가 최종 상태"라고 가정할 수 없다.

### 2. reinvocationPolicy — 왜 mutating만 "재호출"이 필요한가

단일 pass로 여러 mutating plugin을 한 번씩만 돌리는 방식은 다음 시나리오에서 깨진다: 어떤 mutating webhook이 파드에 컨테이너를 새로 추가했는데, 그 이전에 이미 실행되고 지나간 "모든 컨테이너에 imagePullPolicy를 기본값으로 채우는" 내장 plugin은 새로 추가된 컨테이너를 보지 못한 채 끝나버린다. 이 문제(issue 64333)를 풀기 위한 두 가지 장치:

- **내장 mutating plugin은 무조건 재실행된다** — 어떤 mutating webhook이든 객체를 수정하면, API 서버는 내장 plugin들을 다시 돈다.
- **mutating webhook은 옵션이다** — `reinvocationPolicy: Never`(기본값)면 이번 admission 평가에서 절대 두 번 호출되지 않는다. `IfNeeded`면, 자신이 호출된 이후 다른 admission plugin이 객체를 추가로 수정했을 경우 재호출될 "수" 있다.

`IfNeeded`를 선택해도 다음 세 가지는 보장되지 않는다는 점이 문서에 명시적으로 나온다:
- 추가 호출 횟수가 정확히 1번이라는 보장 없음.
- 재호출 결과 또 다른 수정이 생겨도, 그에 대해 다시 호출된다는 보장 없음(무한 수렴을 강제하지 않음).
- `IfNeeded`를 쓰는 webhook들은 호출 횟수를 최소화하는 방향으로 **순서 자체가 재배치될 수 있음**.

즉 `IfNeeded`는 "네가 최신 상태를 보고 싶다면 요청은 하겠지만, 몇 번을, 어떤 순서로 줄지는 API 서버 재량"이라는 계약에 가깝다. 그래서 문서는 **reinvocation을 쓰는 mutating webhook은 반드시 멱등(idempotent)해야 한다** — 이미 자신이 적용한 패치가 들어있는 객체를 다시 받아도 같은 패치를 중복 적용하지 않고 안전하게 처리할 수 있어야 한다 — 고 강조한다.

이 재호출 흔적은 실제로 audit 로그에 라운드 단위로 남는다. Metadata 레벨 이상 audit에서 `mutation.webhook.admission.k8s.io/round_{round}_index_{order}` 키로, 어떤 webhook이 몇 번째 라운드·몇 번째 순서로 불렸고 실제로 객체를 바꿨는지(`"mutated": true/false`)가 JSON으로 기록된다. Request 레벨 이상이면 `patch.webhook.admission.k8s.io/round_{round}_index_{order}`에 실제 적용된 JSON Patch까지 남는다.

### 3. matchConditions — 네트워크 호출 이전의 CEL 필터

`rules`/`objectSelector`/`namespaceSelector`만으로 부족할 때, webhook 정의에 최대 64개의 CEL(`matchConditions`)을 걸 수 있다. 모든 조건이 true여야 실제로 webhook에 HTTP 요청이 나간다. 흥미로운 점은 **에러 처리 순서**다:

- 조건 평가 중 에러가 나도, **어떤 조건이든 false로 평가되면(다른 에러 여부와 무관하게) webhook은 스킵된다.**
- 그 외의 경우 `failurePolicy: Fail`이면 webhook을 부르지 않고 즉시 요청을 거부하고, `Ignore`면 webhook을 건너뛰고 요청을 통과시킨다.

이 설계 덕분에 예컨대 `authorizer.group(...).check("breakglass").allowed()` 같은 CEL 식으로 "이 사용자는 이 webhook을 우회할 권한이 있는가"를 **실제 webhook 서버를 왕복하지 않고** 판정할 수 있다 — RBAC 체크 자체는 비싸므로, 두 번째 webhook으로 분리해 RBAC 관련 리소스에만 걸리게 하는 패턴이 문서 예시에 그대로 나온다.

### 4. 응답 프로토콜 — allowed/patch/warnings

Webhook은 `AdmissionReview`를 그대로 돌려주며 `response.uid`, `response.allowed`가 필수다. Mutating webhook이 객체를 바꾸려면 `patchType: JSONPatch` + base64 인코딩된 RFC 6902 JSON Patch 배열을 `patch` 필드에 담아 돌려준다 — **패치를 실제로 적용하는 주체는 webhook이 아니라 API 서버**다. 거부 시 `status.code`/`status.message`로 사용자에게 보여줄 HTTP 코드·메시지를 지정할 수 있고, `allowed` 여부와 무관하게 `warnings` 배열(항목당 사실상 256자에서 잘릴 수 있고 전체 4096자 넘으면 이후 경고는 버려짐, `"Warning:"` 접두사 넣지 말 것)로 HTTP `Warning` 헤더 경고를 클라이언트에 전달할 수 있다.

### 5. failurePolicy가 실제로 커버하는 범위

`failurePolicy: Fail`(기본값)/`Ignore`는 다음 "에러"들에만 적용된다: 네트워크 오류·타임아웃·연결 실패, non-2xx/malformed 응답, API 서버가 요청을 직렬화하거나 내부 HTTP 클라이언트를 만드는 데 실패한 경우, (mutating 한정) 디코딩 불가능하거나 지원하지 않는 patch 타입. 반대로 **webhook이 정상적으로 응답했고 그 응답이 `allowed: false`인 명시적 거부**라면, `failurePolicy` 설정과 무관하게 그 거부는 항상 유효하다 — `failurePolicy`는 "webhook에 도달/응답받는 과정 자체의 실패"를 다루는 것이지 "webhook의 판단"을 무력화하는 스위치가 아니다.

## 검증

문서에 나온 JSON Patch 예시(`[{"op": "add", "path": "/spec/replicas", "value": 3}]` → base64)는 로컬에서 그대로 재현해 값이 일치함을 확인했다:

```bash
echo -n '[{"op": "add", "path": "/spec/replicas", "value": 3}]' | base64
# W3sib3AiOiAiYWRkIiwgInBhdGgiOiAiL3NwZWMvcmVwbGljYXMiLCAidmFsdWUiOiAzfV0=
```

reinvocation과 mutating webhook 순서는 실제 클러스터에서 다음으로 관찰할 수 있다: audit policy를 `level: Metadata` 이상으로 설정한 API 서버에 mutating webhook 2개 이상을 등록하고(하나는 `reinvocationPolicy: IfNeeded`), 한쪽이 다른 쪽이 추가한 필드를 바꾸도록 구성한 뒤 `kubectl apply`를 실행한다. audit 로그에서 `mutation.webhook.admission.k8s.io/round_0_index_*`와 `round_1_index_*` 키가 함께 나타나면 재호출이 실제로 일어난 것이고, 각 항목의 `"mutated"` 값으로 어떤 라운드에서 실제 패치가 적용됐는지 확인할 수 있다.

## 잘못 알고 있던 것

- **"mutating이든 validating이든 admission webhook은 그냥 한 뭉치로, 등록한 순서대로 한 번씩 불린다"** — 아니다. mutating 단계는 "더 이상 아무도 객체를 바꾸지 않을 때까지"를 향한 반복(내장 plugin은 강제 재실행, webhook은 `reinvocationPolicy` 여부에 따라 선택적 재실행) 구조이고, validating은 그 수렴된 최종 객체에 대해 정확히 한 번만 실행된다. "부수효과(side-effect)가 있는 로직은 validating에 둬라"는 권고는 이 비대칭 때문에 나온다.
- **"`reinvocationPolicy: IfNeeded`면 내가 항상 최신 상태를 두 번째 호출에서 받는다"** — 아니다. 추가 호출이 정확히 1회라는 보장도, 그 이후 또 바뀐 상태를 다시 받는다는 보장도 없다. 순서 자체가 재배치될 수도 있다. 그래서 이 옵션을 쓰는 webhook은 애초에 멱등하게 짜야 하고, "마지막 상태를 반드시 검증해야 한다"는 요구가 있다면 validating webhook으로 옮겨야 한다.
- **"`failurePolicy: Ignore`로 설정해두면 이 webhook이 요청을 막는 일은 절대 없다"** — 절반만 맞다. `Ignore`는 webhook을 부르는 과정 자체가 실패했을 때(타임아웃, 연결 실패 등)만 관대하다. webhook이 정상 응답하며 명시적으로 `allowed: false`를 내려주면, `failurePolicy`와 무관하게 그 거부는 그대로 유효하다.
- **"와일드카드 rule을 걸어두면 TokenReview·SubjectAccessReview 같은 인증/인가용 가상 리소스까지 다 가로챌 수 있다"** — Kubernetes 1.37부터(베타, 기본 활성화) 이런 비영속 가상 리소스는 webhook rule이 명시적으로 매치해도 제외된다. 그 이유 자체가 흥미로운데, 이런 리소스를 가로채는 webhook이 실패하면 클러스터가 스스로의 인증/인가 경로를 authorize하지 못해 락아웃될 수 있기 때문이다. 다만 `*` 와일드카드로만 매치되는 경우는 의도가 모호하다고 보고 경고 대상에서 제외한다.

## 더 파고들 만한 것

- `ValidatingAdmissionPolicy`/`MutatingAdmissionPolicy` — 별도 webhook 서버 없이 CEL만으로 admission 로직을 API 서버 내부에서 직접 평가하는 최신 대안. webhook 왕복 지연·가용성 의존을 없앤다는 점에서 이 노트의 reinvocation/timeout 문제 자체를 우회하는 접근이라 비교해볼 가치가 있다.
- kube-apiserver의 내장 mutating admission plugin 목록과 각 plugin이 정확히 "재실행 대상"으로 어떻게 식별되는지(플러그인 체인 구현 관점)는 이 문서 범위 밖이라 소스 레벨로 더 들어가 볼 만하다.

## 참고

- Kubernetes 공식 문서: Reference / Access Authn Authz / Extensible Admission Controllers (Dynamic Admission Control)

---

<!-- velog 글로 발전 후 -->
**velog 글:** {link}
