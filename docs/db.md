# DB 함수 사용 분담 분석

[src/mysqldb.js](../src/mysqldb.js)는 4개의 DB 헬퍼와 2개의 로그 헬퍼를 export 한다.
이 문서는 코드베이스 전체 호출 패턴을 분석해서 **어떤 함수가 어떤 업무를 담당하는지** 정리한 것이다.

분석 시점 기준 `src/` 트리 호출 빈도:

| 함수            | 풀 / DB                     | `await` 호출 수 | 사용 파일 수 |
| --------------- | --------------------------- | --------------- | ------------ |
| `DB`            | admin (primary, pier)       | **249**         | 17           |
| `slaveDB`       | slave (replica, pier)       | **79**          | 13           |
| `transactionDB` | admin (primary, 트랜잭션)   | **29**          | 9            |
| `logDB`         | log (gamelog 별도 DB)       | 1 (await)       | 1 (await)    |

> `logDB`는 대부분 `await` 없이 호출하는 fire-and-forget 패턴이라 카운트가 적게 보인다. 실제로는 `logAction()`/`logAD()`/`reportRequestError()` 내부에서 매 요청마다 호출됨.

---

## 1. `DB(sql, params)` — 메인 admin 풀

**연결 대상**: `MYSQL_HOST/PORT/USER/PWD` (primary). 게임 운영 DB(`pier` 스키마).

**기본 옵션**: `multipleStatements: true` → 여러 SQL을 한 번에 실행 가능.

### 담당 업무

- 모든 **단일 write** (`INSERT`, `UPDATE`, `DELETE`). 트랜잭션이 필요 없는 경우 기본 선택지.
- **자기 쓰기 직후 읽기 (read-after-own-write)** — slave 복제 지연을 피해야 하는 read도 여기로 보냄.
- 저장 프로시저 호출 (`CALL sp_*`)이 단일 호출일 때.
- `multipleStatements: true`를 활용한 **다중 INSERT 일괄 처리** (단, 원자성 보장이 불필요한 경우).
- 운영성 배치 작업 — 예: [src/controllers/statController.js](../src/controllers/statController.js) `collectAllProjectRetention`에서 `DELETE` + `INSERT` 다건을 `DB`로 처리.

### 예시 호출 위치

- [src/controllers/clientController.js](../src/controllers/clientController.js) `unlockProjectHiddenElements` — 갤러리/엔딩 해금 다중 INSERT
- [src/controllers/accountController.js](../src/controllers/accountController.js) — 유저 진행 정보 update (write 위주)
- [src/controllers/couponController.js](../src/controllers/couponController.js) — 쿠폰 사용 후 `unreadMailCount`/`com_localize` 메시지 읽기 (write 직후 read)
- [src/controllers/packageController.js](../src/controllers/packageController.js) — 로그인/플레이 기록 update 다수

### 사용 가이드

| 시나리오                              | `DB` 사용 적절성 |
| ------------------------------------- | ---------------- |
| 단일 `INSERT`/`UPDATE`/`DELETE`       | ✅ 표준          |
| 방금 쓴 데이터를 즉시 다시 SELECT     | ✅ 권장          |
| 마스터/기준정보 SELECT                | ⚠️ `slaveDB` 우선 |
| 여러 statement의 원자적 커밋          | ❌ `transactionDB` 사용 |

---

## 2. `slaveDB(sql, params)` — 읽기 전용 replica 풀

**연결 대상**: `SLAVE_MYSQL_HOST/PORT/USER/PWD`. primary와 동일한 `pier` 스키마의 읽기 복제본.

### 담당 업무

- **마스터/기준정보 SELECT** — 변하지 않거나 변경 주기가 느린 데이터 조회.
- **부팅 시 캐시 로딩** — [src/com/cacheLoader.js](../src/com/cacheLoader.js)의 거의 모든 캐시 워밍업 쿼리가 `slaveDB`를 거친다(서버 마스터, 로컬라이즈 텍스트, 말풍선, 패키지 텍스트 등).
- **로그인 / 클라이언트 초기화 단계의 lookup** — [src/com/centralControll.js](../src/com/centralControll.js) `initializeClient`에서 `com_package_master`, `com_package_client`, `com_build_hash` 조회.
- **쿠폰 검증 read 경로** — 쿠폰 마스터 정보, 보상 재화, 해금 DLC 조회.
- **웹/리포팅 read 엔드포인트** — [src/controllers/webController.js](../src/controllers/webController.js) (공개 작품 목록, 공지, 로컬라이즈 텍스트).
- **광고/상점/능력치 정보 read** — `shopController.getInappProductDetail`, `abilityController.getProjectAbility` 등.

### 예시 호출 위치

