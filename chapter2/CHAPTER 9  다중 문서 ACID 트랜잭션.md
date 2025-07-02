# 📘 마스터링 MongoDB 7.0 - 9장: 다중 문서 ACID 트랜잭션 #

## ✅ ACID란?
ACID는 데이터베이스 트랜잭션에서 지켜야 하는 4가지 속성

약어	뜻	설명
- A	Atomicity (원자성) : 트랜잭션이 두 가지 결과중 하나만을 가질 수 있음을 의미.
  - 예시: A계좌에서 B계좌로 이체 시, A계좌에서 출금만 되고 B계좌에 입금이 안 되면 안 됨.
- C	Consistency (일관성) : 모든 데이터베이스 작어은 성공하거나 실패하더라도 반드시 일관된 상태를 유지.
  - 최종 일관성 : 분산 데이터 시스템에서 가장 널리 사용되는 모델. 
    - 모든 노드가 시간이 지남에 따라 일관된 상태로 수렴.
  - 강한 일관성 : 모든 후속 읽기 작업에서 항상 최근에 커밋된 쓰기 값을 확인 할 수 있도록 보장.
- I	Isolation (격리성)	여러 트랜잭션이 동시에 실행돼도 서로 영향을 주지 않아야 함. 
  - 격리성과 관련된 문제
    - 팬텀 읽기 Phantom Read : 한 트랜잭션이 실행되는 동안 다른 트랜잭션이 해당 결과 세트에 영향을 주는 데이터를 추가/삭제
    - 반복 불간으한 읽기 non-repeatable read : 하나의 트랜잭션에서 동일한 데이터를 여러 번 읽을때 다른 트랜잭션에 의해 해당 데이터가 변경되어 서로 다른 결과값을 리턴
    - 더티 읽기 Dirty Read : 트랜잭션이 커밋되지 않은 데이터를 읽는 경우. 데이터 불일치.
- D	Durability (지속성)	완료된 트랜젝션 결과가 손실되지 않고 영구적으로 저장되어야 함.
  - 예시: 트랜잭션이 성공적으로 커밋되면, 시스템 장애가 발생하더라도 해당 데이터는 복구 가능해야 함.
  - 관계형 DB에서는 선행기록로그 ( WAL, Write-Ahead Logging )를 사용하여 트랜잭션의 지속성을 보장.
  - MongoDB에서는 WiredTiger 스토리지 엔진을 사용하여 트랜잭션의 지속성을 보장.

## ✅ MongoDB의 ACID 구현
- 단일 문서 연산에 관해 원자성을 기본적으로 보장.
- 다중 문서 ACID 트랜잭션이 필요한 주된 이유는 단일 문서 크기 제한인 16MB 초과하는 경우
- 읽기 보장 수준
  - local : 서버의 최신 데이터를 즉시 반환하지만, 롤백 가능성 있음
  - majority : 복제본 세트 구성원의 과반수가 해당 데이터를 승인. 효력을 가지려면 majority 쓰기 보장 수준으로 커밋 
  - snapshot : majority 쓰기 보장으로 커밋된 데이터의 일관된 스냅샷 제공.
- 쓰기 보장 수준
  - majority : 몽고 5.0의 기본 설정. 복제본 세트 과반수의 승인을 요구
- 제약 사항
  - capped(고정된 사이즈) 컬렉션, 시스템 컬렉션에서는 쓰기 작업이 불가능
  - 몽고 4.2 부터는 killCursors를 트랜잭션의 첫 번째 작업으로 지정할 수 없음.
  - 트랜잭션 내에ㅓ 컬렉션 생성과 같은 DDL 작업은 제한적
  - 샤딩 클러스터에서 local 읽기 보장 수준을 사용할 경우, 모든 샤드에서 동일한 스냅샷 뷰의 데이터를 보장 받지 못함. 데이터의 일관된 스냅샷이 필요한 경우 snapshot 읽기 보장 수준을 사용해야 함.
  - listCollections, listIndexes 명령과 관련 도우미 메소드는 트랜잭션 내에서 사용할 수 없음.
  - 스냅샷 읽기 보장 수준은 트랜잭션이 majority 쓰기 보장으로 커밋된 되었을때만 과반수가 승인한 데이터 스냅샷으로 제공.

## ✅ 왜 다중 문서 트랜잭션이 필요할까?
예를 들어, airbnb 주문 처리 시 다음 두 작업 필요
- users 컬렉션에 사용자 정보를 저장
- listingsAndReview 특정 숙소의 예약 날짜 정보 갱신.

이 작업은 항상 함께 실행되어야 하고, 한쪽만 성공하면 데이터가 망가진다.

## ✅ MongoDB에서 트랜잭션 사용하기
MongoDB의 트랜잭션은 session을 이용해 수행한다.

▶ 코드 (JavaScript 예시)
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

▶ 코드 (kotlin 예시)
```kotlin
import com.mongodb.client.MongoClients
import com.mongodb.client.ClientSession
import com.mongodb.client.MongoCollection
import com.mongodb.client.MongoDatabase
import org.bson.Document

fun main() {
    val client = MongoClients.create("mongodb://localhost:27017")
    val database: MongoDatabase = client.getDatabase("shop")
    val orders: MongoCollection<Document> = database.getCollection("orders")
    val stock: MongoCollection<Document> = database.getCollection("stock")

    val session: ClientSession = client.startSession()

    try {
        session.startTransaction()

        // 1. 주문 문서 삽입
        val orderDoc = Document("userId", 1)
            .append("item", "책")
            .append("quantity", 1)
        orders.insertOne(session, orderDoc)

        // 2. 재고 감소
        stock.updateOne(
            session,
            Document("item", "책"),
            Document("\$inc", Document("quantity", -1))
        )

        // 3. 트랜잭션 커밋
        session.commitTransaction()
        println("트랜잭션 성공!")

    } catch (e: Exception) {
        println("트랜잭션 실패: ${e.message}")
        session.abortTransaction()
    } finally {
        session.close()
        client.close()
    }
}
```



## ✅ 모범사례
- 체계적인 모델링을 통해서 품부한 문서구조를 구현하면 다중 문서 ACID 트랜잭션 필요성을 최소화 할 수 있다.
- 트랜잭션의 기본 제한 시간은 60초. mongod 레벨에서 transactionLifetimeLimitSeconds 옵션을 통해 조정 가능.
- 트랜잭션 처리에서는 단일작업으로 수정할 수 있는 문서의 수가 1000개로 제한. 그 이상인 경우 여러단계로 나누어 배치 처리.
- oplog 특성 고려. oplog는 트랜잭션당 하나의 항목만 기록하며 16MB BSON 문서 크기 제한을 준수해야 합니다.
- 몽고DB는 트랜잭션 처리를 위해 Core API와 Callback API를 제공. Callback API가 더 포괄적인 기능을 제공하여 선호 된다. 최적화됨.

## ✅ 요약 정리
- 기능	여러 문서를 하나의 트랜잭션으로 처리
- 지원 버전	MongoDB 4.0 이상
- 주요 메서드	startTransaction(), commitTransaction(), abortTransaction()
- 장점	데이터 정합성 보장
- 단점	성능 저하 가능성 있음
- 권장 사용	짧고 핵심적인 처리에만 사용

