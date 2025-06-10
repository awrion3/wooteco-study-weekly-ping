# API 문서 자동화 필요성

## 문제점

- 비효율적인 커뮤니케이션
- Production 코드와의 불일치
- 문서에 대한 신뢰성 문제

---

## 도구

### Swagger

- **정의**
    - RESTful API를 설계, 문서화, 시각화, 테스트할 수 있게 돕는 OpenAPI Spec 기반 도구 모음
- **OpenAPI Spec**
    - REST API 자체를 정의하는 표준 포맷 (YAML 또는 JSON)
    - API의 경로, 파라미터, 응답 등을 기계가 읽을 수 있는 구조로 표현
- **장점**
    - Annotation 기반으로 간편하게 문서 자동 생성 (@Operation, @ApiResponse 등)
    - 별도 테스트 필요 없이 화면 기반 인터페이스로 API 확인 및 테스트 가능
- **단점**
    - 문서 생성을 위해 Production 코드에 애노테이션이 다수 삽입되어 코드 혼잡성 증가
    - 애노테이션만으로 생성되기 때문에 실제 API와 불일치하는, 검증되지 않은 API 문서 생성 가능

### REST Docs

- **정의**
    - 테스트 코드 기반으로 문서 조각(Snippet)을 자동 생성하고, 이를 조합해 최종 RESTful 문서 생성하는 도구
- **핵심 원리**
    - MockMvc, RestAssured 등을 활용한 테스트 수행 중 HTTP 요청/응답 내용을 캡처
    - 해당 내용을 .adoc 조각 파일(Snippet)로 저장
        - 테스트로 생성된 snippet을 사용해서 snippet이 올바르지 않으면 생성된 테스트가 실패해 정확성 보장
    - Asciidoctor를 통해 메인 문서(index.adoc 등)와 함께 HTML, PDF 문서로 렌더링
- **Asciidoctor**
    - .adoc 문법으로 작성된 문서를 HTML, PDF, DocBook 등으로 변환하는 텍스트 프로세서
- **Snippet**
    - Spring REST Docs에서 API 문서화에 필요한 요청/응답/본문 필드 등의 문서 조각
    - 테스트 코드 실행 시 build/generated-snippets/ 아래에 .adoc 형식의 조각 파일들이 자동 생성
    - 이 조각들은 index.adoc 같은 메인 문서에서 include:: 지시어로 포함되어 사용
    - 최종적으로 Asciidoctor에 의해 .html 문서로 변환되는 대상은 메인 문서이고, 조각은 그 일부로 포함
- **장점**
    - 테스트가 성공해야 문서를 생성하기 때문에 실제 동작과 일치하는 최신화 강제
    - API 명세가 테스트를 기반으로 해야 하기 때문에 신뢰도가 높음
    - 문서 생성을 위해 Production 코드에 영향 없음
- **단점**
    - 문서 생성을 위한 테스트 작성 비용이 높음 (요청/응답 모두 상세히 기술 필요)
    - Swagger에 비해 시각적인 UI 제공이 부족하고, 설정이 다소 복잡
    - 학습 자료 및 레퍼런스가 Swagger에 비해 적음

### Swagger & REST Docs

- Swagger와  REST Docs를 동시에 사용 가능
    - REST Docs를 OpenAPI Spec으로 변환 후 Swagger UI에서 사용 가능
    - https://github.com/ePages-de/epages-docs

---

# REST Docs

## REST Docs 환경 설정

### build.gradle 설정

- **코드**
    - build.gradle
        
        ```bash
        plugins {
            id 'org.springframework.boot' version '3.4.4'
            id 'io.spring.dependency-management' version '1.1.7'
            id 'org.asciidoctor.jvm.convert' version '4.0.2'
            id 'java'
        }
        
        group = 'nextstep'
        version = '0.0.1-SNAPSHOT'
        
        java {
            sourceCompatibility = JavaVersion.VERSION_21
            targetCompatibility = JavaVersion.VERSION_21
        }
        
        repositories {
            mavenCentral()
        }
        
        configurations {
            asciidoctorExt
        }
        
        dependencies {
            // Spring Boot starters
            implementation 'org.springframework.boot:spring-boot-starter'
            implementation 'org.springframework.boot:spring-boot-starter-web'
            implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
            implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
            implementation 'org.springframework.boot:spring-boot-starter-validation'
        
            // JWT
            implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
            runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
            runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
        
            // DB
            runtimeOnly 'com.h2database:h2'
        
            // REST Docs
            asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
            testImplementation 'org.springframework.restdocs:spring-restdocs-restassured'
            testImplementation 'io.rest-assured:rest-assured:5.3.1'
        
            // Testing
            testImplementation 'org.springframework.boot:spring-boot-starter-test'
        }
        
        ext {
            set('snippetsDir', file("build/generated-snippets"))
        }
        
        test {
            outputs.dir snippetsDir
            useJUnitPlatform()
        }
        
        asciidoctor {
            configurations 'asciidoctorExt'
            sources {
                include 'index.adoc'
            }
            baseDirFollowsSourceFile()
            inputs.dir snippetsDir
            dependsOn test
        }
        
        tasks.register('createDocument', Copy) {
            doFirst {
                delete file('src/main/resources/static/docs')
            }
            dependsOn asciidoctor
            from file("build/docs/asciidoc")
            into file("src/main/resources/static")
        }
        
        bootJar {
            dependsOn createDocument
            from("${asciidoctor.outputDir}") {
                into 'static/docs'
            }
        }
        ```
        