- [src/com/cacheLoader.js](../src/com/cacheLoader.js) `refreshCacheLocalizedText`, `refreshCacheFixedData`
- [src/com/centralControll.js](../src/com/centralControll.js) `initializeClient` — 패키지/버전/해시 검증
- [src/controllers/couponController.js](../src/controllers/couponController.js) `requestSingleGameCoupon` — 유저 lang, 쿠폰 마스터, 보상 재화, DLC 정보 lookup
- [src/controllers/clientController.js](../src/controllers/clientController.js) `requestOP_CalcPackUser` — 패키지 구매 유저 집계 read

### 주의사항

- **복제 지연(replication lag)** 이 존재한다. 방금 `DB`/`transactionDB`로 INSERT/UPDATE 한 데이터를 즉시 다시 읽으려면 **반드시 `DB`로 조회**할 것. `slaveDB`로 읽으면 stale data 가능.
- `slaveDB`는 read-only 인덱스/뷰가 정상이라는 전제로 동작 — DDL 변경 시 master/slave 동기화 상태를 확인.

---

## 3. `transactionDB(sql, params)` — 원자적 다중 statement

**연결 대상**: admin 풀과 동일 (primary, `pier`). 차이점은 호출마다 `beginTransaction()` → `query()` → `commit()` / 실패 시 `rollback()`을 감싸는 것.

### 담당 업무

원자성이 필요한 **모든 다중 statement 쓰기**. 즉 "중간에 실패하면 전부 롤백되어야 하는 작업".
대표 도메인은 **재화 정산, 메일/구매/쿠폰 처리, 게임 진행 리셋, 통계 적재**.

### 예시 호출 위치 (도메인별)

| 도메인              | 위치 / 함수                                                                                              | 처리 내용                                       |
| ------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 쿠폰                | [couponController.js](../src/controllers/couponController.js) `useCoupon`, `requestSingleGameCoupon`     | 쿠폰 사용 INSERT + 잔여 카운트 차감 + 메일 발송 |
| 인앱 / DLC 구매     | [packageController.js](../src/controllers/packageController.js) `purchaseDLC`, `purchasePackageInappProduct` | 구매 기록 + 재화 지급 + DLC 해금                |
| 메일 수신           | [packageController.js](../src/controllers/packageController.js) `readNovelPackageUserSingleMail` 외      | `user_property` 적재 + `user_mail.is_receive` 갱신 |
| 게임 진행 리셋      | [packageController.js](../src/controllers/packageController.js) `resetDLC`, `resetOtomeGameProgress`     | 진행 기록 다건 삭제/초기화                       |
| 호감도              | [abilityController.js](../src/controllers/abilityController.js) (호감도 reset 경로)                       | 능력치 reset 일괄 update                         |
| 선택지 구매         | [selectionController.js](../src/controllers/selectionController.js) `purchaseSelection`                  | 재화 차감 + 선택지 구매 기록                     |
| 설문조사 보상       | [surveyController.js](../src/controllers/surveyController.js) `receiveSurveyReward`                      | 보상 재화 다건 적재                              |
| 통계 적재           | [statController.js](../src/controllers/statController.js) `setStatList`, `collectProjectRetention` 등    | 통계 테이블 다건 INSERT (`gamelog.*` 포함)        |
| 갤러리 공유 보상    | [accountController.js](../src/controllers/accountController.js) `requestGalleryShareBonus`               | 일러스트 share_bonus update + 재화 지급          |
| 예약 메일 발송      | [src/schedule.js](../src/schedule.js) `reservationSend`                                                   | `user_mail` 다건 INSERT (스케줄러)              |
| 운영 메일 일괄 발송 | [clientController.js](../src/controllers/clientController.js) `requestOP_CalcPackUser`                   | `CALL sp_send_user_mail` 다건                    |

### 호출 패턴

대부분 다음 형태로 사용된다:

```js
let query = ``;
items.forEach((item) => {
  query += mysql.format(`INSERT INTO ...;`, [item.x, item.y]);
});
query += mysql.format(`UPDATE ...;`, [...]);

const result = await transactionDB(query);
if (!result.state) {
  logger.error(`xxx Error ${result.error}`);
  respondDB(res, 80019, result.error);
  return;
}
```

`mysql.format(...)`으로 statement를 누적시킨 뒤 한 번에 `transactionDB`로 보낸다. `multipleStatements: true` 옵션이 활성화되어 있기에 가능한 패턴.

### 사용 가이드

