# [팀 과제] 노션에서 브로드 크럼스(Breadcrumbs) 만들기 - 면접 4팀

## 요구사항

**페이지 정보 조회 API**: 특정 페이지의 정보를 조회할 수 있는 API를 구현하세요.

- 입력: 페이지 ID
- 출력: 페이지 제목, 컨텐츠, 서브 페이지 리스트, **브로드 크럼스 ( 페이지 1 > 페이지 3 > 페이지 5)**
- 컨텐츠 내에서 서브페이지 위치 고려 X

## 시나리오

> A > B > C  
> D > E

참고) **D > B(다른 경로로 동일한 페이지에 접근 허가 X)**

## 예상 출력

> 1 depth

A라는 페이지 탐색했을 때

```json
{
  "pageId": 1,
  "title": "hihihi",
  "content": "hihihihasdfsdfsdf",
  "subPages": ["B"],
  "breadcrumbs": ["A"]
}
```

> 2 depth

B라는 페이지 탐색했을 때

```json
{
  "pageId": 2,
  "title": "jojo",
  "content": "hihihihasdfsdfsdf",
  "subPages": ["C"],
  "breadcrumbs": ["A", "B"]
}
```

> 3 depth

C라는 페이지 탐색했을 때

```json
{
  "pageId": 3,
  "title": "hoho",
  "content": "hihihihasdfsdfsdf",
  "subPages": [],
  "breadcrumbs": ["A", "B", "C"]
}
```

## ERD

![ERD](https://cdn.discordapp.com/attachments/1146612184655351861/1147864861829763134/2023-09-03_9.04.39.png)

## 데이터 예시

### posts

![posts](https://cdn.discordapp.com/attachments/1146612184655351861/1147863570219008121/2023-09-03_8.59.11.png)

### post_contents

![post_contents](https://cdn.discordapp.com/attachments/1146612184655351861/1147863569409507478/2023-09-03_8.59.42.png)

### breadcrumbs

![breadcrumbs](https://cdn.discordapp.com/attachments/1146612184655351861/1147863569648586804/2023-09-03_8.59.32.png)

### sub_pages

![sub_pages](https://cdn.discordapp.com/attachments/1146612184655351861/1147863569912840252/2023-09-03_8.59.20.png)

## 로직

1. 페이지 ID를 입력 받는다.
2. 입력받은 페이지 ID를 사용하여 `posts` 테이블에서 `title`을 얻는다. `title`은 `breadcrumbs`를 구성하는 요소이다.
3. `page_id`를 사용하여 현재 페이지에서 접속 가능한 하위 페이지들을 `subPages` 테이블에서 검색한다.
4. `page_id`를 사용하여 현재 페이지까지의 경로를 담고있는 `breadcrumbs` 테이블에서 `breadcrumbs`를 얻는다.

## 핵심 쿼리

```sql
select p.id, p.title, sp.title as subPage, pp.breadcrumbs from posts as p
left join sub_pages as sp on p.id = sp.post_id
left join breadcrumbs as pp on p.id = pp.post_id
where p.id = 3;
```

## 실행 결과

![result](https://cdn.discordapp.com/attachments/1146612184655351861/1147862696000237599/2023-09-03_8.51.48.png)

페이지 ID가 3인 경우에 대한 탐색 결과이다.  
`id`, `title`, `subpage`, `breadcrumbs`를 반환한다.  
`A > B > C` 시나리오에 해당하기 때문에,  
C에서부터 접근할 수 있는 페이지가 없으므로 `subpage`는 빈 배열,  
C까지 접근한 경로인 `breadcrumbs`는 `A/B/C`를 반환하게 된다.

## 왜 이 구조가 최선인가?

초기에는 하나의 테이블에서 모든 것을 해결하고자 했다.

![first](https://cdn.discordapp.com/attachments/1146612184655351861/1147773446038749317/2023-09-03_3.01.47.png)

그러나 이는 정규화가 부족한 테이블이며, 하나의 테이블에 모든 col을 몰아넣는 구조는 필요하지 않은 연산을 증가시키는 단점이 있다.  
예를들어, `title`, `subpage`, `breadcrumbs`를 얻기 위해 하나의 테이블을 계속 순회해야하는 문제가 발생하는 것이다.  
이는 SQL을 충분히 활용하지 못할 뿐더러, `while` 등과 같은 언어 문법과 섞여버려서 이해하기 어려울 가능성 또한 높아보였다.  
따라서 이 방법은 폐기하였다.

이런 문제점을 인식한 뒤, 정규화를 최대한으로 진행한 상태에서 데이터 검색을 하는 것으로 방향을 잡게 되었다.  
따라서, 위에서 명시한대로 테이블 구성이 완료되었다.  
정규화가 완료된 테이블을 살펴보면, 각 테이블에서 `post_id`만을 활용하여 빠르게 데이터를 얻어내는 것을 확인할 수 있다.  
앞서 언급한 불필요한 순회를 획기적으로 줄인 것이다.

물론, 그 어떤 방법이든 `tradeoff`는 발생한다.  
이 방법의 경우 `breadcrumbs`를 하나의 문자열로 저장을 한다. 이는 `title`의 수정이 발생할 경우 치명적으로 작용할 수 있다.  
`breadcrumbs`를 얻은 뒤, `split()` 후, 특정 데이터만을 선택해 수정하고, 다시 `join()`해야하기 때문이다.  
그러나, 이 과제는 `GET` 요청을 구현하는 것이다. `PUT`이나 `PATCH`에 대한 고려를 포함하지 않아도 되기 때문에 이 방법이 더 유리하다고 판단하였다.

이런 부분을 우려하여 `breadcrumbs` 테이블을 제거하고 동적으로 생성하도록 하는 방법도 있을 것이다.  
우선, 입력받은 `post_id`로 얻은 `title`을 활용하여 `sub_pages` 테이블을 순회하며 더이상의 상위 페이지가 나오지 않는 경우까지 `subpage`를 수집한다.  
그 후, `subpage`에서 얻은 `post_id`를 가지고 다시 `posts` 테이블에 접근하여 `title`을 얻는다.  
이렇게 두 단계를 계속 반복해나가면 된다.  
이는 현재 테이블 구성에서는 상당히 비효율적인 방법이 될 것이다. 여러 테이블을 오가며 검색을 해야하니 말이다.  
따라서, 수정하는 상황에서 다소 불편함의 여지가 있더라도 정규화가 최대한 진행된 상황에서 필요한 데이터들을 구성해놓은 테이블을 생성하여 접근하는 것이 훨씬 빠르다고 판단된다.

## 전체 코드

```js
const pg = require("pg");
const conn = new pg.Client({ database: "test" });

(async () => {
  await conn.connect();
  const res = await conn.query(
    "select p.id, p.title, sp.title as subPage, pp.breadcrumbs from posts as p " +
      "left join sub_pages as sp on p.id = sp.post_id " +
      "left join breadcrumbs as pp on p.id = pp.post_id where p.id = 2;"
  );
  console.log(res.rows[0]);
  conn.end();
})();
```
