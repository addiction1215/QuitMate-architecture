# Quitmate 전체 아키텍처 설계 문서

## 1. 목적과 범위

Quitmate는 사용자의 흡연 기록, 금연 통계, 챌린지, 알림을 제공하는 모바일 중심 서비스다.
이 문서는 다음 세 저장소의 현재 소스에서 확인한 애플리케이션 구조를 설명한다.

- 프론트엔드: `/Users/chanhong/Desktop/git/addiction-front-app`
- 백엔드: `/Users/chanhong/Desktop/git/addiction`
- 관리자 서버: `https://github.com/addiction1215/quitmate-admin-server`

배포 계정, 네트워크 구성, 실제 DB 인스턴스/백업 정책은 소스만으로 확인되지 않았으므로 이 문서의 범위에서 제외한다.

## 2. 시스템 개요

사용자는 Expo 기반 React Native 앱(iOS, Android, Web)을 통해 Spring Boot API를 호출한다.
앱은 API 이외의 DB나 외부 서비스에 직접 접근하지 않는다. API 서버는 현재 업무 상태를 MySQL에,
과거 일별 흡연 이력을 MongoDB에 보관하며, 이미지·인증·메일·푸시·관측 서비스를 연동한다.
관리자 웹은 별도 Spring Boot 관리자 서버의 `/admin` 컨텍스트 경로로 접속하며, 운영 설정상 동일한 MySQL `addiction` DB를 조회·변경한다.

| 구성 요소 | 책임 | 주 데이터 또는 통신 |
| --- | --- | --- |
| Quitmate 앱 | 화면, 로컬 인증 정보, 서버 데이터 캐시, 푸시 수신 | HTTPS REST API, Expo Push |
| Quitmate API 서버 | JWT 인증, 도메인 규칙, 트랜잭션, 스케줄 작업 | MySQL, MongoDB, 외부 HTTP |
| 관리자 웹 | 운영자용 사용자·챌린지·문의 관리 화면 | 관리자 API |
| 관리자 API 서버 | 관리자 JWT/권한 확인, 운영 데이터 조회·변경 | `/admin`, MySQL, S3 |
| MySQL | 현재 상태와 트랜잭션 데이터 | 회원, Refresh Token, 기기 Push Token, 당일 흡연 기록, 챌린지, 미션, 알림, Push Outbox |
| MongoDB | 일별 시계열 통계와 과거 상세 이력 | `cigarette_history` |
| API 서버 내부 배치 | 날짜 전환과 예약 발송 준비 | 매일 자정 통계 이관, 매 분 피드백 Outbox 생성 |
| API 서버 내부 Outbox 워커 | 푸시 발송의 재시도/상태 관리 | 5초 단위 후보 처리, 실패 재시도 |
| 외부 서비스 | 각 기능의 전문 처리 | OAuth, SMTP, S3, Expo Push, Slack/Log Agent |

## 3. 아키텍처 다이어그램

- 인터랙티브 다이어그램: `quitmate-architecture.html`
- Archify 원본 명세: `quitmate-architecture.json`

다이어그램은 사용자의 일반 요청 경로를 강조하고, 통계 배치와 재시도 가능한 푸시 발송을 보조 흐름으로 분리한다.

## 4. 주요 데이터 흐름

### 4.1 로그인과 세션 유지

1. 앱은 이메일 또는 소셜 로그인 시 기기 ID와 Expo Push Token을 API에 전달한다.
2. API는 사용자 확인 후 Access Token과 Refresh Token을 발급하고, Refresh Token 및 기기 Push 정보를 저장한다.
3. 앱은 모바일 환경에서는 Expo SecureStore에 토큰을 보관하고, 일반 API 요청에 Access Token을 Bearer 헤더로 붙인다.
4. Access Token 오류로 401을 받으면 동시에 발생한 요청도 하나의 refresh 요청만 공유하고, 새 토큰으로 원 요청을 한 번 재시도한다.
5. Refresh Token 갱신이 실패하면 앱과 서버 양쪽의 세션을 종료한다.

### 4.2 흡연 기록과 통계 조회

1. 사용자가 흡연을 추가/취소하면 앱이 `PATCH /api/v1/cigarette/change`을 호출한다.
2. API는 당일 원본 기록을 MySQL에 저장하고, 마지막 흡연 시각·장소를 사용자 상태에 반영한다.
3. 앱은 React Query 캐시를 즉시 갱신한 뒤 서버의 최종 값을 다시 조회한다.
4. 자정 배치는 전날 MySQL 원본 기록을 사용자별 일 단위로 집계해 MongoDB에 저장한다.
5. 통계/캘린더 조회는 당일이면 MySQL 원본을, 과거 날짜면 MongoDB 일별 이력을 사용한다.

### 4.3 예약 피드백과 푸시 알림

