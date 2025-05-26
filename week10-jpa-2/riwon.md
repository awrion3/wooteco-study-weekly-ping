## CascadeType (+ orphanRemoval)

### Cascade 옵션이란?

JPA에서는 어떠한 엔티티를 영속 상태로 만들고, 이와 연관된 엔티티도 함께 영속 상태로 만들고자 할 때 영속성 전이를 사용합니다.
즉 부모 엔티티를 통해 자식 엔티티까지 영향을 줄 수 있습니다.

```java
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @OneToMany(mappedBy = "book", fetch = FetchType.LAZY, cascade = CascadeType.REMOVE) // 영속성 전이 옵션
    private List<Page> pages = new ArrayList<>();
}
```

- Cascade의 6가지 옵션:

| CascadeType | 설명                                              | 적용 시점(부모 → 자식)                     |
| ----------- | ----------------------------------------------- | ---------------------------------- |
| **ALL**     | PERSIST, MERGE, REMOVE, REFRESH, DETACH를 모두 전파  | 모든 엔티티 상태 변화 전파                    |
| **PERSIST** | `EntityManager.persist()` 호출 시 자식도 함께 `persist` | 새 엔티티 저장 시 연관 자식도 저장               |
| **MERGE**   | `EntityManager.merge()` 호출 시 자식도 함께 `merge`     | 준영속(detached) 상태의 부모를 병합할 때 자식도 병합 |
| **REMOVE**  | `EntityManager.remove()` 호출 시 자식도 함께 `remove`   | 부모 삭제 시 연관 자식도 삭제                  |
| **REFRESH** | `EntityManager.refresh()` 호출 시 자식도 함께 `refresh` | DB에 있는 최신 상태로 갱신할 때 자식도 갱신         |
| **DETACH**  | `EntityManager.detach()` 호출 시 자식도 함께 `detach`   | 영속성 컨텍스트 분리 시 자식도 준영속 상태로          |

### CascadeType.PERSIST

CascadeType.PERSIST는 부모와 자식 엔티티를 한 번에 영속화할 수 있습니다. 
즉 아래의 코드에서 부모인 Book을 영속화 했을 경우, Book에 담긴 자식 Page들도 함께 영속화가 됩니다.

> 부모 엔티티

```java
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @OneToMany(mappedBy = "book", fetch = FetchType.LAZY, cascade = CascadeType.REMOVE)
    private List<Page> pages = new ArrayList<>();

    public void addPage(Page page) {
        pages.add(page);
    }
}
```

> 자식 엔티티

```java
@Entity
public class Page {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String content;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(nullable = false)
    private Book book;
}
```

### CascadeType.REMOVE

위에서 PERSIST로 영속화 했던 부모와 함께, 자식 엔티티들도 모두 제거하고자 할 때에는 CascadeType.REMOVE를 사용하면 됩니다.

```java
//...
@Autowired
private BookRepository bookRepository;
@Autowired
private PageRepository pageRepository;

void cascade_type_remove() {
    // given
    Page page1 = new Page();
    Page page2 = new Page();

    Book book = new Book();

    book.addPage(page1);
    book.addPage(page2);

    bookRepository.save(book);

    // when
    bookRepository.delete(book);

    // then
    List<Book> books = bookRepository.findAll();
    List<Page> pages = pageRepository.findAll();

    assertThat(books).hasSize(0);
    assertThat(pages).hasSize(0);
}
```

이처럼 CascadeType.REMOVE로 설정하면, 부모 엔티티가 삭제됨에 따라 자식 엔티티도 같이 삭제됩니다.
다시 말해 부모 엔티티가 자식 엔티티의 삭제 생명 주기를 관리하게 되는 것입니다.

```genericsql
Hibernate: 
    insert 
    into
        book
        (id, title) 
    values
        (null, ?)
Hibernate: 
    insert 
    into
        page
        (id, content, book_id) 
    values
        (null, ?, ?)
Hibernate: 
    insert 
    into
        page
        (id, content, book_id) 
    values
        (null, ?, ?)

Hibernate: 
    delete 
    from
        page 
    where
        id=?
Hibernate: 
    delete 
    from
        page
    where
        id=?
Hibernate: 
    delete 
    from
        book 
    where
        id=?
```

