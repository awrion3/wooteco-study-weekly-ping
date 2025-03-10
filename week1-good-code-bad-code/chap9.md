# Chapter 9. 코드를 재사용 하고 일반화 할 수 있도록 하라

---

- **간결한 추상화 계층 만들기** & **코드 모듈화**

  ⇒ 하위 문제에 대해 **느슨한 결합**을 하는 해결책 코드 생성 가능


## 9.1 가정을 주의하라

---

- 코드에 대해 상황을 가정하기
    - ex ) 뉴스 기사에는 사진이 한 개만 들어 있을 것이다
    - 활용도가 낮아짐 → 재사용 하기에는 불안정
    - 코드의 어떤 부분에 어떤 가정 사항이 들어있었는지 정확한 추적이 어려움
- 따라서 코드 작성 시 가정을 하기 전에 해당 가정이 초래할 비용과 이점 고려 필요

### 가정은 코드 재사용 시 버그를 초래할 수 있다

---

- ex ) 웹사이트 기사 클래스인 `Article` 에서의 `getAllImages()` 메서드

    ```java
    public class Article {
        private List<Section> sections;
        
        ...
        
        List<Image> getAllImages(){
            for(Section section : sections){
                if(section.containsImages()){ // 해당 섹션에 이미지가 포함되어 있으면
                    return section.getImages(); // 해당 섹션의 이미지들만 가져오기
                }
            }
            return List.of();
        }
    }
    ```

    - 메서드 이름에서 기대하는 행동 : 기사에서 이미지가 포함된 섹션은 모두 가져오기
    - 코드에 가정된 사항 : 기사 내에 이미지를 포함한 섹션은 최대 1개이다.
    - 가정이 초래한 비용 : 개발자들이 이 메서드의 의미를 파악하기 어려움

### Solution 1. 불필요한 가정을 피하라

---

- 가정에 대해 비용 - 이익 간 상충 관계 고려하기
- ex ) `Article`의 `getAllImages()` 메서드
    - 가정 사항 : 기사 내에 이미지를 포함한 섹션은 최대 1개이다.
        - 비용 : 코드 재사용성 하락, 요구사항 변경 시 버그 가능성 높음
        - 이익 : 약간의 성능 향상
    - **이익 < 비용** ⇒ 가정이 불필요 ⇒ **가정을 제거**
    - 가정을 제거한 상황 : 이미지가 포함 된 모든 섹션을 반환한다
        
        ```java
        public class Article {
            private List<Section> sections;
            
            ...
            
            List<Image> getAllImages(){
        			  List<Image> images = new ArrayList<>(); 
                for(Section section : sections){ // 모든 섹션에서 이미지를 가져오도록 변경
                    if(section.containsImages()){
                        images.addAll(section.getImages());
                    }
                }
                return images;
            }
        }
        ```


- 가정이 충분히 가치있는 케이스

  : 특정한 가정으로 인해 성능이 눈에 띄게 향상되거나 코드가 크게 단순해질 경우


<aside>

**[ 섣부른 최적화 ]**

- 최적화 된 해결책이 초래할 수 있는 단점
    1. 더 많은 시간 & 노력 & 비용 필요
    2. 가독성 떨어질 수 있음
    3. 유지 관리가 더 어려워짐
    4. 가정으로 인해 견고함 떨어질 수 있음
- 프로그램 내 수천 ~ 수백만 번 실행되는 코드 부분에 대해 이뤄질 때만 유의미한 이점이 있음
- 따라서 효과 미미한 최적화에 집중해서 힘쓰지 말자
    - 가독성 & 유지보수성 & 견고함 고려하는 것이 더 중요
</aside>

### Solution 2. 가정이 필요하면 강제적으로 하라

---

- 가정 했을 때 이득 > 비용이 될 경우, 충분히 가치있는 가정이 됨
- 하지만 다른 개발자들은 이러한 가정 사항을 모를 수 있음
- 따라서 가정을 강제화 해야 함

