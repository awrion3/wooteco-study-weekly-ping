## 5장. 가독성 높은 코드를 작성하라

### 서술형 명칭 사용

- 서술적이지 않은 이름은 코드를 읽기 어렵게 만든다.
- 주석문이나 문서를 추가함으로써 서술적이지 않은 이름을 개선할 수 없지만, 서술적인 이름을 대체할 수 없다.
- 결국 서술적인 이름을 지음으로써 별도의 설명이 필요 없게 되고 주석문 관리가 불필요해지며 가독성이 향상된다.

---

### 주석문의 적절한 사용

- **주석문 및 문서화는 코드가 무엇을 하는지, 왜 그 일을 하는지, 그리고 사용 지침 및 기타 정보를 제공하기 위해 사용된다.**
    - 클래스와 같이 큰 단위의 코드가 무엇을 하는지는 요약하는 높은 수준의 주석문은 유용하나,
    하위 수준에서 줄 단위의 코드는 서술적인 이름으로 작성되어 주석문을 통한 설명이 불필요하다.
    - 만약 낮은 수준의 코드에 대한 주석이 많이 추가된다면 이는 코드 자체의 가독성이 떨어진다는 의미이다.
    - 반면 낮은 층위의 코드가 왜 그 일을 하는 지에 대한 이유나 배경을 설명하는 주석문은 가능하며, 
    비트 논리와 같이 최적화를 목적으로 한 복잡한 코드에 대한 주석으로 유연하게 사용할 수 있다.
- **중복된 주석문은 유해할 수 있다.**
    - 코드가 수행하는 작업을 설명하는 주석문
        
        ```java
        String generateId(String firstName, String lastName) {
          // "{이름}.{성}"의 형태로 ID를 생성한다.
          return firstName + "." + lastName;
        }
        ```
        
- **주석문으로 가독성 높은 코드를 대체할 수 없다.**
    - 주석문이 있는 이해하기 어려운 코드
        
        ```java
        String generateId(String[] data) {
          // data[0]는 유저의 이름이고 data[1]은 성이다.
          // "{이름}.{성}"의 형태로 ID를 생성한다.
          return data[0] + "." + data[1];
        }
        ```
        
    - 가독성이 더 좋아진 코드
        
        ```java
        String generateId(String [] data) {
          return firstName(data) + "." + lastName(data);
        }
        
        String firstName(String [] data) {
          return data[0];
        }
        
        String lastName(String [] data) {
          return data[1];
        }
        ```
        
- **주석문은 코드의 이유를 설명하는 데 유용하다.**
    - 주석문을 사용해 코드가 존재하는 이유를 설명하면 좋다.
        - 제품 또는 비즈니스 의사 결정
        - 이상하고 명확하지 않은 버그에 대한 해결책
        - 의존하는 코드의 예상을 벗어나는 동작에 대처
    - 코드가 존재하는 이유를 설명하는 주석문
        
        ```java
        public class User {
        
            private final String username;
            private final String firstName;
            private final String lastName;
            private final Version signupVersion;
        
            public User(String username, String firstName, String lastName, Version signupVersion) {
                this.username = username;
                this.firstName = firstName;
                this.lastName = lastName;
                this.signupVersion = signupVersion;
            }
        
            private String getUserId() {
                if (signupVersion.isOlderThan("2.0")) {
                    // (v2.0 이전에 등록한) 레거시 유저라는 이름으로 ID가 부여된다.
                    // 자세한 내용은 #4218 이슈를 보라.
                    return firstName.toLowerCase() + "." + lastName.toLowerCase();
                }
                // (v2.0 이후로 등록한) 새 유저는 username으로 ID가 부여된다.
                return username;
            }
        }
        ```
        
- **주석문은 유용한 상위 수준의 요약 정보를 제공할 수 있다.**
    - 코드 기능에 대한 상위 수준에서의 개략적인 문서를 제공하면 좋다.
        - 클래스가 수행하는 작업, 다른 개발자가 알아야 할 중요한 내용을 개괄적으로 설명
        - 함수에 대한 입력 매개변수 또는 기능 설명
        - 함수의 반환값이 무엇을 나타내는지 설명
    - 클래스에 대한 상위 수준의 문서화
        
        ```java
        /**
         * 스트리밍 서비스의 유저에 대한 자세한 사항을 갖는다.
         * 
         * 이 클래스는 데이터베이스에 직접 연결하지 않는다. 대신 메모리에 저장된 값으로 
         * 생성된다. 따라서 이 클래스가 생성된 이후에 데이터베이스에서 이뤄진 변경 사항을 
         * 반영하지 않을 수 있다.
         */
        public class User {
          ...
        }
        ```
        

---

### 코드 줄 수를 고정하지 말라

