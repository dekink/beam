---
title: "MongoDB 인덱스 및 컬렉션 정리"
description: "Wildcard Index, Compound Index 문법, Sparse Index, Clustered Collection, Time Series Collection"
publishDate: "13 Apr 2026"
tags: ["mongodb", "mongodb index", "mongodb collection"]
---

## 1. 와일드카드 인덱스 (Wildcard Index)

### 개념

`$**` 패턴으로 컬렉션의 모든 필드에 인덱스를 생성하는 방식이다. `wildcardProjection`으로 특정 필드를 포함하거나 제외할 수 있다.

```jsx
db.products.createIndex(
  { "$**": 1 },
  { wildcardProjection: { stock: 0, prices: 0 } }
)
```

위 예시는 `stock`과 `prices`를 제외한 나머지 필드를 인덱싱한다.

### 와일드카드 프로젝션 규칙

- `_id`를 제외하면 포함(1)과 제외(0)를 **섞을 수 없다.**
- `_id`는 특수 필드로, 제외 패턴과 함께 `_id: 1`을 명시해도 문법 오류가 나지 않는다.
- 하지만 `_id`는 기본 인덱스가 이미 존재하므로 와일드카드 프로젝션에서 `_id: 1`은 사실상 의미가 없다.

가능한 조합은 두 가지뿐이다:

- **제외 방식**: `{ stock: 0, prices: 0 }` — 특정 필드만 빼고 전부 인덱싱
- **포함 방식**: `{ stock: 1, prices: 1 }` — 특정 필드만 골라서 인덱싱

### 단점

- **쿼리 제한**: 복합 인덱스처럼 여러 필드를 하나의 쿼리 플랜에서 결합할 수 없다. 필터링되지 않은 필드의 정렬도 인덱스를 활용하지 못한다.
- **쓰기 성능 저하**: 모든 필드 경로를 인덱싱하므로 삽입/업데이트 시 갱신할 인덱스 항목이 많다.
- **인덱스 크기**: 모든 필드 경로를 저장하므로 전용 인덱스 대비 저장 공간을 많이 차지한다. 중첩 구조가 깊을수록 급격히 늘어난다.

### 사용 사례

IoT 데이터(센서마다 필드가 다른 경우), CMS/이커머스의 사용자 정의 속성 등 필드 구조가 유동적인 경우에 유용하다. 다만 실무에서 이런 케이스는 흔하지 않으며, 조회 패턴이 조금이라도 잡히면 복합 인덱스가 훨씬 효율적이다.

---

## 2. 복합 인덱스 (Compound Index) 문법

복합 인덱스는 반드시 각 필드에 정렬 방향(`1` 오름차순, `-1` 내림차순)을 지정해야 한다. 점(`.`)이 포함된 필드명은 따옴표로 감싸야 한다.

```jsx
// ❌ 잘못된 문법 (값 누락, 따옴표 없음)
db.people.createIndex({ metadata.likes, metadata.status })

// ✅ 올바른 문법
db.people.createIndex({
  "metadata.likes": 1,
  "metadata.status": 1
})
```

---

## 3. 스파스 인덱스 (Sparse Index)

인덱스 대상 필드가 **아예 존재하지 않는 문서**는 인덱스에서 제외하는 방식이다.

```jsx
db.users.createIndex({ email: 1 }, { sparse: true })
```

- `{ email: "a@b.com" }` → 인덱스에 **포함**
- `{ email: null }` → 인덱스에 **포함** (null도 값이다)
- `{ }` (email 필드 자체 없음) → 인덱스에서 **제외**

선택적 필드에 유니크 인덱스를 걸고 싶을 때 유용하다. `sparse: true` + `unique: true`를 함께 쓰면, 필드가 있는 문서끼리는 중복 불가로 하되 필드가 없는 문서는 여러 개 허용할 수 있다.

---

## 4. 클러스터형 컬렉션 (Clustered Collection)

### 개념 (MongoDB 5.3+)

일반 컬렉션은 데이터와 `_id` 인덱스를 별도 파일에 저장하지만, 클러스터형 컬렉션은 `_id` 인덱스 순서대로 문서 자체를 하나의 WiredTiger 파일에 함께 저장한다.

```jsx
db.createCollection("orders", {
  clusteredIndex: { key: { _id: 1 }, unique: true }
})
```

### 장점

- 일반 컬렉션은 삽입/업데이트/삭제 시 쓰기 2번, 쿼리 시 읽기 2번이 필요하지만 클러스터형 컬렉션은 각각 1번으로 줄어든다.
- `_id` 기준 범위 스캔이나 동등 비교 쿼리가 보조 인덱스 없이도 빨라진다.
- 별도의 인덱스 파일이 없어 저장 공간이 줄어들고, 보조 TTL 인덱스도 필요 없다.

### 제약

- 클러스터형 인덱스 키는 반드시 `{ _id: 1 }`이어야 한다.
- 기존 컬렉션을 클러스터형으로 변환하거나 반대로 변환할 수 없다.
- 고정 사이즈(capped) 컬렉션과 함께 쓸 수 없다.
- 클러스터형 인덱스는 숨길 수 없다.
- 보조 인덱스를 추가하면 오히려 저장 공간이 더 커질 수 있다.
- 무작위 키 값(UUID 등)은 성능을 저하시킨다. 순차 증가하는 키 값을 써야 한다.
- 클러스터형 인덱스 키 하나의 최대 크기는 8MB이지만, 가능한 한 작게 유지해야 한다.
- MongoDB 6.1 이전 버전에서는 쿼리 옵티마이저가 클러스터형 인덱스를 자동 선택하지 않아 힌트를 직접 줘야 했다.

