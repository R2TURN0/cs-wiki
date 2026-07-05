## 식별 관계 vs 비식별 관계

### 식별 관계 (Identifying Relationship)
자식 엔티티의 PK(기본키)에 부모의 FK가 포함됩니다.
즉, 자식은 부모의 식별자 없이는 자기 자신을 식별할 수 없는 존재입니다.
예: 주문 (PK: order_id) — 주문상세 (PK: order_id + line_no). 주문상세는 주문 없이는 존재 의미가 없고, PK 자체가 부모 키에 의존.

### 비식별 관계 (Non-identifying Relationship)
자식 엔티티의 FK가 일반 컬럼으로만 존재하고, PK와는 무관합니다.
자식은 자기 고유의 PK(보통 surrogate key)를 따로 가지며, 부모는 그냥 "참조"만 함.
예: 부서 (PK: dept_id) — 직원 (PK: emp_id, FK: dept_id). 직원은 부서가 바뀌어도 여전히 같은 emp_id로 식별 가능.