- **일반적으로 코드의 줄 수가 적을수록 좋다.**
    - 그러나 코드 줄 수는 반드시 지켜야 할 엄격한 규칙은 아니다.
    - 때로는 매우 이해하기 어려운 코드 한 줄보다는 같은 일을 하는 쉬운 코드 10줄이 더 좋을 때도 있다.
    - 코드 줄 수는 우리가 실제로 신경 쓰는 것들을 간접적으로 측정해줄 뿐이다.
        - 이해하기 쉽다.
        - 오해하기 어렵다.
        - 실수로 작동이 안 되게 만들기 어렵다.
- **간결하지만 이해하기 어려운 코드는 피하라.**
    - 간결하지만 이해하기 어려운 코드
        
        ```java
        Boolean isIdValid(UInt16 id){
        	return countSetBits(id & 0x7FFF) % 2 == ((id & 0x8000) >>15);
        }
        ```
        
- **해결책: 더 많은 줄이 필요하더라도 가독성 높은 코드를 작성하라**
    - 코드 줄 수가 많은 것은 기존 코드를 재사용하지 않거나 필요 이상으로 복잡하다는 의미일 수 있다.
    - 그러나 이것보다 더 중요한 것은 코드가 이해하기 쉽고, 어떤 상황에서도 잘 동작하는 것이다.
    - 이것을 효과적으로 하기 위해 많은 코드가 필요하다면 그것은 문제가 되지 않는다.
    - 코드의 양은 더 많지만 가독성은 높은 코드
        
        ```java
        Boolean isIdValid(UInt16 id){
        	return extractEncodedParity(id) == calculateParity(getIdValue(id));
        }
        
        private const UInt16 PARITY_BIT_INDEX = 15;
        private const UInt16 PARITY_BIT_MASK = (1 << PARITY_BIT_INDEX);
        private const UInt16 VALUE_BIT_MASK= ~PARITY_BIT_MASK;
        
        private UInt16 extractEncodedParity(UInt16 id){
        	return (id & PARITY_BIT_MASK) >> PARITY_BIT_INDEX;
        }
        ```
        

---

### 일관된 코딩 스타일을 고수하라

- **일관적이지 않은 코딩 스타일은 혼동을 일으킬 수 있다.**
    - 일반적으로 클래스 이름을 파스칼 케이스(PascalCase)로, 변수 이름을 캐멀 케이스(camelCase)로 작성한다.
    - 이를 반대로 작성한 코드를 읽는다면 많은 시간과 에너지를 낭비하며 오해로 인한 심각한 버그를 초래할 수도 있다.
- **해결책: 스타일 가이드를 채택하고 따르라.**
    - 일관된 코딩 스타일을 따라 코드를 작성하면 가독성이 좋아지고 버그를 예방하는 데 도움이 된다.
    - 코딩 스타일은 명명법 이상의 많은 측면을 다룬다.
        - 언어의 특정 기능 사용
        - 코드 들여쓰기
        - 패키지 및 디렉터리 구조화
        - 코드 문서화 방법
    - 팀이나 조직에서 코딩 스타일 가이드를 따르는 것이 중요하다.

---

### 깊이 중첩된 코드를 피하라

- **일반적으로 코드는 서로 중첩되는 블록으로 구성된다.**
    - 함수가 호출되면 그 함수가 실행되는 코드가 하나의 블록이 된다.
    - if 문의 조건이 참일 때 실행되는 코드는 하나의 블록이 된다.
    - for 루프의 각 반복 시 실행되는 코드는 하나의 블록이 된다.
- **깊이 중첩된 코드는 읽기 어려울 수 있다.**
    - 중첩이 깊어지면 가독성이 떨어지기 때문에 중첩을 최소화하도록 코드를 구성하는 것이 바람직하다.
    - 깊이 중첩된 if문
        
        ```java
        private Address getOwnersAddress(Vehicle vehicle) {
          if (vehicle.hasBeenScraped()) {
            return SCRAPYARD_ADDRESS;
          } else {
            Purchase mostRecentPurchase = vehicle.getMostRecentPurchase();
            if (mostRecentPurchase == null) {
              return SHOWROOM_ADDRESS;
            } else {
              Buyer buyer = mostRecentPurchase.getBuyers();
              if (buyers != null) {
                return buyer.getAddress();
              }
            }
          }
        
          return null;
        }
        ```
        
- **해결책: 중첩을 최소화하기 위한 구조 변경**
    - 중첩된 모든 블록에 반환문이 있다면 중첩을 피하기 위해 논리를 쉽게 재배치할 수 있다.
    - 중첩된 블록에 반환문이 없다면, 대개 함수가 너무 많은 일을 하고 의미다.
    - 중첩이 최소화된 코드
        
        ```java
        private Address getOwnersAddress(Vehicle vehicle) {
          if (vehicle.hasBeenScraped()) {
            return SCRAPYARD_ADDRESS;
          }
          Purchase mostRecentPurchase = vehicle.getMostRecentPurchase();
          if (mostRecentPurchase == null) {
            return SHOWROOM_ADDRESS;
          }
          Buyer buyer = mostRecentPurchase.getBuyers();
          if (buyers != null) {
            return buyer.getAddress();
          }
          return null;
        }
        ```
        
