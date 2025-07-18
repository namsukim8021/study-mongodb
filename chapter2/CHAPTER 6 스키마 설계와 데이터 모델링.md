# 📘 Mastering MongoDB 7.0 - Chapter 6
## 스키마 설계와 데이터 모델링 (Schema Design and Data Modeling)

---

**주요 내용**
- 스키마 설계의 기본 개념
- 중첩 문서 vs 참조 설계
- 고급 모델링 기법
- 실전 적용 예시 및 성능 고려사항

---

## 🧩 MongoDB 설계 철학

**MongoDB의 특징**
- 스키마 유연성 (Schema-less)
- 문서 지향(Document-oriented)
- JSON 기반의 BSON 저장 구조

**설계 전략 핵심**
- 애플리케이션 중심 설계
- 데이터 액세스 패턴 기반 설계

> MongoDB는 자유롭지만, 체계 없이 설계하면 나중에 비용이 커진다.

---

## 🧮 정규화 vs 비정규화

| 설계 방식 | 장점 | 단점 |
|----------|------|------|
| 정규화 | 중복 제거, 데이터 일관성 | 조인 성능 저하, 쿼리 복잡 |
| 비정규화 | 읽기 성능 향상, 문서 자체 완결성 | 중복 증가, 업데이트 복잡 |

> MongoDB는 비정규화를 선호하지만, 상황에 따라 조절 필요

---

## 🧱 중첩 vs 참조

**문서 중첩 (Embedded)**
- 빠른 조회, 단일 문서 처리
- 예시: 댓글, 주소, 사용자 프로필

**문서 참조 (Reference)**
- 데이터 재사용, 독립적 관리
- 예시: 사용자 - 주문, 상품 - 카테고리

> 중첩은 조회 최적화, 참조는 구조 안정성 확보에 유리

---

## 🧠 고급 설계 기법

- **Bucket 패턴**: 시간 기반 그룹핑 (로그, 타임시리즈)
- **Tree 구조 표현**: 재귀 참조, 중첩 배열 활용
- **Document Versioning**: 스키마 진화 대응
- **Polymorphic Documents**: 다양한 타입의 문서 공존

> 실제 운영 환경에서는 다양한 패턴을 조합하여 사용

---

## 📂 스키마 진화 전략

**MongoDB의 유연성 활용**
- 필드 추가/삭제 자유
- 명시적 스키마 없이 운영 가능

**스키마 버전 관리 전략**
- `schemaVersion` 필드 사용
- 점진적 변환 → 전체 마이그레이션 불필요

---

## 📊 성능 고려사항

- BSON 문서 크기 제한: 16MB
- 배열 크기 및 중첩 깊이 주의
- 쿼리 성능을 고려한 인덱스 설계 필요
- 불필요한 중복 및 구조적 비효율 방지

---

## 🛒 실전 예제 - e커머스 시스템

**주요 Entity**
- 사용자(User), 상품(Product), 주문(Order)

**설계 전략**
- 주문 내 상품 정보 중첩
- 사용자 정보는 참조
- 상태 기록은 배열로 관리

```json
{
  "userId": "1234",
  "products": [
    { "productId": "p1", "qty": 2 },
    { "productId": "p2", "qty": 1 }
  ],
  "statusHistory": [
    { "status": "ordered", "timestamp": "2024-01-01T12:00:00Z" },
    { "status": "shipped", "timestamp": "2024-01-02T09:00:00Z" }
  ]
}
```

## 📝 마무리
**우리팀이 사용하고 있는 MongoDB 스키마 설계는?**
- collection : display_goods 
- field 레벨로 상품 정보와 상품 가격정보 적재

***개선한다면?***
- collection : display_goods_v2
- field 레벨로 필요한 상품 필드, 가격 필드를 꺼낸다.
- lookup을 통해 타 collection과 조인가능. aggregate를 통해 집계 쿼리 가능
