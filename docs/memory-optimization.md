# MySQL InnoDB Buffer Pool 메모리 최적화 분석

> 대상: AWS RDS for MySQL — `pier` (게임 운영 DB) / `gamelog` (분석 로그 DB)
> 목적: 게임 컨텐츠 업데이트 동결을 전제로 `innodb_buffer_pool_size` 축소 여지 검증

---

## 1. 출발 가설

> 신규 유저 관련 DB를 제외하면 게임 컨텐츠는 업데이트 계획이 없으므로 데이터의 상한이 정해져 있다.
> 유저 측에서 흔히 호출하는 데이터가 buffer pool에 캐싱되니, 그 상한에 맞춰 `innodb_buffer_pool_size`를 축소해 메모리를 절감할 수 있을 것이다.

이 문서에서는 위 가설의 타당성, 한계, 그리고 실측 기반의 대안을 정리한다.

---

## 2. 가설의 맞는 부분

- `com_*`, `list_*`, `dlc_*`, `admin_*` 류 **컨텐츠/기준정보 테이블은 동결 상태**이면 `data_length + index_length` 합계의 이론적 상한이 고정된다.
- 이 상한은 `information_schema.tables`로 즉시 측정 가능하므로 "컨텐츠 측 워킹셋의 최대치"는 의미 있는 숫자다.
- 컨텐츠 동결이 곧 "buffer pool에 무한정 페이지가 쌓이지는 않는다"는 점은 옳다.

---

## 3. 가설의 빠진 부분 / 잘못된 가정

### 3.1 buffer pool ≠ 전체 데이터 크기 (working set 크기)

InnoDB buffer pool은 "자주 접근되는 페이지"를 캐싱하는 메커니즘이지 모든 데이터를 담는 캐시가 아니다.
- 컨텐츠 데이터가 100GB여도 hot 페이지가 5GB라면 buffer pool은 5GB + 여유면 충분.
- 거꾸로 컨텐츠를 동결해도 hot 페이지가 메모리에 못 올라오면 hit ratio가 떨어진다.

→ "데이터 상한 = 필요한 buffer pool 크기" 등식은 성립하지 않는다.

### 3.2 user_* 테이블이 진짜 변수다

`database/pier.sql`을 보면 `user_` 접두사 테이블이 86개이며, 이들은 유저 수·플레이 활동에 비례해 **계속 증가**한다:

- `user_episode_progress`, `user_scene_hist`, `user_scene_progress`
- `user_selection_progress`, `user_selection_current`, `user_selection_purchase`
- `user_mail`, `user_purchase`, `user_coin_purchase`
- `user_property`, `user_favor`, `user_achievement`
- `user_project_*` 류 진행 상태들

운영 중인 서비스에서 **buffer pool 사이즈를 결정하는 진짜 요인은 컨텐츠가 아니라 이쪽의 hot working set** (최근 로그인 유저들의 행/인덱스 페이지)이다. 컨텐츠 동결과 무관하게 이 영역은 DAU·총 유저수·이벤트 트래픽에 따라 변동한다.

### 3.3 컨텐츠 캐시는 이미 Node 앱 LRU가 흡수 중

`src/init.js` + `src/com/cacheLoader.js`에서 부팅 시 LRU 캐시를 warm-up 한다:

- `com_server`, `com_localize`, `com_package_localize`
- `com_notice`, `com_promotion`, `com_intro`
- `com_bubble_master`, `com_bubble_group`, `com_bubble_sprite`
- `com_ad`, `com_premium_timedeal`, `com_currency`

이 테이블들은 **앱 단에서 통째로 메모리에 상주**하므로 MySQL까지 호출 자체가 거의 가지 않는다. 즉 가설이 가정하는 "유저가 흔히 호출하는 컨텐츠 데이터"의 상당 부분은 이미 buffer pool 밖에서 처리된다.

반면 다음 테이블들은 LRU에 없고 `sp_*` / `fn_*` 내부에서 매 요청 시 직접 조회되므로 **buffer pool에 hot 상태로 머물러야 한다**:

- `list_episode`, `list_episode_detail`
- `list_scene_*` 류
- `list_dress_*`, `list_illust_*`, `list_live_*`
- `list_emoticon_*`, `list_design`, `list_bg`

→ "컨텐츠 = 모두 buffer pool에 캐싱" 도 아니고, "컨텐츠 = 모두 앱 LRU에 캐싱" 도 아니다. 두 영역의 경계를 구분해서 봐야 한다.

### 3.4 stat_*, gamelog는 write-heavy / read-cold

