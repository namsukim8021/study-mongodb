📘 마스터링 MongoDB 7.0 - 9장: 다중 문서 ACID 트랜잭션

✅ ACID란?
ACID는 데이터베이스 트랜잭션에서 지켜야 하는 4가지 속성

약어	뜻	설명
- A	Atomicity (원자성) : 모든 작업이 전부 실행되거나 전혀 실행되지 않아야 함. 
  - 예시: 계좌 이체 시, 출금만 되고 입금이 안 되면 안 됨.
- C	Consistency (일관성) : 데이터는 항상 정해진 규칙을 따라야 함. 
  - 예시: 이체 전후의 전체 금액은 같아야 함. 
- I	Isolation (고립성)	여러 트랜잭션이 동시에 실행돼도 서로 영향을 주지 않아야 함. 
- D	Durability (지속성)	트랜잭션이 성공하면, 시스템 장애가 발생해도 결과는 영구 저장돼야 함.

✅ MongoDB의 트랜잭션 지원

MongoDB는 원래 단일 문서에 대해서만 ACID를 보장했지만,
MongoDB 4.0부터 다중 문서 트랜잭션을 지원하기 시작했습니다.
MongoDB 7.0에서는 더 안정적이고 성능이 개선된 트랜잭션 기능을 제공합니다.

✅ 왜 다중 문서 트랜잭션이 필요할까?

예를 들어, 쇼핑몰 주문 처리 시 다음 두 작업이 필요

- orders 컬렉션에 주문 정보를 저장
- stock 컬렉션에서 재고 감소

이 작업은 항상 함께 실행되어야 하고, 한쪽만 성공하면 데이터가 망가진다.

✅ MongoDB에서 트랜잭션 사용하기
MongoDB의 트랜잭션은 session을 이용해 수행한다.

▶ 기본 구조 (JavaScript 예시)
```javascript
const session = await client.startSession();

session.startTransaction();
try {
await orders.insertOne({ userId: 1, item: '책' }, { session });
await stock.updateOne({ item: '책' }, { $inc: { quantity: -1 } }, { session });

await session.commitTransaction();  // 성공 시 커밋
} catch (err) {
await session.abortTransaction();   // 실패 시 롤백
} finally {
await session.endSession();
}
```

✅ 트랜잭션 사용 시 주의사항
- 성능: 트랜잭션은 내부적으로 복잡한 처리를 하기 때문에 속도가 느려질 수 있음.
- 간단하게: 트랜잭션은 짧고 단순하게 구성하는 것이 좋음.
- 모든 작업은 세션 안에서: 트랜잭션에 포함되는 모든 작업은 같은 session 객체를 통해 수행해야 함.

✅ 트랜잭션이 필요한 대표적인 상황
- 은행 계좌 이체 (출금 + 입금)
- 쇼핑몰 주문 처리 (주문 생성 + 재고 감소)
- 회원 가입 (유저 정보 + 초기 설정 데이터 삽입)
- 예약 시스템 (좌석 차감 + 예약 정보 기록)

✅ 요약 정리
- 기능	여러 문서를 하나의 트랜잭션으로 처리
- 지원 버전	MongoDB 4.0 이상
- 주요 메서드	startTransaction(), commitTransaction(), abortTransaction()
- 장점	데이터 정합성 보장
- 단점	성능 저하 가능성 있음
- 권장 사용	짧고 핵심적인 처리에만 사용

