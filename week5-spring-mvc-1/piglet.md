# AtomicLong
- `AtomicLong`은 원자적으로 갱신될 수 있는 Long 타입이다.
- 동시성을 고려해야하는 환경에서 값을 안전하게 increase / decrease 하기 위해 사용한다.
- 서로 영향을 받지 않도록 thread-safe를 지원한다.

### CAS (Compare And Swap) 알고리즘

락을 사용하지 않는 non-blocking 방식으로 동시성을 보장한다.

- 알고리즘 원리 ****
    - 기존 값 : 스레드가 갖고 있던 공유변수 value 기존 값
    - 현재 메모리가 가지고 있는 값 : 반영하고자 하는 메모리에 있던 값
    1. 파라미터로 기존 값과 변경할 값을 전달한다.
    2. 기존 값 == 현재 메모리가 가지고 있는 값 → 다른 스레드에서 메모리 값 변경 안했음 의미

       ⇒ 변경할 값을 반영하고 true를 반환한다.

    3. 기존 값 ≠ 현재 메모리가 가지고 있는 값 → 다른 스레드에서 메모리 값 변경 했음 의미

       ⇒ 반영하려던 값 반영 없이 false를 반환한다.

    4. false일 경우, 무한루프를 돌면서 변경된 값을 읽고 위 과정을 반복한다.

### AtomicLong이 어떻게 thread-safe를 지원하는가?

- ex ) `getAndIncreament()`

    ```java
    public final long getAndIncrement() {
       return U.getAndAddLong(this, VALUE, 1L);
    }
    ```

- U : `Unsafe` 객체
    - Java의 low-level을 관리하는 클래스로, 메모리 직접 접근, 객체 생성, 필드 접근, CAS 연산 등 JVM 내부 동작에 가까운 기능을 제공한다.
    - 말 그대로 안전하지 않은 클래스다.

      → 잘못 사용하면 JVM이 예상 못한 동작을 할 수 있다.

- `U.getAndAddLong()` : CAS 연산 부분

    ```java
    @IntrinsicCandidate
    public final long getAndAddLong(Object o, long offset, long delta) {
            long v;
            do {
                v = getLongVolatile(o, offset); // 현재 읽어온 기존 값
            } while (!weakCompareAndSetLong(o, offset, v, v + delta));
            return v;
    }
    ```

- `getLongVolatile()` : 현재 객체 o에 대한 최신 값을 읽어온다.
    - `AtomicLong` 클래스 내에서는 아래처럼 `volatile`이라는 키워드로 변수의 메모리 가시성을 보장한다.

        ```java
        private volatile long value;
        ```

    - 한 스레드에서 변경한 값이 즉시 다른 모든 스레드에게 보이도록 한다.
    - 캐시 일관성 문제 해결을 위해 필요하다.
- `weakCompareAndSetLong()`
    - o의 값이 여전히 현재 값인 v와 같을 경우, v+delta, 즉 위에서는 Long 값에 +1 한 값으로 변경한다.
    - 만약 v 값이 현재 값과 다를 경우, 다른 스레드가 중간에 값을 바꿨음을 의미하므로 do-while로 다시 시도한다.
    - get으로 기존 값을 반환하기에 원래 있던 값인 v를 반환한다.

# List<Reservation\>의 동시성 제어

### 기존 ArrayList<Reservation>에서 동시성 이슈가 발생할 경우

---

```java
public class ArrayListConcurrency {
    public static void main(String[] args) {
        // 스레드 1
				for (Reservation r : reservations) {
				    // 반복 중간에 스레드 2가 리스트를 수정하면
				    processReservation(r);
				}
				
				// 스레드 2 (동시에 실행)
				reservations.add(new Reservation("고객C"));  // ConcurrentModificationException 발생 가능
    }
}
```

- A 스레드가 for-each 루프로 reservations 리스트를 순회하는 동안, B 스레드가 reservations에 새로운 항목을 추가하는 경우

  ⇒ 컬렉션을 순회하는 동안 구조적 변경 (원소 추가/제거)이 감지될 때 `ConcurrentModfiicationException`이 발생한다.

## 1. CopyOnWriteArrayList

---

```java
List<Reservation> reservations = Collections.copyOnWirteArrayList();
reservations.add(reservation); // 기존 reservations를 복사해서 새로 만들고 교체 함
```