- **중첩은 너무 많은 일을 한 결과물이다.**
    - 너무 많은 일을 하는 함수를 더 작은 함수로 나누면 중첩되는 문제를 해결할 수 있다.
    - 너무 많은 일을 하는 함수
        
        ```java
        private SentConfirmation sendOwnerALetter(Vehicle vehicle, Letter letter) {
          Address ownersAddress = null;
          if (vehicle.hasBeenScraped()) {
            ownersAddress = SCRAPYARD_ADDRESS;
          } else {
            Purchase mostRecentPurchase = vehicle.getMostRecentPurchase();
          }
          if (mostRecentPurchase == null) {
            return SHOWROOM_ADDRESS;
          } else {
            Buyer buyer = mostRecentPurchase.getBuyers();
            if (buyers != null) {
              return buyer.getAddress();
            }
          }
          if (ownersAddress == null) {
            return null;
          }
          return sendLetter(ownersAddress, letter);
        }
        ```
        
- **해결책: 더 작은 함수로 분리**

> "2장에서는 하나의 함수가 너무 많은 일을 하면 추상화 계층이 나빠진다는 점을 살펴봤다"에 해당하는 예시에 대해서 각자 생각해보면 좋을 것 같습니다. 🤔

    
    - 함수가 너무 많은 일을 하면 추상화 계층이 나빠진다.
    - 그러므로 추상화를 위해 중첩이 없더라도 메서드를 더 작은 메서드로 분리하는 것이 좋다.
    - 많은 일을 하는 코드에 중첩마저 많을 때 메서드를 분리하는 것이 더욱 더 중요하다.
    - 더 작은 함수
        
        ```java
        private SentConfirmation sendOwnerALetter(Vehicle vehicle, Letter letter) {
          Address ownersAddress = getOwnersAddress(vehicle);
          if (ownersAddresss != null) {
            return sendLetter(ownersAddress, letter);
          }
          return null;
        }
        
        private Address getOwnersAddress(Vehicle vehicle) {
          if (vehicle.hasBeenScraped()) {
            ownersAddress = SCRAPYARD_ADDRESS;
          }
          Purchase mostRecentPurchase = vehicle.getMostRecentPurchase();
          if (mostRecentPurchase == null) {
            return SHOWROOM_ADDRESS;
          }
          Buyer buyer = mostRecentPurchase.getBuyers();
            if (buyers == null) {
              return null;
            }
          return buyer.getAddress();
        }
        ```
        
    
> 중첩된 if문 대신 반환문을 사용하고 마지막에 `return null`을 사용하는 방식에 대한 의견이 궁금합니다. 🤔


---

### 함수 호출도 가독성이 있어야 한다

- **함수의 이름 뿐만 함수 호출도 가독성이 있어야 한다.**
    - 함수의 인수 개수와 함수가 담당하는 역할도 가독성에 영향을 준다.
    - 함수나 생성자가 많은 수의 매개변수를 가지고 있으면 추상화 계층이나 모듈화에 근본적인 문제가 있을 수도 있다.
- **매개변수는 이해하기 어려울 수 있다.**
    - 함수 호출 시 각 인수의 값이 무엇을 의미하는지 알려면 함수 정의를 확인해야 한다.
        
        ```java
        sendMessage("hello", 1, true);
        ```
        
- **해결책: 명명된 매개변수 사용**
    - 인수 목록 내 위치가 아닌 이름으로 일치하는 매개변수를 찾는 방식이다.
    - 명명된 인수를 사용한다면, 함수 정의를 확인하지 않고도 함수 호출을 쉽게 할 수 있다.
    - 그러나 모든 언어에서 지원되는 것은 아니다.
    - 명명된 매개변수로 만드는 것을 **객체 구조 분해**라고 하는데, 이는 타입스크립트 또는 자바스크립트에서 흔히 볼 수 있다.
        
        ```java
        sendMessage(message: "hello", priority: 1, allowRetry: true);
        ```
        
- **해결책: 서술적 유형 사용**
    - 명명된 매개변수 지원 여부와 상관없이 함수를 정의할 때 서술적인 유형을 사용하는 것이 바람직하다.
    - 특정 유형을 만들어 그 매개변수들이 나타내는 바를 설명함으로써 함수 정의를 알지 못해도 이해하기 쉽다.
        
        ```java
        sendMessage("hello", new MessagePriority(1), RetryPolicy.ALLOW_RETRY);
        ```
        
