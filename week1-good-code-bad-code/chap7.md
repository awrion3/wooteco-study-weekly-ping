# 코드를 오용하기 어렵게 만들라

<aside>
### ✅ *Check List*  

- [ ]  객체를 불변으로 생성했는가?
- [ ]  객체를 불변으로 반환했는가?
- [ ]  데이터 전용 객체를 정의했는가?
- [ ]  시간 처리 시, 적절한 자료구조를 활용했는가?
- [ ]  객체 정보에 기본 데이터만 담았는가?
- [ ]  수행하는 일이 비슷한 기능들을 하나의 파일로 묶었는가?
</aside>

## 불변 객체로 만드는 것을 고려하라.

불변 객체  : 외부에서 불변 객체의 값을 수정할 수 없는 객체

가변 객체 : 객체가 생성된 후에 상태를 바꿀 수 있음

필연적으로 상태 변화를 추적해야 하는 경우라면, 가변적인 자료구조가 필요하기도 함

가변(mutability) 객체의 문제점

1. 추론하기 어렵다
    1. 상자의 내용물을 수정할 수 있다면, 상자의 내용물이 어떻게 바뀌었을 지 외부에서 판단하기 어렵다.

    ```java
    // CustomDateTime은 가변
    CustomDateTime beforeDateTime = CustomDateTime.now(); 
    afterDateTime = beforeDateTime.modifyDateTime(); 
    // beforeDateTime과 afterDateTime의 수정 내용을 출력하고 싶지만, 둘의 출력내용이 같아짐
    ```

2. 다중 스레드에서 문제 발생
    1. 여러 스레드가 동시에 사용하면 훼손됨
    2. 한 스레드가 리스트에서 마지막 요소를 제거하는동안, 다른 스레드에서 그 요소를 읽는다면?

### 해결책 → 불변으로 생성하기

1. 객체를 생성할 때만 값을 할당하기. → 멤버 변수를 `final` 로 설정 및 `setter` 같은거 안열기
2. 불변성에 대한 디자인 패턴을 사용하기.
    1. 빌더 → builder().build(), `@Builder`

    ```java
    User user = User.builder()
    						.name("제프리")
                .age(15)
                .height(250)
                .iq(200).build();
    ```

   2. 쓰기 시 복사 패턴 → 값을 변경하면서 새로 생성 → withDay()

    ```java
      public LocalDate withDayOfMonth(int dayOfMonth) {
          if (this.day == dayOfMonth) {
              return this;
          }
          return of(year, month, dayOfMonth); // of -> create -> return new ... 
      }
    ```


## 객체를 깊은 수준까지 불변적으로 만드는 것을 고려하라.

클래스가 무심코 가변적으로 될 수 있는 상황

클래스의 어떤 객체에 대한 참조를 클래스 외부에서도 가지고 있으면 문제가 생길 수 있음

https://tecoble.techcourse.co.kr/post/2021-04-26-defensive-copy-vs-unmodifiable/

### 해결책

1. 방어적 복사 → return List.copyOf()
    1. 복사비용 높고
    2. 클래스 내부에서 발생하는 변경은 못막음
        1. 클래스 내부에서는 add 등을 호출할 수 있음
2. 불변적 자료구조 사용
    1. `ImmutableList` → 클래스 내에서도 내용을 변경할 수 없음.

## 지나치게 일반적인 데이터 유형을 피하라

정수, 문자열, 리스트의 문제점

→ 데이터 유형 자체만으로는 무언가를 설명할 수 없고,
가질 수 있는 값에 대해서도 지나치게 관대하다.

→ List<Integer, Integer> 로는 무엇을 뜻하는지 알 수 없어 주석을 관리해야하고,
넣는 순서가 바뀌어도 잘 모르고,
형식 안정성이 거의 없다 (두 개 이상의 값이 들어와도 , 컴파일에는 문제가 없다.)

```java
List<Double> location = [51.xxxxx, 52,xxxxx]; // 위도, 경도
-> location[0]. location[1]

void markLocationsOnMap(List<List<Double>> locations) {
		for (List<Double> location in locations) {
				map.markLocation(location[0], location[1])
		}
}
```

```java
class MapFeature {
		private final Double latitude;
		private final Double longitude;
		
		List<Double> getLocation() {
			return [latitude, longitude]; // 이건 괜찮을까? 
		}
```

한 번 이렇게 쓰기 시작하면 막을 수 없다..

