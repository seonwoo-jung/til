# JWS 알고리즘 혼동(Algorithm Confusion) 공격: RS256 공개키가 HS256 비밀키로 둔갑하는 원리

> **Primary source:** RFC 7515 (JSON Web Signature) §2, §5.1–5.2, §10.6–10.7 / RFC 8725 (JWT Best Current Practices) §2.1, §3.1–3.2
> **Secondary:** RFC 7519 (JWT) / RFC 7518 (JWA)
> **Date:** 2026-09-07
> **Status:** draft
> 블로그: https://velog.io/@jungseonw00/rs256-hs256-algorithm-confusion

## 왜 봤나

"JWT는 서명이 붙어 있으니 위조가 불가능하다"는 말은 절반만 맞다. 서명 알고리즘을 검증자가 아니라 **토큰 자신이 선언**하게 설계되어 있다는 사실 — `alg` 헤더 — 이 왜 위험한 공격 표면이 되는지, 그리고 "공개키는 공개돼도 안전하다"는 RS256의 전제가 어떻게 무너지는지를 스펙 레벨에서 정리해 둘 필요가 있었다. 흔히 "라이브러리 버그"로 취급하고 넘어가지만, 실제로는 RFC가 명시적으로 경고하는 설계상의 함정이다.

## 핵심 한 문장

> JWS 검증은 서명 값 자체가 아니라 **"어떤 알고리즘으로 검증할지"를 결정하는 로직**에 신뢰의 뿌리를 두는데, 검증자가 그 결정을 공격자가 채울 수 있는 `alg` 헤더에 그대로 맡기면, 원래 비대칭(RS256) 검증에 쓰던 **공개키 바이트열이 대칭(HS256) 비밀키로 재해석**되어 공격자가 스스로 유효한 서명을 만들어낼 수 있다.

## 내부 동작

### 1. JWS의 구조와 서명 대상

JWS Compact Serialization은 `BASE64URL(Header) || '.' || BASE64URL(Payload) || '.' || BASE64URL(Signature)` 세 부분이다. 서명(또는 MAC)이 실제로 덮는 입력은 RFC 7515 §2가 "JWS Signing Input"으로 정의한 값 하나뿐이다.

```
JWS Signing Input = ASCII( BASE64URL(UTF8(Header)) || '.' || BASE64URL(Payload) )
```

여기서 중요한 점: **Header 자신도 서명 대상에 포함된다.** `alg` 필드가 Header 안에 있으므로, `alg`를 바꾸면 서명 대상 바이트열도 바뀐다 — 즉 공격자가 `alg`를 조작하면 "새로운 입력에 대해 새로운 서명"을 처음부터 다시 만들어야 하는 게 원칙이다. 문제는 검증 쪽 구현이 이 원칙을 어떻게 강제하느냐에 있다.

### 2. 검증 절차가 신뢰를 위임하는 지점

RFC 7515 §5.2는 검증 단계를 8단계로 정의한다. 그중 8단계가 핵심이다.

> "Validate the JWS Signature against the JWS Signing Input ... in the manner defined for the algorithm being used, **which MUST be accurately represented by the value of the "alg" ... Header Parameter**" (§5.2 step 8)

스펙은 "알고리즘은 `alg` 값이 정확히 나타내야 한다"고 서명 생성자에게 요구할 뿐, **검증자가 그 `alg` 값을 무조건 신뢰해도 된다고 말하지 않는다.** 오히려 §10.6(Algorithm Validation)과 §10.7(Algorithm Protection)에서 정반대를 경고한다.

- §10.6: 서명 값 내부에 알고리즘 정보가 인코딩되는 경우, 구현체는 "그 인코딩된 알고리즘이 `alg` 헤더와 일치하는지" 검증해야 한다(MUST) — 안 하면 공격자가 강한 해시를 쓴 척하면서 실제로는 약한 해시로 서명값을 위조할 수 있다.
- §10.7 (Algorithm Protection / substitution attack): "검증자가 여러 알고리즘을 동시에 지원"하고 "공격자가 다른 알고리즘으로 같은 서명값을 만족하는 페이로드를 찾을 수 있으면" 치환 공격이 성립한다고 정의한다. 대응책 중 하나로 "`alg`를 반드시 JWS Protected Header에 실어(서명 대상에 포함시켜) 위조 시 서명 자체가 깨지게 하라"고 명시한다.

즉 스펙 저자들은 이미 "검증기가 alg 헤더를 곧이곧대로 믿고 알고리즘을 선택하는 구현"을 위험 패턴으로 못 박아 두었다.

### 3. RS256 → HS256 혼동이 실제로 일어나는 경로

RFC 8725 §2.1(Weak Signatures and Insufficient Signature Validation)은 실제로 보고된 두 가지 사례를 명시한다.

1. `alg`를 `"none"`으로 바꾸면 일부 라이브러리가 서명 검증 자체를 건너뛰고 "유효"로 판정.
2. **`"RS256"`을 `"HS256"`으로 바꾸면, 일부 라이브러리가 RSA 공개키를 HMAC 공유 비밀(shared secret)로 그대로 사용해 HMAC-SHA256 검증을 시도**한다(CVE-2015-9235로 알려짐).

이 두 번째 경로를 단계별로 풀면:

```
정상 흐름 (RS256, 비대칭)
  서버: privateKey로 서명 생성 → 클라이언트/공격자에게 공개키(pubKey) 배포(JWKS 등, "공개"라 안전 가정)
  검증자: verify(token, pubKey)  // 내부적으로 alg 헤더를 보고 RSA-SHA256 검증기로 분기

공격 흐름 (혼동)
  1) 공격자가 pubKey 값(바이트/PEM 문자열)을 그대로 손에 넣는다 (원래 공개 정보라 문제 없다고 여겨짐)
  2) 공격자가 원하는 Header={"alg":"HS256",...}, Payload={"sub":"admin",...} 를 만든다
  3) 공격자가 HMAC-SHA256(SigningInput, key=pubKey바이트) 를 "직접" 계산해 Signature로 붙인다
  4) 검증자가 verify(token, pubKey) 호출
     → alg 헤더를 읽고 "HS256"이니 HMAC 검증기로 분기 (신뢰 위임 지점)
     → 같은 pubKey를 이번엔 "HMAC 비밀키"로 사용해 HMAC 계산
     → 공격자가 3)에서 이미 같은 키로 같은 값을 계산해 뒀으므로 일치 → 통과
```

핵심은 **키 자체는 그대로인데, 그 키를 "비대칭 검증용 공개 정보"로 쓰느냐 "대칭 MAC용 비밀 정보"로 쓰느냐를 결정하는 것이 오직 공격자가 통제하는 `alg` 필드라는 점**이다. RSA 공개키는 정의상 비밀이 아니므로, 그것이 "비밀"로 취급되는 순간 전체 신뢰 모델이 무너진다.

### 4. 스펙이 제시하는 차단 지점

RFC 8725 §3.1(Perform Algorithm Verification)은 이렇게 요구한다.

> "Libraries MUST enable the caller to specify a supported set of algorithms and MUST NOT use any other algorithms ... each key MUST be used with exactly one algorithm, and this MUST be checked when the cryptographic operation is performed."

즉 안전한 설계는 "토큰의 `alg`를 보고 알고리즘을 고르는" 것이 아니라, **검증자가 미리 정한 허용 알고리즘 목록(그리고 키-알고리즘 1:1 바인딩)을 기준으로, 토큰의 `alg`가 그 목록·바인딩과 일치하는지만 확인**하는 것이다. 알고리즘 선택의 주도권이 토큰(공격자 통제 영역)에서 검증자 설정(서버 통제 영역)으로 넘어와야 한다.

```java
// 위험한 패턴 (개념적 예시): alg 헤더가 곧 검증 알고리즘을 결정
Object claims = genericJwtLibrary.parse(token, key); // 내부에서 header.alg로 알고리즘 자동 선택

// 안전한 패턴: 허용 알고리즘을 서버가 고정하고, 키도 알고리즘별로 분리
Object claims = jwtLibrary.parseBuilder()
        .requireAlgorithm("RS256")      // 허용 목록을 서버가 소유
        .verifyWith(rsaPublicKey)       // 이 키는 오직 RS256 검증에만 바인딩
        .parse(token);
```

## 검증

이 스펙 조항은 코드 실행 없이도 직접 손으로 추적할 수 있다.

1. RFC 7515 §5.1의 8단계 서명 생성 절차와 §5.2의 8단계 검증 절차를 나란히 놓고, "어느 단계에서 `alg` 값이 검증기 선택에 쓰이는지"를 표시해 보면 8단계(검증)가 유일하게 그 값을 소비하는 지점임을 확인할 수 있다.
2. 아무 JWT 디버거(예: 토큰 구조를 디코딩해 보여주는 도구)에 RS256으로 발급된 토큰을 붙여넣고, Header의 `"alg":"RS256"`을 `"HS256"`으로 바꿔 보면 Payload는 그대로인데 서명 검증에 필요한 "키 해석 방식"만 바뀐다는 것을 시각적으로 확인할 수 있다 — 실제 위조 서명을 만들려면 별도로 HMAC 계산이 필요하지만, 구조상 "같은 값이 다른 용도로 재해석"되는 지점은 이 실습만으로도 드러난다.
3. RFC 8725 §3.1의 "each key MUST be used with exactly one algorithm"이라는 요구를, 사용 중인 JWT 라이브러리의 검증 API가 "허용 알고리즘 목록"을 필수 인자로 받는지, 아니면 토큰의 `alg`를 그대로 신뢰하는지로 대입해 점검할 수 있다.

## 잘못 알고 있던 것

- **"JWT는 서명이 있으니 위조 불가능하다"** — 서명의 존재 여부가 아니라 **"검증자가 알고리즘을 어떻게 결정하는가"**가 안전성의 핵심이다. 알고리즘 선택권이 토큰 쪽(공격자 통제)에 있으면 서명은 있어도 무력화될 수 있다.
- **"공개키는 유출돼도 상관없다"** — RS256 단독으로는 맞는 말이지만, 검증자가 알고리즘 혼동에 취약하면 "공개"라는 성질 자체가 무기가 된다. 공개키가 광범위하게 배포되어 있을수록(JWKS 엔드포인트 등) 공격자가 그 값을 손에 넣기는 오히려 더 쉽다.

## 더 파고들 만한 것

- JWKS의 `kid`(Key ID) 헤더를 검증자가 그대로 파일 경로·URL로 사용할 때 생기는 주입 공격 (RFC 8725 §2.9 계열).
- JWE(암호화)에서 `alg`/`enc` 조합을 조작하는 유사한 혼동 공격 패턴.

## 참고

- RFC 7515 JSON Web Signature (JWS) — §2 Terminology, §5.1–5.2, §10.6–10.7
- RFC 8725 JSON Web Token Best Current Practices — §2.1, §3.1–3.2
- RFC 7519 JSON Web Token (JWT), RFC 7518 JSON Web Algorithms (JWA)
- CVE-2015-9235 (RFC 8725 §2.1에서 참조하는 RS256/HS256 혼동 취약점 사례)

---

<!-- velog 글로 발전 후 -->
**velog 글:** {link}
