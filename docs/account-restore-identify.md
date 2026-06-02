# 계정 데이터 복원 — 옛 계정 특정 가이드

구글 로그인으로 데이터 복원이 안 된다는 문의가 들어왔을 때, **옛 `uid`(`#pincode-userkey`)를 모르는 상태**에서 유저가 제공 가능한 정보만으로 옛 계정(`userkey`)을 특정하는 방법을 정리한 문서다.

기준 코드: [src/controllers/packageController.js](../src/controllers/packageController.js) `loginSinglePackageVer2`, 스키마: [database/pier.sql](../database/pier.sql) `table_account`.

---

## 1. 왜 복원이 안 되는가 (구조상 원인)

`loginSinglePackageVer2`([packageController.js:2942-2983](../src/controllers/packageController.js#L2942-L2983))의 로그인 매칭 순서:

1. `gamebaseid = ugsid`(구글/UGS ID) + `package` 로 먼저 조회
2. 없으면 `deviceid` + `package` (최근 로그인순 `limit 1`)
3. 둘 다 없으면 **신규 계정 생성** (`registerPackageAccount`)
4. 디바이스로 찾았으면 그 계정에 현재 `ugsid`를 `gamebaseid`로 기록

즉 구글 로그인 복원 실패는 → **옛 계정의 `gamebaseid`가 NULL이거나 현재 구글 ID와 다르고**, 디바이스도 안 맞아서(기기 변경·재설치) 새 계정이 생성된 상황이다.

추가 제약: `table_account`에 `UNIQUE KEY (gamebaseid, package)`가 있다([pier.sql:3166](../database/pier.sql#L3166)). 따라서 현재 구글 ID는 이미 **새(빈) 계정**에 점유된 상태일 가능성이 크다 — 옛 계정에 같은 `gamebaseid`를 그대로 붙이려면 새 계정 쪽을 먼저 떼어내야 한다.

---

## 2. 유저 제공 정보로 옛 계정을 특정하는 필드 (강도순)

### 🟢 가장 강함 — 결제 영수증 (유일 식별)

유저가 유료 결제를 한 적 있으면 Gmail에 구글플레이 주문번호/영수증이 남는다. [user_purchase](../database/pier.sql#L4212-L4227):

| 필드 | 설명 |
| --- | --- |
| `receipt` | 영수증 원문 |
| `purchase_token` | 게임베이스 식별자2 |
| `payment_seq` | 게임베이스 식별자1 |
| `purchase_date` + `price` + `product_currency` | 결제일/금액/통화 |

→ `userkey`를 거의 100% 특정 가능. **복원 케이스의 정석 앵커.**

### 🟡 중간 — 본인 설정값 / 메타데이터

`table_account` 기준:

- `alter_name` / `nickname` — 유저가 직접 지은 별명. (단, 기본값은 uid라 안 바꿨으면 무의미 — [packageController.js:2996-3005](../src/controllers/packageController.js#L2996-L3005))
- `os`, `current_lang`, `package` — 접속 플랫폼/언어/패키지
- `createtime` / `lastlogintime` — 대략의 가입 시점 / 마지막 플레이 날짜

> ⚠️ **`country` 컬럼은 식별에 쓰지 말 것.** 로그인 갱신 쿼리([Q_UPDATE_CLIENT_ACCOUNT_WITH_INFO](../src/QStore.js#L5-L16))가 매 로그인마다 리터럴 `'ZZ'`를 하드코딩해 덮어쓴다([packageController.js:333](../src/controllers/packageController.js#L333), [2859](../src/controllers/packageController.js#L2859), [3033](../src/controllers/packageController.js#L3033)). 실제 국가가 아니라 사실상 전 계정이 `ZZ`(미지정)다.
>
> `current_culture`의 `ZZ`는 의미가 다르다 — [com_culture](../database/pier.sql#L659-L667) 조회에서 `lang`/`country_code` 매칭 실패 시의 fallback 값([packageController.js:318](../src/controllers/packageController.js#L318))이라, 좁히기 단서로는 `current_lang` 쪽이 낫다.

### 🟡 진행도 기반 (단독으론 약함, 조합 시 유효)

- `table_account.current_level`, `current_experience`, `grade`
- 진행 위치: `user_episode_progress`, `user_ending`, `user_all_clear` — "어떤 작품 몇 화/엔딩까지 봤다"

### 🟢 문의 이력

- [user_inquiry](../database/pier.sql#L3728) — 예전에 인게임 문의를 넣었다면 그 `userkey`에 연락처가 묶여 있을 수 있음

### 🔴 deviceid

- 옛 기기를 아직 보유 중이면 `table_account.deviceid`로 매칭 가능. 단, 유저가 텍스트로 불러줄 수 있는 값이 아니라 실무상 거의 불가.

---

## 3. 권장 식별 절차

```
1순위: 결제 영수증/주문번호 → user_purchase.receipt / purchase_token → userkey 확정
2순위: 결제일 + 금액      → user_purchase.purchase_date / price 로 후보 좁히기
3순위: 가입/마지막접속 시기 + OS/언어(current_lang) + 진행도(레벨·엔딩) 조합으로 압축
        (country 컬럼은 항상 ZZ라 제외)
보조:  현재 구글 ID(gamebaseid)가 붙은 "새 계정"을 먼저 찾고,
        같은 deviceid·유사 닉네임으로 옛 userkey 를 역추적
```

---

## 4. 복원 방식 (특정 이후)

옛 `userkey`를 확정한 뒤, 복원은 두 가지 중 하나:

1. **데이터 이관** — 옛 `userkey`의 데이터를 새 계정으로 옮김
2. **gamebaseid 재연결** — 옛 계정에 현재 구글 ID를 붙임
   - ⚠️ `UNIQUE KEY (gamebaseid, package)` 때문에 **새 계정의 `gamebaseid`를 먼저 NULL 처리/해제**해야 충돌이 안 난다.

> 실제 복원 실행 전, `userkey` 후보가 1건으로 확정됐는지 반드시 검증할 것. 진행도/메타데이터만으로 좁힌 경우 동일 조건 계정이 복수일 수 있다.
