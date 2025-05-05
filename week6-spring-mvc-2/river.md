# 레이어드 아키텍처

레이어드 아키텍처(Layered Architecture)란 소프트웨어 시스템을 관심사에 따라 여러 계층으로 나누고, 이를 수직적으로 구성한 아키텍처이다. 일반적으로 Spring Boot에서는 Controller Layer, Service Layer, Repository Layer의 3계층 구조를 따르며, 경우에 따라 Presentation Layer, Application Layer, Domain Layer, Infrastructure Layer로 구성된 4계층 구조를 사용하기도 한다.

이 아키텍처의 핵심은 관심사의 분리를 통해 각 계층의 역할을 명확히 하고, 전체 시스템의 결합도를 낮추는 것에 있다. 계층 간 결합을 줄이기 위해 모듈화를 강조하지만, 계층 간 데이터 전달 시 DTO 변환이나 캐스팅과 같은 부가적인 처리가 필요해 일정 수준의 오버헤드가 발생할 수 있다.

## 각 계층의 역할

- **Presentation Layer(표현 계층)**: 사용자 인터페이스(UI)와 직접 상호작용하는 계층으로, 일반적으로 Controller가 이 역할을 담당한다. 사용자의 요청을 받아 필요한 Service를 호출하고, 처리 결과를 사용자에게 전달할 DTO를 만들어 응답하는 역할을 한다. 비즈니스 로직 자체는 수행하지 않는다.
- **Service Layer(응용 계층)**: 애플리케이션의 핵심 비즈니스 로직을 처리하는 계층이다. 전달받은 데이터를 기반으로 비즈니스 규칙을 실행하고, 필요한 경우 Repository를 호출해 DB와 상호작용한다. 비즈니스 로직이 복잡해질 경우 다른 서비스를 호출하기도 하며, 도메인 객체를 직접 생성하고 도메인 로직을 호출하기 때문에, 4계층 구조에서는 Domain Layer의 일부로 간주되기도 한다.
- **Repository Layer(데이터 계층)**: 데이터베이스 혹은 기타 저장소와 상호작용하는 계층이다. 주로 도메인 객체의 CRUD 작업을 수행하며, 쿼리를 통해 데이터를 조작한다. 기술적으로는 Infrastructure Layer에 속한다고 볼 수 있으나, 도메인 객체의 저장과 조회를 책임진다는 점에서 Domain Layer에 포함되기도 한다.

# 레이어드 아키텍처에서 DTO

## Domain을 어느 계층까지 노출해도 될까?

- DTO와 Domain(Entity) 간의 변환 작업은 어디에서 수행되어야 할까?
- DTO를 어느 레이어까지 전달해서 사용해야 할까?

## Repository

> *”…a cohesive set of responsibilities for providing access to the roots of AGGREGATES from early life cycle through the end” - Eric Evans*
> 

Repository는 Domain 객체 중 Aggregate Root(Domain 객체들의 그룹에서 중심이 되는 엔티티)에 일관되고 응집력 있는 방식으로 접근할 수 있도록 책임지는 구성 요소이며, 객체의 생성부터 소멸까지 전 생명주기를 관리하는 역할을 한다. 따라서 Repository 레이어는 Entity의 영속성을 관장하는 역할이기 때문에 표현 계층에서 사용할 객체(DTO)들을 Repository에 사용하는 것을 지양해야 한다.

## Service

### Service가 Domain Model 반환

Service가 Domain Model을 Controller에 반환하고, Controller가 Entity를 DTO로 변환하는 경우 몇 가지 문제가 발생할 수 있다. 우선 View에 반환할 필요가 없는 데이터까지 Domain 객체에 포함되어 표현 계층(Controller)으로 전달된다. Controller가 여러 Domain 객체를 직접 조회하고 이를 가공하여 DTO를 생성하는 과정에서 원래 Service 계층이 맡아야 할 비즈니스 로직이 Controller로 이동하게 되어, 계층 간 책임이 모호해질 수 있다. 또한 여러 Domain 객체를 조회해야 하므로 하나의 Controller가 지나치게 많은 Service에 의존하게 되는 문제가 발생할 수 있다.

