# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 컨텍스트

IFyou 스토리 게임 플랫폼의 게임 서버. 현재 브랜치 `PackageGameSystem`은 단일 게임 지원 시스템으로 리팩토링된 버전이며, 구 멀티 게임 플랫폼은 `플랫폼-서버` 브랜치에 있다. 코드베이스는 한국어 기반(주석·로그 메시지·DB 에러 문구가 모두 한국어). Node.js 18.16.0 / CentOS 7.8 기준으로 동작.

## 명령어

```bash
npm run dev      # nodemon + babel-node로 src/init.js 실행 (2초 delay)
npm run build    # babel src --out-dir dist --copy-files
npm run start    # node dist/init.js (build 이후 실행)
```

테스트 프레임워크는 구성되어 있지 않음 (`npm test`는 placeholder). 린트는 `npx eslint src/` (airbnb-base + prettier, `prettier/prettier` 규칙은 `off`).

프로덕션 배포는 PM2: `pm2 start ecosystem.config.js` (cluster 모드 2 instance) 또는 `cluster.config.js` (max instance). 둘 다 `./dist/init.js`를 구동하므로 사전에 반드시 `npm run build` 필요.

## 런타임 구조

- **엔트리**: [src/init.js](src/init.js)는 HTTP 리스너를 열기 **이전에** LRU 캐시(`refreshCacheServerMaster`, `refreshCacheLocalizedText`, `refreshCacheFixedData`, `refreshCachePackageEvent`)를 모두 로딩한다. 기준정보 캐시에 의존하는 모든 코드는 캐시가 warm 상태임을 전제로 작성됨.
- **HTTPS 토글**: `process.env.HTTPS > 0`이면 `./cert/{ca-chain-bundle.pem,key.pem,crt.pem}`로 HTTPS 구동. PORT 기본값은 7606.
- **캐시**: `init.js`에서 export하는 모듈 레벨 `cache = new LRU(...)`를 컨트롤러들이 전역 key-value 스토어처럼 공유한다. 실제로는 LRU로 동작하지 않음 — 부팅 시 한 번 로딩하고 계속 사용하며 `dispose`는 no-op. 재로딩은 `loadingRegularCacheData()`를 통해서만.
- **캐시 자동 갱신**: [src/com/cacheLoader.js](src/com/cacheLoader.js)의 `schedule.scheduleJob("*/20 * * * *")`가 **20분마다** `loadingRegularCacheData()`를 호출(env 게이트 없이 항상 동작) → `refreshCacheServerMaster`/`refreshCachePackageEvent` 갱신. 공지(`com_notice`)·서버마스터 등 DB 직접 수정분은 재시작 없이 최대 20분 내 반영. PM2 클러스터에서는 인스턴스마다 각자 갱신.

## 요청 디스패치 패턴 (중요)

이 서버는 사실상 **HTTP 엔드포인트가 verb당 하나뿐**이다. [src/routers/globalRouter.js](src/routers/globalRouter.js)에서 다음과 같이 연결됨:

- `POST /client`  → `clientHome`
- `PATCH /client` → `patchClient`
- `PUT /client`   → `putClient`
- `GET /rep`      → liveness 체크 (Morgan은 `/rep` 로그를 skip)

세 핸들러는 모두 [src/controllers/clientController.js](src/controllers/clientController.js)에 있으며 거대한 `switch (req.body.func)` 디스패처 구조다. 새 API 액션을 추가할 때는:

1. 해당 도메인 컨트롤러(`packageController`, `accountController` 등)에 핸들러 구현
2. `clientController.js`로 import
3. 적절한 디스패처(`clientHome` / `patchClient` / `putClient`)에 `case "<funcName>":` 추가 — REST 의미론이 아니라 의도에 따라 verb 선택. 읽기·쓰기 대부분은 POST/`clientHome`을 통해 들어옴

`func`가 없거나 매칭되지 않으면 `respondFail(..., "no func"/"Wrong Request", 80019)`를 반환. `routers/clientRouter.js`는 의도적으로 비어있음 — "버그"로 오해하고 채우지 말 것.