| 시나리오                                                   | `transactionDB` 사용 적절성 |
| ---------------------------------------------------------- | --------------------------- |
| 단일 INSERT/UPDATE/DELETE                                  | ❌ `DB` 사용                |
| 여러 statement가 "전부 성공" / "전부 실패" 여야 함         | ✅ 표준                     |
| 다른 풀(`slaveDB`, `logDB`)과 statement를 섞고 싶음        | ❌ 풀 경계 못 넘음          |
| 보상 지급 + 사용 기록 + 메일 발송처럼 연관된 쓰기 묶음     | ✅ 강력 권장                |

---

## 4. `logDB(sql, params)` — gamelog 분석 DB

**연결 대상**: `LOG_MYSQL_HOST/PORT/USER/PWD`. 데이터베이스명은 코드에 `gamelog`로 하드코딩되어 있다.

### 담당 업무

게임 외부의 **분석/감사/장애 추적용 로그 적재**. 운영 DB(`pier`)와 분리된 별도 DB.

호출 진입점은 거의 대부분 [src/mysqldb.js](../src/mysqldb.js)에 export된 helper를 통해 들어간다:

| Helper                                                        | 대상 테이블            | 호출 방식       |
| ------------------------------------------------------------- | ---------------------- | --------------- |
| `logAction(userkey, action_type, log_data, project_id)`       | `gamelog.log_action`   | fire-and-forget |
| `logAD(userkey, project_id, episode_id, ad_type)`             | `gamelog.log_ad`       | fire-and-forget |
| `reportRequestError` ([logController.js](../src/controllers/logController.js)) | `gamelog.log_request_error` | fire-and-forget |
| `getUserPropertyHistory` ([logController.js](../src/controllers/logController.js)) | `gamelog.log_property` (read) | `await logDB(...)` |

### 특이 사항

- `logAction`은 `log_data` 길이가 **520자를 넘으면 `substr(0, 520)`으로 자른다**. 큰 페이로드를 그대로 던지면 손실되니 주의.
- `logDB`는 일반적으로 `await` 하지 않고 호출된다. 즉 **로그 적재 실패가 응답을 막지 않는다** — 의도된 설계지만, 디버깅 시 로그가 누락될 수 있음.
- `gamelog` 스키마는 [statController.js](../src/controllers/statController.js)에서는 **`DB`/`transactionDB`로 직접** 접근하기도 한다 (예: `DELETE FROM gamelog.log_first_episode_user`). 즉 cross-schema 쿼리는 primary 풀의 권한으로 처리된다 — `logDB` 풀은 주로 application-level 로그 INSERT 전용.
- `log_action`은 [statController.js](../src/controllers/statController.js)의 retention 분석에서 입력 소스로도 활용된다.

---

## 5. 의사결정 플로우 — 어떤 함수를 쓸지

```
┌──────────────────────────────────────────────────────────┐
│ 작업 종류는?                                              │
└──────────────────────────────────────────────────────────┘
    │
    ├─ 분석/감사 로그 적재 (log_action, log_ad, log_request_error)
    │     → logDB (or 헬퍼: logAction / logAD)  ※ await 불필요
    │
    ├─ 쓰기 (INSERT / UPDATE / DELETE / CALL sp_)
    │     │
    │     ├─ 여러 statement가 원자적으로 묶여야 함
    │     │     → transactionDB
    │     │
    │     └─ 단일 statement 또는 원자성 불필요
    │           → DB
    │
    └─ 읽기 (SELECT)
          │
          ├─ 방금 같은 트랜잭션에서 쓴 데이터를 다시 읽음
          │     → DB  (slave lag 회피)
          │
          ├─ 마스터/기준정보 / 캐시 워밍업 / 공개 리스트
          │     → slaveDB
          │
          └─ 그 외 일반 read
                → 도메인 컨벤션 따름 (write 직전·직후면 DB, 단순 lookup이면 slaveDB)
```

---

## 6. 공통 반환 형태

네 함수 모두 동일한 결과 객체를 반환한다:

```ts
{
  state: boolean,   // 성공: true, 실패: false
  row?: any[],      // 성공 시 결과 rows. multipleStatements면 2차원 배열
  error?: Error,    // 실패 시 mysql2 에러 객체
}
```

호출자 패턴은 거의 항상:

```js
const result = await DB(/* or slaveDB / transactionDB */)(query, params);
if (!result.state) {
  logger.error(`xxx Error ${result.error}`);
  respondDB(res, 80019, result.error);   // 또는 적절한 코드
  return;
}
// 정상 처리
```

`multipleStatements`로 여러 SELECT을 묶었을 경우 `result.row`는 **2차원 배열**이 된다 — `result.row[0]`이 첫 번째 쿼리의 rows, `result.row[1]`이 두 번째 쿼리의 rows. [src/controllers/clientController.js](../src/controllers/clientController.js)의 `nestedQuery` 함수가 이 동작을 시연해두었다.
