---
title: "Transactional Outbox 패턴 정리"
description: "DB 쓰기와 메시지 발행을 원자적으로 처리하는 패턴. dual-write 문제, 동작 방식, Polling vs CDC, 주의점"
publishDate: "5 May 2026"
tags: ["msa", "distributed-system", "outbox-pattern", "kafka"]
---

## 1. 어떤 문제를 푸는가 — Dual Write Problem

마이크로서비스에서 자주 마주치는 상황:

> 주문을 저장하고 → Kafka에 `OrderCreated` 이벤트를 발행해야 한다.

순진하게 짜면 이렇다.

```js
await db.orders.insert(order);        // 1) DB 저장
await kafka.publish("OrderCreated", order); // 2) 이벤트 발행
```

이 코드의 문제는 **둘 사이가 원자적이지 않다**는 점이다.

- 1)은 성공했는데 2) 직전에 서버가 죽으면? → DB에는 주문이 있는데 이벤트는 유실.
- 2)는 성공했는데 1)이 롤백되면? → 존재하지 않는 주문에 대한 이벤트가 떠다님.

DB와 메시지 브로커는 **서로 다른 시스템**이라 한 트랜잭션으로 묶을 수 없다. 분산 트랜잭션(2PC)을 쓰면 가능하긴 하지만 느리고 운영 부담이 크다. 이걸 **dual-write problem**이라고 한다.

## 2. Outbox 패턴의 아이디어

핵심은 단순하다.

> 메시지 브로커에 직접 보내지 말고, **같은 DB 트랜잭션 안에서 `outbox` 테이블에 기록**한 뒤, 별도 프로세스가 그걸 읽어서 발행한다.

이러면 비즈니스 데이터(`orders`)와 발행할 이벤트(`outbox`)가 **하나의 로컬 트랜잭션**으로 묶인다. DB가 ACID를 보장해주므로 둘은 항상 같이 커밋되거나 같이 롤백된다.

```sql
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (aggregate_id, event_type, payload, created_at)
    VALUES (...);
COMMIT;
```

이후 별도 워커(또는 CDC)가 `outbox`를 폴링해서 Kafka로 발행하고, 발행 성공한 row는 처리 완료 표시를 한다.

## 3. Outbox 테이블 스키마 예시

```sql
CREATE TABLE outbox (
  id           BIGSERIAL PRIMARY KEY,
  aggregate_id VARCHAR(64) NOT NULL,   -- 예: order_id
  event_type   VARCHAR(64) NOT NULL,   -- 예: OrderCreated
  payload      JSONB       NOT NULL,   -- 이벤트 본문
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at TIMESTAMPTZ              -- 발행 완료 시각, NULL이면 미발행
);

CREATE INDEX idx_outbox_unprocessed
  ON outbox (created_at)
  WHERE processed_at IS NULL;
```

미발행 row를 빠르게 찾기 위해 부분 인덱스(`WHERE processed_at IS NULL`)를 거는 게 일반적이다.

## 4. 발행 방식 — Polling vs CDC

Outbox에서 Kafka로 옮기는 방식은 크게 두 가지다.

### 4-1. Polling Publisher

워커가 주기적으로 `outbox`에서 미처리 row를 읽어 발행한다.

```
SELECT * FROM outbox
WHERE processed_at IS NULL
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

- 장점: 구현이 단순. 추가 인프라 필요 없음.
- 단점: 폴링 주기만큼 지연(latency). 주기를 짧게 하면 DB 부하 증가.

### 4-2. CDC (Change Data Capture)

DB의 트랜잭션 로그(WAL, binlog 등)를 읽어 outbox 테이블의 변경을 실시간 스트림으로 만든다. **Debezium**이 사실상 표준.

- 장점: 거의 실시간. 애플리케이션 코드와 분리됨. DB 부하 적음.
- 단점: Debezium + Kafka Connect 등 인프라 필요. 운영 복잡도 ↑.

규모가 작으면 Polling, 처리량이 크고 지연 민감하면 CDC.

## 5. 보장하는 것과 보장 못 하는 것

### 보장: At-least-once delivery

이벤트는 **최소 한 번** 발행된다. 발행 후 `processed_at` 업데이트 직전 워커가 죽으면 다음 워커가 같은 row를 다시 발행한다.

따라서 **컨슈머는 반드시 멱등(idempotent)** 해야 한다. 이벤트에 고유 ID(`event_id`)를 넣고 컨슈머 측에서 중복 처리한다.

### 보장 안 함: 순서

워커를 여러 개 띄우거나 파티션을 나누면 순서가 뒤섞일 수 있다. 같은 aggregate(예: 같은 `order_id`)에 대한 이벤트 순서가 중요하다면:

- Kafka 파티션 키를 `aggregate_id`로 지정
- 또는 같은 aggregate는 같은 워커가 처리하도록 라우팅

### Exactly-once는 아님

"정확히 한 번"은 일반적으로 불가능하다. At-least-once + 멱등 컨슈머 조합으로 **사실상 exactly-once 효과**를 낸다.

## 6. 자주 하는 실수

- **outbox INSERT를 다른 트랜잭션으로 분리** — 패턴의 의미가 사라진다. 반드시 비즈니스 로직과 같은 트랜잭션 안에서.
- **payload에 엔티티를 그대로 직렬화** — 스키마 변경에 취약하다. 이벤트는 별도 스키마(versioning 포함)로 정의.
- **outbox 테이블을 안 비움** — 무한정 쌓이면 인덱스/스토리지 부담. 발행 완료된 row는 일정 기간 후 삭제하거나 아카이브.
- **컨슈머의 멱등성 가정 안 함** — 운영 중 반드시 중복 발행이 발생한다.

## 7. 언제 쓰고 언제 안 쓰나

**쓸 때**
- DB 트랜잭션과 외부 발행(이벤트, API 호출)을 함께 처리해야 할 때
- Saga 패턴의 각 스텝에서 다음 스텝으로 안전하게 이벤트를 넘길 때
- 이벤트 유실이 비즈니스 임팩트가 큰 도메인 (결제, 주문, 재고 등)

**안 써도 될 때**
- 발행 실패해도 무방한 비핵심 이벤트 (예: 분석용 로그)
- 단일 서비스 내에서 끝나는 작업

## 정리

- Outbox 패턴은 **dual-write 문제**를 해결하는 패턴이다.
- 비즈니스 데이터와 outbox row를 **하나의 로컬 트랜잭션**으로 묶어 원자성을 확보한다.
- 별도 워커(Polling) 또는 CDC(Debezium)가 outbox를 읽어 브로커로 발행한다.
- 보장은 **at-least-once**이며, 컨슈머의 멱등성이 전제되어야 한다.
- Saga 같은 분산 트랜잭션 패턴의 **신뢰성 있는 이벤트 발행 메커니즘**으로 자주 함께 쓰인다.