## DB 접근 ([src/mysqldb.js](src/mysqldb.js))

총 4개의 export, 모두 `{ state: boolean, row?, error? }` 형태를 반환:

- `DB(sql, params)` — admin 풀 (read/write primary)
- `slaveDB(sql, params)` — read replica
- `logDB(sql, params)` — 별도 `gamelog` DB (분석용 로그 쓰기)
- `transactionDB(sql, params)` — `beginTransaction`/`commit`/`rollback` 래핑

세 풀 모두 **`multipleStatements: true`**가 활성화되어 있다. 컨트롤러들은 이걸 적극적으로 활용해서 `mysql.format(...)`으로 여러 쿼리를 하나의 큰 문자열로 합친 뒤 한 번의 `DB()` 호출로 실행하는 패턴을 표준으로 사용한다. 다건 statement 일괄 커밋이 필요하면 `transactionDB`가 정석. 환경변수: `MYSQL_*`, `LOG_MYSQL_*`, `SLAVE_MYSQL_*`, `MYSQL_CONN_LIMIT`, `MYSQL_ADMIN_DB`.

`logAction(userkey, action_type, log_data, project_id)`과 `logAD(...)`는 fire-and-forget 헬퍼이며 `log_data`는 520자에서 잘림.

저장 프로시저(`CALL sp_*`)와 DB 함수(`fn_*`)가 매우 많이 사용된다 — 도메인 로직이 Node에만 있는 것이 아니라 DB에도 분산되어 있음을 항상 염두에 둘 것. 스키마는 `database/{pier,gamelog}.sql`에 있음.

## 응답 헬퍼 ([src/respondent.js](src/respondent.js))

`res.status().json()` 직접 호출 대신 항상 다음을 사용:

- `respondSuccess(res, data)` — `data.result = 1` 세팅 후 200
- `respondFail(res, data, devMessage, textID)` — **200**으로 응답하지만 `result: 0` (클라이언트가 비즈니스 실패로 처리)
- `respondError(res, error, localizedTextID, koMessage)` — **400**으로 응답
- `respondDB(res, errorCode, serverError, lang)` — `com_localize`에서 로컬라이즈 메시지를 조회한 뒤 `respondError` 호출

`respondFail`이 200 + `result:0`을 쓰는 규약은 의도된 설계 — Unity 클라이언트가 transport 실패와 비즈니스 실패를 구분하기 위함이다.

## 스케줄링 작업 ([src/schedule.js](src/schedule.js))

- `scheduleMail` — 매 분 실행, `list_reservation` 메일 발송 처리. `process.env.SCHEDULE_JOB_ON > 0`일 때만 동작
- `scheduleAdCharge` — 매일 00:00 실행, `table_account.ad_charge`를 0으로 초기화. `process.env.MAIL_SCHEDULE == 1`일 때만 동작

PM2 클러스터 환경에서는 중복 실행을 막기 위해 **하나의 인스턴스에만** `MAIL_SCHEDULE`/`SCHEDULE_JOB_ON` 환경변수를 세팅해야 한다.

## 컨벤션

- 소스는 ES 모듈(`import`/`export`), Babel `@babel/preset-env` + `@babel/plugin-transform-runtime`로 `dist/`에서 CommonJS로 트랜스파일
- 한국어 주석 마커: `// *` 중요 표시, `// !` 주의·경고, `// ?` 섹션 종료, `// TODO`/`// Deprecated`는 상태 표기. 코드 수정 시 동일한 스타일 유지
- 상당수 핸들러는 미들웨어에서 세팅된 `global.user`로 body 파라미터를 받는다 — caller 식별을 `req.body`만으로 가정하지 말 것
- AWS S3와 Google Translate 자격증명은 [src/com/com.js](src/com/com.js)에 wiring되어 있음. `google_credential.json`은 레거시로 체크인되어 있으므로 키 로테이션 시 주의
- 로그는 `./logs/%DATE%.log`와 `./logs/error/%DATE%.error.log`에 적재 (winston-daily-rotate, 30일 보관)