MapFeature를 다른 용도로 사용해야하는 개발자가 왔을때, 앞서 List<Double> 이 선택되어 있으면 어쩔 수 없이 List<Double> 을 선택해야 한다.

그럼 페어유형은 어떨까?

```java
void markLocationsOnMap(List<Pair<Double, Double>> locations) {
		for (Pair<Double, Double> location in locations) {
				map.markLocation(location.getFirst(), location.getSecond())
		}
}
```

이래도 결국 위도와 경도의 순서에 대해 혼동하기 쉽고, Pair<Double, Double> 만으로는 파악하기 어려움..

### 해결책 제시 :  전용 클래스 사용하기!

불필요해 보여도, 생각보다 노력이 덜 들어가고, 다른 개발자가 봤을 때 이해하기 쉽다.

```java
class LatLong { 
		private final Double latitude;
		private final Double longitude;
		
		LatLong(Double latitude, Double longitude) {
				this.latitude = latitude;
				this.longitude = longitude;
		}
		
		... getter
}
```

```java
void markLocationsOnMap(List<LatLong> locations) {
		for (LatLong location in locations) {
				map.markLocation(location.getLatitude(), location.getLongitude())
		}
}
```

→ 데이터를 그룹으로 묶기만 하는 간단한 객체 정의 : 자바에서는 Record를(자바14 이상) 사용할 수 있음

→ 이건 관점따라 다름..

데이터 전용 객체를 정의하는것을 잘못된 관행으로 간주하면 → 데이터와 데이터에 대한 기능이 모두 동일한 클래스에 캡슐화되어야된다고 생각함. → 데이터가 기능과 밀접하게 연관되어 있는 경우 상당히 타당한 주장임

```java
class BankAccount {
    private int balance;

    public BankAccount(int initialBalance) {
        this.balance = initialBalance;
    }

    public void deposit(int amount) {
				...
    }

    public void withdraw(int amount) {
				...
    }
}
```

→ 하지만, 기능에 데이터를 연결하지 않고도, 데이터를 그룹화하는것이 유용한 상황이 있는데, 이럴경우 데이터 전용 객체가 유용함.

## 시간 처리

XX년 몇월 며칠 몇시 UTC 처럼 절대적인 시간을 나타내거나,

5분 내 와 같이 상대적인 시간을 나타냄

or 30 분 굽기 처럼 시간의 양을 언급할 수도 있고

윤년, 윤초, 표준시간대 등 다양한 종류와 복잡함을 자랑함.

→ 정수로 나타내지마라

- 한 순간의 시간인가, 아니면 시간의 양인가?
- 단위가 일치하는가? → 초를 쓰거나, 밀리초를 쓰거나…
- 시간대 처리가 일치하는가? 베를린에서 계산하는가, 뉴욕에서 계산하는가?

해결책 : 적절한 자료구조를 사용하라.

→ java.time

- 한 순간의 시간, 시간의 양 → java.time의 Instant (타임스탬프) 와 Duration(시간 간격)이라는 클래스가 있음
- Instant → UTC기준 특정 시점
- Duration → 두 Instant 간의 차이를 계산

```java
        Instant now = Instant.now(); // 현재 시간
        System.out.println("현재 시각: " + now);

        Instant past = now.minusSeconds(3600); // 1시간 전
        System.out.println("1시간 전: " + past);
```

```java
Duration duration1 = Duration.ofSeconds(5);
        Duration duration2 = Duration.ofMinutes(2);
        Duration duration3 = Duration.ofMillis(10);
```

- 시간대 처리 ? → LocalDateTime 을 기준으로

```java
public static LocalDateTime now() {
        return now(Clock.systemDefaultZone());
    }
    
        public static LocalDateTime now(Clock clock) {
        Objects.requireNonNull(clock, "clock");
        final Instant now = clock.instant();  // called once -> 현재 시각(UTC 기준) 
        ZoneOffset offset = clock.getZone().getRules().getOffset(now);
        // ex) Clock.system(ZoneId.of("Asia/Seoul")) ->  ZoneOffset.of("+09:00")
        return ofEpochSecond(now.getEpochSecond(), now.getNano(), offset);
    }
```

## 데이터의 진실의 원천은 하나만

- 기본 데이터 : 코드에 제공해야 할 데이터.
- 파생 데이터 : 기본 데이터에 기반해서 계산하는 데이터