1. 사용자는 네 개 시간대의 일일 피드백 예약을 설정한다.
2. 매 분 배치는 발송 시각이 도래하고 알림 설정이 켜진 사용자에 대해 `PushOutbox` 대기 건을 만든다.
3. 워커는 5초마다 대기/재시도 건을 선점한 뒤 Expo Push API로 보낸다.
4. 성공 시 Outbox를 `SENT`로 표시하고 앱 내 알림 이력을 저장한다.
5. 실패 시 재시도 시각을 정하고, 최대 시도 횟수에 도달하면 `FAILED`로 종료한다. 10분 이상 `PROCESSING` 상태에 남은 건은 복구 대상이다.

### 4.4 이미지 업로드

1. 앱은 이미지 파일을 멀티파트로 `/api/v1/storage/{bucketKind}`에 전송한다.
2. API는 S3에 업로드하고 객체 키를 반환한다.
3. 도메인 데이터에는 객체 키를 보관하며, 조회 시 필요한 경우 API가 제한 시간 Presigned URL을 만든다.

### 4.5 관리자 요청

1. 관리자 웹은 `admin.quitmate.co.kr`에서 관리자 API에 요청한다. 이 도메인은 관리자 서버의 CORS 허용 목록에서 확인된다.
2. 관리자 로그인 서비스는 `ROLE_ADMIN` 사용자만 토큰을 발급한다.
3. 관리자 API는 운영 설정에서 MySQL `addiction` DB를 사용해 사용자, 챌린지, 미션 이력, 공지·FAQ·문의 등을 조회·변경한다.
4. 관리자 서버의 MongoDB 의존성은 선언돼 있으나, 운영 설정에는 MongoDB 연결 정보가 없으므로 이 문서에서는 관리자 서버가 MongoDB를 사용한다고 가정하지 않는다.

## 5. 데이터 소유권과 설계 의도

### MySQL

관계와 상태 변경이 중요한 현재 데이터의 원천이다. 회원, 토큰, 기기 Push Token, 챌린지/미션 이력,
알림 설정/이력/Outbox, 그리고 자정 이전 당일 흡연 원본을 담당한다.

### MongoDB

`cigarette_history` 시계열 컬렉션은 `smokeDate`를 시간 필드, `userId`를 메타 필드로 사용한다.
과거 일자의 그래프, 캘린더, 상세 이력을 사용자·기간 조건으로 조회하는 목적이다.

### 두 DB 사이의 이관

자정 배치는 MySQL 원본을 사용자별 일별 문서로 압축해 MongoDB에 저장한 뒤 원본을 삭제한다.
따라서 과거 통계의 조회 비용과 MySQL 원본 테이블의 누적을 줄이는 의도다.

이 이관은 서로 다른 DB에 걸쳐 실행되므로 단일 트랜잭션이 아니다. 현재 코드는 개별 Mongo 저장 실패를 기록한 뒤에도
마지막에 전날 MySQL 원본 전체를 삭제한다. 운영 전에는 성공한 사용자 기록만 삭제하는지, 재실행/중복 방지/복구 정책이 있는지를 별도로 검토해야 한다.

## 6. 클라이언트 상태 경계

| 위치 | 보관 대상 | 역할 |
| --- | --- | --- |
| Expo SecureStore | Access/Refresh Token, 기기 ID | 로그인 세션 유지 |
| React Query | API 조회 결과 | 서버 데이터 캐시, 변경 후 무효화/재조회 |
| Zustand | 설문 입력, UI 상태, 인증 UI 상태 | 화면 간 임시 상태 |
| MySQL/MongoDB | 서비스의 영속 데이터 | 모든 기기에서 공유되는 기준 데이터 |

## 7. Prometheus 적용 상태와 관리자 노출 방식

### 현재 확인된 상태

메인 API 서버에는 `spring-boot-starter-actuator`와 `micrometer-registry-prometheus`가 포함돼 있다.
운영 프로필은 `/actuator/prometheus`를 노출하고 Prometheus endpoint를 활성화한다. WAS는 Blue/Green 배포 시 내부 `127.0.0.1:8080` 또는 `127.0.0.1:8081`에서 실행되고, Nginx가 활성 포트로 프록시한다.

그러나 이 저장소에는 Prometheus 서버의 `scrape_configs`, 컨테이너/인프라 설정, Targets 상태가 없다. 서버 접속 권한도 없으므로 **실제 Prometheus가 해당 exporter를 수집 중인지는 확인 불가**다. 다이어그램의 `Prometheus exporter` 표기는 수집 서버가 존재한다는 뜻이 아니라, API 서버가 수집 가능한 형식으로 메트릭을 노출한다는 뜻이다.

### 관리자가 메트릭을 보는 권장 구조

가능하다. 다만 관리자 웹이 `/actuator/prometheus` 또는 Prometheus API를 직접 호출하면 인증 정보와 임의 PromQL 실행 권한이 브라우저에 노출된다. 다음 경로로 제한한다.

```text
관리자 웹 → 관리자 API → Prometheus HTTP API (/api/v1/query, /api/v1/query_range) → Prometheus → Quitmate API exporter
```