### Service가 DTO 반환

> *A Service Layer defines an application’s boundary [Cockburn PloP] and its set of available operations from the perspective of interfacing client layers. It encapsulates the application’s business logic, controlling transactions and coor-dinating responses in the implementation of its operations. - Martin Fowler*
> 

마틴 파울러는 Service를 애플리케이션의 경계를 정의하고 비즈니스 로직 등 Domain을 캡슐화하는 역할로 정의한다. 즉, Service는 Domain을 보호하는 계층이며, Domain Model을 표현 계층(Controller)에서 직접 사용하는 경우 레이어 간 결합도가 높아지고, Domain 변경 시 Controller까지 함께 수정해야 하는 문제가 발생할 수 있다. 따라서 레이어 간 데이터를 전달하기 위한 DTO를 사용하는 것이 적절하며, 요청에 대한 응답 처리는 Service의 책임에 속하므로 DTO는 Service 레이어에서 정의되고 반환되는 것이 바람직하다.

## DTO-Entity 변환 위치

1. **Controller에서 받은 DTO를 Repository까지 사용**
    
    Controller에서 받은 DTO를 Repository까지 사용한다면 DTO는 필요한 정보만을 담고 있어 데이터를 주고받는 절차가 간단하고 명확해지지만, Controller와 Service, Repository가 강하게 결합되어 있어 API가 변경된다면 Controller, Service, Repository를 모두 수정해야 한다는 문제가 생길 수 있다.
    
2. **Service에서 Domain 객체로 변환 후 Repository로 전달**
    
    Service에서 DTO를 Domain으로 변환하고 Repository는 Domain 객체만을 주고받도록 구성하면, Repository는 별도의 DTO ↔ Entity 변환 없이 DB 작업에만 집중할 수 있어 역할이 명확해지고 책임이 분리된다. 이로 인해 Service는 전체적인 비즈니스 흐름을 관리하고 Repository는 온전히 데이터 저장소와의 소통을 담당하게 되어 구조적인 이점을 가지지만, Service가 Controller에 종속되는 형태가 되기 때문에 Controller가 어떤 데이터를 전달하느냐에 따라 Service가 받는 데이터 형태가 결정되고, 이로 인해 Service가 원하는 방식으로 유연하게 정보를 처리하지 못하는 한계가 생길 수 있다.
    
3. **Client, Controller, Service 간에 별도의 DTO 사용**
    
    Client와 Controller, Controller와 Service 간에 데이터를 전달할 때, 각 계층에 맞는 별도의 DTO를 사용하는 방식은 계층 간의 의존도를 줄이고 역할을 명확하게 분리할 수 있다. 다만 다음 레이어로 데이터를 전달할 때 계층마다 DTO를 구분하면 DTO ↔ DTO 간 변환 로직이 추가되어 구조가 복잡해지고 오버엔지니어링이 될 수 있으므로 이와 같은 설계는 애플리케이션의 복잡도와 기능 요구사항을 충분히 고려한 뒤 선택해야 한다.
    
4. **Controller에서 Mapper를 사용해 DTO ↔ Entity 변환**
    
    Controller에서 Client 요청을 받아 처리할 때 DTO와 Entity 간 변환이 자주 발생하는데 변환을 Controller 내부에서 처리하게 되면 코드가 복잡해지고 표현 계층이 Domain 객체에 대한 너무 많은 책임을 지게된다. 이러한 문제는 Mapper 클래스를 통해 해결할 수 있다. Mapper는 DTO와 Entity 간 변환을 담당하는 클래스로 변환 책임을 분리해 Controller는 요청과 응답에만 집증할 수 있다. 다만 Mapper에서 변환한 Entity는 아직 DB에 저장되지 않았기 때문에 ID가 없는 불완전한 상태일 수 있다. 특히 Entity 생성이 단순히 데이터 매핑이 아니라 복잡한 비즈니스 로직을 수행해야 하는 경우 이를 Controller에서 수행하는 것은 바람직하지 않고, 이런 경우 Entity 생성 책임은 Service 계층으로 옮기고 Mapper는 단순한 변환 도구로 사용해도 좋다.
