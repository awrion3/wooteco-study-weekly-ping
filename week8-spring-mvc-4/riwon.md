## JWT 개요

JWT(Json Web Token)은 사용자 인증 정보를 JSON 형식으로 안전하게 담아 **클라이언트와 서버 간 주고받는 토큰**입니다.<br>
서버가 로그인된 사용자에게 토큰을 발급하면, 클라이언트는 이후 요청 시 토큰을 함께 보내 인증을 수행합니다.<br>
JWT는 자체적으로 인증 정보를 포함하므로, 서버가 별도 세션 저장소를 두지 않아도 되는 **Stateless 인증 방식**입니다.<br>
JWT는 헤더와 페이로드, 시그니처로 구성되는데, 이때 각 부분은 점(`.`)으로 구분되며 Base64Url로 인코딩됩니다.

```
헤더(Header).페이로드(Payload).시그니처(Signature)
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJ1c2VySWQiLCJyb2xlIjoiUk9MRV9VU0VSIn0.
AbCDefGhIjKlMnOpQrStUvWxYz1234567890
```

## JWT 각 부분 상세 설명

### 1. 헤더 (Header)

헤더는 일반적으로 두 부분으로 구성됩니다:
- `typ`: 토큰의 유형 (JWT)
- `alg`: 사용된 해싱 알고리즘 (예: HMAC SHA256, RSA 등)

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
이 JSON을 Base64Url로 인코딩하면 JWT의 첫 부분이 완성됩니다.

### 2. 페이로드 (Payload)

페이로드에는 클레임(claim)이라 불리는 정보가 포함됩니다. 클레임은 세 가지 유형이 있습니다:
- 등록된 클레임: 표준 클레임 (예: iss(발행자), exp(만료시간), sub(주제) 등)
- 공개 클레임: 사용자 정의 클레임
- 비공개 클레임: 당사자들 간에 합의된 정보

```json
{
  "sub": "1234567890",
  "name": "홍길동",
  "admin": true,
  "iat": 1516239022,
  "exp": 1516242622
}
```
이 JSON 역시 Base64Url로 인코딩하여 JWT의 두 번째 부분을 구성합니다.

> 등록된 클레임

JWT 표준 스펙에서 정의한 클레임들입니다. 
사용은 선택 사항이지만, 의미와 사용법이 미리 정의되어 있어서 많이 사용됩니다.
    - iss (Issuer): 토큰을 발행한 주체 (예: 서버 이름)
    - sub (Subject): 토큰의 주제 (예: 사용자 ID)
    - aud (Audience): 토큰의 수신 대상
    - exp (Expiration Time): 만료 시각 (토큰이 더는 유효하지 않은 시간)
    - iat (Issued At): 토큰이 발행된 시간

> 공개 클레임

등록된 클레임 외에 공개적으로 사용될 수 있는 사용자 정의 클레임입니다.
    - user_name, role, email 등.

> 비공개 클레임

특정한 사람들에게만 공유되어야 하는 비공개 정보를 말합니다. 
이때 비공개 정보는 JWT 표준에 따라 정의되지 않은 사용자 정의 정보이며, 민감한 정보를 포함할 때 유용합니다.
즉, JWT 사양이나 공개 이름 공간에 등록되지 않고, 내부 용도로 자유롭게 정의하는 값입니다.
    - user_id, department_id 등.

### 3. 시그니처 (Signature)

시그니처는 JWT의 무결성을 보장하는 핵심 부분입니다. 헤더와 페이로드가 변조되지 않았음을 검증합니다.

> 시그니처 생성 과정

1. 인코딩된 헤더와 페이로드를 점(`.`)으로 연결
2. 지정된 알고리즘(예: HMAC SHA256)과 비밀 키를 사용하여 해시 생성
3. 생성된 해시를 Base64Url로 인코딩

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

> Base64Url 인코딩

Base64는 바이너리 데이터를 ASCII 문자열로 인코딩하는 방식입니다. 
JWT에서는 URL과 쿠키에서 안전하게 사용하기 위해 Base64Url을 사용합니다.

> HMAC-SHA256 알고리즘

HMAC(Hash-based Message Authentication Code)는 메시지의 무결성과 인증을 제공하는 메커니즘입니다.
SHA256은 SHA-2 계열의 해시 함수로, 어떤 길이의 데이터로부터 256비트 해시 값을 생성합니다.

> 시그니처의 역할

1. 무결성 보장: 토큰의 내용이 전송 과정에서 변조되지 않았음을 확인
2. 인증: 토큰이 신뢰할 수 있는 소스에서 생성되었음을 증명
3. 부인 방지: 발급자가 토큰 발행을 부인할 수 없음

서버는 JWT를 검증할 때 자신이 가진 비밀 키로 헤더와 페이로드를 다시 해싱하여 시그니처와 비교합니다. 
일치하면 토큰이 유효하다고 판단합니다.

## JWT 인증 흐름

1. 클라이언트가 로그인 요청
2. 서버가 사용자 정보 확인 → JWT 생성 후 반환
3. 클라이언트는 JWT를 저장 (보통 LocalStorage 또는 Cookie)
4. 이후 요청 시 JWT를 Authorization 헤더에 포함
5. 서버는 토큰의 서명을 검증하고, 사용자 인증 및 권한 판단

- 헤더에 포함된 JWT 예)
```http
Authorization: Bearer eyJhbGciOi...
```

> JWT 장단점

| 항목    | 설명                                                                    |
| ----- |-----------------------------------------------------------------------|
| ✅ 장점  | - 서버 세션 저장소 불필요<br> - 확장성 좋음<br> - 서비스 간 인증 공유 가능                     |
| ⚠️ 단점 | - 토큰 무효화 어려움<br> - 탈취 시 위험(HTTPS 필요)<br> - Payload는 암호화되지 않음(민감 정보 X) |

## JWT 코드 예시

```java
public class JwtExample {
    public static void main(String[] args) {
        // 보안 키 생성
        Key key = Keys.secretKeyFor(SignatureAlgorithm.HS256);
        
        // JWT 생성
        String jwtToken = Jwts.builder()
            .setSubject("user123")             // 토큰 서브젝트
            .setExpiration(new Date(System.currentTimeMillis() + 3600000)) // 만료 시간 (1시간)
            .claim("name", "홍길동")             // 사용자 정의 클레임
            .signWith(key)                     // 서명에 사용할 키와 알고리즘
            .compact();                        // JWT 문자열로 변환
        
        System.out.println("생성된 JWT: " + jwtToken);
        
        // JWT 검증
        boolean isValid = Jwts.parserBuilder()
            .setSigningKey(key)                // 서명 검증에 사용할 키
            .build()
            .parseClaimsJws(jwtToken)          // JWT 파싱 및 서명 검증
            .getBody()
            .getSubject().equals("user123");   // 주제 확인
        
        System.out.println("JWT 유효 여부: " + isValid);
    }
}
```