1. **가정이 ‘깨지지 않게’ 만든다**
    - 가정이 깨지면 컴파일 되지 않는 방식으로 코드를 작성하기 → 가정의 항상성 유지 가능 (3장 & 7장)
        - 컴파일러로 계약을 만드는 것이 가장 신뢰할 수 있는 방법

        <aside>

      3장 일부 내용 (67p)

        - 인자 유형 : 호출하는 쪽에서 인자 유형을 잘 못 사용하면 컴파일조차 되지 않음
        - 반환 유형 : 호출하는 쪽에서 반환 유형과 일치 하지 않는 유형을 사용하면 코드는 컴파일조차 되지 않음
        - checked exception : 호출하는 코드가 이를 처리하지 않을 경우 컴파일조차 되지 않음
        </aside>

    
2. **오류 전달 기술을 사용한다**
    - 가정을 깨는 것을 불가능하게 만들 수 없는 경우, 오류를 감지하고 신속하게 실패하도록 코드를 작성 (3장 & 4장)

        <aside>

      4장 일부 내용 (86p)

      >  오류가 발생했을 때 신속하게 실패하지 않으면, 오류는 실제 위치에서 멀리 떨어진 코드에서 나타날 수 있다. 이는 곧 오류의 실제 위치를 찾기 위해 스택 트레이스를 역추적 하며 찾아가야 한다.

        </aside>


- 문제의 소지가 있는 강제되지 않은 가정
    - ex ) ‘기사 내에 이미지를 포함하는 섹션은 최대 1개 있다.’ 가정 시,
    1. **가정을 포함하는 코드**

        ```java
        class Article{
        	private List<Section> sections;
        	...
        	
        	Section getImageSection(){
        		// 기사 내 이미지를 포함한 섹션은 최대 1개이다.
        		return sections
        						.stream()
        						.filter(section -> section.containsImages())
        						.findFirst()
        						.orElse(null);
        	}
        }
        ```

        - 위 가정이 깨져도 (여러 개의 섹션이 이미지를 가져도) 코드는 실패하지 않음
            - 경고 생성도 없이 정상 작동 될 것

              ⇒ **‘신속하게 실패’** 할 수 없는 코드가 됨 (호출한 곳에서 NPE 발생 가능)

    2. **가정에 의존하는 호출자**

        ```java
        class ArticleRender{
        	...
        	
        	void render(Article article){
        		...
        		Section imageSection = article.getImageSection();
        		if(imageSection != null){
        			templateData.setImageSection(imageSection);
        		}
        	}
        }
        ```

        - 위처럼 호출하려면 기사에 이미지를 갖는 섹션이 최대 하나만 있다는 가정이 필요
            - 다른 개발자가 이를 호출하면, 호출한 곳에서 실제로 처음 한 장 외에는 모두 이미지가 누락되는 결과를 초래할 것

- 가정의 강제적 확인 ( 이해 안감 자바에서는 어떻게 한다는거지? )
    - assertion을 사용해 이미지 섹션이 최대 한 개 있다고 가정하는 코드를 보여줌
    - 이렇게 만듦으로써 이 가정을 원치 않을 경우 함수를 호출하지 않도록 할 수 있음

    ```java
    class Article{
    	private List<Section> sections;
    	...
    	
    	Section getOnlyImageSection(){
    		List<Section> imagesSections = sections.stream()
    																			.filter(section -> section.containsImages())
    																			.toList();
    		
    		assertThat(imagesSections).hasSize(1);
    		
    		return imagesSections;
    	}
    }
    ```


## 9.2 전역 상태를 주의하라

---

- 전역 상태 : 코드를 취약하게 만들고 재사용 하기에도 불안전함 ( 일반적으로 이득 < 비용 )

```java
public static int GLOBAL_VALUE = 10;
```

### 전역 상태를 갖는 코드는 재사용 하기에 안전하지 않을 수 있다

---

당연함

전역 변수로 장바구니를 갖고 있으면 어떻게 될까요?