따라서 위와 같이 부모 엔티티 삭제와 더불어 자식 엔티티도 삭제됨에 따라, 총 3번의 delete 쿼리가 나가게 됩니다.

### CascadeType.REMOVE vs orphanRemoval = true

다만 CascadeType.REMOVE의 경우 아래처럼 부모 엔티티가 자식 엔티티와의 관계를 제거할 시, 해당 자식 엔티티는 삭제되지 않습니다.

```java
//...
@Autowired
private BookRepository bookRepository;
@Autowired
private PageRepository pageRepository;

void cascade_type_remove_page() {
    // given
    Page page1 = new Page();
    Page page2 = new Page();

    Book book = new Book();

    book.addPage(page1);
    book.addPage(page2);

    bookRepository.save(book);

    // when
    book.getPages().remove(0);     // 부모 엔티티 Book pages에서 자식 엔티티 첫 Page와의 관계를 제거합니다. 

    // then
    List<Book> books = bookRepository.findAll();
    List<Page> pages = pageRepository.findAll();

    assertThat(books).hasSize(1);
    assertThat(pages).hasSize(2);  // 자식 엔티티가 여전히 delete 되지 않았음을 알 수 있습니다.
}
```

이렇게 `CascadeType.REMOVE`는 오직 부모 엔티티를 삭제하는 경우에만 자식 엔티티도 삭제되며, 위의 경우에서는 Page가 데이터베이스에서 삭제되지 않습니다.
그리고 이에 대해 `orphanRemoval = true`를 적용하면, 부모 엔티티와 관계가 끊긴 자식 엔티티가 자동으로 삭제되게 됩니다.

```java
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @OneToMany(mappedBy = "book", fetch = FetchType.LAZY, cascade = CascadeType.REMOVE, orphanRemoval = true)
    private List<Page> pages = new ArrayList<>();

    public void addPage(Page page) {
        pages.add(page);
    }
}
```

이렇게 orphanRemoval을 true로 설정해두면 아래처럼 고아 객체가 되었을 경우, 해당 자식 엔티티의 자동 삭제가 이루어지게 됩니다.

```java
//...
@Autowired
private BookRepository bookRepository;
@Autowired
private PageRepository pageRepository;

void cascade_type_remove_page_orphan_removal() {
    // given
    Page page1 = new Page();
    Page page2 = new Page();

    Book book = new Book();

    book.addPage(page1);
    book.addPage(page2);

    bookRepository.save(book);

    // when
    book.getPages().remove(0);     // 부모 엔티티 Book pages에서 자식 엔티티 첫 Page와의 관계를 제거합니다. 

    // then
    List<Book> books = bookRepository.findAll();
    List<Page> pages = pageRepository.findAll();

    assertThat(books).hasSize(1);
    assertThat(pages).hasSize(1);  // 이번에는 해당 자식 엔티티가 delete 되었음을 알 수 있습니다.
}
```

다시 말해 부모 엔티티를 삭제한 경우에는 CascadeType.REMOVE와 orphanRemoval = true 옵션 모두 동일하게 자식 엔티티를 삭제하게 됩니다.
하지만 부모 엔티티에서 자식 엔티티를 제거하는 경우, CascadeType.REMOVE만 사용한다면 자식 엔티티가 그대로 남아있는 반면,
orphanRemoval = true와 함께 사용하면 부모 엔티티와 자식 엔티티의 관계가 끊어졌을 때 해당 자식 엔티티를 제거해줄 수 있게 됩니다.

### Cascade 옵션 사용 시 주의할 점

만일 CascadeType.ALL을 사용하고 orphanRemoval = true를 같이 적용할 경우, 부모 엔티티를 통해 자식 엔티티의 모든 생명 주기를 관리할 수 있게 됩니다.
그러나 ManyToMany와 같이 엔티티 간의 연관 관계에 있어 부모 엔티티가 둘 이상으로 있을 시에는, 
한 곳에서 REMOVE를 하더라도 다른 곳에서 PERSIST가 되어 실패할 수 있는 문제 등의 발생할 수 있기에 주의해야 합니다

---