- `stat_*` 테이블 17개와 `gamelog` DB 전체는 **로그/분석 적재 위주**다.
- write 시 잠시 dirty page로 buffer pool을 쓰지만 flush 후엔 cold.
- 사이즈가 크다고 buffer pool을 키울 이유가 없다 (정기 리포트만 read).

### 3.5 저장 프로시저의 working set이 정적 분석으로 안 잡힌다

`CLAUDE.md`에 명시된 대로 도메인 로직이 `sp_*` / `fn_*`에 대거 분산되어 있다.
프로시저 내부의 임시 결과셋·다중 조인·임시 테이블도 buffer pool 페이지를 사용한다. 이 비용은 스키마/SQL 정적 분석만으로는 추정이 어렵고 실측이 필요하다.

---

## 4. 권장하는 실측 기반 검증 절차

가설을 "테이블 크기 합산으로 상한을 구한다"에서 **"실측 hit ratio + 실제 page 점유량으로 right-sizing 한다"** 로 재정의한다.

### 4.1 현재 hit ratio 확인

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- hit_ratio = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)
```

- 99.9% 이상이면 **buffer pool이 working set보다 충분히 크다** → 축소 여지 있음.
- 99% 미만으로 떨어지면 working set보다 작음 → 축소 위험.

AWS RDS는 Performance Insights / CloudWatch `BufferCacheHitRatio` 메트릭으로도 확인 가능.

### 4.2 카테고리별 실제 데이터 크기 측정

```sql
SELECT
  CASE
    WHEN table_name LIKE 'com\_%' ESCAPE '\\' THEN 'content_com'
    WHEN table_name LIKE 'list\_%' ESCAPE '\\' THEN 'content_list'
    WHEN table_name LIKE 'dlc\_%' ESCAPE '\\' THEN 'content_dlc'
    WHEN table_name LIKE 'admin\_%' ESCAPE '\\' THEN 'admin'
    WHEN table_name LIKE 'user\_%' ESCAPE '\\' THEN 'user'
    WHEN table_name LIKE 'stat\_%' ESCAPE '\\' THEN 'stat'
    WHEN table_name LIKE 'table\_%' ESCAPE '\\' THEN 'core'
    ELSE 'other'
  END AS category,
  SUM(data_length + index_length) / 1024 / 1024 AS size_mb
FROM information_schema.tables
WHERE table_schema = 'pier'
GROUP BY category
ORDER BY size_mb DESC;
```

이로써 "컨텐츠 vs 유저 vs 로그" 비율을 정량화 한다.

### 4.3 실제 buffer pool 점유 테이블 확인

```sql
SELECT
  table_name,
  COUNT(*) AS pages,
  ROUND(COUNT(*) * 16 / 1024, 2) AS mb_in_pool