```java
public static Cart cart = new Cart(); 
```

모든 유저가 하나의 장바구니를 공유하게 되겠죠 (thread-safe 하지 않음)

메서드 파라미터로 항상 객체를 넘기는게 귀찮다고 함부로 static에 박아두면 안됩니다

### Solution 1. 공유 상태에 의존성 주입하라

---

우리가 가장 많이 하는 의존성 주입

- 전역 상태 사용할 경우

```java
class Customer{
		...
		public void addItem(Item item){
			Basket.add(item); // 전역 상태 장바구니 사용
		}
}
```

- 상태를 의존성 주입할 경우

```java
class Customer{
		private Basket basket;
		
		public Customer(Basket basket){
			this.basket = basket;
		}
		
		...
		
		public void addItem(Item item){
			basket.add(item);
		}
}
```

## 9.3 기본 반환 값을 적절하게 사용하라

---

- 예시로 Word 프로그램 열면 글꼴, 텍스트 크기, 텍스트 색, 배경색, 줄 간격 등 정확히 우리가 선택해야 프로그램이 시작되진 않음

  ⇒ 다 default 값이 설정되기 때문!

- 기본 값 제공을 위해선 아래 가정이 필요함
    1. 어떤 기본 값이 합리적인지
    2. 더 상위 계층 코드는 기본값이든 명시적으로 설정된 값이든 상관하지 않음

### 낮은 층위 코드의 기본 반환 값은 재사용성을 해칠 수 있다

---

- 하위 계층 코드는 상위 계층에 영향을 끼침
- 보통 낮은 층위 코드는 근본적인 하위 문제를 해결하는 데 더 많이 사용됨
    - 따라서 상위 계층 문제를 해결하기 위해 재사용 될 가능성이 높음
- 하위 계층 코드의 가정은 그 위에 있는 모든 상위 계층의 코드들에 영향을 끼침
- 기본 값을 정의하는 가정의 코드 계층이 낮을수록, 해당 가정이 적용되는 상위 계층은 더 많아짐

### Solution 1. 상위 수준 코드에서 기본 값을 제공하라

---

- 사용자가 제공한 값이 없을 때 null 값을 반환하는 것이 더 좋은 설계가 될 수 있음
- 기본 값을 반환 하던 기존 코드
    - ARIAL 폰트가 지정된 기본 값이었는지, 사용자가 선택한 값이었는지 알 수 없음

    ```java
    class UserDocumentSettings{
    	private final Font font;
    	...
    	Font getPreferredFont(){
    		if(font != null){
    			return font;
    		}
    		
    		return Font.ARIAL; // 기본 값 반환 코드 
    	}
    }
    ```

- null을 리턴하면 호출한 쪽에서 null을 처리할 수 있음 → 상위 계층에서 처리하도록 두기

    ```java
    class UserDocumentSettings{
    	private final Font font;
    	...
    	Font getPreferredFont(){
    		return font; 
    	}
    }
    ```

- 출석에서도 비슷한 경험 했음!!

    ```java
    int hour = -1;
    int minute = -1;
    
    vs
    
    Integer hour = null;
    Integer minute = null;
    ```


- **널 병합 연산**
    - `nullableValue ?? defaultValue`
    - nullableValue가 null이면 defaultValue를 갖게 됨
    - 보통 C#, JS, Swift에서 사용 가능한 연산임

        ```java
        // 사용자 선호 언어가 없을 경우, 기본 세팅의 font를 가져옴
        Font getFont(){
        	return userSettings.getPreferredFont() ?? defaultSettings.getFont();
        }
        ```


- 어떤 기본 값을 사용할지에 대한 결정을 호출하는 쪽에서 하도록 함

  ⇒ 코드 재사용성 향상

    - 널 병합 연산을 지원하지 않는 연산에서는 널 처리 위한 반복적인 코드 작성이 필요해짐
    - Java도 Map에서 비슷한 기능을 제공함

        ```java
        String value = map.getOrDefault(key,"default");
        ```

        - 호출한 쪽에서 별도의 널 처리 필요 없이 적절한 기본 값 결정 가능

