## Alternatives to JSON Web Tokens (JWTs)


# 수많은 토큰중 왜 하필 JWT인가?

Date: 2025년 5월 12일
Status: In progress

## JWT 에 대한 주요 비판들

## 상세 브리핑 문서: JWT의 대안과 사용 시 주의사항 검토

### 개요

본 문서는 제공된 세 가지 소스("Alternatives to JSON Web Tokens (JWTs)", "No Way, JOSE! Javascript Object Signing and Encryption is a Bad Standard That Everyone Should Avoid", "Stop using JWT for sessions", "Stop using JWT for sessions, part 2: Why your solution doesn't work")를 분석하여 JWT(JSON Web Tokens)에 대한 비판적인 시각과 제안된 대안, 그리고 JWT의 부적절한 사용으로 인해 발생하는 문제점을 상세히 검토합니다. 특히 세션 관리 목적으로 JWT를 사용하는 것의 위험성을 강조하며, 더 안전하고 실용적인 접근 방식을 제시합니다.

### 주요 테마 및 핵심 아이디어/사실

### 1. JWT에 대한 비판 (Why the Hate on JWTs? / Why You Don't Want Javascript Object Signing and Encryption)

- **알고리즘 선택 (Ciphersuite Agility)의 문제점:** JWT 헤더의 alg 값은 토큰이 서명되거나 암호화된 방식을 지정하며, 이 값이 공격자에 의해 none(서명 없음), 약한 알고리즘 또는 심지어 RSA 공개 키를 사용한 대칭 서명으로 변경될 수 있는 취약점이 있습니다. 이는 특히 OAuth 및 OpenID Connect와 같은 프로토콜에서 추가적인 표준화가 이루어져 완화될 수 있지만, 무시할 수 없는 수많은 취약점을 야기했습니다.
    - *"Taking the JWT specification at face value, **this value could be changed to none** (no signature), to a weaker algorithm, or even to a symmetric signature using an RSA public key."*
    - *"This isn't just an implementation bug, this is the result of a failed standard that shouldn't be relied on for security. If you adhere to the standard, you **must** process and "understand" the header. You are explicitly forbidden, by the standard, to just disregard the header that an attacker provides."*
- **JWT의 오용 (Misuse of JWTs):** JWT는 상태 비저장(stateless) 세션 또는 저장소에 적합하거나 설계되지 않았습니다. 특히 SPA(Single Page Apps)에서 서버 측 코드나 OAuth, OpenID Connect와 같은 프로토콜을 회피하기 위해 사용되는 경우가 많습니다. 이러한 오용은 JWT 자체의 문제라기보다는 잘못된 설계 선택에서 비롯됩니다.
    - *"JWTs are not suitable or designed for stateless sessions or storage. This one seems to be prevalent in Single Page Apps (SPAs), and it strikes me as a way of avoiding server-side code or protocols such as OAuth and OpenID Connect."*
    - *"Don't use JWT for session management, as discussed in other articles"*
- **JWE(JSON Web Encryption)의 문제점:** JWE는 암호화에 사용되는 알고리즘 선택의 폭이 넓어 구현 오류의 여지가 많습니다. 특히 RSA와 PKCS #1v1.5 패딩은 패딩 오라클 공격에 취약하며, NIST 곡선을 사용하는 ECDH는 무효 곡선 공격(invalid-curve attacks)의 위험이 있습니다. 암호학 알고리즘 선택은 개발자가 아닌 전문 암호학자에 의해 이루어져야 합니다.
    - *"JSON Web Encryption is a Foot-Gun"*
    - *"Let's look at some of the **key encryption** choices in detail... Almost all of these options have one or more security issues."*
    - *"Cryptography algorithm selections should be made by professional cryptographers, not developers. Letting developers mix-and-match protocols and algorithms is the hallmark of error-prone cryptographic designs."*

### 2. JWT의 대안 (JWT Alternatives / A Better Solution than JOSE)

