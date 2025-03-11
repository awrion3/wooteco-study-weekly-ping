## 8. 코드를 모듈화해라

### 의존성 주입의 사용을 고려하라.

- 하드 코드화된 의존성은 문제가 될 수 있다.

```java
class RoutePlanner {
	private final RoadMap roadMap;
	
	RoutePlanner() {
		this.roadMap = new NorthAmericaRoadMap();
	}
}
```

❓ 만약 NorthAmericaRoadMap 생성자에 매개변수가 필요하다면???

- 인터페이스에 의존하라
    - 구체적인 구현에 의존하면 적응성이 제한된다.

```java
interface RoadMap {
	List<Road> getRoads();
	List<Junction> getJunctions();
}

class NorthAmericaRoadMap implements RoadMap {}

class RoutePlanner {
	private final NorthAmericaRoadMap roadMap;
	
	RoutePlanner(NorthAmericaRoadMap roadMap) {
		this.roadMap = roadMap;
	}
}
```

🤔 의존성 역전 원리

DIP 원칙 → 객체에서 어떤 Class를 참조해서 사용해야하는 상황이 생긴다면, 그 Class를 직접 참조하는 것이 아니라 그 대상의 상위 요소(추상 클래스, 인터페이스)로 참조하라.

### 의존성 주입의 사용을 고려하라.

- 클래스 상속은 문제가 될 수있다.

```java
interface FileValueReader {
	String getNextValue();
	void close();
}

class CsvFileHandler implements FileValueReader{

	CsvFileHandler(File file) {}
	override String getNextValue();
	override void close();
}

class IntFileReader extends CsvFileHandler {
	IntFileReader(File file) {
		super(file);
	}
	
	Int getNextInt() {
		String nextValue = getNextValue();
		if(nextValue == null) {
			reutrn null;
		}
		return Int.parse(nextValue);
	}
}
```

❌상속은 추상화 계층에 방해가 될 수 있다 → 내 의도는 IntFileReader는 CsvFileHanlder의 getNextValue를 사용하여 Integer를 반환하도록 함. 하지만 IntFileReader 메서드에는 CsvFileHandler 메서드가 모두 노출되어 있음

❌상속은 적응성 높은 코드의 작성을 어렵게 만들 수 있다. → ? 만약 위 코드에서 쉼표 기준이 아니라 세미콜론 기준이 추가되었다. 근데 다른 개발자가 CsvFileHandler와 마찬가지로 SemicolonFileHandler를 만들었다.

```java
class SemicolonFileHandler implements FileValueReader {

	SemicolonFileHandler(File file) {}
	override String getNextValue();
	override void close();
}

class SemicolonIntFileReader extends SemicolonFileHandler {
	SemicolonIntFileReader(File file) {
		super(file);
	}
	
	Int getNextInt() {
		String nextValue = getNextValue();
		if(nextValue == null) {
			reutrn null;
		}
		return Int.parse(nextValue);
	}
}
```

💡구성을 사용해 해결

```java
class IntFileReader {
	private final FileValueReader valueReader;
	
	IntFileReader(FileValueReader valueReader) {
		this.valueReader = valueReader;
	}
	
	Int getNextInt() {
	 String nextValue = valueReader.getNextValue();
	 if(nextValue == null) {
		 return null;
		}
		return Int.parse(nextValue);
	}
	
	void close() { // 포워딩(전달)
		valueReader.close();
	}
}
```

### 클래스는 자신의 기능에만 집중해야한다.

- 다른 클래스와 지나치게 연관되어 있으면 문제가 될 수 있다.
- 추후 변경 사항이 나오면 하나만 수정하게끔 만들자.

```java
class Book {
	private final List<Chapter> chapters;
	
	Int wordCount() {
		return chapters
                .map(getChapterWordCount())
                .sum();
	}
	
	private Int getChapterWordCount(Chapter chapter) {
		return chapter.getPrelude().wordCount() +
			chapter.getSections()
				.map(section -> section.wordCount())
				.sum();
	}
}

class Chapter {
	private final TextBlock textBlock;
	private final List<TextBlock> sections;
	
	TextBlock getPrelude() {...}
	List<TextBlock> getSections() {...}
}
```

🤔디미터의 법칙 → 한 객체가 다른 객체의 내용이나 구조에 대해 가능한 한 최대한으로 가정하지 않아야한다. 즉, 다른 객체가 어떠한 자료를 갖고 있는지 속사정을 몰라야 한다는 것을 의미한다.

예) Book 클래스는 Chapter와만 상호작용하는 것이 좋다.

### 관련 있는 데이터는 함께 캡슐화해라

- 캡슐화되지 않은 데이터는 취급하기 어렵다.

```java
class TextBox {
	...
	void renderText( -> 특정 문자를 문자 꾸밈과 함께 출력하는 기능
		String text,
		Font font,
		Double fontSize,
		Double lineHeight,
		Color textColor) { ... }
}

class UiSettings {
	...
	Font getFont();
	Double getFontSize();
	Double getLineHeight();
	Color getTextColor();
}

class UserInterface {
	private final TextBox messageBox;
	private final UiSettings uiSettings;
	
	void displayMessage(String message) {
		messageBox.renderText( -> displayMessage 함수는 UiSettings 클래스의 일부 정보를 renderText 함수로 전달한다.
			message,
			uiSettings.getFont(),
			uiSettings.getFontSize(),
			uiSettings.getLineHeight(),
			uiSettings.getTextColor());
	}
}
```

