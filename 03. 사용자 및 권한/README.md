## **3.1 사용자 식별**

- MySQL은 **사용자 계정 + 접속 호스트(IP)** 두 가지를 함께 사용자 식별 정보로 사용한다.
- 같은 사용자 이름이라도 접속 호스트가 다르면 **완전히 다른 사용자로 취급**한다.
- 계정 정의 방식 예시
    - 'svc_id'@'127.0.0.1' → 로컬 접속 전용
    - 'svc_id'@'192.168.0.10' → 특정 서버에서만 접속 허용
    - 'svc_id'@'%' → 모든 외부 접속 허용(보안상 위험)
- 접속 가능한 호스트 범위는 보안을 위해 **가급적 좁게 설정할 것**을 권장한다.
- MySQL은 클라이언트가 접속할 때 서버는 가장 정확하게 일치하는 계정을 선택한다.
- 따라서 같은 사용자 이름이라도 IP 범위가 넓은 계정이 우선되지 않는다.

## **3.2 사용자 계정 관리**

### **3.2.1 시스템 계정과 일반 계정**

MySQL 8.0부터 계정에는 **SYSTEM_USER 권한을 가진 시스템 계정**과 **일반 계정**이 존재한다.

- **시스템 계정**
    - MySQL 서버 유지관리용 계정
    - 서버 시작/중지, 백업 복구, 스키마 변경 등 핵심 작업 수행
    - 대부분의 관리 권한 포함
- **일반 계정**
    - 애플리케이션에서 사용하는 일반 사용자 계정
    - 시스템 계정을 변경하거나 서버 관리 작업 수행 불가
    - DB 접근과 DML 중심의 권한을 가짐

이 구분은 보안을 위해 필수이며, **DBA 권한이어도 SYSTEM_USER를 임의로 부여하면 안 된다.**

또한 MySQL 서버에는 내부적으로 사용하는 기본 계정이 몇 개 있다.

- 'mysql.sys'@'localhost'
- 'mysql.session'@'localhost'
- 'mysql.infoschema'@'localhost'

이 계정들은 서버 기능에 필요한 계정이며 삭제하거나 비밀번호를 변경해서는 안 된다.

### **3.2.2 계정 생성**

MySQL 8.0에서 계정 생성은 **CREATE USER** 명령으로만 수행한다.

(5.7까지의 GRANT 방식 계정 생성은 deprecated)

- 기본 옵션
    - 인증 방식
    - 비밀번호 정책
    - 기본 역할(Role)
    - SSL 사용 여부
    - 계정 잠금 여부
- 예시

```sql
CREATE USER 'user'@'%'
IDENTIFIED WITH 'mysql_native_password' BY 'password'
REQUIRE NONE
PASSWORD EXPIRE INTERVAL 30 DAY
ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REUSE INTERVAL DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT;
```

### **인증 방식 IDENTIFIED WITH**

- MySQL은 다양한 인증 플러그인을 지원한다.
- 대표 인증 방식
    - **Native Authentication**
    - **Caching SHA-2 Authentication**(기본값)
    - **LDAP, PAM 등의 외부 인증 플러그인**
- 8.0에서는 Caching SHA-2 방식이 기본이며, Native 방식으로 바꾸고 싶다면

```sql
SET GLOBAL default_authentication_plugin='mysql_native_password';
```

### **3.2.3 비밀번호 관련 옵션**

### **REQUIRE**

- SSL 사용 여부 설정
- REQUIRE SSL 설정 시 암호화된 채널에서만 접속 허용

### **PASSWORD EXPIRE**

- 비밀번호 유효 기간 설정
- NEVER, DEFAULT, INTERVAL(n DAY)

### **PASSWORD HISTORY**

- 이전에 사용했던 비밀번호 재사용 제한
- 비밀번호 변경 시 히스토리를 mysql.password_history 테이블로 관리

### **PASSWORD REUSE INTERVAL**

- 같은 비밀번호 재사용까지 필요한 기간 설정

### **PASSWORD REQUIRE**

- 비밀번호 변경 시 기존 비밀번호를 요구할지 여부 설정
- PASSWORD REQUIRE CURRENT는 보안성이 높아 생산 환경에 권장

## **3.3 비밀번호 관리**