- **코드 설명**
    - Asciidoctor 파일을 컨버팅하고 Build 폴더에 복사하기 위한 플러그인 추가
        
        ```bash
        plugins {
            id 'org.springframework.boot' version '3.4.4'
            id 'io.spring.dependency-management' version '1.1.7'
            id 'org.asciidoctor.jvm.convert' version '4.0.2'
            id 'java'
        }
        ```
        
    - asciidoctorExt를 configurations으로 지정
        
        ```bash
        configurations {
            asciidoctorExt
        }
        ```
        
    - .adoc 파일에서 사용할 snippets 속성이 자동으로 build/generated-snippets를 가리키도록 지정
        
        ```bash
        dependencies {
            ...
            
            // REST Docs
            asciidoctorExt 'org.springframework.restdocs:spring-restdocs-asciidoctor'
            testImplementation 'org.springframework.restdocs:spring-restdocs-restassured'
            testImplementation 'io.rest-assured:rest-assured:5.3.1'
            
        		...
        }
        ```
        
    - snippets 파일이 저장될 경로를 snippetsDir로 변수 설정
        
        ```bash
        ext {
            set('snippetsDir', file("build/generated-snippets"))
        }
        ```
        
    - 출력할 디렉터리를 snippetsDir로 설정
        
        ```bash
        test {
            outputs.dir snippetsDir
            useJUnitPlatform()
        }
        ```
        
    - Asciidoctor 설정
        
        ```bash
        asciidoctor {
            configurations 'asciidoctorExt'
            sources {
                include 'index.adoc'
            }
            baseDirFollowsSourceFile()
            inputs.dir snippetsDir
            dependsOn test
        }
        ```
        
        - Asciidoctor에서 asciidoctorExt 설정 사용
        - src/docs/asciidoc/index.adoc 파일 하나만 문서 변환 대상에 포함 설정
        - .adoc 파일에서 다른 .adoc을 include 해 사용하는 경우 경로를 동일한 경로를 baseDir로 동일하게 설정
        - input 디렉터리를 snippetsDir로 설정
        - Gradle 빌드시 test가 성공해야지 asciidoctor 순으로 진행
    - 문서 생성 설정
        
        ```bash
        
        tasks.register('createDocument', Copy) {
            doFirst {
                delete file('src/main/resources/static/docs')
            }
            dependsOn asciidoctor
            from file("build/docs/asciidoc")
            into file("src/main/resources/static")
        }
        ```
        
        - 실행할 task를 createDocument로 정의하고, type을 복사로 정의
        - 문서를 생성할 때 처음으로 해당 경로에 있는 파일 삭제
        - Gradle 빌드시 asciidoctor가 성공해야지 createDocument 순으로 진행
        - from에 위치한 파일들을 into로 복사
    - bootJar 설정
        
        ```bash
        
        bootJar {
            dependsOn createDocument
            from("${asciidoctor.outputDir}") {
                into 'static/docs'
            }
        }
        ```
        
        - Gradle 빌드시 createDocument가 성공해야지 bootJar 순으로 진행
        - Gradle 빌드시 asciidoctor.outputDir에 Html 파일이 생기고 jar 안에 /resources/static 폴더에 복사
    - build.gradle 실행 흐름

---

## build.gradle 실행 흐름

### test 실행

- **REST Docs 조각 생성 (.adoc)**
    - Spring REST Docs 테스트들이 실행됨 (RestAssured, MockMvc 등 사용 가능)
    - .adoc 문서 조각(snippet)이 생성됨
        - 생성 위치: build/generated-snippets/
    - 테스트가 실패하면 → asciidoctor는 실행되지 않음
    - 실행 결과 예시
        
        ```bash
        build/generated-snippets/
        └── reservation-create/
            ├── http-request.adoc
            ├── http-response.adoc
            ├── request-fields.adoc
            └── response-fields.adoc
        ```
        

### **asciidoctor 실행**

- **조각 + index.adoc → HTML 생성**
    - index.adoc 등의 메인 문서를 기반으로, snippets 조각들을 include::로 불러옴
    - 최종적으로 HTML 문서를 생성
    - 실행 결과 예시
        - .adoc + snippets를 조합해서 HTML 문서로 변환
        
        ```bash
        build/docs/asciidoc/index.html
        ```
        

### **createDocument 실행**

- **HTML 문서를 static/docs로 복사**
    - 기존 문서는 먼저 삭제해서 최신화
    - asciidoctor가 만든 HTML 문서를 정적 리소스(static) 폴더로 복사
    - 앱을 실행하면 /docs/index.html으로 접근 가능하게 됨
    - 복사 대상
        
        ```bash
        src/main/resources/static/docs/index.html
        ```
        

### bootJar 실행

- **앱 + 문서를 실행 가능한 JAR로 패키징**
    - 실행 가능한 Spring Boot JAR 파일을 생성함
    - /static/docs/index.html이 포함된 상태로 빌드됨
    - java -jar build/libs/app.jar로 실행 시 API 문서도 함께 제공 가능
    - 포함 위치 (JAR 내부)
        
        ```bash
        BOOT-INF/classes/static/docs/index.html
        ```