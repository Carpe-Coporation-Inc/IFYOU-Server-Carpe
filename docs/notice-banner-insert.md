# 공지사항(배너) 등록 가이드

DBeaver로 공지사항을 직접 등록하는 절차. 운영 어드민을 거치지 않고 DB에 직접 넣을 때 사용한다.

대상 테이블: [list_design](../database/pier.sql#L1743) · [com_notice](../database/pier.sql#L1061) · [com_notice_detail](../database/pier.sql#L1087). 런타임 조회: [src/com/cacheLoader.js](../src/com/cacheLoader.js#L180) `getCachePackageNotice`.

---

## 구조 요약

```
list_design        배너 이미지 메타 (URL/key/bucket)  →  design_id 생성
com_notice         공지 마스터 (노출기간/공개여부)     →  notice_no 생성
com_notice_detail  언어별 상세 (title/contents/배너연결)
```

- 배너 URL은 컬럼이 아니라 `fn_get_design_info(design_id, 'url')`로 `list_design.image_url`을 꺼낸다.
- 공지/프로모션 배너처럼 특정 작품에 안 묶인 공통 리소스는 `list_design.project_id = -1`.
- 노출 조건: `com_notice.is_public = 1` **AND** `now()`가 `start_date`~`end_date` 사이.

---

## 절차

### 1) 배너 이미지를 S3에 업로드

S3 Browser 등으로 `carpestore` 버킷의 `assets/com/-1/` 아래에 이미지 파일을 올린다.
→ 이후 `image_url`(전체 URL) / `image_key`(`assets/` 뺀 경로)에 그대로 쓴다.

### 2) SQL 실행 (같은 에디터 탭에서, autocommit ON 권장)

```sql
-- (1) 배너 디자인
INSERT INTO list_design
  (project_id, design_type, image_name, image_url, image_key, bucket)
VALUES
  (-1,
   'notice_banner',
   '<배너이름>_KO',
   'https://carpestore.s3.dualstack.ap-northeast-2.amazonaws.com/assets/<경로>/<파일>.png',
   '<경로>/<파일>.png',
   'carpestore/assets');
SET @design_id = LAST_INSERT_ID();

-- (2) 공지 마스터
INSERT INTO com_notice
  (notice_type, notice_name, sortkey, is_public, start_date, end_date, os, exception_culture, connected_project)
VALUES
  ('notice',
   '<공지이름>',
   <정렬순서>,
   1,
   '<YYYY-MM-DD HH:MM:SS>',
   '<YYYY-MM-DD HH:MM:SS>',
   'all',
   '<제외문화권 or NULL>',
   <연결프로젝트 or -1>);
SET @notice_no = LAST_INSERT_ID();

-- (3) 공지 상세 (언어별로 행 추가)
INSERT INTO com_notice_detail
  (notice_no, lang, title, contents, design_id, url_link, detail_design_id)
VALUES
  (@notice_no,
   'KO',
   '<제목>',
   '<내용>',
   @design_id,
   '<링크 or NULL>',
   -1);
```

> 변수(`@design_id`, `@notice_no`)는 **같은 커넥션(에디터 탭)**에서만 유지된다. 끊기면 NULL이 되니, 불안하면 각 INSERT 후 나온 숫자를 (3)에 직접 박아 넣어도 된다.

### 3) 검증

```sql
SELECT cn.notice_no, cn.notice_name, cn.is_public,
       cnd.lang, cnd.title, cnd.design_id,
       fn_get_design_info(cnd.design_id, 'url') AS banner_url
FROM com_notice cn
JOIN com_notice_detail cnd ON cnd.notice_no = cn.notice_no
WHERE cn.notice_no = @notice_no;
```

`banner_url`이 입력한 S3 URL로 나오면 연결 정상.

---

## 주의

- **S3 파일 필수**: `image_url`이 가리키는 파일이 실제로 S3에 있어야 한다. DB만 넣으면 깨진 이미지.
- **캐시 반영**: 공지는 캐시에서 서빙된다. [cacheLoader.js](../src/com/cacheLoader.js#L511)의 스케줄이 **20분마다** 자동 갱신하므로, 최대 20분 내 노출된다 (즉시 반영하려면 서버 재시작).
- **컬럼 값 메모**:
  - `notice_type` — `'notice'`(공지) / `'event'`(이벤트)
  - `exception_culture` — 제외할 문화권 코드(예: `'AR,JP'`), 없으면 `NULL`
  - `connected_project` — 연결 프로젝트 ID, 없으면 `-1` (런타임 공지 조회는 이 값으로 필터링하지 않음)
  - `detail_design_id` — 상세 이미지, 없으면 `-1`
