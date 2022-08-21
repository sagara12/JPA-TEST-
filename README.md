# Spring,JPA - 전체 테스트 오류 해결

## 오류 상황
* Junit5을 통해 테스트 오류를 수행
* 글 여러개 조회 & 페이징 처리 부분에서 전체 테스트에 테스트 값이 기대했던 값과 다르게 나옴
* 엔티티에 PK값을 자동 생성 해주고 그 값을 근거로 가장 최신 값(가장 큰 값)을 맨처음 보여주는 테스트 였는데 지정했던 값보다 +2가 더 높게 나옴
* 단위 테스트를 수행 한 결과 단위 테스트에서는 기대했던 값과 일치한 값이 나옴
* 전 테스트를 수행한 결과가 영향을 미치는 것으로 판단하고 테스트를 고립시키고 테스트가 끝나면 결과를 비우는 코드 작성
* 해당 코드를 작성 했음에도 결과에 변화 가 없음




> WriteControllerTest
```java
  @Test
    @DisplayName("글 여러개 조회 & 페이징 처리")
    void test5() throws Exception {

        //given
        List<Write> requestWrite = IntStream.range(1, 31)
                .mapToObj( i -> {
                    return Write.builder()
                            .title("글 제목 " + i)
                            .content("글 내용 " + i)
                            .build();
                })
                .collect(Collectors.toList());

        writeRepository.saveAll(requestWrite);


        //expected

        mockMvc.perform(get("/writes?page=1&size=10") // /write?page=1&sort=writeId,desc&size=5
                        .contentType(MediaType.APPLICATION_JSON)
                )
                .andExpect(jsonPath("$.length()", Is.is(10)))
                                    .andExpect(jsonPath("$.[0]writeId").value(30))
                .andExpect(jsonPath("$.[0]title").value("글 제목 30"))
                .andExpect(jsonPath("$.[0]content").value("글 내용 30"))
                .andExpect(status().isOk())
                .andDo(MockMvcResultHandlers.print());

        dataBaseCleaner.cleanWriteDBDataBase();
    }
```


> Write(Entity 클래스)
```java
package com.datePage.request.domain;

import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.*;


@Entity
@NoArgsConstructor(access = AccessLevel.PUBLIC)
@Getter
public class Write {

    @Id
    @GeneratedValue(strategy = GenerationType.Auto)
    @Column(name = "ID")
    private Long writeId;

    @Column(name = "write_title")
    private String title;

    @Lob
    private String content;

    @Builder
    public Write(String title, String content) {
        this.title = title;
        this.content = content;
    }

    public WriteEditor.WriteEditorBuilder toEditor() {
       return WriteEditor.builder()
                .title(title)
                .content(content);

    }

    public void edit(WriteEditor writeEditor) {
        title = writeEditor.getTitle();
        content = writeEditor.getContent();
    }
}
```

## 오류 원인 
* @GeneratedValue(strategy = GenerationType.Auto) 이 부분이 문제
* @GeneratedValue(strategy = GenerationType.IDENTITY)로 바꿈

## 오류 해결
> Write(Entity)
```java
package com.datePage.request.domain;

import lombok.AccessLevel;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.*;

@Entity
@NoArgsConstructor(access = AccessLevel.PUBLIC)
@Getter
public class Write {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long writeId;

    @Column(name = "write_title")
    private String title;

    @Lob
    private String content;

    @Builder
    public Write(String title, String content) {
        this.title = title;
        this.content = content;
    }

}
```

> strategy에서 GenerationType.Auto 와 GenerationType.IDENTITY 차이점

* @GeneratedValue(strategy = GenerationType.AUTO) (Default)
- insert 쿼리 전에 hibernate_sequence 테이블의 데이터에 대해서 select, update 쿼리가 실행된것을 확인 할 수 있다.
- id 생성을 위해 hibernate_sequence 테이블의 시퀀스 값을 가져와 업데이트하고, 그 값으로 id를 생성하여 insert 쿼리를 사용한다.
- 이 설정이 @Id @GeneratedValue default로 동작한다.

* @GeneratedValue(strategy = GenerationType.IDENTITY) 
- insert 쿼리가 pk 값 없이 수행된다.
- 데이터베이스의 auto_increment 동작이 수행된다.
- ddl-auto: create 을 사용중이라면 pk 옵션이 auto_increment로 생성된다.

참고 사이트
https://lion-king.tistory.com/entry/JPA-JPA-Id-GenerationTypeAUTO-IDENTITY