### **3.3.1 고수준 비밀번호 정책**

- MySQL은 **validate_password 컴포넌트**를 사용해 비밀번호 정책을 강화할 수 있다.
- 컴포넌트 설치 후 다음과 같은 변수들을 사용할 수 있다.
    - validate_password_length
    - validate_password_mixed_case_count
    - validate_password_special_char_count
    - validate_password_policy (LOW, MEDIUM, STRONG)
- STRONG 레벨에서는 금칙어 파일(dictionary)을 지정해 특정 단어 포함 여부도 검증 가능
- 금칙어 예시는 책에서 제공하는 Common Credentials 리스트 참고

### **3.3.2 이중 비밀번호(Dual Password)**

- MySQL 8.0은 두 개의 비밀번호를 동시에 설정할 수 있다.
- 비밀번호 변경 시 서비스 중단을 최소화하려는 목적
- Primary / Secondary 두 종류로 관리되며 아래처럼 기존 비밀번호를 유지한 채 새 비밀번호를 추가할 수 있다.

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;
```

## **3.4 권한(Privilege)**

MySQL 권한은 크게 **정적 권한**과 **동적 권한**으로 나뉜다.

### **정적 권한**

- MySQL 서버 설치 시 고정적으로 제공되는 권한
- 테이블, 뷰, 스키마, 프로시저 등 객체 단위 권한 포함
- 예: SELECT, INSERT, UPDATE, DELETE, CREATE, DROP 등

### **동적 권한**

- MySQL 8.0에서 추가된 권한
- 특정 서버 기능 제어, 백업, 레플리케이션, 암호화 키 관리 등 특수 작업에 사용
- 예:
    - BACKUP_ADMIN
    - ENCRYPTION_KEY_ADMIN
    - REPLICATION_APPLIER
    - ROLE_ADMIN
    - SYSTEM_USER

### **GRANT 명령**

- 권한 부여는 다음과 같이 수행한다.

```sql
GRANT SELECT, INSERT ON db.table TO 'user'@'host';
```

- 글로벌 권한과 객체 권한의 구분이 명확해야 한다.
- 전체 DB에 권한을 주는 잘못된 패턴을 피해야 한다.
- 테이블 특정 컬럼 단위 권한도 설정 가능하다.

## **3.5 역할(Role)**

MySQL 8.0에서 역할은 권한 묶음으로 제공된다.

### **역할 생성**

```sql
CREATE ROLE role_emp_read, role_emp_write;
```

### **역할에 권한 부여**

```sql
GRANT SELECT ON employees.* TO role_emp_read;
GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
```

### **계정에 역할 부여**

```sql
GRANT role_emp_read TO 'reader'@'127.0.0.1';
```

### **역할 활성화**

- 로그인 후 사용자가 직접 역할을 활성화해야 한다.
- 기본값은 활성화되지 않은 상태

```sql
SET ROLE role_emp_read;
```

### **자동 활성화**

- 모든 사용자에게 로그인 시 자동으로 역할을 활성화하려면

```sql
SET GLOBAL activate_all_roles_on_login=ON;
```

### **호스트별 역할 처리**

- 'reader'@'127.0.0.1'과 'reader'@'%'는 서로 다른 계정이며, 역할 부여도 별개로 처리된다.
- 역할과 계정의 host 부분은 서로 독립적으로 평가된다.

# **3장 핵심 정리**

- **사용자 계정은 ID + 호스트로 식별**하며, 호스트 범위 관리가 보안의 핵심이다.
- MySQL 8.0부터 **SYSTEM_USER**가 도입되어 관리 계정과 일반 계정을 명확히 구분한다.
- 계정 생성은 **CREATE USER**로만 수행하며, 비밀번호 정책 관련 옵션이 대폭 강화되었다.
- validate_password 컴포넌트를 통해 **비밀번호 복잡도 정책**을 강제할 수 있다.
- MySQL 8.0의 **이중 비밀번호** 기능은 서비스 중단 없이 비밀번호 변경을 가능하게 한다.
- 권한은 정적/동적으로 나뉘며, 동적 권한은 8.0의 주요 변화 중 하나다.
- 역할(Role)은 권한 관리의 편의성을 크게 높여주며, 자동 활성화 설정 여부가 중요하다.