1. Prometheus는 서버 내부망에서 메인 API의 `/actuator/prometheus`를 scrape한다.
2. 관리자 API에 `GET /admin/api/v1/monitoring/summary` 같은 관리자 전용 endpoint를 만든다.
3. 이 endpoint는 CPU/JVM heap/HTTP 오류율/응답 시간/DB 커넥션 등 사전에 허용한 쿼리만 실행해 DTO로 반환한다.
4. Prometheus URL·인증 정보는 관리자 서버 환경 변수로 주입하고, Prometheus 자체는 외부 인터넷에 노출하지 않는다.
5. `ROLE_ADMIN` 권한, 짧은 조회 범위, timeout·rate limit·감사 로그를 적용한다.

Prometheus가 아직 없다면 exporter를 다시 만드는 것이 아니라 Prometheus 서버와 scrape 설정을 추가하면 된다. 실제 scrape 대상 포트는 운영 Nginx/방화벽 구성 확인 후 내부 8080/8081 또는 프록시 경로 중 하나로 확정해야 한다.

## 8. Confluence 공유·적용 방식

Confluence Cloud는 HTML의 JavaScript를 페이지 본문에서 실행하지 않으므로, 인터랙티브 HTML을 그대로 붙여넣지 않는다.

권장 구성은 다음과 같다.

1. `addiction1215` 조직의 별도 문서 저장소(예: `quitmate-architecture`)에 Archify JSON, HTML, 설계 문서를 버전 관리한다. 현재처럼 기존 HTML을 갱신해도 Git 이력으로 이전 버전과 비교할 수 있다.
2. 조직 정책상 공개 가능한 경우 GitHub Pages로 HTML을 배포한다. 비공개 Pages는 GitHub Enterprise Cloud가 필요하므로, 플랜/조직 정책을 먼저 확인한다.
3. Confluence 페이지에는 최신 구조의 PNG 또는 SVG 정적 이미지와 핵심 요약을 넣고, `인터랙티브 다이어그램 열기` 링크를 GitHub Pages 주소로 연결한다. 외부 도메인이 embed를 허용하지 않으면 Smart Link는 일반 링크로 표시될 수 있다.
4. HTML·JSON·Markdown은 Confluence 첨부 파일에도 함께 올린다. 동일 파일명을 다시 첨부하면 Confluence가 새 버전으로 관리한다.

권장 Confluence 페이지 제목은 `Quitmate 전체 아키텍처`이며, 본문에는 `목적 및 범위`, `구조 이미지`, `인터랙티브 링크`, `주요 데이터 흐름`, `Prometheus 적용 상태`, `운영 확인 항목` 순서로 배치한다.

## 9. 확인이 필요한 운영 전제

- API 기본 주소는 프론트 환경 변수 `EXPO_PUBLIC_API_URL`이며, 미설정 시 `https://api.quitmate.co.kr`를 사용한다.
- 실제 운영의 API Gateway, Load Balancer, Nginx, DB 네트워크 격리, 백업/복구, 모니터링 수집 경로는 이 문서에서 확정하지 않는다.
- `addiction1215` 조직에 문서 저장소 및 Pages를 만들 권한과, GitHub Pages 공개/비공개 정책은 확인이 필요하다.
- S3 버킷 이름, OAuth Client 설정, SMTP 자격 증명, MongoDB URI, JWT 키는 환경 변수로 주입되는 운영 비밀값이다.
- 프론트는 Web 실행도 지원하지만, Push Token과 SecureStore의 기본 흐름은 iOS/Android 네이티브 환경을 중심으로 구현돼 있다.

## 10. 근거 소스

- 백엔드 시작점/스케줄: `src/main/java/com/addiction/AddictionApplication.java`
- 인증: `src/main/java/com/addiction/global/security/config/SecurityConfig.java`, `src/main/java/com/addiction/jwt/filter/JwtAuthenticationFilter.java`
- 흡연 원본 및 이관: `src/main/java/com/addiction/user/userCigarette/service/impl/UserCigaretteServiceImpl.java`, `src/main/java/com/addiction/batch/userCigaretteHistory/userCigaretteHistoryBatch.java`
- Mongo 시계열 설정: `src/main/java/com/addiction/global/config/MongoConfig.java`
- Push Outbox: `src/main/java/com/addiction/batch/dailySmokingPush/DailySmokingPushBatch.java`, `src/main/java/com/addiction/pushOutbox/service/PushOutboxDispatcher.java`
- 외부 연동: `src/main/java/com/addiction/storage/service/S3StorageService.java`, `src/main/java/com/addiction/expo/ExpoNotiService.java`
- 프론트 API/세션: `src/services/api/instanceApi.ts`, `src/services/auth/authStorage.ts`, `src/services/auth/authApi.ts`
- 프론트 푸시/상태: `src/lib/pushToken.ts`, `src/app/_layout.tsx`
- Prometheus exporter: `build.gradle`, `src/main/resources/application-prd.yml`, `src/main/java/com/addiction/global/security/config/SecurityConfig.java`
- Blue/Green 포트 전환: `scripts/prd/run_new_was.sh`, `scripts/prd/switch.sh`
- 관리자 서버: `quitmate-admin-server/src/main/resources/application-prd.yml`, `quitmate-admin-server/src/main/java/com/quitmate/global/security/config/SecurityConfig.java`, `quitmate-admin-server/src/main/java/com/quitmate/user/users/service/impl/LoginServiceImpl.java`