**동시성 제어 방법**

- 읽기 작업에서 lock 사용 X
- 읽기 작업은 스냅샷을 사용한다.
1. 쓰기 작업에서 배열 전체를 복사해 새로운 배열을 만든다.
2. 복사본에서 수정 작업을 수행한다.
3. 원본 참조를 새 배열로 교체한다.
- 쓰기 작업을 할 때마다 새로운 배열을 생성하고 기존 배열을 수정하지 않는다.

  ⇒ 다른 스레드에서 읽기 작업을 안전하게 수행할 수 있다.

  ⇒ 읽기 작업은 쓰기 작업의 영향 안받는다.


**장점**

- 읽기 작업이 많은 환경에서 List보다 최적화 된다.
    - 읽기 작업 시 잠금이 필요가 없기 때문이다.
- 다수의 스레드가 자주 리스트를 읽고, 수정하는 일이 드물 경우에 쓰면 좋다.

**단점**

- 쓰기 작업이 빈번한 경우라면 성능이 크게 저하된다.
    - 쓰기 작업 시마다 배열을 복사하기 때문이다.

### Reservations에 적합한가? → X

적합하지 않다고 판단했다.

예약 정보는 static한 데이터가 아니고 변경 가능성이 높다. 조회 비중만큼 데이터 C,U,D도 빈번할 것이다. 쓰기 작업이 잦은 Reservations에 대해서는 성능 저하가 발생할 수 있으므로 CopyOnWriteArrayList는 적절하지 않다고 생각했다.

조회가 위주인 로직에는 써 봐도 될 듯하다.

## 2. SynchronizedList

---

```java
List<Reservation> reservations = Collections.synchronizedList(new ArrayList<>());
reservations.add(reservation);
```

**동작 방법**

- 모든 읽기와 쓰기 동작에 대해 `synchronized` 키워드로 동기화 되어있다.
    - `synchronized` 란?

        ```java
        public synchronized void increment() {
            count++;
        }
        ```

        - 인스턴스 메서드에 붙이면 해당 객체에 대해 lock을 걸 수 있다.
        - 동시에 두 스레드가 increment()를 호출하려고 하면 하나가 끝날 때까지 나머지는 기다린다.

          ⇒ blocking 방식


```java
static class SynchronizedList<E>
        extends SynchronizedCollection<E>
        implements List<E> {
        ...
        public boolean equals(Object o) {
            if (this == o)
                return true;
            synchronized (mutex) {return list.equals(o);}
        }
        public int hashCode() {
            synchronized (mutex) {return list.hashCode();}
        }

        public E get(int index) {
            synchronized (mutex) {return list.get(index);}
        }
        public E set(int index, E element) {
            synchronized (mutex) {return list.set(index, element);}
        }
```

- 하나의 스레드만이 해당 리스트에 대해 읽기나 쓰기를 할 수 있다.
- 애초에 두 스레드가 충돌 발생할 일이 없다.
- 대신 list의 iterate(for-each)를 쓸거면 `synchronized (reservations)` 로 감싸줘야 한다.

**장점**

- List를 래핑하는 방식이라 생성 비용이 적다.
- 읽기보다 쓰기 동작이 많고, 크기가 큰 리스트의 경우 적합하다.

**단점**

- 모든 메서드가 동일한 잠금을 사용해서 한 번에 하나의 스레드만 접근 가능하므로 병렬적 처리가 어렵다.

- 왜 쓰기 작업이 많은 케이스에 적합할까?
    - CopyOnWriteArrayList와 **비교**했을 때 좋단거임
    - 쓰기 작업에 특화 된 특징이 있다거나 그런건 아님

### Reservations에 적합한가? → O

정확하게는 CopyOnWriteArrayList보다 적합할 것 같다.

blocking 방식이라 성능은 안좋을 것 같지만, 추가/삭제를 많이 하는 예약 정보 목록을 관리하려면 SynchronizedList가 더 나을 것 같다.

## CopyOnWriteArrayList vs SynchronizedList

---

|  | 읽기 (20) < 쓰기 (100) | 읽기 (20) > 쓰기 (5) |
| --- | --- | --- |
| CopyOnWriteArrayList | 16s 297ms | 2s 506ms |
| SynchronizedList | 17s 595ms | 3s 198ms |

아직까진 유의미한 차이점을 찾지 못했다.