- **Fernet:** AES 128-CBC 암호화와 HMAC-SHA256 인증(encrypt-then-MAC)을 사용하는 사용자 지정 토큰 형식입니다. 대칭 암호화를 사용하므로 양 당사자가 암호화 및 복호화할 수 있는 내부 시스템에 적합합니다. 하지만 공개 키 암호화를 지원하지 않으며, 표준화되어 있지 않고, 사용 가능한 알고리즘이 제한적입니다. 2013년에 마지막으로 업데이트된 Fernet GitHub 사양과 활동이 적은 프로젝트 상태는 사용을 망설이게 합니다.
    - *"Fernet meets some JWT use cases. It uses symmetric encryption (a shared key), so that means it would **replace a JWE, where both parties can both encrypt and decrypt**."*
    - *"At the spec level, Fernet only supports AES-128-CBC, which is a little old hat now."*
    - *"Fernet itself is something of an unknown... The GitHub spec was created in 2013 and hasn’t really been modified since."*
- **Branca:** Fernet의 현대화된 버전으로 XChaCha20-Poly1305를 사용합니다. Base62 인코딩과 Unix epoch 타임스탬프를 사용한다는 차이점이 있습니다. Fernet과 마찬가지로 대칭 암호화를 사용하며 내부 시스템에 적합합니다. 최신 암호화 알고리즘을 사용하고 사양이 활발히 업데이트되고 있다는 장점이 있지만, 표준화되어 있지 않고 Base62 인코딩은 널리 사용되지 않습니다.
    - *"Branca is a modernized version of Fernet... uses XChaCha20-Poly1305 for authenticated encryption."*
    - *"Branca is definitely more up to date than Fernet, but it still falls into the same category of use cases, where it is only suitable for internal, high-trust systems."*
- **PASETO (Platform-Agnostic Security Tokens):** JWT 및 JOSE 사양에 대한 직접적인 대응으로 생성되었습니다. 알고리즘 선택의 유연성(ciphersuite agility)을 제거하고 버전 관리를 강제합니다. 버전과 용도(public/local)에 따라 특정 알고리즘(v1.local: AES-256-CTR + HMAC-SHA384, v2.local: XChaCha20-Poly1305, v1.public: RSASSA-PSS using SHA-384, v2.public: Ed25519)을 사용합니다. JWS와 JWE를 모두 대체하는 데 적합하지만, 대칭 키와 비대칭 키 간의 타입 혼동 방지 기능이 부족하고, JWT와 동일하게 라이브러리 구현에 의존해야 합니다. 표준화 노력은 있었으나 현재는 IETF 드래프트가 만료되어 공식적으로 채택되지 않았습니다.
    - *"PASETO was created in direct response to its author’s concerns with JWTs and the various JOSE specifications."*
    - *"Unlike Fernet and Branca, PASETO is **suitable to replace both JWS and JWE**."*
    - *"Versioning brings the idea of unambiguous cipher suites."*
    - *"PASETO fixes the concerns around JOSE’s ciphersuite agility, but, in my opinion, it doesn’t do itself any favors by phrasing itself as a replacement."*
    - *"Unfortunately, the draft has expired, it was never formally adopted by the working group, nor is anyone currently championing it."*
- **"Just sign some JSON" (일부 JSON에 서명만 하기):** 특정 서명 알고리즘이나 암호화 라이브러리를 직접 사용하여 JSON에 서명하는 접근 방식입니다. 추상화를 사용하지 않고 직접 구현하는 방식으로, JWT와는 다른 사용 사례입니다. 표준화가 되지 않고 오케스트레이션 문제가 발생할 수 있으며, JWT의 목적을 놓치는 접근 방식입니다.
    - *"The argument is to just do it yourself instead of using an abstraction that gives you the possibility of getting it wrong."*
    - *"To be as flippant as the Twitter arguments: if this seems like an alternative to JWTs to you, then you were probably using JWTs wrong."*

### 3. 세션 관리 목적으로 JWT 사용의 문제점 (Stop using JWT for sessions / Stop using JWT for sessions, part 2)