- **때로는 훌륭한 해결책이 없다.**
    - 마땅한 해결책이 없는 경우, 인라인 주석문을 사용할 수 있다.
    - 인라인 주석문은 생성자 호출 코드의 가독성을 높인다.
        
        ```java
        BoundingBox box = new BoundingBox(
                /*top=*/ 10,
                /*right=*/ 50,
                /*bottom=*/ 20,
                /*left=*/ 5
                );
        ```
        
- **IDE는 어떤가?**
    - IDE(통합 개발 환경)에서 제공하는 함수의 매개변수 표시 기능을 활용할 수 있다.
    - 그러나 모든 개발자가 이러한 기능을 제공하는 IDE를 사용하지 않는다.
    - 또한 IDE 도움 없이 코드 그 자체만 봐야 하는 경우를 고려하여 IDE에 의존해서는 안된다.
    - 실제 코드
        
        ```java
        sendMessage("hello", 1, true);
        ```
        
    - IDE가 보여주는 코드
        
        ```java
        sendMessage(message: "hello", priority: 1, allowRetry: true);
        ```
        

---

### 설명되지 않은 값을 사용하지 말라

- 하드 코드로 작성된 모든 값에는 2가지 중요한 정보가 있다.
    - 값이 무엇인지: 컴퓨터가 이해하는 의미
    - 값이 무엇을 의미하는지: 개발자 이해하는 의미
- **설명되지 않은 값은 혼란스러울 수 있다.**
    - 코드에 설명되지 않은 값이 있으면 이렇게 혼란을 초래하고 버그를 발생시킬 수 있다.
    - 따라서 그 값이 무엇을 의미하는지 명확하게 하는 것이 중요하다.
- **해결책: 잘 명명된 상수를 사용하라**
    - 값을 설명하기 위해 상수를 정의하고 상수 이름을 통해 값을 설명할 수 있다.
- **해결책: 잘 명명된 함수를 사용하라**
    - 코드의 가독성을 높이기 위해 함수를 사용할 수 있는 방법은 2가지가 있다.
        - 상수를 반환하는 **공급자 함수**(Provider Function)
        - 변환을 수행하는 **헬퍼 함수**(Helper Function)

---

### 익명 함수를 적절하게 사용하라

- 익명 함수(anonymous function)은 이름이 없는 함수이며, 일반적으로 코드 내의 필요한 지점에서 인라인으로 정의된다.
- 익명 함수는 간단한 경우 코드의 가독성을 높여주지만 복잡하거나 재사용해야 하는 경우 문제가 될 수 있다.
- **익명 함수는 간단한 로직에 좋다.**
- **익명 함수는 가독성이 떨어질 수 있다.**
- **해결책: 대신 명명 함수를 사용하라**
- **익명 함수가 길면 문제가 될 수 있다.**
- **해결책: 긴 익명 함수를 여러 개의 명명 함수로 나누라**

---

### 프로그래밍 언어의 새로운 기능을 적절하게 사용하라

- 새로운 기능을 사용하고 싶다는 생각이 들 때, 그것이 정말로 그일에 적합한 도구인지 솔직하게 생각해봐야 한다.
- **새 기능은 코드를 개선할 수 있다.**
    - 전통적인 자바 코드로 작성된 리스트 필터링
        
        ```java
        List<String> getNotEmptyStrings(List<String> strings) {
          List<String> nonEmptyStrings = new ArrayList<>();
          for (String str: strings) {
            if (!str.isEmpty()) {
              nonEmptyStrings.add(str);
            }
          }
          return nonEmptyStrings;
        }
        ```
        
    - 스트림을 사용한 리스트 필터링
        
        ```java
        List<String> getNotEmptyStrings(List<String> strings) {
          return strings
            .streams()
            .filter(str -> !str.isEmpty())
            .collect(toList());
        }
        ```
        
- **불분명한 기능은 혼동을 일으킬 수 있다.**
    - 일반적으로 코드 품질을 개선한다면 언어가 제공하는 기능을 사용하는 것이 바람직하다.
    - 개선사항이 적거나 다른 개발자가 그 기능에 익숙하지 않을 경우 지양하는 것이 좋을 때도 있다.
- **작업에 가장 적합한 도구를 사용하라.**
    - map에서 키 값을 찾을 때 아래는 가장 합리적인 코드이다.
        
        ```java
        String value = map.get(key);
        ```
        
    - 하지만 이 과정을 스트림을 사용한다면 가독성이 훨씬 떨어질 뿐만 아니라 효율성도 훨씬 낮아진다.
        
        ```java
        Optional<String> value = map
          .entrySet()
          .stream()
          .filter(entry -> entry.getKey().equals(key))
          .map(Entry::getValue)
          .findFirst();
        ```