FROM information_schema.innodb_buffer_page
WHERE table_name IS NOT NULL
GROUP BY table_name
ORDER BY pages DESC
LIMIT 30;
```

- 이 결과가 **실제 buffer pool 사용 분포**.
- 컨텐츠 테이블이 상위에 안 보인다면 → 앱 LRU가 잘 흡수하고 있다는 뜻 (가설의 빈틈).
- 상위가 `user_*` 위주라면 → buffer pool 사이즈는 유저 활동에 의해 지배된다는 결정적 증거.

> 주의: RDS에서 `information_schema.innodb_buffer_page`는 인스턴스 영향이 있을 수 있으니 트래픽 한산한 시간대에 실행.

### 4.4 점진적 축소 절차

1. RDS 파라미터 그룹에서 `innodb_buffer_pool_size`를 현재값의 75%로 조정.
2. 재시작 후 24~72시간 모니터링:
   - `BufferCacheHitRatio` (목표 ≥ 99.5%)
   - `ReadIOPS` (급증 여부)
   - `DBLoad` / wait events (`io/file/innodb/innodb_data_file` 증가 여부)
3. 안정적이면 다시 75% → 반복.
4. hit ratio가 떨어지거나 read IOPS가 급증하면 직전 값이 적정선.

### 4.5 사이드 체크: 자주 호출되는 컨텐츠가 정말 LRU에 있는가

`src/init.js`에서 warm-up 되는 캐시 키를 기준으로, 같은 테이블이 컨트롤러나 `sp_*`에서 **직접 SELECT 되는 경로**가 없는지 확인.
- 직접 조회 경로가 있으면 그만큼 buffer pool 부담으로 잡힌다.
- 발견 시 (a) 앱 캐시에서 가져오게 리팩토링 하거나 (b) buffer pool 산정에 포함.

---

## 5. 실측 결과 (1차 측정)

### 5.1 측정값

```
Innodb_buffer_pool_pages_total      32,768
Innodb_buffer_pool_pages_free        5,313
Innodb_buffer_pool_read_requests   140,435,625
Innodb_buffer_pool_reads                24,172
Innodb_buffer_pool_wait_free                 0
```

### 5.2 해석

| 지표 | 계산 | 값 | 판정 |
|---|---|---|---|
| Buffer pool 크기 | 32,768 × 16KB | **512MB** | — |
| Free 비율 | 5,313 / 32,768 | **16.2%** | 여유 있음 |
| Hit ratio | 1 − (24,172 / 140,435,625) | **99.9828%** | 매우 양호 |
| Wait free | — | **0** | 정상 (블록된 쿼리 없음) |

- 1억 4천만 건 읽기 중 디스크까지 내려간 건 24,172건 — 사실상 워밍업 단계의 미스로 추정.
- `pages_free`가 16% 남아있으면서 `wait_free`가 0이라는 것은 **워킹셋이 buffer pool 안에 모두 들어가고도 자리가 남는다**는 결정적 신호.
- 즉, 현재 512MB는 **현 트래픽 기준 과잉 할당** 상태.

### 5.3 가설 검증

3장에서 제기한 "user_* 류가 진짜 변수다", "list_* 류가 hot working set의 핵심이다"라는 주장과 측정값을 교차하면:

- hit ratio 99.98%는 **현재 DAU/유저수 수준에서는** working set이 buffer pool 안에 안정적으로 수용되고 있다는 뜻.
- 다만 이 수치는 **현재 시점의 누적값**이므로 다음을 추가 검증해야 한다:
  1. 서버 uptime 확인 (`SHOW GLOBAL STATUS LIKE 'Uptime'`) — 너무 짧으면 부팅 직후 효과
  2. 시간 간격을 둔 2회 측정으로 **구간별 hit ratio** 산출 (피크 타임 포함)
  3. `information_schema.innodb_buffer_page`로 실제 점유 분포 확인 (4.3 절차)

### 5.4 축소 여지 추정

- `pages_data` 기준 실제 사용량은 약 `(32768 − 5313 − misc) × 16KB ≈ 420~440MB`.
- 안전 마진(피크 트래픽, DAU 변동, sp_* 워크스페이스)을 30~40% 더하면 **약 384MB가 하한 후보**.
- 단, 위 추정은 1차 측정의 스냅샷이므로 **4.4의 점진적 축소 절차(75% → 75% → ...)** 를 그대로 적용해 검증해야 한다.

### 5.5 다음 액션

1. `Uptime`과 시간 간격을 둔 2차 측정으로 구간 hit ratio 재계산
2. `innodb_buffer_page`로 상위 점유 테이블 확인 — 예상대로 `user_*` 우위인지, 아니면 `list_*` 우위인지 확인
3. 위 두 데이터가 일관되게 "여유 있음"을 가리키면 RDS 파라미터 그룹에서 `innodb_buffer_pool_size`를 384MB로 1차 축소 후 24~72시간 모니터링 (FreeableMemory 부족 시)

---

## 6. 결론

- 가설 그대로의 "컨텐츠 동결 → 컨텐츠 크기 합산 = 필요 buffer pool" 등식은 **성립하지 않는다**.
- 다만 "컨텐츠 워킹셋에 상한이 있다"는 관찰 자체는 옳고, 축소 검토의 **출발점**으로 유효하다.
- 실제 사이즈 결정 요인은:
  1. `list_*` 류 런타임 참조 컨텐츠 (앱 LRU 미포함분) — 동결되어 상한 존재
  2. `user_*` 류의 hot working set — DAU/유저수에 비례, **이쪽이 지배적**
  3. 저장 프로시저 실행 워크스페이스 — 정적 분석 불가, 실측 필요
- 따라서 **"테이블 크기 합으로 상한을 정한다"** 대신 **"hit ratio + `innodb_buffer_page` 실측으로 점진 축소한다"** 가 안전한 방법.
- 축소 자체는 합리적 방향이며, 4.4의 점진적 축소 절차로 진행하면 운영 리스크를 통제하면서 메모리 절감이 가능하다.

---

## 참고 위치

- 스키마: [database/pier.sql](../database/pier.sql), [database/gamelog.sql](../database/gamelog.sql)
- 앱 LRU 워밍: [src/init.js](../src/init.js), [src/com/cacheLoader.js](../src/com/cacheLoader.js)
- DB 풀 설정: [src/mysqldb.js](../src/mysqldb.js)
