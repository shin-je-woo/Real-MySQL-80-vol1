# 10. 실행 계획

## **10.1 통계 정보**

- MySQL 옵티마이저는 비용 기반 최적화를 수행하며 이를 위해 통계 정보를 사용한다.
- 통계 정보가 부정확하면 실행 계획이 크게 왜곡될 수 있다.
- MySQL 서버의 실행 계획은 EXPLAIN 명령으로 확인할 수 있으며 실행 계획을 이해하려면 MySQL 서버의 내부 처리 로직과 통계 정보의 의미를 알아야 한다.
- MySQL 8.0에서는 실행 계획의 정확도를 높이기 위해 통계 정보 관리 방식과 히스토그램 기능이 개선되었다.

### **10.1.1 테이블 및 인덱스 통계 정보**

- 비용 기반 최적화에서 가장 중요한 요소는 통계 정보다.
- 통계 정보가 정확하지 않으면 옵티마이저는 실제와 전혀 다른 실행 계획을 선택할 수 있다.
- MySQL 5.7 이전에는 테이블과 인덱스에 대해 제한적인 통계 정보만 사용했으며 데이터 분포를 알 수 없었다.
- MySQL 8.0부터는 인덱스와 컬럼의 **데이터 분포를 저장하는 히스토그램** 기능이 도입되었다.

### **MySQL 서버의 통계 정보 관리 방식**

MySQL 5.6 이전까지 InnoDB 통계 정보는 메모리에만 저장되었으며 서버 재시작 시 모두 초기화되었다.

MySQL 5.6부터는 InnoDB 통계 정보가 영구적으로 관리되도록 개선되었으며 다음 테이블에 저장된다.

- mysql.innodb_index_stats
- mysql.innodb_table_stats

이로 인해 서버 재시작 후에도 통계 정보가 유지된다.

### **통계 정보 컬럼 의미**

innodb_index_stats 주요 항목

- n_diff_pfx
    
    인덱스가 가진 유니크 값의 개수
    
- n_leaf_pages
    
    인덱스 리프 노드 페이지 수
    
- size
    
    인덱스 전체 페이지 수
    

innodb_table_stats 주요 항목

- n_rows
    
    테이블 전체 레코드 수
    
- clustered_index_size
    
    프라이머리 키 인덱스 크기
    
- sum_of_other_index_sizes
    
    프라이머리 키를 제외한 인덱스 크기 합
    

### **통계 정보 갱신**

다음 경우 통계 정보가 자동으로 갱신될 수 있다.

- 테이블 오픈
- 대량 INSERT UPDATE DELETE 발생
- ANALYZE TABLE 실행
- SHOW TABLE STATUS 또는 SHOW INDEX 실행
- InnoDB 모니터 활성화

통계 정보 자동 갱신은 innodb_stats_auto_recalc 시스템 변수로 제어할 수 있다.

### **10.1.2 히스토그램**

- 기존 통계 정보는 유니크 값 개수만 제공하므로 데이터 분포의 불균형을 반영하지 못했다.
- MySQL 8.0부터는 컬럼 단위 데이터 분포를 저장하는 히스토그램 기능이 도입되었다.
- 히스토그램은 자동 수집되지 않으며 ANALYZE TABLE … UPDATE HISTOGRAM 명령으로 수동 생성한다.
- 히스토그램 정보는 information_schema.column_statistics 테이블에 저장된다.

### **히스토그램과 인덱스 관계**

- 인덱스가 존재하는 컬럼에 대해 일치하는 레코드 건수를 예측하기 위해 옵티마이저는 실제 인덱스의 B-Tree를 샘플링해서 살펴본다. 이 작업을 인덱스 다이브라고 하며, 이 작업이 히스토그램보다 우선된다.
- 히스토그램은 주로 인덱스가 없는 컬럼의 조건 선택도 추정에 사용된다.

### **10.1.3 코스트 모델**

- MySQL 서버는 쿼리 실행 시 필요한 작업을 여러 단위 작업으로 분해하고 각 작업 비용을 합산하여 실행 계획을 선택한다.
- 이때 사용하는 비용 기준을 코스트 모델이라 한다.

### **비용 관리 테이블**

MySQL 8.0에서는 코스트 모델 설정값이 다음 테이블에 저장된다.

- server_cost
    
    인덱스 비교
    
    레코드 평가
    
    임시 테이블 처리 비용
    
- engine_cost
    
    데이터 페이지 읽기 비용
    