- **주장된 이점의 반박:** 세션 관리에 JWT를 사용하는 사람들이 주장하는 대부분의 이점(확장 용이성, 사용 편의성, 유연성, 보안성, 만료 기능, 쿠키 동의 불필요, CSRF 방지, 모바일 호환성, 쿠키 차단 사용자 지원)은 잘못되었거나 오해의 소지가 있습니다. 상태 저장(stateful) 세션 관리를 위한 기존 솔루션(Redis, 스티키 세션 등)은 대부분의 애플리케이션 요구 사항을 충족하며, JWT는 이러한 솔루션에 비해 실제적인 이점을 제공하지 않습니다.
    - *"This is a terrible, **terrible** idea, and in this post, I'll explain why."*
    - *"This is the only claim in the list that is **technically** somewhat true, but only if you are using **stateless** JWT tokens. The reality, however, is that almost nobody actually **needs** this kind of scalability"*
    - *"A lot of people think that JWT tokens are "more secure" because they use cryptography. While signed cookies **are** more secure than unsigned cookies, this is in no way unique to JWT..."*
    - *"Completely wrong. There's no such thing as a "cookie law" - the various laws concerning cookies actually cover **any** kind of persistent identifier that isn't strictly necessary for the functioning of the service."*
    - *"It doesn't, really... The **only** correct CSRF mitigation is a CSRF token. The session mechanism is not relevant here."*
- **단점:더 많은 공간 차지:** 특히 상태 비저장 JWT는 모든 세션 데이터를 포함하므로 토큰 크기가 커져 쿠키 또는 URL의 크기 제한을 초과할 수 있습니다.
    - *"They take up more space"*
- **더 낮은 보안성:** JWT를 Local Storage와 같은 쿠키 이외의 장소에 저장하면 XSS(Cross-Site Scripting) 공격에 취약해져 공격자가 토큰에 접근하고 탈취할 수 있게 됩니다. 쿠키는 이러한 공격에 대해 더 나은 보호 기능을 제공합니다.
    - *"They are **less** secure"*
    - *"Local storage, unlike cookies, doesn’t send the contents of your data store with every single request... any attacker supplied JavaScript that passes the Content Security Policy can access and exfiltrate it."*
- **개별 JWT 토큰 무효화 불가:** 상태 비저장 JWT는 서버에서 개별적으로 무효화할 수 없습니다. 토큰은 만료될 때까지 유효하며, 공격자의 세션을 차단하거나 비밀번호 변경 시 기존 세션을 무효화하는 등의 작업이 어렵습니다.
    - *"You cannot invalidate individual JWT tokens"*
    - *"By design, they will be valid until they expire, no matter what happens."*
- **데이터 부패 (stale):** 상태 비저장 토큰의 데이터는 최신 상태를 반영하지 못하고 '부패'할 수 있습니다. 이는 민감한 정보(예: 사용자 역할)에 대해 보안 문제를 야기할 수 있습니다.
    - *"Data goes stale"*
- **검증되지 않은 또는 존재하지 않는 구현:** 상태 저장 JWT는 기본적으로 일반 세션 쿠키와 유사하지만, 오랫동안 사용되어 보안이 검증된 기존 세션 관리 라이브러리(예: express-session)에 비해 구현이 덜 성숙하고 검증되지 않았을 수 있습니다.
    - *"Implementations are less battle-tested or non-existent"*
- **마이크로서비스 아키텍처에서의 세션 vs. 토큰:** 마이크로서비스 아키텍처에서도 세션 관리에 JWT를 사용하는 것은 잘못된 접근 방식입니다. 상태 비저장 서비스의 경우 짧은 수명의 일회성 토큰을 사용하고, 상태 저장 서비스의 경우 각 서비스별 세션을 사용해야 합니다. JWT를 세션 자체로 사용하는 것은 적절하지 않습니다.

### 4. JWT의 적절한 사용 사례 (So... what is JWT good for, then?)

