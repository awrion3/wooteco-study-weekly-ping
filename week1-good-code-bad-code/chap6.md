## 6장. 예측 가능한 코드를 작성하라

---

### 6.0 왜 예측 가능해야 하는가?

- 코드가 예측을 벗어나 동작하는 경우들이 존재하므로, 이를 피하기 위한 방법을 마련해야 한다

### 6.1 (우연한) 매직값 반환 금지

- 예시1: 아래의 getAge()에서 값이 없음을 나타내고자 -1을 반환한다면?

```java
    for(User user :users){
        sumOfAges +=user.getAge();
    }
```

- 예시2: 만약 전달받은 값이 빈 목록인 경우, 최소값은 디폴트 값으로 반환될 것이다.

```java
    Int minValue = Int.MAX_VALUE;
```

- 해결: 널, 옵셔널 또는 오류를 반환한다
    - 다만 널을 반환하는 경우, NPE 발생 가능
    - 옵셔널 혹은 오류를 반환하는 경우, 별도 처리 필요

### 6.2-1 널 객체 패턴을 널 대신 사용

- 널 값의 대체품들:
    - 빈 컬렉션을 반환한다

```java
    if(attribute ==null){
        return new Set();
    }
```

- 빈 문자열을 반환한다
    - 다만, 문자열이 ID로 사용되는 경우, 문자열이 정말 없는지를 파악해야 하므로 차라리 널 값이 나을 수 있음에 유의.

### 6.2-2 복잡한 널 객체 패턴과 구현은 오히려 예측 불가

- 복잡한 널 객체는 오히려 예측 불가

```java
    if(mugs.isEmpty()){
        return new CoffeeMug(diamter:0.0, height:0.0);
    }
// 머그잔이 없는 경우에 
// 크기가 0인 커피 머그잔 객체를 생성해서 반환한다.
// 이를 정상적인 객체로 착각할 수 있음에 유의하자.
```

- 널 객체 구현도 예측 불가

```java
    class NullCoffeeMugImpl implements CoffeeMug {
        Double getDiameter() {
            return 0.0;
        }
    
        Double getHeight() {
            return 0.0;
        }
    
        void reportMugBroken() {
            // 아무 일도 하지 않는다.
        }
    }
// CoffeeMug의 널 객체 구현 클래스
// NullCoffeMug는 reportMugBroken()을 호출했을 때
// 정말 아무 일도 하지 않는다는 문제가 존재한다.
```

### 6.3-1 사이드 이펙트를 피해라

- 함수가 반환하는 값 외에 다른 효과가 존재한다면, 이를 부수 효과(Side Effect)라고 한다.
    - 그런 행위를 기대하지 않았음에도 일어난다면 말이다.

- 일반적으로 클래스를 불변으로 만드는 것 == 부수 효과 가능성을 최소화하는 좋은 방법이다.

- But, 분명하고 의도적인 부수 효과는 괜찮다.

```java
    public void displayErrorMessage(String message) {
        canvas.drawText(message, Color.RED);
    }
```

> 사용자에게 출력을 표시하는 부수 효과를 가지고 있지만, 괜찮다.
> 클래스명과 함수명이 부수 효과를 일으킨다고 이미 명시하기 때문이다.

### 6.3-2 예기치 않은 부수 효과의 예, 그리고 해결책: 예고하라

```java
    Color getPixel(int x, int y) {
        canvas.redraw();
        Pixel pixel = canvas.getPixel(x, y);
        return new Color(
                pixel.getRed(),
                pixel.getGreen(),
                pixel.getBlue()
        );
    }
```

- 스레드1: 캔버스 다시 그리기 의도치 않게 호출 중…
- 스레드2: 어? 왜 픽셀을 가져오려 하는데, 캔버스 그림이 어디갔지?

> 이 함수를 호출하는 사람은 canvas를 다시 그릴 것이라고는 상상하지 못할 것이다.
> 따라서 코드 한 줄 때문에 문제가 발생해도 어디서 문제가 발생했는지 쉽게 찾아내지 못하게 된다.

- 부수 효과는 비용이 많이 들거나, 호출하는 쪽에 예상치 못한 동작을 야기하거나, 멀티 스레드 코드의 버그를 일으킬 수 있다.
    - 하지만 부수 효과는 소프트웨어를 작성하며 반드시 일어나는 행위다.
    - 일어난다면, 일어난다는 사실을 분명하게 알려야 한다.

### 6.4~5 오해를 일으키는 함수는 작성하지 말라