### 보조 인덱스와의 관계

클러스터형 인덱스가 `_id`로만 가능하다는 건 데이터의 물리적 정렬 기준이 `_id`뿐이라는 뜻이다. 다른 필드로 조회해야 한다면 보조 인덱스를 별도로 추가할 수 있다.

```jsx
db.orders.createIndex({ customerId: 1 })
```

### 대용량 쓰기

- `_id`가 순차 증가하는 경우(ObjectId, 날짜): 새 문서가 항상 끝에 붙으므로 대용량에서도 괜찮다.
- `_id`가 랜덤한 경우(UUID): 페이지 분할이 발생하여 데이터가 많아질수록 쓰기 성능이 저하된다.

---

## 5. 시계열 컬렉션 (Time Series Collection)

### 개념 (MongoDB 5.0+)

시간 기반 데이터를 저장하기 위한 전용 컬렉션 타입이다. 같은 소스에서 발생한 데이터를 비슷한 시간대끼리 묶어서 저장하며, 내부적으로 컬럼형 저장 포맷과 자동 클러스터형 인덱스를 사용한다.

```jsx
db.createCollection("weather", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "hours",
  },
})
```

### 구조

- `timeField` — 타임스탬프 필드 (필수)
- `metaField` — 데이터 소스를 식별하는 메타데이터 (선택, 권장)
- `granularity` — 데이터 수집 빈도 힌트: `seconds`, `minutes`, `hours` (선택)
- 나머지 필드 — 측정값(measurement), 자유롭게 추가 가능

### 클러스터형 컬렉션과의 차이

| 항목 | 클러스터형 컬렉션 | 시계열 컬렉션 |
| --- | --- | --- |
| 저장 방식 | 문서 그대로 저장 | 시간 기준 버킷에 묶어서 저장 |
| 저장 포맷 | 행 기반 | 컬럼형 (압축률 높음) |
| 스키마 제약 | 없음 | timeField, metaField 지정 필요 |
| 업데이트 | 자유롭게 CRUD 가능 | metaField만 수정 가능 |
| 내부 구조 | 실제 컬렉션 그 자체 | 내부 컬렉션 위의 비구체화 뷰 |

### metaField

데이터의 출처를 식별하는 태그 역할이다. MongoDB는 metaField 값을 기준으로 데이터를 그룹화하여 같은 버킷에 저장한다.

```jsx
// 센서 A 데이터 → 같은 버킷
{ timestamp: ISODate("10:00"), metadata: { sensorId: "A", location: "서울" }, temp: 25 }
{ timestamp: ISODate("10:01"), metadata: { sensorId: "A", location: "서울" }, temp: 26 }

// 센서 B 데이터 → 다른 버킷
{ timestamp: ISODate("10:00"), metadata: { sensorId: "B", location: "부산" }, temp: 22 }
```

metaField에 여러 필드가 있으면 **객체 전체가 일치해야** 같은 버킷에 들어간다. 필드가 하나라도 다르면 별도 버킷으로 분리된다. 따라서 metaField에는 데이터 소스 식별에 필요한 최소한의 필드만 넣는 것이 좋다.

### 버킷 제한

- 버킷 하나의 최대 크기는 1000개 측정값 또는 125KB 중 작은 쪽이다.
- 카디널리티가 높은 경우(고유한 metaField 값이 너무 많은 경우) MongoDB가 버킷 크기를 더 줄일 수 있다.

### 인덱스 주의사항

- MongoDB 6.3부터 metaField + timeField에 복합 인덱스가 자동 생성된다.
- timeField, metaField, 측정값 필드 모두 복합 인덱스에 포함할 수 있다.
- 개별 문서가 아닌 버킷 단위로 인덱싱되며, 각 필드의 최솟값과 최댓값을 인덱싱한다.
- 멀티키 인덱스, 2d 인덱스, 스파스 인덱스는 metaField에만 생성 가능하다.
- metaField와 timeField에는 부분 인덱스(partial index)를 만들 수 없다.
- `_id` 필드에 기본 인덱스가 자동 생성되지 않는다.
- metaField 인덱스는 필터링/동등 비교에, timeField 인덱스는 범위 쿼리에 사용하는 것이 권장된다.
- `distinct` 명령은 비효율적이므로 `$group` 집계를 대신 사용해야 한다.

### explain과 $cursor

시계열 컬렉션은 내부적으로 aggregation 파이프라인으로 실행된다. explain 출력에서 `$cursor`는 내부 버킷 컬렉션에서 데이터를 읽어오는 스테이지이다.

```jsx
// 일반 컬렉션
db.normal.find({...}).explain()
// → queryPlanner.winningPlan

// 시계열 컬렉션
db.weather.find({...}).explain()
// → stages[0].$cursor.queryPlanner.winningPlan
```

`CLUSTERED_IXSCAN`이면 내부 클러스터형 인덱스를, `IXSCAN`이면 보조 인덱스를 사용한 것이다.
