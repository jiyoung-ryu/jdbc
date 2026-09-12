# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 이 저장소의 성격

스프링 DB 접근 기술을 단계별로 학습하는 **학습용 저장소**다. 프로덕션 코드가 아니라 진도에 따라 같은 기능을 점진적으로 개선한 버전들(`V0` → `V5`)이 **모두 남아 있는** 것이 정상이다.

- 구버전(`MemberRepositoryV0`, `MemberServiceV1` 등)은 죽은 코드가 아니다. **지우거나 리팩터링해 통합하지 말 것.** 각 버전은 그 단계의 문제점을 보여주기 위해 존재한다.
- 커밋은 학습 단계 단위로 하나씩 쌓는다. 사용자가 "N. 제목"만 말하면 그 단계에 해당하는 워킹 트리 변경만 커밋하면 된다.
- 다음 단계용 코드가 미리 주석 처리돼 있는 경우가 많다(예: `MemberServiceV4Test`의 저장소 빈 선택). 주석을 임의로 정리하지 말 것.

## 개발 명령

```bash
# 빌드 / 컴파일 확인 (커밋 전 검증용으로 주로 사용)
./gradlew compileTestJava
./gradlew build

# 전체 테스트 (H2 서버가 떠 있어야 함)
./gradlew test

# 단일 테스트 클래스 / 메서드
./gradlew test --tests 'hello.jdbc.service.MemberServiceV4Test'
./gradlew test --tests 'hello.jdbc.service.MemberServiceV4Test.accountTransfer'

# 애플리케이션 실행
./gradlew bootRun
```

포매터·린터 설정은 없다.

## H2 데이터베이스 (테스트 실행 전 필수)

대부분의 테스트가 **실제 H2 서버에 TCP로 접속**한다. 서버가 떠 있지 않으면 전부 실패한다.

```bash
h2/bin/h2.sh   # 로컬에 내려받은 H2 배포판 (git에는 포함되지 않음)
```

- 접속 정보는 두 곳에 중복돼 있다: `ConnectionConst`(순수 JDBC 예제·테스트용)와 `application.properties`(스프링 부트 자동 등록용). **URL을 바꿀 때는 둘 다 고쳐야 한다.**
- 두 URL 모두 `jdbc:h2:tcp://localhost/<절대경로>/data/test` 형태의 **로컬 절대경로**다. 다른 머신에서는 그대로 동작하지 않는다.
- `member` 테이블 DDL은 저장소에 없다. H2 콘솔에서 직접 만들어야 한다:
  ```sql
  create table member (
      member_id varchar(10) primary key,
      money integer not null default 0
  );
  ```

## 아키텍처: 버전별로 무엇이 달라지는지

핵심 도메인은 `Member`(memberId, money) 하나뿐이고, 모든 버전이 같은 CRUD와 계좌이체 로직을 구현한다. 버전 간 차이는 **커넥션을 어떻게 얻고, 트랜잭션을 어떻게 유지하고, 예외를 어떻게 다루는가**에 있다.

### 저장소 (`hello.jdbc.repository`)

| 버전 | 커넥션 획득 | 예외 | 인터페이스 |
|---|---|---|---|
| `V0` | `DriverManager` 직접 | `throws SQLException` | 없음 |
| `V1` | `DataSource` 주입 + `JdbcUtils`로 정리 | `throws SQLException` | 없음 |
| `V2` | 위 + `Connection`을 파라미터로 받는 오버로드 추가 | `throws SQLException` | 없음 |
| `V3` | `DataSourceUtils.getConnection/releaseConnection` | `throws SQLException` | 없음 |
| `V4_1` | `DataSourceUtils` | `MyDbException`(직접 정의한 런타임 예외) | `MemberRepository` |
| `V4_2` | `DataSourceUtils` | `SQLExceptionTranslator` → `DataAccessException` | `MemberRepository` |
| `V5` | `JdbcTemplate`이 전부 처리 | `JdbcTemplate`이 전부 처리 | `MemberRepository` |

`MemberRepositoryEx`는 구현체 없는 인터페이스다. 체크 예외(`throws SQLException`)가 인터페이스까지 오염시키는 문제를 보여주기 위한 대비용이므로 삭제하면 안 된다.

`V2`의 `findById(Connection, ...)` / `update(Connection, ...)` 오버로드는 커넥션을 파라미터로 넘겨 트랜잭션을 유지하는 방식 전용이라 **저장소에서 커넥션을 닫지 않는다**. `V3` 이후로는 `DataSourceUtils`가 트랜잭션 동기화 매니저에 보관된 커넥션을 대신 찾아주므로 이 오버로드가 사라진다.

### 서비스 (`hello.jdbc.service`)

| 버전 | 트랜잭션 처리 |
|---|---|
| `V1` | 없음 (예외 시 데이터 정합성이 깨지는 것을 보여줌) |
| `V2` | `Connection`을 직접 다뤄 `setAutoCommit(false)` / `commit` / `rollback`, 파라미터로 전달 |
| `V3_1` | `PlatformTransactionManager` |
| `V3_2` | `TransactionTemplate` |
| `V3_3` | `@Transactional` AOP |
| `V4` | `@Transactional` + `MemberRepository` 인터페이스 의존 (`SQLException` 완전 제거) |

`V2`의 `release()`는 커넥션 풀 반환을 고려해 닫기 전에 `setAutoCommit(true)`로 되돌린다.

### 예외 (`hello.jdbc.repository.ex`, 테스트의 `hello.jdbc.exception`)

`MyDbException` ← `MyDuplicateKeyException` 계층은 "직접 만든 데이터 접근 예외"에 해당한다. `V4_2` 이후 스프링의 `DataAccessException`으로 대체되지만 `V4_1`과 `ExTranslatorV1Test`가 아직 쓰므로 남겨둔다.

`src/test/.../exception/basic`의 테스트들은 프로덕션 코드를 검증하지 않는 **학습용 테스트**다. 체크/언체크 예외 동작, 예외 전환, 스택 트레이스 유지를 보여주기 위해 테스트 클래스 안에 `Controller`/`Service`/`Repository` 정적 중첩 클래스를 직접 정의한다.

### 테스트에서 구현체 갈아끼우기

`MemberServiceV4Test`는 `@SpringBootTest` + `@TestConfiguration`으로 빈을 등록하고, `memberRepository()` 빈에서 `V4_1` / `V4_2` / `V5` 중 하나만 주석을 풀어 선택한다. 서비스 코드를 건드리지 않고 저장소를 교체할 수 있다는 것이 이 테스트의 요점이다.

`MemberServiceV3_3Test`는 `AopUtils.isAopProxy()`로 `@Transactional`이 프록시를 만들었는지 확인한다. `MemberServiceV3_4Test`부터는 `DataSource`와 `PlatformTransactionManager`를 직접 등록하지 않고 스프링 부트의 자동 등록에 맡긴다.

## 커밋

전역 지침대로 Conventional Commits를 따른다. 이 저장소의 기존 커밋은 **제목과 본문을 한국어로** 쓰고, 스코프는 패키지 이름(`connection`, `repository`, `service`, `config`, `exception`)을 쓴다. 학습용 테스트만 추가하는 단계는 `test(...)`, 기능 단계는 `feat(...)`를 쓴다.
