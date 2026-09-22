# wlsdn-lab.github.io
Locky study report

# [락키 스터디 2주차] AI로 PHP 사이트 만들고 스스로 보안 점검해보기

> 본 글은 보안 스터디 교육 목적으로, **본인이 만든 사이트**에서만 점검·수정한 기록입니다.

## TL;DR
- claude AI로 PHP+MySQL (로그인/게시판/업로드) 사이트를 만들어 무료 호스팅에 배포했다.
- 보안 점검 5대 관점으로 스스로 훑어 3개를 찾았고, 그중 1개를 패치했다.
- 느낀 점 한 줄: ai의 성능이 대단해서 감탄했고, 나중에 일을 할 때 이런 식으로 보고서를 많이 쓴다고 들었는데 그때를 대비해 이런 연습을 많이 해야겠단 생각이 들었다.

## 1. 무엇을 만들었나
- 사이트 주제 / 기능:
- 스택: PHP + MySQL (mysqli), InfinityFree
- 사용한 AI와 핵심 프롬프트:
  ```
  내가 학과 동아리 스터디에서 진행하는 공격 방어하기 위한 사이트를 작성하려고해
  취약점은 있어도 상관이 없어
  컨셉은 쇼핑몰이고
  환경은 infinityfree에서 만들거야
  백엔드는 php, mysql이고
  기능은 아래와 같아
  회원가입, 로그인, 로그아웃, 회원탈퇴, 게시판(CRUD), 댓글기능, 마이페이지, 프로필 조회 •수정, 관리자 페이지
  이를 인피니티프리에 업로드 편하게 하나의 폴더로 만들어줘 
  ```
- 화면 캡처 (URL 표시줄은 가리기)
<img src="1.png" width="80%" alt="설명">


## 2. 배포하면서 막힌 점
- 문제 → 해결: 사이트를 만드는 과정에 MySql에 스키마에 문자셋이 달라서 오류가 발생했었지만, 수정하여 해결함
- MySql 비밀번호 때문에 문제가 발생했었지만, 서브 도메인 비밀번호를 그대로 사용한다는 것을 알고 해결함

## 3. 5대 관점 셀프 점검

| # | 관점 | 확인 결과 | 조치 |
| --- | --- | --- | --- |
| 1 | 입력검증 | 취약 (SQLi·Stored XSS 발견) | XSS만 패치 / SQLi 미패치 |
| 2 | 인증세션 | 안전함 | 패치 안함 |
| 3 | 접근제어 | 대체로 안전. 관리자 삭제가 GET이라 CSRF 결합 시 위험 | 미패치 |
| 4 | 파일업로드 | 안전 | 패치 안함 |
| 5 | 설정노출 | SQL 에러 노출 | 미패치 |

> 공방전이 끝날 때까지 **미패치 항목은 비워 두고**, S4 이후 업데이트합니다.

## 4. 패치 기록 (3개)
### 4-1. Stored XSS — 댓글 출력
- 문제: 댓글 내용을 이스케이프 없이 그대로 출력 → 저장된 'script'가 다른 사용자 브라우저에서 실행됨 (Stored XSS)
- Before
  ```php
  <div class="comment-body"><?= nl2br($c["content"]) ?></div>
  ```
- After
  ```php
  <div class="comment-body"><?= nl2br(h($c["content"])) ?></div>
  ```
- 확인: 패치 전엔 굵게 표시되고 alert(1) 실행됨, 패치 후엔 글자 그대로 표시되고 실행 안 됨.

```html
<b>test</b>
<script>alert(1)</script>​
```

패치 전엔 굵게 표시되고 alert(1) 실행됨, 패치 후엔 글자 그대로 표시되고 실행 안 됨.

### 4-2. SQL Injection — 게시판 검색 (board.php) [미패치 · 공방전용]
- 문제: 검색어를 쿼리에 문자열로 직접 결합 + SQL 에러 원문 노출
- Before
  ```php
  $sql = "SELECT ... WHERE p.title LIKE '%$q%' ORDER BY p.id DESC";
  $res = mysqli_query($conn, $sql);
  if (!$res) { $err = mysqli_error($conn); }
​  ```
- After: 처음부터 패치가 너무 많이 되어서 부득이하게 공방전을 위해서 미패치 상태로 두었습니다. S4에서 수정하겠습니다!
  
### 4-3. CSRF — 상태변경 요청에 토큰 없음 [미패치 · 공방전용]
- 문제: 요청이 우리 폼에서 온 건지 검증하지 않음 → 외부 페이지가 로그인된 피해자 대신 댓글 작성·삭제를 유발 가능
- Before
  
```php
<form class="comment-form" action="post.php" method="post">
  <input type="hidden" name="action" value="add_comment">
  <input type="hidden" name="post_id" value="...">
  <textarea name="content"></textarea>
  <button type="submit">post comment</button>
</form>
```

- After: 이것도 공방전을 위해서 s4에서 패치 하겠습니다 죄송합니다.

## 5. AI가 짠 코드에서 느낀 점
- AI가 기본으로 놓친 것: 공방전 학습용 이라고 말을 했지만, 거의 모든 것을 다 막아놓은 것이 아쉬웠습니다. 제가 다시 한번 말을 해야지 학습용으로 취약점을 남겨두었었습니다. 그 점이 조금 아쉬웠습니다.
- 잘한 것: 순식간에 php와 mysql 전부다 엄청나게 높은 퀄리티로 오류 없이 원하는 대로 구현 해줘서 놀랐습니다.
- 다음에 AI에게 요청할 때 바꿀 점: 내가 원하는 것을 명확하게 이해 가능하게 질문 자체를 세세하게 해야겠다는 생각이 들었습니다.

## 6. 다음 주
- 공방전 1차(공격) — 남의 사이트에서 무엇을 찾아볼지