계좌를 생성할 때, 대변(debit), 차변(credit), 잔액(balance)로 구성된다면?

- 대변, 차변

  대차대조표(재무상태표) 계정에서, 오른쪽(대변)은 **부채**와 **자본**이고 왼쪽(차변)은 **자산**이다. 전자는 현금이 장부로 들어오는 것, **자금**의 **조달** 즉, 사업을 할 때 필요한 자금이 어떠한 원천에서 조달되었는지를 보여주며 후자는 현금이 장부에서 나가는 것, **자금**의 **운용** 즉, 자금을 어떤 용도로 활용하는지 자금의 운용 상태를 나타낸다.


잔액은 대변과 차변에서 파생되므로, 잘못된 인스턴스 생성

```java
// 차액 데이터에 대한 원천이 2개인 경우
class UserAccount {
    private final Double credit;
    private final Double debit;
    private final Double balance;
    ...
}
-> new UserAccount(credit, debit, debit - credit); // 차변에서 대변을 빼는 잘못된 방식

// 차액 데이터에 대한 원천이 1개인 경우
class UserAccount {
    private final Double credit;
    private final Double debit;
    ...
    Double getBalance() {
        return credit - debit;
    }
}
```

만약 이런 계산과정이 비용이 많이 든다면?

→ lazy evaluation후 결과를 캐싱 !

계산의 결과값이 필요할 때까지 계산을 늦추는 기법

```java
class UserAccount {
    private final List<Transaction> transactions; // 이 transaction간의 연산에 비용 큼 

    private Double? cachedCredit;
    private Double? cachedDebit;

    UserAccount(ImmutableList<Transaction> transactions) {
        this.transactions = transactions;
    }

    ...

    Double getCredit() { // 저장되어있지 않으면, 계산을 하고 캐시로 저장
        if (cachedCredit == null) { // 호출되었을 때(lazy), 값이 빈 경우에만 연산한다.
        cachedCredit = transactions
                .map(transaction -> transaction.getCredit())
                .sum();
        }
        return cachedCredit;
    }
}
```

여기서, 클래스가 불변이 아니라면? → 클래스가 변경될 때 마다 cachedCredit, cachedDebit을 다시 재설정 해야함.

## 논리에 대한 진실의 원천은 하나

코드의 한 부분에서 수행되는 일이 다른 부분에서 수행되는 일과 일치해야 하는 경우.

```java
// 값을 직렬화 하고 저장하는 코드, 정수 리스트 저장 (직렬화)
class DataLogger {
    private final List<Int> loggedValues;
    ...

    saveValues(FileHandler file) {
        String serializedValues = loggedValues
                .map(value -> value.toString(Radix.BASE_10)) // 값을 10진수 문자열로 변환
                .join(","); // 값을 쉼표로 결합
        file.write(serializedValues);
    }
}

// 값을 읽고 역직렬화 하는 코드, 정수 리스트 불러오기 (역직렬화)
class DataLoader {
    ...

    List<Int> loadValues(FileHandler file) {
        return file.readAsString()
                .split(",") // 쉼표로 분할해 문자열의 목록으로 변환
                .map(str -> Int.parse(str, Radix.BASE_10)); // 각 문자열을 십진수 정수로
    }
} 
```

이게 다른 클래스에 정의되어 있을 경우,

10진수 대신 16진수로 코드가 변경된다면,  쉽표 대신 다른 문자열로 분할한다면,

각각의 파일을 수정해야 한다.

```java
class IntListFormat {
    private const String DELIMITER = ",";
    private const Radix RADIX = Radix.Base_10;

    Strint serialize(List<Int> values) {
        return values
                .map(value -> value.toString(RADIX))
                .join(DELIMITER);
    }

    List<Int> deserialize(String serialized) {
        return serialized
                .split(DELIMITER)
                .map(str -> Int.parse(str, RADIX));
    }
}

// 값을 직렬화 하고 저장하는 코드, 정수 리스트 저장 (직렬화)
class DataLogger {

		private final List<Integer> loggedValues;
		private final IntListFormat intListFormat;

    saveValues(FileHandler file) {
        file.write(intListFormat.serialize(loggedValues);
    }
}

// 값을 읽고 역직렬화 하는 코드, 정수 리스트 불러오기 (역직렬화)
class DataLoader {

    private final IntListFormat intListFormat;

    List<Int> loadValues(FileHandler file) {
        return intListFormat.deserialize(file.readAsString());
    }
} 
```