- 기본 값을 어느 레벨에서 사용할지는 고민해보고 사용해야 한다!

## 9.4 함수의 매개변수를 주목하라

---

- 하나의 클래스에 대해 필드가 여러 개 존재할 때, 모든 필드가 있어야 하는 경우에는 생성자가 객체나 클래스의 인스턴스를 매개변수로 받는 것이 타당함

    ```java
    Setting{
    	Font font;
    	int fontSize;
    	Color color;
    }
    
    void setSetting(Setting setting){
    	setFont(setting.getFont());
    	setFontSize(setting.getFontSize());
    	setColor(setting.getFontColor());
    }
    ```

- 하지만 한 두 가지 정보만 필요로 할 때는 객체나 클래스의 인스턴스를 매개변수로 사용하는 것은 코드 재사용성을 해칠 수 있음

    ```java
    void setFont(Setting setting){
    	this.font = setting.getFont();
    }
    
    vs
    
    void setFont(Font font){
    	this.font = font;
    }
    ```


### 필요 이상으로 매개변수를 받는 함수는 재사용하기 어려울 수 있다

---

- 아래와 같은 텍스트 옵션 클래스가 있다고 가정한다

```java
class TextOptions{
	private final Font font;
	private final Double fontSize;
	private final Color textColor;
	...
}
```

- 이 때 아래 같이 `textColor` 만 사용하게 되면 `font`, `fontSize`는 불필요하지만 존재 해야 하는 정보들이 됨 (메서드 파라미터로 `options` 인스턴스를 넘겨줘야 하기 때문)

```java
class TextBox{
	private final Element textContainer;
	...
	void setTextColor(TextOptions options){ // 메서드 파라미터로는 TextOptions를 받지만
		textContainer.setStyleProperty( // 실제 세팅되는 값은 textColor만 사용됨
			"color", options.getTextColor().asHexRgb()
		);
	}
}
```

- 빨간색으로 설정하고싶으면 `font`, `fontSize`는 관련 없는 값들로 의무적으로 초기화 해서 `TextOptions` 객체를 만들어야 함

```java
TextOptions style = new TextOptions(
	Font.ARIAL,
	12.0,
	14.0,
	Color.RED
);
textBox.setTextColor(style);
```

### Solution 1. 함수는 필요한 것만 매개변수로 받도록 하라

---

- 기존에 `TextOptions` 받던 것을 `Color` 로 받도록 수정하는 것이 재사용성에 좋다

```java
void setTextColor(Color color){
	textElement.setStyleProperty("color", color.asHexRgb());
}
```

```java
textBox.setTextColor(Color.RED);
```

## 9.5 제네릭 사용을 고려하라

---

- 다른 클래스를 참조하는 코드를 작성할 때 해당 클래스가 어떤 클래스인지 신경 쓰지 않는다면 제네릭 사용 고려 필요

### 특정 유형에 의존하면 일반화를 제한한다

---

### Solution 1. 제네릭을 사용하라

---

- 랜덤 문제가 문자열로 표현될 수 있는 단어를 저장하는 클래스일 경우

```java
class QuizQueue{
	private final List<String> values = new ArrayList<>();
		
	void add(String value){
		values.add(value);
	}
	
	// 큐로부터 한 항목을 삭제하고 이를 반환함
	String getValue(){
		return values.removeFirst();
	}
}
```

- 제네릭을 사용해 단어 문제나 사진 문제처럼 문제 목록을 변경할 수 있음

```java
class QuizQueue<T>{
	private final List<T> values = new ArrayList<>();
	
	void add(T value){
		values.add(value);
	}
	
	// 큐로부터 한 항목을 삭제하고 이를 반환함
	T getValue(){
		return values.removeFirst();
	}
}
```

- 제네릭을 사용해서 아래처럼 퀴즈 큐 생성 코드를 재활용할 수 있음