### **주요 비용 항목 의미**

engine_cost

- io_block_read_cost
- memory_block_read_cost
- disk_temptable_create_cost
- disk_temptable_row_cost

server_cost

- key_compare_cost
- memory_temptable_create_cost
- memory_temptable_row_cost
- row_evaluate_cost

### **코스트 모델 설정 주의사항**

- 코스트 모델 값은 MySQL 서버 내부 동작과 밀접하게 연결되어 있다.
- 기본값은 오랜 기간 실전에서 검증된 값이므로 명확한 목적 없이 변경하지 않는 것이 바람직하다.

## **10.2 실행 계획 확인**

MySQL 서버의 실행 계획은 DESC 또는 EXPLAIN 명령으로 확인할 수 있다.

MySQL 8.0부터는 EXPLAIN 명령에 다양한 FORMAT 옵션이 추가되었다.

### **10.2.1 실행 계획 출력 포맷**

MySQL 8.0에서는 다음 출력 포맷을 제공한다.

- 기본 테이블 포맷
- TREE 포맷
- JSON 포맷

FORMAT 옵션을 통해 출력 형태만 달라질 뿐 실행 계획의 본질은 동일하다.

### **10.2.2 쿼리의 실행 시간 확인**

MySQL 8.0.18부터 EXPLAIN ANALYZE 기능이 추가되었다.

EXPLAIN ANALYZE는 실제 쿼리를 실행한 후 다음을 출력한다.

- 단계별 실제 실행 시간
- 처리된 레코드 수
- 반복 횟수

실행 시간이 오래 걸리는 쿼리를 분석할 때 유용하지만 먼저 EXPLAIN으로 실행 계획을 검토한 후 사용하는 것이 권장된다.

## **10.3 실행 계획 분석**

실행 계획 분석에서 중요한 것은 출력 포맷이 아니라 각 컬럼이 의미하는 바를 이해하는 것이다.

이 절에서는 기본 테이블 포맷 기준으로 실행 계획 컬럼을 설명한다.

### **10.3.1 id 컬럼**

id 컬럼은 단위 SELECT 쿼리의 식별자다.

- 실행 순서를 의미하지 않는다.
- 조인된 테이블은 동일 id 값을 가질 수 있다.
- 서브쿼리가 포함되면 id 값이 증가한다.

실제 실행 순서는 TREE 포맷을 통해 확인해야 한다.

### **10.3.2 select_type 컬럼**

select_type은 단위 SELECT 쿼리의 유형을 나타낸다.

### **SIMPLE**

- UNION이나 서브쿼리를 사용하지 않은 단순 SELECT
- 조인이 포함되어 있어도 SIMPLE로 표시

### **PRIMARY**

- UNION이나 서브쿼리를 포함한 쿼리에서 가장 바깥에 있는 SELECT

### **UNION**

- UNION으로 결합된 두 번째 이후 SELECT

### **DEPENDENT UNION**

- UNION으로 결합된 단위 SELECT 중에서 **외부 SELECT 쿼리의 컬럼 값에 의존하는 경우** 표시
- 외부 쿼리 결과에 따라 반복 실행된다.

### **UNION RESULT**

- UNION 결과를 저장하는 임시 테이블
- 실제 SELECT 쿼리는 아니며 id 값이 없다.
- MySQL 8.0 버전부터 UNION만 임시 테이블이 필요하고, UNION ALL은 임시 테이블이 필요 없다.

### **SUBQUERY**

- FROM 절 이외에서 사용된 서브쿼리

### **DEPENDENT SUBQUERY**

- 외부 SELECT의 컬럼 값을 참조하는 서브쿼리
- 외부 쿼리 결과에 따라 반복 실행된다.

### **DERIVED**

- FROM 절에 사용된 서브쿼리
- DERIVED는 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블(파생 테이블)을 생성하는 것을 의미한다.

### **DEPENDENT DERIVED**

- 외부 컬럼을 참조하는 FROM 절 서브쿼리
- MySQL 8.0부터 LATERAL JOIN으로 지원된다.

### **UNCACHEABLE SUBQUERY**

- 캐시가 불가능한 서브쿼리
- 대표적인 원인
    - 사용자 변수 사용
    - 비결정적 함수 사용
    - RAND(), UUID() 등

### **MATERIALIZED**

- IN 서브쿼리 최적화를 위해 서브쿼리 결과를 임시 테이블로 구체화하여 처리