- **일회성 권한 부여 토큰:** JWT는 두 당사자 간에 클레임(권한 부여 등)을 전달하기 위한 짧은 수명의 일회성 토큰으로 사용될 때 적합합니다. 예를 들어, 인증 서버에서 다운로드 서버에 대한 파일 다운로드 권한을 부여하는 데 사용될 수 있습니다. 이러한 경우 토큰은 단기간만 유효하며, 한 번 사용된 후 폐기됩니다.
    - *"the usecases where JWT is particularly effective are typically usecases where they are used as a **single-use authorization token**."*
    - *"When using JWT in this manner, there are a few specific properties: **The tokens are short-lived.** ... **The token is only expected to be used once.**"*

### 5. 향후 발전 방향 (How Can We Replace JWTs? / Versioned JWTs)

- **JWT 대체보다는 개선:** JWT의 문제를 완전히 대체하기보다는 PASETO나 Branca와 같은 대안에서 발견되는 기능을 JWT에 추가하여 개선하는 것이 더 현실적입니다.
    - *"**Instead of replacing JWTs, we need to build upon them with the features found in alternatives** such as PASETO and Branca."*
- **버전 관리된 JWT:** JOSE 워킹 그룹에서 제안된 것처럼, 알고리즘을 명시하는 대신 특정 알고리즘 집합을 지정하는 "v1.local", "v1.public" 등의 새로운 알고리즘을 RFC로 정의하고 다른 알고리즘을 금지하는 방식은 PASETO의 핵심 기능을 JWT에 통합하는 효과적인 방법입니다. 이는 가장 중요한 비판인 알고리즘 선택 문제를 해결할 수 있습니다.
    - *"You could write an RFC that proposes 4 new JOSE algorithms: “v1.local”, “v1.public”, “v2.local”, “v2.public” and marks all others (and associated headers) as prohibited. Wouldn’t that be pretty similar to the core of PASETO?"*
- **필수적인 클레임 유효성 검사:** PASETO와 Branca의 문제점 중 하나는 페이로드 유효성 검사 프로세스에 대한 사양 부족입니다. 발급자, 대상, 만료 등의 표준 클레임 유효성 검사를 필수로 만들어야 합니다.
    - *"Combining a **versioned JWT with mandatory claim validation** sounds like a robust solution to me, and I would love to see this idea explored further."*

### 결론

제공된 소스는 JWT가 특히 세션 관리 목적으로 사용될 때 심각한 보안 위험과 실용적인 문제를 내포하고 있음을 강력히 시사합니다. 알고리즘 선택의 유연성, 잘못된 사용 사례, 검증되지 않은 구현 등 다양한 단점이 지적됩니다. Fernet, Branca, PASETO와 같은 대안들이 제시되지만, 각기 다른 제약과 단점을 가지고 있습니다.

가장 중요한 메시지는 대부분의 웹 애플리케이션에서 JWT를 상태 비저장 세션 관리 메커니즘으로 사용하는 것은 피해야 한다는 것입니다. 기존의 상태 저장 세션 관리 솔루션은 훨씬 더 안전하고 실용적이며, 대부분의 확장 요구 사항을 충족합니다.

JWT의 적절한 사용 사례는 짧은 수명의 일회성 권한 부여 토큰으로 제한됩니다.

향후 JWT의 발전은 PASETO의 장점(버전 관리 및 알고리즘 고정)을 통합하고 필수적인 클레임 유효성 검사를 포함하는 방식으로 이루어져야 합니다. 하지만 현재로서는 세션 관리 목적으로 JWT를 사용하는 것보다 검증된 세션 관리 기술을 사용하는 것이 압도적으로 권장됩니다.


> https://www.scottbrady.io/jose/alternatives-to-jwts

> https://paragonie.com/blog/2017/03/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid

> http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/

> http://cryto.net/%7Ejoepie91/blog/2016/06/19/stop-using-jwt-for-sessions-part-2-why-your-solution-doesnt-work/