해결 방안

```java
class TextOptions {
	 private final Font font;
	 private final Double fontSize;
	 private final Double lineHeight;
	 private final Color textColor;
}

class UiSettings {
	...
	TextOptions getTextStyle()
}

class TextBox {
	...
	void renderText(String text, TextOptions textStyle) {...}
}

class UserInterface {
	private final TextBox messageBox;
	private final UiSettings uiSettings;
	
	void displayMessage(String message) {
		messageBox.renderText(
			message,
			uiSettings.getTextStyle()
	}
}
	 
```

### 반환 유형에 구현 세부 정보가 유출되지 않도록 주의하라

- 반환 형식에 구현 세부 사항이 유출될 경우 문제가 될 수 있다.

프로필 사진 데이터를 가져오는 HttpFetcher를 사용해 이루어진다.

```java
class ProfilePictureService {
	private final HttpFetcher httpFetcher;
	...
	
	ProfilePictureResult getProfilePicture(Int userId) {...}
}

class ProfilePictureResult {
	HttpResponse.Status getStatus() {...}
	HttpResponse.Payload getImageData() {...}
}
```

❌문제점 : ProfilePictrueService가 HTTP 연결을 사용하여 프로필을 가져온다는 사실을 유출한다.

- 다른 개발자가 ProfilePictureService를 사용하려면 HttpResponse와 같은 개념을 처리해야한다.
- ProfilePictrueService의 구현을 변경하는 것이 어렵다. ProfilePictureService.getProfilePicture)를 호출하는 코드는 함수의 반환 값을 처리하기 위해 status, payload를 다뤄야한다. 하지만 만약 http가 아닌 웹소켓으로 바꾸고 싶다면 해당 메서드를 사용하는 모든 코드를 수정해야한다.

💡해결방안

- HttpResponse.Statuts 열거형을 사용하는 대신 사용자 지정 열거형을 사용하자.
- HttpResponse.Payload를 반환하는 대신 바이트 리스트를 반환하자.

```java
class ProfilePictureResult {
	...
	
	enum Status {
		SUCCESS,
		USER_DOES_NOT_EXIST,
		OTHER_ERROR
	}
	
	Status getStatus() {...}
	
	List<Byte> getImageData() {...}
}
```

### 반환 유형에 구현 세부 정보가 유출되지 않도록 주의하라

- 예외 처리 시 구현 세부 사항이 유출되면 문제가 될 수 있다.

```java
class TextSummarizer {
	private final TextImportanceScorer importanceScorer;
	...
	String summarizeText(String text) {
		return paragraphFinder.find(text)
						.filter( p -> importanceScorer.isImportant(p))
						.join("\n\n");
	}
}

interface TextImportanceScorer {
	Boolean isImportant(String text);
}

class ModelBasedScorer implements TextImportanceScorer {
	private final Model model;
	override Boolean isImportant(String text) { -> PredictionModelExcpetion 언체크 예외 발생
		return model.predict(text) >= MODEL_THRESHOLD;
	}
}

void updateTextSummary(UserInterface ui) {
	String userText =  ui.getUserText();
	try {
		String summary = textSummarizer.summarizeText(userText);
		ui.getSummaryField().setValue(summary);
	} catch (PredictionModelException e) {
		ui.getSummaryField().setError("Unable to summarize text");
	}
```

❌문제점 : 현재 예외로 Model를 기준으로 스코어가 계산되는지 예측이 가능하다. 또한 ModelBasedScorer가 아닌 다른 구현체에서 예외가 발생한 경우 해당 예외에 맞게 catch 해야한다.

💡 해결책 : 추상화 계층에 적절한 예외를 만들라

```java
class TextSummarizerException extends Exception {
	TextSummarizerException(Throwable cause) {...}
}

class TextImportanceScoreException extends Exception {
	TextImportanceScoreException(Throwable cause) {...}
}

class TextSummarizer {
	private final TextImportanceScorer importanceScorer;
	...
	String summarizeText(String text) {
		try{
			return paragraphFinder.find(text)
							.filter( p -> importanceScorer.isImportant(p))
							.join("\n\n");
		} catch (TextImportanceScoreException e) {
			throw new TextSummarizeException(e);
		}
	}
}

class ModelBasedScorer implements TextImportanceScorer {
	
	override Boolean isImportant(String text) {
		try{
			return model.predict(text) >= MODEL_THRESHOLD;
		} catch (PredictionModelException e) {
			throw new TextImportanceScorerException(e);
		}
	}
}

void updateTextSummary(UserInterface ui) {
	String userText =  ui.getUserText();
	try {
		String summary = textSummarizer.summarizeText(userText);
		ui.getSummaryField().setValue(summary);
	} catch (TextSummarizerException e) {
		ui.getSummaryField().setError("Unable to summarize text");
	}
```