- 건네준 매개변수를 함수 내에서 수정하는 경우도, 부수 효과 발생
    - 변경하기 전에 해당 객체 복사해두기
    - 혹은 방어적으로 불변 객체로 만들어 놓기

- 중요한 매개변수의 입력이 누락되었을 때, 아무 대응도 안하는 함수는 좋지 않다

```java
    public void doSomething(String value) {
        if (value == null)
            return;
        // 값이 없으면 아무 작업도 수행하지 않음
        System.out.println("Message: " + value);
    }
```

```java
    public static void main(String[] args) {
        Example ex = new Example();
        ex.doSomething(null);
        // 아무것도 출력되지 않음
        ex.doSomething("Hello");
        // "Message: Hello" 출력
    }
```

> 매개변수가 없더라도 호출할 수 있고, 해당 매개변수가 없으면 아무 작업도 수행하지 않는 함수가 있다면:
> 중요한 입력 매개변수는 필수 항목으로 처리하게끔 고치거나, 혹은 호출하는 측에서 null 여부를 확인하도록 만들자.

- “이름대로 행동하게 함수를 쓰자. 코드 계약의 명백한 부분에 오해의 소지가 있는 경우를 줄이자.”

### 6.6-1 미래를 대비한 열거형 처리

- Enum에 더 많은 값이 추가될 수 있음을 예상하고, 코드를 작성하라.

```java
    enum Status {
        START,
        END;
    }

    Boolean isStatus(Status status) {
        if (status == Status.START) {
            return false;
        }
        return true;
    }
```

> Status에 새로운 결과인 Ready가 추가된다면? 아래처럼 미리 가정하여 설계하자.

- 모든 경우를 처리하는 스위치 문을 사용

```java
    enum Status {
        START,
        END;
    }

    Boolean isStatus(Status status) {
        switch (status) {
            case START:
                return false;
            case END:
                return true;
        }
        throw new UncheckedException();
    }
```

> 이렇게 사용할 경우 Status 열거형에 새 값을 추가한 개발자는 isStatus()도 변경해야 함을 알 수 있다.

### 6.6-2 열거형의 기본 케이스를 주의하라

- default 사용하여 새로운 값에 대해 암시적으로 처리하게 된다
    - 때문에 아래처럼 기본 케이스를 사용함으로 인해 예상을 벗어나는 동작과 버그를 초래할 수 있다.

```java
    Boolean isStatus(Status status) {
        switch (status) {
            case START:
                return false;
            case END:
                return true;
            default:
                return true;
        }
    }
```

- 따라서 기본 케이스를 활용하고자 한다면, default 파트에서 예외를 발생시킬 수 있다.

> 다만 C++과 같은 일부 언어에서는 스위치 문이 모든 값을 처리하지 않을 때 컴파일러 경고를 할 수 있다.
> 근데 기본 케이스가 있게 되면, 컴파일러는 모든 값이 처리되는 것으로 판단하고
> 나중에 새로운 열거값이 추가되어도 스위치 문이 모든 값을 처리해준다고 잘못 판단해버린다.

- 따라서 차라리 스위치 문이 끝나고 throw new UncheckedException()을 두는 것이 낫다

### 6.7 그냥 이 모든 것을 테스트로 해결할 수는 없는가?

- 예상을 벗어나는 코드를 방지하기 위한, 이러한 품질 향상 노력을 반대하는 사람들이 종종 있다.
- 그들에 의하면, 테스트가 이런 모든 문제를 결국 잡기에, 예측 가능한 코드를 작성하기 위한 노력은 시간 낭비라는 것이다.

> 물론, 코드를 작성하는 시점에서는 코드를 어떻게 테스트할지 제어할 수 있다.
> 테스트는 매우 중요하지만, 이를 부지런하게 관리하는 것은 쉽지 않다.
> 특히 직관적이지 않거나 예상을 벗어나는 코드에 숨어있는 오류의 경우, 테스트만으로는 방지하기 어렵다.

### 6장. 결론

- 우리는 다른 개발자가 작성하는 코드에 의존하는 상황을 마주할 수 있으며, 반대로 우리가 작성한 코드를 그들이 사용할 수도 있다.

> 코드를 잘못 해석할 수 있는 여지를 주게 되면, 버그의 발생 가능성이 크다.

- 예상대로 동작하기 위한 좋은 방법은, 세부사항이 코드 계약의 명백한 부분에 포함되게 하는 것이다.

> 우리가 사용하는 코드에 대해, 허술하게 가정하지 말고 미래를 예상하며 대비하자.

- 의존해서 사용 중인 코드가 가정을 벗어나면, 컴파일을 중지하게 혹은 테스트를 실패하게 만들도록 하자.

---
