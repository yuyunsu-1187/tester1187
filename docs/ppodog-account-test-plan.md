# 뽀독(ppodog) 계정별 테스트 계획서

> 작성일 2026-09-30 · 작성 QA 리드 · 대상 라운드: v2.6 QA(2026-09-16~17, yong) 다음 라운드
> 역할 명칭은 `account-groups-and-permissions.md`(참조 R1) 정본 표기(슈퍼마스터·마스터·지점장·미용사)와 고객을 사용한다.
> 이 문서의 기대결과는 **정본 문서 + main 코드**에서 직접 확인한 규칙을 근거로 한다. 코드에서 확인하지 못한 항목은 `(확인 필요)`로 표시한다.

---

## 1. 문서 정보

### 1.1 목적

- 5개 계정 그룹(슈퍼마스터·마스터·지점장·미용사·고객)별로 **보이는 화면, 할 수 있는 행위, 볼 수 있는 데이터 범위**를 검증한다.
- v2.6 QA에서 슈퍼마스터 계정으로 막혔던 모바일(`/app`, SCR-*)·고객 앱(APP-*) 영역을 계정별로 채운다.
- v2.6 발견 이슈를 재판정하고, 이번 작성 중 발견한 **정본-코드 불일치**를 권한 경계 TC로 고정한다(§1.6).

### 1.2 기준 버전

| 구분 | 기준 |
|---|---|
| 화면설계서 | v2.9 (Phase 27 반영, SCR-RES-018/019 신설·SCR-RES-005~008 삭제) |
| IA | v3.1 |
| 기능요건정의서 | v2.4 |
| 역할·권한 정본 | `docs/account-groups-and-permissions.md` (2026-09-03) |
| 화면 ID 정본 | `docs/screen-id-map*.md` — SCR-* 58 · WEB-* 82 · SUP-* 23 · APP-* 21 (합계 184) |
| 코드 | `ppodog` main 최신 (HEAD `7dcb721`, 마이그레이션 head 0067) |
| 자동 회귀 기준선 | pytest 1106 / jest 280 / vitest 92 / Playwright e2e 330 (STATE.md Operator Next Steps) |

### 1.3 대상 환경

| 환경 | 구성 | 계정 | 이번 라운드 용도 |
|---|---|---|---|
| **개발-E2E (주 환경)** | Mac Mini Docker, `mysql-test/ppodog_test` + `backend-e2e`(8002), 프런트 `VITE_API_BASE_URL=http://localhost:8002 ... npm run dev -- --port 5175` | 시드 8계정(비번 `123456`) + 신규 가입 고객 | 계정별 단독·교차·E2E 전 TC. 파괴적 TC(정지·등급 변경·점검 모드) 허용 |
| 개발-DEV | `mysql-dev/ppodog`, Tailscale `https://macmini.tail4a2b1d.ts.net` | 시드 8계정 | 폰 Chrome(360px) 실기기 모바일 웹 부가 확인. 포트 매핑은 `(확인 필요)` |
| 스테이징 | 서버 8080, `ppodog_staging` | 고객 테스트 계정 0929/0931/0932 | 회귀 스모크(REG-*), 0067 반영 확인. 파괴적 TC 금지 |
| 운영 | `ppodog.com` | 슈퍼마스터(실계정), 고객 `010-0000-0929`(UAT Tester) | 읽기 전용 스모크 + 고객 앱 실기기(EAS 69dcdb05). **운영에는 테스트 샵·샵 소속 계정이 없음** — 쓰기 TC는 대장님이 SUP-BIZ-006으로 테스트 샵 생성을 승인한 경우에만 |

- 모바일 실기기: SM-S901N(360 CSS px). 폰 Chrome으로 로컬 웹 확인 시 `adb reverse tcp:5175 tcp:5175`·`tcp:8002 tcp:8002`.
- 고객 앱: EAS internal 빌드는 `EXPO_PUBLIC_API_BASE_URL=https://ppodog.com`(운영) 고정. 개발 DB 대상 고객 TC는 `EXPO_PUBLIC_API_BASE_URL`을 개발 백엔드로 지정한 로컬 빌드/Expo 실행 또는 API 직접 호출로 수행한다.

### 1.4 범위 내

- PC 웹 `/admin`: WEB-*(82), SUP-*(23) — 역할별 접근·행위·데이터 범위.
- 모바일 웹 `/app`: SCR-*(58) — 마스터·지점장 겸직, 미용사 주 사용. Phase 27 신규(SCR-RES-018/019, 드래그·빈 시간 즉시예약, 색상 토글, 상태 4분류) 포함.
- 고객 앱(Expo, APP-* 21): 가입·로그인·매장 연결·반려동물·예약 신청·이력·알림장·탈퇴.
- 백엔드 인가 경계: API 직접 호출로 403/404/409/423/503 확인.
- 구독 등급 게이트(BASIC/PRO/PREMIUM)와 지점장 매출 grant(`staff_sales_grants`).

### 1.5 범위 외 (상세는 §8)

- 실 SMS 수신(SOLAPI 미계약), 실 푸시 수신(Firebase/FCM 미설정), PG 실결제(토스 미계약), 카카오 알림톡 실채널, Play Console 배포(Phase 16), 알림장 사진 기기 갤러리 저장(SCR-CX-007 스코프 제외), 네이버 예약 연동.
- 성능·부하, 브라우저 호환성 매트릭스(Chrome 최신 + 폰 Chrome만 수행).

### 1.6 작성 중 확인한 정본-코드 불일치 (이번 라운드 P0 확인 대상)

| # | 항목 | 정본(R1) | 코드 실제 | 관련 TC |
|---|---|---|---|---|
| D1 | 지점장이 지점장 초대/등록 | ✗ | `POST /staff/invites`·`POST /staff`·`PATCH /staff/{id}`(role)는 `require_role(SHOP_MANAGER, SHOP_MASTER)` + PREMIUM 게이트만 검사 → **지점장도 지점장 초대 가능**. 프런트 `StaffInviteContent`도 역할과 무관하게 지점장 옵션 노출 | TC-MG-008, X-11 |
| D2 | 지점장·미용사의 샵 신규 등록 | ✗ | `POST /shops`는 역할 검사 없이 `ADDITIONAL_SHOP`(PREMIUM)만 검사. `test_create_shop_reuses_account_and_returns_enterable_membership`가 **미용사로 201**을 기대. `/app/shops/new`도 역할 가드 없음 | TC-MG-011, TC-GR-026, X-12 |
| D3 | 미용사 사고 기록 등록 | ✗(추후 허용 예정) | `POST /incidents`는 역할 검사 없음(PRO 이상 게이트만). `test_pro_groomer_can_create_and_list_incidents`가 허용을 기대. SCR-LAW-002 라우트 무가드 | TC-GR-022, X-13 |
| D4 | 지점장 사고 기록 열람·등록(PC) | ✓ | PC `WEB-SYS-005~007` 라우트가 `roles={['SHOP_MASTER']}` → 지점장은 `/app/home`으로 리다이렉트. 모바일 SCR-LAW-002/003은 허용 | TC-MG-016 |
| D5 | 미용사 고객 정보 "개인정보 마스킹 열람" | 마스킹 열람 | 범위 밖 고객은 **목록에서 제외**(사고 기록에서만 이름 마스킹). 범위 안 고객(본인 배정 BOOKED/DONE 예약 고객 + 첫 예약 전 본인 생성 고객)은 **평문 연락처** 응답 | TC-GR-015, TC-GR-016 |
| D6 | 고객 계정 연결 승인 주체 | (정본 미기재) | `link-requests` 전 API가 `require_role(SHOP_MASTER, SHOP_MANAGER)` — **미용사 승인 불가**. 반면 예약 신청 확정·거절은 미용사도 가능(본인 배정 범위) | TC-GR-018, TC-CU-009 |
| D7 | SUP-BIZ-005 계정 정지 효과 | "소속 직원 로그인 즉시 차단"(screen-id-map-admin) | `Account.is_suspended`는 **정지된 계정 본인**의 login/refresh/select-shop만 차단. 같은 업체의 다른 소속 계정(지점장·미용사) 차단 로직 없음. 이미 발급된 access 토큰은 `_resolve_current_principal`이 정지 여부를 보지 않아 만료 전까지 유효 가능 | TC-SM-019, X-06 |
| D8 | 테스트 계정 문서 | 샵A·샵B, 서비스 4종, grant 0건 | 시드는 **샵A-2**(shop01 두 번째 샵)·**등급**(shop01=PREMIUM, paid_until 2026-12-31 / shop02=BASIC)·**shop02 BASIC→PRO PENDING 변경 신청**·샵당 고객 8·예약 14·결제 7(부분취소 2)·마일리지·가능시간·사고 2건까지 생성. `test-accounts.md`에 미기재 | §2 |
| D9 | 샵B 직원 한도 | — | BASIC 직원 한도 2 < 샵B 비-owner 소속 3명(manager02·groomer01·groomer02). 소속 계정 로그인 시 lazy reconcile로 **GRACE(1개월) 시작**, 만료 후 구독 job 실행 시 최근 합류 계정부터 비활성화. 또 BASIC은 `SHOP_MANAGER_ROLE`(PREMIUM 전용)이 없어 manager02 존재 자체가 등급 정책 밖(시드 우회) | §2.4, TC-MA-030 |
| D10 | 휴대폰 인증 문서 | plan 문서: mock 고정 코드 `000000` | 현행: 난수 6자리, `PHONE_VERIFICATION_DEBUG_MODE=true`면 응답 `debug_code`로 노출(프런트가 자동 입력) | TC-MA-001, TC-CU-001 |
| D11 | 고객 앱 logout-all·매장 탈퇴·예약금 | — | `POST /customer/auth/logout-all`, `POST /customer/shops/{id}/leave`는 API만 있고 앱 UI 호출처 없음. 예약금은 샵측(예약 생성 `deposit_amount`)에만 있고 고객 앱 표시 없음 | TC-CU-021, TC-CU-024 |
| D12 | 시드 고객 연락처 해시 형식 | — | 시드는 `hash_pii("010-9101-0003")`(하이픈 포함)로 저장하나, 고객 가입·직원 고객 등록 API는 숫자만(`01091010003`)으로 정규화 후 해시(`hash_pii`는 공백만 제거). → **시드 고객 번호로 고객 앱 가입해도 휴대폰 매칭 0건 → `UNLINKED`**. 매칭 승인 흐름은 직원 화면에서 새로 등록한 고객으로만 재현 가능 | §2.3, TC-CU-006·027 |

---

## 2. 테스트 계정 · 픽스처

### 2.1 시드 계정 (개발·테스트 DB 전용, 비밀번호 전부 `123456`)

| 그룹 | 이메일 | 소속 샵 (역할) | 업체 등급 | 이번 라운드 용도 |
|---|---|---|---|---|
| 슈퍼마스터 | `yuyunsu87@gmail.com` | 없음(플랫폼 전역) | — | SUP-* 전 화면, 대행, 점검 모드. 대장님 실주소 — 개발 DB 한정 |
| 슈퍼마스터 | `dark1514@gmail.com` | 없음 | — | SUP-SYS-002 상호 권한회수·잠금해제 대상, v2.6 재현 계정 |
| 마스터 | `shop01@gmail.com` | 샵A·샵A-2 (SHOP_MASTER, owner) | **PREMIUM** (paid_until 2026-12-31) | 전 기능·샵 추가·지점장 초대·멀티샵 매출 합산 |
| 마스터 | `shop02@gmail.com` | 샵B (SHOP_MASTER, owner) | **BASIC** | 등급 잠금 UX, 구독 변경 신청(PENDING 1건 보유), v2.6 비교 |
| 지점장 | `manager01@gmail.com` | 샵A (SHOP_MANAGER) | PREMIUM | 매출 grant 항목별 토글, D1·D2 확인 |
| 지점장 | `manager02@gmail.com` | 샵B (SHOP_MANAGER) | BASIC | grant 무효(PRO 미만), v2.6 WEB-MGR-001~003 재판정 |
| 미용사 | `groomer01@gmail.com` | 샵A + 샵B (SHOP_GROOMER) | PREMIUM / BASIC | 샵 선택 화면, 샵별 등급 차이, 데이터 스코프 |
| 미용사 | `groomer02@gmail.com` | 샵B (SHOP_GROOMER) | BASIC | 동료 데이터 격리 대조군 |

### 2.2 시드 픽스처 (`backend/app/db/seed_test_accounts.py` 끝까지 확인)

| 샵 | 소유 | 소속 | 샘플 데이터 |
|---|---|---|---|
| 샵A | shop01 | manager01, groomer01 | 고객 8명(연락처 `010-9101-0001~0008`)·반려동물 12마리, 예약 14건(DONE 7·BOOKED 5·CANCELED 2, 시드 실행일 기준 −28~+13일), 결제 7건(CARD/CASH/EASY 순환, 2건 PARTIALLY_CANCELED), 마일리지 적립 5%·사용, manager01·groomer01 평일 10–13·14–19 가능시간 + manager01 +7일 특정일, 사고 2건(작성 manager01) |
| 샵B | shop02 | manager02, groomer01, groomer02 | 샵A와 동일 구조(연락처 `010-9102-0001~0008`), 미배정 예약 포함 |
| 샵A-2 | shop01 | shop01만 | 고객 "멀티샵 대조 고객"(`010-9201-0001`)·반려동물 "대조견", DONE 예약 3건·결제 31,000(CARD)/42,000(CASH)/53,000(EASY), order_id `W4B6-A2-01~03` — 멀티샵 매출 합산 대조용 |
| 공통 | — | — | 샵별 기본 서비스·알림 템플릿(`ensure_default_shop_content`), shop02 `BASIC→PRO` 구독 변경 신청 PENDING 1건 |

- 고객 생성자(`created_by_staff_id`) 분포: 샵A 1·2번 고객=groomer01, 3·6번=manager01, 4·7번=shop01, 5·8번=없음. 샵B 1·2번=groomer01, 3번=groomer02 … (미용사 데이터 스코프 TC에 사용).
- 예약 담당 키: `primary`=groomer01, `secondary`=샵A manager01 / 샵B groomer02, `manager`, `master`, 미배정(None).
- **주의**: 예약 날짜는 시드 실행 시점 기준 상대일이고 재실행 시 "기존 샘플 skip"이다. 시드가 오래됐으면 BOOKED 예약이 과거가 된다 → 사전 준비에서 예약 날짜를 확인하고, 필요하면 TC 안에서 새 예약을 만든다.
- 기본 서비스 수: 문서 "4종" vs v2.6 관측 "5개 서비스" `(확인 필요)` — 사전 준비에서 실제 개수 기록.

### 2.3 고객 앱 테스트 계정 (시드 없음 → 신규 가입으로 생성)

| 계정 | 생성 방법 | 휴대폰 | 용도 |
|---|---|---|---|
| CUST-A (매칭) | ① manager01이 WEB-CUS-002/SCR-CUS-002로 샵A 고객 "QA고객A"(`010-7777-0001`)·반려동물 "QA콩" 선등록 ② 같은 번호로 APP-ONB-002 가입 | `010-7777-0001` | 가입코드로 샵A 가입 → 휴대폰 1건 매칭 → `PENDING_APPROVAL` → 승인 E2E. 시드 고객 번호는 D12로 매칭되지 않음 |
| CUST-B (비매칭) | APP-ONB-002 신규 가입 | 샵에 등록되지 않은 번호(예: `010-9999-0001`) | `UNLINKED`(앱 표시 NOT_LINKED) → 수동 연결 경로 |
| CUST-S (소셜) | 카카오/네이버 → APP-ONB-003 | 시드에 없는 번호 | 소셜 가입·연동 관리(APP-PRF-004) |
| 운영 UAT | 기존 `010-0000-0929`(UAT Tester) | 미개통 대역 | 운영 스모크 전용. 테스트 종료 후 탈퇴로 정리(STATE 지시) |

절차: 휴대폰 인증 `POST /api/v1/auth/phone-verification/send` → DEBUG 모드면 응답 `debug_code`를 앱이 자동 입력 → 이름·비밀번호(8자+)·생년월일·필수 약관 2종 동의 → 가입. 가입코드는 샵A 마스터가 `WEB-09`(`/admin/customers/join-code`)에서 발급.

### 2.4 사전 준비 체크리스트

| # | 항목 | 명령·확인 방법 | 완료 |
|---|---|---|---|
| P-01 | 시드 실행(가드 통과 확인) | `docker compose exec -T -e PPODOG_ENV=development -e ALLOW_TEST_ACCOUNT_SEED=true -e DB_HOST=mysql-dev -e DB_NAME=ppodog backend python -m app.db.seed_test_accounts` (mysql-test는 `DB_HOST=mysql-test -e DB_NAME=ppodog_test`, 대상 서비스명 `(확인 필요)`). 허용 조합 외 DB·플래그 누락 시 `RuntimeError`로 차단되는지 1회 확인 | ☐ |
| P-02 | 약관 버전 시드 | 재초기화한 DB면 `python -m app.db.seed` 선행(약관 3종 없으면 가입 503) | ☐ |
| P-03 | 구독 등급 확인 | `account_subscriptions`: shop01=PREMIUM(paid_until 2026-12-31), shop02=BASIC. 다르면 SUP-SUB-001로 맞춤 | ☐ |
| P-04 | 변경 신청 확인 | SUP-SUB-002 대기 탭에 shop02 BASIC→PRO 1건 | ☐ |
| P-05 | 지점장 매출 grant 0건 | `SELECT * FROM staff_sales_grants;` 0행 또는 WEB-STF-004에서 두 지점장 모두 미체크 | ☐ |
| P-06 | 샵B 직원 한도 상태 | `subscription_staff_quota_states`에서 shop02 행 확인. GRACE 만료 전인지, 비활성화된 소속이 없는지 기록(D9) | ☐ |
| P-07 | 샵 정지·계정 정지 해제 | `shops.is_suspended=0`, `accounts.is_suspended=0` 전 계정 | ☐ |
| P-08 | 점검 모드 OFF | SUP-SYS-001 "정상 운영 중" | ☐ |
| P-09 | 예약 날짜 신선도 | 샵A·샵B BOOKED 예약이 오늘 이후인지. 아니면 TC 내 신규 예약 생성으로 대체 | ☐ |
| P-10 | 소셜 로그인 모드 | 개발: `SOCIAL_AUTH_ENABLED=true`, `SOCIAL_AUTH_MOCK=true`(FakeProvider) / 실기기: 실키 | ☐ |
| P-11 | 휴대폰 인증 모드 | 개발·운영 모두 `PHONE_VERIFICATION_DEBUG_MODE=true`(운영은 active_risk) 확인 | ☐ |
| P-12 | Playwright 로그인 상태 | `npx playwright test --project=setup`으로 `e2e/.auth/*.json` 갱신 | ☐ |
| P-13 | 자동 회귀 1회 | 백엔드 pytest 전체 + e2e 전체 결과를 기준선으로 첨부(공유 mysql-test 오염 주의: 초기화 후 pytest) | ☐ |
| P-14 | 계정 잠금 해제 | 5회 실패 TC 후 15분 대기 또는 SUP-SYS-002 잠금해제(슈퍼마스터 계정만). 샵 계정 잠금은 DB `accounts.locked_until` 초기화 | ☐ |

---

## 3. 테스트 전략

### 3.1 진행 단계

| 단계 | 내용 | 산출 |
|---|---|---|
| S1 계정별 단독 | 각 계정으로 로그인해 메뉴·화면·행위 확인. 화면 ID 기준 체크 | §4 TC 결과 |
| S2 계정 간 교차 | 같은 기능을 5그룹으로 반복(허용/거부), 타 샵 ID 직접 접근, API 직접 호출, 정지·만료·비활성 상태 | §5 매트릭스 |
| S3 플로우 E2E | 고객→미용사→지점장→마스터→슈퍼마스터로 이어받는 릴레이 | §6 시나리오 |
| S4 회귀 | v2.6 이슈 재판정 + 09-29~30 UAT 수정분 | §7 표 |

- UI에서 막힌 행위는 반드시 **API 직접 호출**(브라우저 DevTools 또는 `curl` + 쿠키/CSRF)로 서버 거부까지 확인한다. UI만 숨기고 서버가 허용하면 P0 결함이다.
- 등급 게이트와 역할 게이트를 구분해 기록한다: 등급 잠금은 `403 {"code":"TIER_LOCKED"}`, 한도는 `409 {"code":"TIER_LIMIT"}`, 역할 거부는 `403 "권한이 없습니다"` 계열.

### 3.2 우선순위

| 등급 | 정의 | 예 |
|---|---|---|
| P0 | 권한 경계·테넌트 격리·개인정보·결제 금액·계정 보안. 실패 시 릴리스 차단 | 타 샵 데이터 노출, 역할 우회, 정지 우회 |
| P1 | 계정의 주 업무 플로우. 실패 시 해당 역할 업무 불가 | 예약 등록·확정·완료, 가입, 초대 |
| P2 | 표시·문구·편의 | 배지, 안내 문구, 정렬 |

### 3.3 상태 표기 (yong 보고서 표기 계승)

| 표기 | 의미 |
|---|---|
| ✅ Pass | 기대결과 일치 |
| ❌ Fail | 불일치 → 이슈 ID 필수 |
| ⚠️ 주의 | 기능은 되나 정본·설계와 차이(판정 보류) → 이슈 ID(Info) |
| ⛔ Blocked | 선행 조건·외부 의존으로 수행 불가(사유 기재) |
| — 미수행 | 이번 라운드 미실행 |
| N/A | 해당 계정에 적용 안 됨 |

### 3.4 자동화 열 표기

- `pytest: 파일명` — `backend/tests/` 기존 자동 테스트가 같은 규칙을 검증함. 수동은 **UI 확인만** 한다.
- `e2e: 화면ID` — `frontend/e2e/{admin,app}/<화면ID>.spec.ts` 스모크 존재.
- `jest` — `customer-app/__tests__/` 컴포넌트 테스트(실기기 E2E 하네스 없음).
- `수동` — 자동화 없음.

---

## 4. 계정 그룹별 테스트 케이스

공통 사전조건: §2.4 체크리스트 완료, 개발-E2E 환경, 각 TC 시작 시 해당 계정으로 새로 로그인(쿠키 초기화).

| 그룹 | TC 수 | P0 | 비고 |
|---|---:|---:|---|
| 슈퍼마스터 TC-SM | 28 | 16 | SUP-* 16개 화면 파일 + SUP-DSH-001(=WEB-PLT-002), 대행 |
| 마스터 TC-MA | 31 | 13 | 가입·샵·초대·매출·구독 |
| 지점장 TC-MG | 21 | 12 | grant·D1/D2/D4 |
| 미용사 TC-GR | 30 | 17 | 모바일·데이터 스코프·Phase 27·오프라인 초안 |
| 고객 TC-CU | 27 | 9 | APP-* 21 |
| 교차 TC-X (§5.2) | 16 | 15 | 부정·경계 |
| **합계** | **153** | **82** | + E2E 5 시나리오(§6), 회귀 16건(§7) |

### 4.1 슈퍼마스터 (TC-SM) — `yuyunsu87@gmail.com` / `dark1514@gmail.com`

근거: `core/deps.py`(`require_platform_master`, `resolve_platform_act_as_claims`), `api/v1/platform.py`, `api/v1/auth.py`(`/act-as/exit`, `/change-password`), `ProtectedRoute.tsx`, `AdminLayout.tsx`.

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-SM-001 | WEB-COM-001 | 플랫폼 로그인 | 소속 샵 없음 | `/admin/login`에서 yuyunsu87 로그인 | `principal_kind=platform` 토큰 발급, `/admin/platform/overview` 진입, 사이드바에 플랫폼 메뉴 10개(통합 대시보드~마스터 계정)만 표시 | P0 | pytest: test_platform_master.py · e2e: WEB-COM-001 |
| TC-SM-002 | SUP-DSH-001 (=WEB-PLT-002) | 통합 대시보드 | TC-SM-001 | 기간 변경 조회, 시작일>종료일 입력 | 전 샵 고객·예약·순매출·정지 샵 수·등급 분포 표시. 시작일>종료일은 422 안내 | P1 | e2e: WEB-PLT-002 · pytest: test_platform_stats.py |
| TC-SM-003 | WEB-PLT-001 | 전체 매장 목록 | — | 검색("샵A"), 페이지 이동, 등급 컬럼 확인 | 샵A·샵A-2(PREMIUM)·샵B(BASIC) 표시, 활성 직원 수에 대행 마커 미포함 | P1 | e2e: WEB-PLT-001 |
| TC-SM-004 | WEB-PLT-001 | 샵 정지/재개 | 샵B 정상 | 샵B 정지 → groomer02·shop02 로그인 시도 → groomer01 로그인 → 샵B 재개 | 정지 중 shop02·groomer02 로그인 403 "정지된 샵에는 로그인할 수 없습니다", groomer01은 샵 선택 없이 샵A로 바로 진입(가용 샵 1개). 재개 즉시 정상 | P0 | pytest: test_platform_master.py::test_suspend_blocks_login_select_shop_and_existing_staff_token |
| TC-SM-005 | WEB-COM-007 | 대행 진입·배너 | — | `/admin/select-shop`에서 샵A 선택(대행 진입) | `POST /platform/shops/{id}/act-as` 성공, `/admin` 진입, 상단 호박색 고정 배너에 "샵A"와 나가기 버튼, `/auth/me`에 `acting_shop` 포함 | P0 | pytest: test_platform_act_as.py · e2e: WEB-COM-007 |
| TC-SM-006 | WEB-COM-007 | 대행 나가기 | TC-SM-005 | 배너의 나가기 클릭 | `POST /auth/act-as/exit` → platform 토큰 재발급, `/admin/platform/*` 복귀. 일반 staff 토큰으로 exit 호출 시 403 | P1 | pytest: test_platform_act_as.py |
| TC-SM-007 | WEB-COM-007 | 정지 샵 대행 차단 | 샵B 정지 | 샵B 대행 진입 시도 | 409 "정지된 샵은 먼저 재개해야 합니다". 대행 중 샵이 정지되면 다음 요청 403 | P0 | pytest: test_platform_act_as.py |
| TC-SM-008 | ShopCreate(`/admin/shops/new`) | 대행 금지 — 샵 추가 | 샵A 대행 중 | 지점 화면에서 샵 추가 시도 + `POST /api/v1/shops` 직접 호출 | 403 "슈퍼마스터 대행 중에는 샵을 추가할 수 없습니다" | P0 | pytest: test_platform_act_as.py |
| TC-SM-009 | WEB-COM-002 | 대행 금지 — 비밀번호 변경 | 샵A 대행 중 | `/admin/password` 진입 및 `POST /auth/change-password` | 대행 중에는 최초 비번 변경 강제 리다이렉트 없음, API 403 "슈퍼마스터 대행 중에는 비밀번호를 변경할 수 없습니다" | P0 | 수동(API) |
| TC-SM-010 | SCR-STF-004 | 대행 금지 — 본인 가능시간 | 샵A 대행 중 | `GET/PUT /api/v1/staff/me/availability` | 둘 다 403 "슈퍼마스터 대행 계정은 개인 가능시간을 사용할 수 없습니다" | P0 | 수동(API) |
| TC-SM-011 | WEB-RES-005 / 직원 동의 | 대행 금지 — 마커 배정·직원 동의 | 샵A 대행 중 | 예약 등록에서 담당 미용사 목록 확인, API로 `groomer_id=<마커 staff_id>` 예약 생성, `POST /consents` | 미용사 선택지에 "슈퍼마스터 대행" 없음, API는 "슈퍼마스터 대행 계정은 담당 미용사로 배정할 수 없습니다", 동의 기록 거부 | P0 | pytest: test_platform_act_as.py |
| TC-SM-012 | WEB-SUB-001 | 대행 중 구독 관리 불가 | 샵A 대행 중 | 사이드바 확인, `/admin/subscription` 접근, `GET /api/v1/subscription` | "구독 관리" 메뉴 미노출, API 403 `ACT_AS_FORBIDDEN` | P0 | pytest: test_platform_act_as.py::test_act_as_uses_target_subscription_and_cannot_manage_it |
| TC-SM-013 | WEB-DSH-003 · WEB-SAL-001~006 | 대행은 대상 업체 등급 적용 | — | ① 샵A(PREMIUM) 대행 → 매출 메뉴·대시보드 매출 탭 ② 샵B(BASIC) 대행 → 동일 확인 | ① 매출 6화면·WEB-DSH-003 표시 ② 매출 메뉴 없음, `/admin/sales` → `/admin` 리다이렉트, WEB-DSH-003 매출 미표시. v2.6 ISSUE-003 재판정 근거(REG-03) | P0 | pytest: test_sales_grants.py · 수동 |
| TC-SM-014 | (UI 없음) | 대행 감사 로그 | TC-SM-005~013 수행 후 | `SELECT account_id, shop_id, method, path, status_code FROM platform_action_logs ORDER BY id DESC LIMIT 30;` | 대행 중 모든 요청이 실제 슈퍼마스터 account_id·대상 shop_id·상태코드와 함께 기록. 거부(403/409) 요청도 기록 | P0 | pytest: test_platform_act_as.py::test_enter_refresh_me_exit_and_tenant_audit |
| TC-SM-015 | WEB-STF-001 · WEB-SAL-005 · WEB-COM-007 | 대행 마커 비노출 | 샵A 대행 1회 이상 | 샵A 직원 목록, 미용사별 실적, shop01 로그인 샵 후보, 직원 수 집계 확인 | "슈퍼마스터 대행" 마커가 어디에도 표시·집계되지 않음 | P1 | pytest: test_platform_act_as.py::test_marker_is_hidden_from_staff_counts_reports_and_historical_names (WINDOWS #19 기존 불안정) |
| TC-SM-016 | SUP-TIR-001 / SUP-TIR-002 | 등급 정의 편집 | 개발 DB 한정 | PRO 직원 한도 5→6, 미용 이력 일수 변경 저장 → shop02 WEB-SUB-001 확인 → 원복 | 비교표·WEB-SUB-001 한도 즉시 반영, 기능 목록은 코드 상수라 편집 대상 아님 | P1 | pytest: test_platform_tiers.py · e2e: SUP-TIR-001/002 |
| TC-SM-017 | SUP-BIZ-001 / SUP-BIZ-002 | 업체 목록·상세 | — | shop01 업체 상세 진입 | 샵 2개(샵A·샵A-2), 직원 수, 구독(PREMIUM·paid_until) 표시. shop02 상세에 대기 중 변경 신청 표시 | P1 | e2e: SUP-BIZ-001/002 · pytest: test_platform_businesses.py |
| TC-SM-018 | SUP-BIZ-006 | 업체 등록 | — | 신규 이메일·샵 정보·등급 PRO로 등록 | Account+Shop+Settings+SHOP_MASTER+구독 원자 생성, 임시 비밀번호 1회 표시. 신규 계정 로그인 시 `must_change_password` → WEB-COM-002 강제. 중복 이메일 409 | P1 | pytest: test_platform_business_register.py · e2e: SUP-BIZ-006-register |
| TC-SM-019 | SUP-BIZ-005 | 계정 정지·해제 | shop02 로그인 세션 유지 | 사유 입력 후 shop02 정지 → shop02 새 로그인·기존 세션 새로고침·샵 선택 → manager02·groomer02 로그인 → 해제 | shop02 login/refresh/select-shop 403 `ACCOUNT_SUSPENDED`(사유 표시). **manager02·groomer02 차단 여부와 shop02 기존 access 토큰 만료 전 API 허용 여부를 기록(D7, 확인 필요)**. 해제 즉시 재로그인 가능 | P0 | pytest: test_platform_business_suspend.py |
| TC-SM-020 | SUP-SUB-001 | 구독 직접 변경(오버라이드) | shop02 BASIC, PENDING 1건 | shop02를 PRO로 직접 변경 | 대기 신청이 무효(superseded) 처리, shop02 세션에서 매출·결제취소·사고기록·마일리지 조정 게이트 즉시 해제(재조회 기준), 직원 한도 5로 GRACE 해제 공지 | P0 | pytest: test_platform_subscription_change_requests.py::test_direct_patch_supersedes_active_but_same_tier_keeps_it |
| TC-SM-021 | SUP-SUB-002 | 변경 신청 승인/반려 | PENDING 1건 | 대기 탭 확인 → 승인(또는 반려 후 shop02 재신청 → 승인) | FIFO 목록, 승인 시 등급·`/auth/me` 즉시 반영 + 대상 업체 공지, 이중 처리 409. 샵 소속 계정 토큰 호출 403 | P1 | pytest: test_platform_subscription_change_requests.py · e2e: SUP-SUB-002 |
| TC-SM-022 | SUP-SUB-003 / SUP-SUB-005 | 변경 이력·만료 예정 | — | 이력 최신순, 만료 예정 목록 | 이력은 `SubscriptionChangeRequest` 최신순. shop01(paid_until 2026-12-31)이 D-N 배지로 표시(기준일 필터에 따라) | P2 | e2e: SUP-SUB-003/005 · pytest: test_platform_subscription_expiring.py |
| TC-SM-023 | SUP-STA-001 / SUP-STA-002 | 통계·업체 활성도 | — | 7/30/90일 전환 | 등급 분포·12개월 가입·등급별 순매출 비중, 활성도는 로그인·예약 빈도 기반 활성/비활성 배지 | P2 | e2e: SUP-STA-001/002 · pytest: test_platform_business_activity_stats.py |
| TC-SM-024 | SUP-NTF-001 / SUP-NTF-002 | 공지 등록·수정·삭제 | — | 대상·게시 기간 지정 공지 등록 → shop01 로그인 → 삭제 | 게시 기간 내 대상 계정 `/admin` 상단 공지 표시·닫기 가능, 삭제 후 미표시. 대상 범위 규칙 `(확인 필요)` | P2 | e2e: SUP-NTF-001/002 · pytest: test_platform_notices.py |
| TC-SM-025 | SUP-SYS-001 | 서비스 점검 모드 | 개발 DB 한정 | 안내 문구·예상 완료 시각 입력 후 ON → shop01·groomer01·고객 앱 요청 → 슈퍼마스터 요청 → OFF | 일반 요청 503 `SERVICE_MAINTENANCE`(문구·예상 시각·Retry-After), `/auth/login`·`/auth/refresh`·`/system/maintenance`는 통과, 슈퍼마스터 토큰은 통과. OFF 즉시 복구, 변경자 기록 | P0 | pytest: test_service_maintenance.py · e2e: SUP-SYS-001 |
| TC-SM-026 | SUP-SYS-002 | 슈퍼마스터 계정 관리 | 두 슈퍼마스터 존재 | 목록 → 신규 추가(임시비번) → dark1514 잠금해제·비번재발급 → 본인 회수 시도 → 마지막 1명 회수 시도 | 추가·재발급은 임시비번 표시, 본인 회수 409 `CANNOT_REVOKE_SELF`, 마지막 1명 409 `LAST_MASTER`, 전 행위 감사 기록. 테스트 후 추가 계정 회수 | P0 | pytest: test_platform_master_accounts.py · e2e: SUP-SYS-002 |
| TC-SM-027 | SCR-HOM-001 (/app) | `/app` 접근 리다이렉트 | — | ① 대행 없이 `/app/home` ② 샵A 대행 중 `/app/home` | ① `/admin/platform/overview` ② `/admin`으로 리다이렉트. 설계 동작(`ProtectedRoute`) — v2.6 Info 이슈 재판정(REG-04) | P1 | 수동 |
| TC-SM-028 | (API) | 순수 platform 토큰의 tenant API 차단 | 대행 없음 | `GET /api/v1/customers`, `GET /api/v1/sales/daily` | 403 "샵 컨텍스트가 필요한 기능입니다" — 플랫폼 전용 API와 테넌트 API 분리 확인 | P0 | pytest: test_platform_master.py |

### 4.2 마스터 (TC-MA) — `shop01@gmail.com`(PREMIUM) / `shop02@gmail.com`(BASIC)

근거: `api/v1/auth.py`(`/signup`, `/login`, `/select-shop`), `social_auth.py`, `shops.py`, `staff.py`, `staff_invites.py`, `sales.py`, `subscription.py`, `payments.py`, `services/subscriptions.py`(`TIER_FEATURES`, `DEFAULT_TIER_LIMITS`), `admin/routes.tsx`.

등급 기능 요약(코드 상수): BASIC=없음·직원 2·알림톡 100/월·미용 이력 90일 / PRO=간편결제·매출·특이사항 자동팝업·사진 ZIP·마일리지 수동조정·적립률 편집·자동배정·완료사진 첨부·결제 취소환불·사고기록, 직원 5·알림톡 500 / PREMIUM=PRO 전부 + 추가 샵 + 지점장 역할, 무제한.

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-MA-001 | WEB-COM-003 (+WEB-COM-006 통합) | 이메일 가입 + 최초 샵 등록 | 미가입 이메일 | 이메일·비밀번호(8자+)·이름·샵명·우편번호·주소·상세주소·휴대폰 입력 → 인증번호 발송 → 약관 3종(필수 2) 동의 → 가입 | DEBUG 모드면 인증번호 자동 입력. 가입 1회로 Account+샵+SHOP_MASTER+BASIC 구독+동의 이력 생성, 즉시 로그인 → `/admin/select-shop`. 샵 정보 없이 가입 불가(필수 필드) | P1 | pytest: test_auth_signup.py · e2e: WEB-COM-003 |
| TC-MA-002 | WEB-COM-003 | 가입 검증 | TC-MA-001 계정 존재 | 같은 이메일 재가입, 인증번호 미입력/오입력, 필수 약관 미동의 | 409 "이미 가입된 이메일입니다", 400 "휴대폰 인증을 완료해 주세요", 필수 미동의는 계정·샵 미생성(orphan 없음). 인증 오입력 5회 시 해당 코드 잠김 | P1 | pytest: test_auth_signup.py, test_phone_verification.py |
| TC-MA-003 | SCR-COM-010 → SCR-COM-006 | 모바일 가입(사업자) | 미가입 이메일 | `/app` 온보딩 → 가입유형 "사업자" → 가입 | TC-MA-001과 동일 결과, 가입 후 `/app` 홈 또는 샵 선택 | P1 | e2e: SCR-COM-010, SCR-COM-006 |
| TC-MA-004 | SCR-COM-007 / SCR-COM-008 | 카카오·네이버 가입 | 개발: mock / 실기기: 실키 | 소셜 버튼 → 콜백 → 온보딩(이름·샵명·약관) → 완료 → 로그아웃 후 소셜 재로그인 | 신규: 샵 포함 원자 생성·로그인. 재로그인: 샵 1개면 자동 진입, 2개 이상이면 샵 선택. 필수 약관 미동의 시 orphan 없음·토큰 미소비. state 재사용·만료 거부 | P1 | pytest: test_auth_social.py · e2e: SCR-COM-007/008 |
| TC-MA-005 | WEB-COM-001 → WEB-COM-007 | 다중 샵 로그인 | shop01 | 로그인 | 샵 선택 화면에 **본인 소속 샵만**(샵A, 샵A-2) 표시, 선택 시 해당 샵 컨텍스트 진입 | P0 | pytest: test_shop_switch_options.py · e2e: WEB-COM-007 |
| TC-MA-006 | WEB-COM-008 / WEB-STF-005 | 지점 목록·전환·관리 | shop01 샵A 진입 | `/admin/shops`에서 샵A-2로 전환 → `/admin/staff/branches`에서 영업정보 수정 | 계정 범위 지점만 표시, 전환 후 모든 화면이 샵A-2 데이터(고객 "멀티샵 대조 고객")로 바뀜 | P1 | e2e: WEB-COM-008, WEB-STF-005 |
| TC-MA-007 | ShopCreate(`/admin/shops/new`) | 샵 추가(등급 한도) | — | ① shop01로 샵 추가 ② shop02로 메뉴 확인 + `POST /api/v1/shops` | ① 201, 신규 샵 SHOP_MASTER 멤버십 생성·진입 가능 ② 추가 버튼 비노출, API 403 `TIER_LOCKED`(ADDITIONAL_SHOP은 PREMIUM) | P0 | pytest: test_subscription_gates.py::test_additional_shop_gate_and_me_entitlements |
| TC-MA-008 | WEB-STF-002 | 초대 링크 생성 | shop01 샵A | 역할 미용사 초대 → 링크 복사, 연락처 입력해 SMS 발송 옵션 사용 | 201, 만료 7일 표시, 응답 `Cache-Control: no-store`. SOLAPI 미계약이라 SMS 실패해도 링크는 생성(발송 결과만 실패) | P1 | pytest: test_staff_invites_shops.py, test_staff_invite_sms.py · e2e: WEB-STF-002 |
| TC-MA-009 | WEB-STF-002 | 지점장 초대(등급) | — | ① shop01 지점장 초대 ② shop02 지점장 초대 | ① 201 ② 옵션 비활성 + API 403 `TIER_LOCKED`(SHOP_MANAGER_ROLE은 PREMIUM) | P0 | pytest: test_subscription_gates.py::test_staff_quota_boundaries_and_manager_role_gate |
| TC-MA-010 | WEB-STF-002 / WEB-STF-003 | 마스터 권한 부여 차단 | — | API로 `role=SHOP_MASTER` 초대·직원 생성·역할 변경 | 초대 409 "마스터 권한은 샵 생성 트랜잭션에서 소유 계정에만 부여됩니다", 생성·변경 409 | P0 | pytest: test_staff_invites_shops.py |
| TC-MA-011 | WEB-STF-002 | 직원 한도(BASIC 2) | shop02, 샵B 비-owner 소속 3명(D9) | 미용사 초대 생성 시도 | 409 `TIER_LIMIT`(한도 초과). 한도·사용량이 WEB-SUB-001과 일치 | P0 | pytest: test_subscription_gates.py |
| TC-MA-012 | WEB-STF-003 | owner 보호 | shop01 | 본인(owner) 역할을 지점장으로, 활성=false로 변경 | 409 "샵 소유 계정의 마스터 권한과 활성 상태는 변경할 수 없습니다" | P0 | pytest: test_platform_master.py::test_shop_ownership_invariants_return_409 |
| TC-MA-013 | WEB-STF-004 | 지점장 매출 grant 토글 | shop01, grant 0건 | manager01에 WEB-SAL-001·003만 체크 저장 → 해제 | 저장 즉시 반영, manager01 `/auth/me.sales_report_keys`가 해당 키만. 비활성·비지점장 대상 409 "활성 지점장에게만 매출 권한을 설정할 수 있습니다" | P0 | pytest: test_sales_grants.py · e2e: WEB-STF-004 |
| TC-MA-014 | WEB-STF-004 | grant 등급 게이트 | shop02(BASIC) | 권한 설정 화면 진입 + `GET /api/v1/staff/sales-grants` | 토글 비활성/안내, API 403 `TIER_LOCKED`(SALES_REPORT은 PRO 이상) | P0 | pytest: test_sales_grants.py |
| TC-MA-015 | WEB-SAL-001~005 | 매출 전 항목 + 멀티샵 범위 | shop01 샵A | 일/주/월/서비스별/미용사별 조회, 지점 범위 "현재 샵" ↔ "전체 소유 샵" 전환 | 전 화면 표시, 전체 범위에서 샵A-2 결제 31,000/42,000/53,000이 합산되고 행 식별 가능. 시작일>종료일 422 | P0 | pytest: test_multi_shop_sales_scope.py, test_phase5_dashboard_sales.py · e2e: WEB-SAL-001~005 |
| TC-MA-016 | WEB-SAL-006 | Excel 내보내기 | shop01 | 기간 지정, 섹션(payments/services/groomers) 선택 다운로드 | xlsx 시트별 헤더·필터, 부분취소 금액 반영 순매출. `=`,`+`,`-`,`@` 시작 텍스트는 `'` 접두 처리. 잘못된 include 422 | P1 | e2e: WEB-SAL-006 · pytest: test_sales_grants.py |
| TC-MA-017 | WEB-DSH-001~004 | 대시보드 4분할 | shop01 / shop02 | 각 탭 확인 | shop01: KPI 4개(예약·완료·오늘 매출·마일리지). shop02(BASIC): 매출 KPI·WEB-DSH-003 미표시(등급) — v2.6 WEB-MGR-003 비교 | P1 | e2e: WEB-DSH-001~004 |
| TC-MA-018 | WEB-PAY-001 | 결제 처리 | 샵A DONE·미결제 예약 | 카드/현금/간편/계좌이체 + 마일리지 차감 + 현금영수증(소득공제/지출증빙) | 결제 생성·마일리지 적립(적립률). shop02는 간편결제 비활성 + API 403 `TIER_LOCKED` | P1 | e2e: WEB-PAY-001 · pytest: test_phase4_payments_mileage.py, test_payment_transfer_method.py |
| TC-MA-019 | WEB-PAY-006~008 | 결제 취소·환불 | 샵A 결제 CONFIRMED | 부분 환불 → 잔액 초과 환불 → 취소 알림 체크 유지 | 부분취소 성공(상태 PARTIALLY_CANCELED, 마일리지 회수), 잔액 초과 422. shop02는 취소 버튼 비활성 + 403 `TIER_LOCKED` | P0 | pytest: test_payment_cancel_refund_gate.py · e2e: WEB-PAY-008 |
| TC-MA-020 | WEB-PAY-009 / WEB-SYS-004 | 예약금 | shop01 | PG 설정 화면의 예약금 섹션 설정 → 예약 등록 시 예약금 입력 → 예약금 내역 탭 확인 | 결제완료(HELD+APPLIED)/환불완료/노쇼-미환불 탭별 표시, 샵 격리 | P1 | e2e: WEB-PAY-009 · pytest: test_reservation_deposit_history.py |
| TC-MA-021 | WEB-SUB-001 | 구독 변경 요청 | shop02, PENDING 1건 | 등급 비교 → 동일 신청 재시도 → 기존 신청 취소 → 재신청 | 비교표·한도·사용량 표시, 중복 신청 차단, 취소 후 재신청 가능(가격 스냅샷). 다운그레이드는 paid_until 시점 예약 | P1 | pytest: test_subscription_change_requests.py · e2e: WEB-SUB-001 |
| TC-MA-022 | WEB-SUB-001 | 구독 관리 접근 범위 | — | shop01 샵A-2에서 접근, manager01로 사이드바·URL·API 확인 | owner 마스터만 "구독 관리" 메뉴. manager01은 `/admin/subscription` → `/app/home`, API 403 "샵 마스터만 구독을 변경할 수 있습니다" | P0 | pytest: test_subscription_change_requests.py::test_same_tier_invalid_body_and_non_owner_permissions |
| TC-MA-023 | WEB-09 · WEB-CUS-013/014 | 가입코드·고객 연결 승인(PC) | CUST-A 가입 대기 | 가입코드 발급/재발급/비활성화 → 연결 요청 상세 → 승인 | 재발급 시 이전 코드 즉시 무효, 상세에 메모·마일리지·평문 연락처 없음(마스킹), 승인 시 LINKED | P0 | pytest: test_shop_join_codes.py, test_customer_link_queue.py · e2e: WEB-09, WEB-CUS-013/014 |
| TC-MA-024 | WEB-MIL-001~004 | 마일리지 | shop01 / shop02 | 적립·사용 내역 조회(`/admin/mileage/ledger`), 수동 조정, 적립률 변경(사유) | shop01 가능·사유 필수·이력에 변경자 이름·사유. shop02 수동조정 403 `TIER_LOCKED`, 적립률 편집 잠금 | P1 | pytest: test_mileage_manual_adjust_gate.py, test_mileage_rate_reason.py · e2e: WEB-MIL-001~004 |
| TC-MA-025 | WEB-SYS-005 / WEB-SYS-006 | 사고 기록(PC) | shop01 / shop02 | 등록·조회 | shop01 가능(시드 2건 표시), shop02 등록 폼 대신 안내 + API 403 `TIER_LOCKED` | P1 | pytest: test_incident_record_gate.py · e2e: WEB-SYS-005/006 |
| TC-MA-026 | WEB-NTF-004 / WEB-NTF-005 | 알림장 발송 정책·승인 | shop01 | 정책 APPROVAL 저장 → 미용사 완료 알림장 → 승인 대기에서 승인/반려 | 정책 상호배타 저장, 대기 목록·승인 발송·반려 동작 | P1 | pytest: test_notification_approval_queue.py, test_notification_send_mode.py · e2e: WEB-NTF-005 |
| TC-MA-027 | SCR-HOM-001 · SCR-RES-019 · SCR-STF-004 | 모바일 겸직 | shop01 | `/app/login` → 샵 선택 → 예약 등록에서 담당자로 본인 선택 → 내 스케줄 가능시간 등록 | `/app` 사용 가능, 본인을 담당 미용사로 배정 가능, 가능시간 저장 | P1 | e2e: SCR-HOM-001, SCR-STF-004 |
| TC-MA-028 | WEB-COM-001 · WEB-SYS-003 | 로그인 잠금·이력 | shop02 | 잘못된 비번 5회 → 정상 비번 → 로그인 이력 화면 | 5회째 423 + `Retry-After`, 15분 잠금 중 정상 비번도 423, 이력에 실패/성공 기록·결과 필터 | P0 | pytest: test_auth_login_logs.py · e2e: WEB-SYS-003 |
| TC-MA-029 | WEB-COM-004 / WEB-COM-005 | 아이디 찾기·비밀번호 재설정 | — | 휴대폰 인증으로 아이디 찾기, 재설정 요청 | 인증 후 아이디 안내(표시 방식 `(확인 필요)`), 재설정 요청 흐름 완료(실메일 발송 여부 `(확인 필요)`) | P2 | pytest: test_auth_account_recovery.py · e2e: WEB-COM-004/005 |
| TC-MA-030 | WEB-SUB-001 · 공지 | 샵B 직원 한도 유예(D9) | shop02, P-06 기록 | shop02 로그인 → 공지·구독 화면 확인 | "직원 계정 한도 초과 유예 시작"(3/2, 만료일) 공지 표시 여부 기록. 유예 만료 후 job 실행 시 최근 합류 계정 비활성화 위험 → 결과 기록 | P1 | pytest: test_subscription_jobs.py, test_subscription_automation.py |
| TC-MA-031 | SCR-COM-005 / WEB 로그아웃 | 로그아웃 | 로그인 상태 | 로그아웃 → 뒤로가기 → 새로고침 | refresh 토큰 폐기, 보호 화면 재진입 시 로그인 화면 | P1 | pytest: test_auth_refresh_tokens.py · e2e: SCR-COM-005 |

### 4.3 지점장 (TC-MG) — `manager01@gmail.com`(샵A·PREMIUM) / `manager02@gmail.com`(샵B·BASIC)

근거: `core/deps.py`(`sales_report_keys_for`, `has_sales_report_access`, `require_sales_report`), `data_scope.py`(`MANAGER_ROLES`), `staff.py`(`_sales_grant_admin`), `staff_invites.py`, `shops.py`, `customer_links.py`, `ProtectedRoute.tsx`(`salesReportKey`), `admin/routes.tsx`.

정본: 배정 샵 안에서 마스터와 동일 권한, 매출은 grant 항목별, 미용사 초대 ✓·지점장 초대 ✗, 샵 정보 수정 ✓·샵 추가 ✗.

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-MG-001 | WEB-COM-001 | 로그인·배정 샵 진입 | — | manager01 로그인 | 샵 선택 없이 샵A 자동 진입(`/admin`), `account_group` 지점장 | P0 | e2e: WEB-COM-001 |
| TC-MG-002 | WEB-COM-008 · WEB-COM-007 | 배정 샵만 노출 | — | 지점 목록, `/admin/select-shop`, `POST /auth/shop-switch-options` | 샵A만. 같은 업체의 샵A-2·타 업체 샵B 미노출 | P0 | pytest: test_staff_invites_shops.py::test_list_shops_is_account_scoped_and_excludes_inactive_membership |
| TC-MG-003 | WEB-SAL-001~006 · WEB-DSH-001/003 | 매출 grant 0건 | P-05 | 사이드바, `/admin/sales` 직접 입력, `GET /api/v1/sales/daily`, 대시보드 | 사이드바 "매출" 없음, URL → `/admin` 리다이렉트, API 403 "이 매출 리포트 열람 권한이 없습니다", DSH-001 매출 KPI 미표시, DSH-003 "매출 조회 권한이 없어…" — **기대 동작**(REG-05~07) | P0 | pytest: test_sales_grants.py::test_report_access_by_role_and_manager_grant_set |
| TC-MG-004 | WEB-SAL-001 / 003 | grant 부분 허용 | TC-MA-013로 001·003 허용 | 재로그인(또는 새로고침) 후 사이드바·각 매출 URL·API | 사이드바 "매출"이 첫 허용 항목(일별)으로 연결, 001·003만 열람, 002·004·005·006 URL 리다이렉트·API 403. DSH-001 매출 KPI 표시(001 기준) | P0 | pytest: test_sales_grants.py::test_me_returns_effective_sales_report_keys |
| TC-MG-005 | WEB-SAL-006 | export grant 없음 | 006 미허용 | `GET /api/v1/sales/export?include=invalid` | include 파싱 전에 403(권한 우선) | P1 | pytest: test_sales_grants.py::test_export_without_006_denies_before_include_parsing |
| TC-MG-006 | WEB-SAL-* | BASIC에서 grant 무효 | manager02, DB에 grant 행 존재(PRO 상태에서 부여 후 BASIC 복귀) | `/auth/me`, 매출 API | `sales_report_keys=[]`, API 403 `TIER_LOCKED`. grant 행은 보존(재승격 시 복원) | P0 | pytest: test_sales_grants.py::test_basic_downgrade_preserves_grants_but_makes_them_ineffective |
| TC-MG-007 | WEB-STF-004 | grant 관리 불가 | manager01 | 권한 설정 화면, `PUT /api/v1/staff/{id}/sales-grants` | 토글 조작 불가(화면 표시 방식 기록), API 403 "마스터 권한이 필요합니다" | P0 | pytest: test_sales_grants.py::test_grant_management_guards_replace_and_role_cleanup |
| TC-MG-008 | WEB-STF-002 · SCR-STF-002 | **지점장 초대 차단(D1)** | manager01(PREMIUM) | 초대 역할 목록 확인 → 지점장 선택 생성, API `POST /staff/invites {role:SHOP_MANAGER}`, `POST /staff`, 미용사→지점장 역할 변경 `PATCH /staff/{id}` | 정본 기대: 지점장 옵션 없음·API 403. **현 코드는 허용(201) 예상 → 결과를 그대로 기록하고 Fail 시 High 이슈** | P0 | 수동(자동화 없음) |
| TC-MG-009 | WEB-STF-002 | 미용사 초대 | manager01 / manager02 | 미용사 초대 링크 생성 | manager01 201. manager02는 샵B 한도 초과로 409 `TIER_LIMIT`(D9) | P1 | pytest: test_staff_invites_shops.py |
| TC-MG-010 | WEB-SYS-001 · WEB-STF-005 · WEB-SYS-004 | 샵 정보 수정 | manager01 | 영업 신고 정보(주소 포함) 수정·저장, PG·예약금 설정 조회 | 저장 성공(`PATCH /shop` 지점장 허용), PG 설정 접근 가능 — 정본 "담당 샵 정보 수정 ✓"과 일치 여부 확인(v2.6 "권한 검토 필요" 재판정) | P1 | e2e: WEB-SYS-001, WEB-SYS-004 |
| TC-MG-011 | ShopCreate · `/app/shops/new` | **샵 추가 차단(D2)** | manager01(샵A PREMIUM 컨텍스트) | 지점 화면 샵 추가 버튼 확인, `POST /api/v1/shops` | 정본 기대: 버튼 없음·API 403. **현 코드는 역할 검사 없이 201로 manager01 소유 신규 샵 생성 가능 예상** → 결과 기록, 생성 시 정리 | P0 | 수동 |
| TC-MG-012 | WEB-SUB-001 | 구독 관리 불가 | — | 메뉴·URL·API | 메뉴 없음, `/app/home` 리다이렉트, API 403 | P0 | pytest: test_subscription_change_requests.py |
| TC-MG-013 | WEB-PAY-008 | 결제 취소·환불 | 샵A/샵B 결제 | manager01 부분 환불, manager02 동일 시도 | manager01 성공. manager02 버튼 비활성 + 403 `TIER_LOCKED` | P0 | pytest: test_payment_cancel_refund_gate.py |
| TC-MG-014 | WEB-CUS-001~012 | 고객·반려동물 전체(배정 샵) | manager01 | 고객 목록(8명 전원), 상세, 수정, 반려동물 등록, 동의 현황 CSV | 배정 샵 고객 전원 평문 조회·수정 가능, 단건 조회·목록·CSV에 PII 감사로그 기록(`pii_access_logs`) | P0 | pytest: test_pii_audit_log.py, test_phase2_customers_pets.py · e2e: WEB-CUS-* |
| TC-MG-015 | WEB-CUS-013/014 · SCR-CUS-012/013 | 고객 연결 승인 | CUST-A 대기 | PC·모바일 양쪽 대기열 확인 후 승인 | 대기열 샵A만, 승인 시 LINKED, 이미 결정된 건 409 | P0 | pytest: test_customer_link_queue.py · e2e: SCR-CUS-012/013 |
| TC-MG-016 | WEB-SYS-005~007 · SCR-LAW-002/003 | **사고 기록 경로(D4)** | manager01 | PC `/admin/system/incidents` 접근 → 모바일 `/app/incidents`에서 조회·등록 | PC는 `/app/home` 리다이렉트(SHOP_MASTER 전용 라우트), 모바일은 조회·등록 가능. 정본 ✓와 PC 차이를 ⚠️로 기록 | P1 | e2e: WEB-SYS-005(지점장 리다이렉트 검증), SCR-LAW-002/003 |
| TC-MG-017 | WEB-RES-001~013 | 예약 전체 관리 | manager01 | 전 미용사 예약 조회, 등록·수정·취소·완료, 고객 신청을 임의 미용사로 확정 | 배정 샵 전체 예약 대상, 임의 미용사 배정 확정 가능 | P1 | pytest: test_customer_reservations.py::test_manager_can_confirm_onto_any_groomer · e2e: WEB-RES-* |
| TC-MG-018 | WEB-MIL-002 · WEB-CUS-009 | 등급 기능 차이 | manager01 / manager02 | 마일리지 수동 조정, 특이사항 자동팝업 토글, 완료사진 ZIP | manager01 가능, manager02 잠금(403 `TIER_LOCKED`/안내) | P1 | pytest: test_mileage_manual_adjust_gate.py, test_pet_auto_popup.py, test_reservation_photo_zip.py |
| TC-MG-019 | SCR-HOM-001 · SCR-STF-004 | 모바일 겸직 | manager01 | `/app` 로그인 → 홈 → 내 스케줄(시드 가능시간 확인·수정) | 반복 슬롯 + 특정일 슬롯 표시, 저장 성공, 잘못된 구간(시작≥종료) 거부 | P1 | pytest: test_staff_availability.py · e2e: SCR-STF-004 |
| TC-MG-020 | WEB-SYS-003 · WEB-NTF-* | 로그인 이력·알림 | manager01 | 로그인 이력, 알림 템플릿 수정, 발송 이력 재발송(SENT 건) | 이력 조회 가능(`require_role` 지점장 포함), SENT 로그 재발송 409 | P2 | e2e: WEB-SYS-003, WEB-NTF-003 |
| TC-MG-021 | WEB-SVC-001~004 | 서비스 관리 | manager01 | 서비스 등록·수정·비활성화(사유) | 성공, 샵 격리 | P2 | pytest: test_services.py · e2e: WEB-SVC-* |

### 4.4 미용사 (TC-GR) — `groomer01@gmail.com`(샵A PREMIUM + 샵B BASIC) / `groomer02@gmail.com`(샵B BASIC)

근거: `data_scope.py`(`reservation_scope_condition`, `customer_scope_condition`, `payment_scope_condition`), `staff_invites.py`(accept), `auth.py`(select-shop), `reservations.py`(requests/confirm/reject — 미용사 허용 D-08), `incidents.py`, `customer_links.py`, `staff.py`(본인 비번·닉네임·색상 모드만), `app/routes.tsx`, `docs/offline_grooming_draft_flow.md`.

데이터 범위(코드): 예약=본인 배정(+조회 시 미배정 포함), 고객=본인 배정 BOOKED/DONE 예약 고객 ∪ 첫 예약 전 본인 생성 고객, 결제=본인 배정 예약 결제.

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-GR-001 | (`/invite/:token`) · SCR-COM-002/003 | 초대링크 신규 가입 | shop01이 샵A-2 미용사 초대 링크 발급(신규 이메일용) | 로그아웃 상태로 링크 진입 → 샵명·역할 표시 확인 → 가입(약관) | 신규 Account + 샵A-2 SHOP_GROOMER 멤버십 생성, 로그인 후 `/app/home`. 링크 재사용 불가(consumed) | P1 | pytest: test_staff_invites_shops.py::test_signup_accept_creates_account_membership_and_consents |
| TC-GR-002 | (`/invite/:token`) · SCR-COM-004 | 기존 계정 초대 수락 → 다중 샵 | groomer02 로그인 상태, 샵A-2 초대 링크 | 링크 진입 → 현재 계정으로 합류 → 로그아웃·재로그인 | 샵 선택에 샵B·샵A-2 표시. 다른 계정 로그인 상태로 수락 시 422 "현재 로그인된 계정으로 합류해 주세요"/"로그아웃 후 새 계정으로 가입해 주세요" 분기 | P1 | pytest: test_staff_invites_shops.py::test_current_account_accept_adds_membership_and_enables_shop_selection |
| TC-GR-003 | (`/invite/:token`) | 만료·소비·중복 초대 | 7일 경과(DB `expires_at` 조정) 또는 소비된 링크 | 진입·수락 | 401 "유효하지 않거나 만료된 초대입니다", 이미 소속 409 "이미 이 업체에 소속된 계정입니다", 동시 수락 1건만 성공 | P0 | pytest: test_staff_invites_shops.py::test_invite_lookup_and_accept_reject_unknown_expired_or_consumed |
| TC-GR-004 | SCR-COM-004 → 샵 선택(`/app/select-shop`) | 다중 샵 선택·전환 | groomer01 | 로그인 → 샵A 선택 → 홈 → 샵 전환으로 샵B | 샵 선택에 샵A·샵B, 전환 후 데이터·등급 기능이 샵B 기준으로 바뀜. 타 계정 staff_id로 select-shop 401 | P0 | pytest: test_shop_switch_options.py · e2e: SCR-COM-004 |
| TC-GR-005 | WEB-* | PC 웹 차단 | groomer02 | `/admin`, `/admin/customers` 직접 입력 | `/app/home` 리다이렉트(v2.6 정상 동작 회귀) | P0 | e2e: WEB-COM-001 |
| TC-GR-006 | SCR-HOM-001 / SCR-HOM-002 | 홈·빠른 예약 | groomer01 샵A | 홈 집계, 빠른 예약 시트(최근 고객·서비스 제안) | 집계가 본인 범위 예약 기준, 제안에 범위 밖 고객 없음 | P1 | pytest: test_quick_reservation_suggestions.py · e2e: SCR-HOM-001/002 |
| TC-GR-007 | SCR-RES-001 / SCR-RES-018 / SCR-RES-002 | 캘린더 범위·탭 순서 | groomer01 샵A | 월→주→일 탭 순서, 각 뷰 예약 목록 | 탭 순서 월→주→일. 본인 배정 + 미배정 예약만, manager01 배정 예약 미표시. 주별(SCR-RES-018)은 요일별 건수 요약·행별 빠른등록·행 탭 시 일별 이동 | P0 | pytest: test_groomer_data_scope.py::test_groomer_reservation_scope_and_assignment_rules · e2e: SCR-RES-001/002/018 |
| TC-GR-008 | SCR-RES-019 | 통합 예약 등록 폼 | groomer01 샵A | "+ 예약 등록" → 고객·반려동물·서비스·일시 선택 → 저장 → SCR-RES-009 완료 | 담당 기본값=본인, 서비스 선택 시 예상 금액 실시간(정액/가변 문구), 가능 날짜·시간 추천, 특이사항 자동 팝업(PRO 이상). 담당에 동료 지정 시 거부 | P1 | e2e: SCR-RES-019, SCR-RES-009 |
| TC-GR-009 | SCR-RES-002 | 빈 시간 탭·드래그 즉시예약 | groomer01 | 일별 06:00~23:00 그리드에서 빈 시간 탭, 드래그로 1시간 블록 지정 | SCR-RES-019가 날짜·시간 프리필로 열림(30분 슬롯 정책 유지), 360px에서 가로 넘침 없음 | P1 | e2e: SCR-RES-002, mobile-360-overflow |
| TC-GR-010 | SCR-RES-001 | 월별 빈 날짜 즉시예약 | groomer01 | 빈 날짜 탭 | 해당 날짜로 SCR-RES-019 진입 | P2 | e2e: SCR-RES-001 |
| TC-GR-011 | SCR-RES-001/002/018 | 캘린더 색상 모드 | groomer01 | 미용사별↔메뉴별 토글 → 로그아웃·재로그인 → groomer02로 확인 | 본인 계정에만 영구 저장(`PATCH` 본인 전용), 타 계정 영향 없음 | P2 | pytest: test_staff_calendar_preference.py |
| TC-GR-012 | SCR-RES-001(신청 탭) · SCR-RES-016 / SCR-RES-017 | 고객 신청 확정·거절 | CUST-A 예약 신청 존재 | 신청 대기 탭 → 본인 배정으로 확정 / 다른 건 사유 코드로 거절 | 확정 시 BOOKED, **동료에게 확정 불가**, 담당 미지정 확정 거부, 겹침 409, 거절 사유 "기타"는 메모 필수 | P0 | pytest: test_customer_reservations.py::test_non_manager_cannot_confirm_onto_a_colleague, test_reservation_queue.py · e2e: SCR-RES-016/017 |
| TC-GR-013 | SCR-RES-013 / SCR-RES-014 / SCR-RES-015 | 완료 처리·사진·카카오 발송 | 본인 배정 BOOKED | 샵A: 완료 → 사진 5장 → 6번째 시도 → 발송 / 샵B: 사진 첨부 시도 | 샵A: 청구 금액 표시, 최대 5장, 발송 결과 표시. 샵B(BASIC): 사진 섹션 대신 안내 + `POST /reservations/{id}/photos` 403 `TIER_LOCKED` | P1 | pytest: test_completion_photo_attach_gate.py · e2e: SCR-RES-013/014/015 |
| TC-GR-014 | SCR-RES-013 (완료 알림장 초안) | 오프라인 미용 draft(Phase 14.1) | 샵A 본인 배정 완료 건, 폰 Chrome | 비행기 모드 → 코멘트·사진 입력 → 화면 이탈·재진입 → "이어서 작성" → 온라인 복귀 → 업로드 재개 → 발송 | 750ms 후 저장 시각 표시, 재진입 시 이어서/삭제 선택, 사진 pending→uploaded(재시도 1·2·4초), 미완료 사진은 발송 차단, SENT 후 초안 삭제, 중복 발송 없음 | P1 | pytest: test_reservation_photo_idempotency.py · 수동(실기기) |
| TC-GR-015 | SCR-CUS-001 / SCR-CUS-003 | 고객 범위 | groomer01 샵A, groomer02 샵B | 고객 목록·검색·상세 URL 직접 입력(범위 밖 고객 ID) | 범위 안 고객만 목록·검색. 범위 밖 ID 직접 접근 404. 시드 기준 groomer01 샵A: 배정 예약 고객 + 1·2번(본인 생성) | P0 | pytest: test_groomer_data_scope.py::test_groomer_customer_pet_and_shared_mileage_scope · e2e: SCR-CUS-001 |
| TC-GR-016 | SCR-CUS-003 / SCR-CUS-004 | **고객 PII 마스킹(D5)** | groomer01 | 범위 안 고객 상세의 연락처·메모 확인 | 정본 기대: 개인정보 마스킹 열람. **현 코드는 범위 안 고객 평문 연락처 응답** → ⚠️/Fail 판정은 기획 확인 후, 관측값 기록 | P0 | 수동 |
| TC-GR-017 | SCR-CUS-002 | 고객 등록·생성자 예외 | groomer01 | 신규 고객 등록 → 목록 확인 → 해당 고객 첫 예약을 동료(manager01) 배정으로 생성(manager01 계정) → groomer01 목록 재확인 | 등록 직후 보임, 첫 예약 생성 후 본인 배정이 아니면 목록에서 사라짐(생성자 예외 종료) | P0 | pytest: test_groomer_data_scope.py::test_creator_exception_ends_when_first_reservation_is_created |
| TC-GR-018 | SCR-CUS-012 / SCR-CUS-013 | 계정 연결 승인 불가(D6) | CUST-A 대기 | 고객 목록 상단 연결 요청 진입점, `/app/customers/link-requests` 직접, `GET /api/v1/link-requests` | 진입점 없음, `/app/home` 리다이렉트, API 403 | P0 | pytest: test_customer_link_queue.py::test_groomer_forbidden |
| TC-GR-019 | SCR-PAY-006 / SCR-PAY-007 / SCR-PAY-008 | 결제 범위·취소 불가 | groomer01 샵A | 결제 이력, 범위 밖 결제 ID 직접, 취소 버튼, `POST /payments/{id}/cancel` | 본인 배정 예약 결제만, 범위 밖 404, 취소 버튼 없음, API 403(PRO 샵에서도 역할 거부) | P0 | pytest: test_payment_cancel_refund_gate.py::test_pro_shop_groomer_still_gets_403_role_forbidden, test_groomer_data_scope.py::test_groomer_payment_scope_and_cash_receipt_policy |
| TC-GR-020 | SCR-PAY-001~005 | 결제 처리 | 본인 배정 DONE 예약 | 마일리지 차감(100단위·전액) → 수단 선택 → 완료 → 현금영수증 | 결제 생성, 샵B에서 간편결제 비활성(`EASY_PAYMENT`) | P1 | e2e: SCR-PAY-001~005 |
| TC-GR-021 | SCR-LAW-003 | 사고 기록 열람 | groomer01 샵A / 샵B | 사고 이력 조회·검색 | 샵A: 동료 범위 고객명 마스킹·반려견·조치 내용은 표시, 범위 밖 고객명 검색 무시. 샵B: 403 `TIER_LOCKED` 안내 | P0 | pytest: test_groomer_data_scope.py::test_groomer_incidents_mask_colleague_customer_but_keep_safety_details, test_incident_record_gate.py |
| TC-GR-022 | SCR-LAW-002 | **사고 기록 등록 차단(D3)** | groomer01 샵A(PRO 이상) | 메뉴·`/app/incidents/new`·`POST /api/v1/incidents` | 정본 기대: 불가. **현 코드·테스트는 허용(201)** → 결과 기록, 정본 개정 여부 기획 확인 | P0 | pytest: test_incident_record_gate.py::test_pro_groomer_can_create_and_list_incidents(현 코드 기준) |
| TC-GR-023 | SCR-STF-004 | 본인 가능시간 | groomer01 샵A | 요일 반복 슬롯 교체, 특정일 슬롯, 겹침·역전 구간 입력 | 저장 시 정렬·교체, 특정일 보존, 잘못된 구간 거부(기존 데이터 유지), 샵별 분리(샵B 슬롯 불변) | P1 | pytest: test_staff_availability.py · e2e: SCR-STF-004 |
| TC-GR-024 | SCR-STF-001 / SCR-STF-003 / SCR-STF-002 | 직원 화면 권한 | groomer01 | 직원 목록, 본인 상세 비번·닉네임 수정, 동료 상세 수정, `/app/staff/new` | 목록 열람, 본인 비번·닉네임·색상만 수정, 동료 수정 403 "본인의 비밀번호와 닉네임만 변경할 수 있습니다", 초대 화면 `/app/home` 리다이렉트 | P0 | e2e: SCR-STF-001~003 |
| TC-GR-025 | SCR-CUS-006 · SCR-MIL-001 | 마일리지 범위 | groomer01 | 마일리지 내역·고객 검색, 수동 조정 API | 범위 안 고객만(동료 전용 고객 숨김), 수동 조정 403(역할) | P0 | pytest: test_groomer_data_scope.py::test_groomer_mileage_ledger_hides_colleague_only_customer, test_mileage_manual_adjust_gate.py |
| TC-GR-026 | `/app/shops/new` | **샵 추가 차단(D2)** | groomer01 샵A(PREMIUM 컨텍스트) | URL 직접 진입·저장, `POST /api/v1/shops` | 정본 기대: 불가. **현 코드·테스트는 201(미용사 소유 신규 샵 + SHOP_MASTER 멤버십)** → 결과 기록, 생성 샵 정리 | P0 | pytest: test_staff_invites_shops.py::test_create_shop_reuses_account_and_returns_enterable_membership(현 코드 기준) |
| TC-GR-027 | SCR-SVC-001~003 | 서비스 관리 | groomer01 | 서비스 목록, 등록·수정 화면 진입·저장 | 목록 열람 가능, 저장 API 403(`require_role` 지점장·마스터). 등록 화면 노출 여부 `(확인 필요)` | P2 | e2e: SCR-SVC-001~003 |
| TC-GR-028 | SCR-COM-005 | 로그아웃·초안 정리 | TC-GR-014 미발송 초안 존재 | 로그아웃 → 확인 대화상자 | 현재 샵·계정 초안만 삭제 확인, 취소 시 로그인 유지 | P1 | e2e: SCR-COM-005 |
| TC-GR-029 | SCR-CUS-011 | 완료 사진 권한 | groomer02 | groomer01 담당 예약 사진 URL 직접 | 범위 밖 404, 타 샵 소속 계정의 사진 접근 불가 | P0 | pytest: test_customer_history.py::test_same_shop_staff_reads_photo_and_other_shop_staff_cannot |
| TC-GR-030 | SCR-RES-011 / SCR-RES-012 | 예약 수정·취소 | 본인 배정 BOOKED | 시간 변경(알림 재발송 체크), 사유 입력 취소, 동료 예약 ID로 수정 API | 본인 건 성공·취소 알림톡 자동 발송 기록, 동료 건 404/403·부작용 없음 | P0 | pytest: test_groomer_data_scope.py::test_denied_reservation_actions_have_no_side_effects · e2e: SCR-RES-011/012 |

### 4.5 고객 (TC-CU) — 고객 앱 APP-* (신규 가입 계정 CUST-A/B/S, 운영 `010-0000-0929`)

근거: `customer_auth.py`, `customer_social_auth.py`, `customer_shops.py`(`join_or_reactivate_membership`, 3값 축약), `customer_links.py`, `customer_reservations.py`(`require_linked_membership`, 대기 3건), `customer_history.py`, `customer_pets.py`, `customer_account.py`(withdraw), `customer_push.py`, `core/phone_verification.py`, `customer-app/app/` 라우트.

전제: 고객 앱 E2E 하네스 없음(jest 컴포넌트 테스트만). 실기기(SM-S901N) 또는 개발 백엔드 지정 로컬 빌드로 수행. 푸시는 Firebase 미설정으로 **수신 검증 범위 외**.

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-CU-001 | APP-ONB-002 | 휴대폰 가입 | DEBUG 모드(P-11) | 이름·휴대폰 → 인증번호 발송(자동 입력 확인) → 비밀번호 8자+·생년월일 → 필수 약관 2종 동의(마케팅 선택) → 가입 | 가입·로그인 완료, 약관 동의 이력 3행. 운영에서는 **테스트 번호만 사용**(DEBUG로 임의 번호 인증 가능 — active_risk) | P1 | pytest: test_customer_auth_signup.py · jest |
| TC-CU-002 | APP-ONB-002 | 가입 검증 | CUST-A 존재 | 같은 번호 재가입, 인증 미완료, 코드 5회 오입력, 5분 경과 코드, 발송 1분 6회 | 409 "이미 가입된 휴대폰 번호입니다", 400 "휴대폰 인증을 완료해 주세요", 5회 오입력 잠김, 만료 거부, 발송 6회째 429 | P1 | pytest: test_phone_verification.py, test_customer_auth_rate_limit.py |
| TC-CU-003 | APP-ONB-001 | 로그인 | CUST-A | 휴대폰+비밀번호 로그인, 오비번, 탈퇴 계정 로그인 | 성공 시 홈, 실패는 동일 문구 401 "휴대폰 번호 또는 비밀번호가 올바르지 않습니다"(계정 존재 비노출), 분당 5회 제한 | P0 | pytest: test_customer_auth_login.py |
| TC-CU-004 | APP-ONB-001 → APP-ONB-003 | 카카오·네이버 가입·재로그인 | 실기기 실키(09-21 검증 완료 항목 회귀) | 소셜 버튼 → 외부 인증 → `auth/callback` 딥링크 → 가입 완료(휴대폰 인증 필수) → 로그아웃 → 소셜 재로그인 | 신규는 휴대폰 인증 후 계정 생성, 재로그인은 즉시 홈. 딥링크 404·레이스 재발 없음 | P1 | pytest: test_customer_auth_social.py, test_customer_social_auth_pkce.py, test_customer_social_auth_mobile.py · 수동(실기기) |
| TC-CU-005 | APP-PRF-004 | 소셜 연동 관리 | CUST-A(로컬 계정), CUST-S(소셜 전용) | CUST-A에 카카오 추가 연동 → 해제, CUST-S 마지막 수단 해제 시도 | 연동·해제 성공, 소셜 전용 계정의 마지막 로그인 수단 해제 409 | P1 | pytest: test_customer_social_link_management.py · jest |
| TC-CU-006 | APP-ONB-004 / APP-ONB-004b | 가입코드 매장 가입(매칭) | CUST-A(`010-7777-0001`, 샵A 선등록 고객과 동일 번호), 샵A 가입코드(TC-MA-023) | QR 스캔 또는 코드 입력 | 샵A 멤버십 ACTIVE + `PENDING_APPROVAL`, 홈 카드 "대기(PENDING)". 잘못된 코드 "유효하지 않은 가입 코드입니다" | P0 | pytest: test_shop_join_codes.py, test_customer_link_gate.py · jest |
| TC-CU-007 | APP-ONB-004a | 매장명 검색 가입 | CUST-B | "샵B" 검색 → 가입 | 검색 결과 `limit≤50`, 가입 후 휴대폰 불일치로 `UNLINKED` → 홈 카드 "NOT_LINKED" | P1 | pytest: test_customer_shop_search.py |
| TC-CU-008 | APP-RES-001 · APP-HIST-001 · APP-ONB-006 | 미승인 데이터 차단 | CUST-A(PENDING), CUST-B(UNLINKED) | 예약 탭·이력 탭·반려동물 등록 진입, API 직접 | 모든 매장 데이터 403 "매장 연결 승인 후 이용할 수 있습니다", 앱은 안내 상태 표시 | P0 | pytest: test_customer_link_gate.py |
| TC-CU-009 | SCR-CUS-013 / WEB-CUS-014 → APP-HOME-001 | 승인 즉시 반영 | CUST-A 앱 실행 중 | manager01이 승인 → 고객 앱 새로고침(재로그인 없음) | 다음 요청부터 LINKED, 카드 "연결됨", 예약·이력 접근 가능 | P0 | pytest: test_customer_link_gate.py::test_gate_opens_immediately_after_staff_approves |
| TC-CU-010 | APP-HOME-001 | 거절 비노출 | 매칭 계정 CUST-A2(샵A 선등록 고객 `010-7777-0002`와 동일 번호) 가입코드 가입 → PENDING(거절은 PENDING에서만 가능) | 앱 확인, 동일 매장 재가입 시도 | 고객에게는 NOT_LINKED로만 표시(거절 사실 비노출), 거절 후 셀프 재신청으로 PENDING 복귀 불가 | P0 | pytest: test_customer_link_gate.py::test_customer_response_hides_rejection, test_customer_link_no_self_reapply.py |
| TC-CU-011 | SCR-CUS-012(UNLINKED 필터) → APP-HOME-001 | 수동 연결 | CUST-B UNLINKED | manager02가 대기열 UNLINKED 필터(UI 위치 `(확인 필요)`)에서 샵B 고객 레코드를 지정해 승인 | CUST-B LINKED, 연결 대상 고객 레코드의 반려동물이 APP-ONB-005에 표시 | P1 | pytest: test_customer_link_queue.py |
| TC-CU-012 | APP-ONB-005 / APP-ONB-006 | 반려동물 확인·등록 | LINKED | 기존 반려동물 확인(이름 수정) → 직접 등록(이름만/등록번호 포함) | 확인 시 `pets_confirmed_at` 기록(멱등), 빈 이름 거부, 등록 반려동물이 샵 화면 SCR-CUS-005에 표시 | P1 | pytest: test_customer_pets.py · jest |
| TC-CU-013 | APP-SHP-001 | 다매장 전환 | CUST-A 샵A LINKED + 샵B 가입 | 매장 전환 시트에서 샵B 선택 | 예약·이력이 선택 매장 기준으로 전환, 다른 계정 멤버십 미노출. 매장 탈퇴(`/customer/shops/{id}/leave`)는 UI 없음(D11) | P1 | pytest: test_customer_link_gate.py::test_membership_list_does_not_leak_other_accounts |
| TC-CU-014 | APP-RES-002 | 예약 신청 | LINKED, 샵A 가능시간 존재 | 반려동물 1·서비스 1 선택 → 날짜(내일~30일) → 서버 제공 슬롯 → 신청 ×4 | 신청 REQUESTED, 오늘·31일 이후 선택 불가, 확정 예약 시간은 슬롯 제외, 4번째 신청 409 "대기 중인 신청이 3건입니다" | P0 | pytest: test_customer_reservations.py, test_reservation_slots.py, test_reservations_booking_window.py |
| TC-CU-015 | APP-RES-001 / APP-RES-003 | 예약 목록·상세 | 신청·확정·거절·완료 건 존재 | 탭(대기/확정/거절·취소/완료) 전환, 상세 | 상태별 분류 정확, 거절 사유 표시 방식 기록, 타 고객 예약 ID 직접 조회 불가 | P1 | pytest: test_customer_reservations.py · jest |
| TC-CU-016 | APP-RES-004 | 신청 취소 | REQUESTED 1건 | 취소 확인 | 취소 반영, 샵 신청 대기열에서 사라짐. 확정 건 취소 가능 여부 `(확인 필요)` | P1 | jest |
| TC-CU-017 | APP-RES-001 | 확정·거절 결과 반영 | TC-GR-012 수행 | 앱 새로고침 | 확정 건 "확정", 거절 건 "거절·취소" 탭. 푸시 알림은 범위 외(⛔ Firebase) | P1 | 수동 |
| TC-CU-018 | APP-HIST-001 | 마일리지·미용 이력·사진 | E2E-01 결제·완료 후 | 이력 탭 확인, 다른 고객 photo_id로 사진 URL 직접 | 현재 매장 마일리지 잔액·반려동물별 완료 이력·사진 표시, 사진은 고객 토큰 인증 경로로만, 타 고객 사진 404 | P0 | pytest: test_customer_history.py::test_history_gate_and_photo_idor_hide_customer_data |
| TC-CU-019 | APP-JNL-001 / APP-JNL-002 | 알림장 | 완료 알림장 발송됨 | 목록 필터(4축)·썸네일 → 상세 | 사진 썸네일·코멘트·담당자·매장 표시. 기기 갤러리 저장 없음(범위 외) | P2 | jest |
| TC-CU-020 | APP-PRF-001 / APP-PRF-005 | 프로필·동의 이력 | LINKED | 프로필 수정, 동의 이력 시트 | 수정 저장, 계정 약관 동의만 시간 역순(방문별 미용 동의서 미포함) | P2 | pytest: test_customer_account_profile.py, test_customer_account_consents.py · jest |
| TC-CU-021 | APP-PRF-001 | 로그아웃·전체 로그아웃 | 2기기(또는 2세션) 로그인 | 한 기기 로그아웃 → API `POST /customer/auth/logout-all` → 다른 세션 refresh | 로그아웃은 해당 refresh만 폐기, logout-all은 본인 계정 전 토큰 폐기(타 계정 무영향). logout-all은 UI 없음(D11) | P1 | pytest: test_customer_auth_tokens.py |
| TC-CU-022 | APP-PRF-003 | 회원 탈퇴 | 테스트 계정(운영은 0929 정리 시) | 탈퇴 확인 → 기존 토큰으로 API → 같은 번호 재가입 | 즉시 익명화(이메일·휴대폰·이름·자격증명 제거), 멤버십 LEFT, 파기 로그, 기존 토큰 401, 같은 번호 재가입 가능 | P0 | pytest: test_customer_account.py · jest: withdraw-confirm |
| TC-CU-023 | (API) | 토큰 종류 격리 | CUST-A·groomer01 토큰 | 고객 토큰으로 `/api/v1/customers`, `/platform/shops` / groomer01 토큰으로 `/customer/auth/me` | 각각 403("샵 컨텍스트가 필요한 기능입니다"/"슈퍼마스터 권한이 필요합니다"/"고객 전용 API입니다"), 우연히 같은 숫자 ID의 staff로 해석되지 않음 | P0 | pytest: test_customer_security_regression.py |
| TC-CU-024 | APP-RES-003 | 예약금 표시(D11) | 마스터·지점장이 예약금 포함 예약 생성(WEB-RES-005/009) 후 고객 연결 | 고객 앱 예약 상세 | 고객 앱에 예약금 정보 없음(현행). 설계 요구 여부 `(확인 필요)` → ⚠️ 기록 | P2 | 수동 |
| TC-CU-025 | (앱 기동) | 푸시 토큰 등록 | 실기기 | 로그인 후 `customer_push_tokens` 행 확인, 로그아웃 시 비활성 | 등록·비활성 API 동작. **수신·알림 탭 딥링크는 ⛔(Firebase 미설정, 15-UAT #4/5/7/8)** | P2 | pytest: test_customer_push_tokens.py, test_push_outbox.py |
| TC-CU-026 | APP-HOME-001 | 홈 구성 | 다매장·LINKED | 홈 확인 | 매장별 연결 상태 3값(PENDING/LINKED/NOT_LINKED)만, 반려동물 요약 카드, 최근 알림 2건 | P2 | jest: (tabs)/index |
| TC-CU-027 | APP-ONB-004b → APP-HOME-001 | 시드 고객 번호 매칭(D12) | 샵A 시드 고객 번호 `010-9101-0003`(박지우)로 신규 가입 | 샵A 가입코드로 가입 | 의도상 `PENDING_APPROVAL`이어야 하나 **해시 형식 불일치로 `UNLINKED`(NOT_LINKED) 예상** → 관측값 기록, 시드 수정(숫자만 해시) 이슈 후보 | P1 | 수동 |

---

## 5. 교차 검증 매트릭스

### 5.1 기능 × 계정 그룹 (정본 권한 매트릭스 + 코드 확인값)

표기: ✓ 허용 / ✗ 거부 / △ 조건부 / ⚠️ 정본과 코드가 다름(괄호에 코드 관측 예상). 셀 뒤는 검증 TC.

| 기능 | 슈퍼마스터 | 마스터 | 지점장 | 미용사 | 고객 |
|---|---|---|---|---|---|
| 플랫폼 대시보드·SUP-* | ✓ SM-001~026 | ✗ X-01 | ✗ X-01 | ✗ X-01 | ✗ CU-023 |
| 임의 샵 데이터 열람·수정 | △ 대행 진입 시만 SM-005, SM-028 | ✗ 본인 샵만 X-02 | ✗ 배정 샵만 X-02 | ✗ X-02 | ✗ |
| 샵 신규 등록 | ✗ 대행 중 SM-008 (SUP-BIZ-006으로 업체 등록 SM-018) | ✓ PREMIUM만 MA-007 | ⚠️ ✗ (코드 ✓ PREMIUM) MG-011 | ⚠️ ✗ (코드 ✓ PREMIUM) GR-026 | ✗ |
| 담당 샵 정보 수정 | ✓ 대행 | ✓ MA-006 | ✓ MG-010 | ✗ X-03 | ✗ |
| 지점장 초대·등록·수정 | ✓ 대행(PREMIUM) | ✓ PREMIUM만 MA-009 | ⚠️ ✗ (코드 ✓ PREMIUM) MG-008 | ✗ GR-024 | ✗ |
| 미용사 초대·등록·수정 | ✓ 대행 | ✓ MA-008 (한도 MA-011) | ✓ MG-009 | ✗ GR-024 | ✗ |
| 마스터 권한 부여 | ✗ | ✗ MA-010 | ✗ MA-010 | ✗ | ✗ |
| 예약 등록·수정·취소·완료·사진 | ✓ 대행(마커 배정 ✗ SM-011) | ✓ | ✓ MG-017 | △ 본인 배정분 GR-007·030 | △ 신청·신청 취소만 CU-014·016 |
| 고객 예약 신청 확정·거절 | ✓ 대행 | ✓ | ✓ 임의 미용사 MG-017 | △ 본인 배정만 GR-012 | ✗ |
| 고객 계정 연결 승인 | ✓ 대행 | ✓ MA-023 | ✓ MG-015 | ✗ GR-018 | ✗ |
| 본인 가능시간 | ✗ 대행 SM-010 | ✓ MA-027 | ✓ MG-019 | ✓ GR-023 | ✗ |
| 고객·반려동물·마일리지 | ✓ 대행 | ✓ | ✓ MG-014 | ⚠️ 마스킹 열람 (코드: 범위 안 평문, 범위 밖 비노출) GR-015·016·025 | △ 본인 것만 CU-012·018 |
| 마일리지 수동 조정 | ✓ 대행(PRO+) | ✓ PRO+ MA-024 | ✓ PRO+ MG-018 | ✗ GR-025 | ✗ |
| 결제 처리 | ✓ 대행 | ✓ MA-018 | ✓ | △ 본인 배정 GR-020 | ✗ |
| 결제 취소·환불 | ✓ 대행(PRO+) | ✓ PRO+ MA-019 | ✓ PRO+ MG-013 | ✗ GR-019 | ✗ |
| 매출·정산 열람 | ✓ 대행(PRO+) SM-013 | ✓ PRO+ MA-015 | △ PRO+ & grant 항목별 MG-003~006 | ✗ X-04 | ✗ X-04 |
| 매출 grant 관리 | ✓ 대행(PRO+) | ✓ PRO+ MA-013 | ✗ MG-007 | ✗ | ✗ |
| 사고 기록 열람 | ✓ 대행(PRO+) | ✓ PRO+ MA-025 | ⚠️ ✓ (PC ✗·모바일 ✓) MG-016 | △ 개인정보 제외 GR-021 | ✗ |
| 사고 기록 등록 | ✓ 대행(PRO+) | ✓ PRO+ MA-025 | ⚠️ ✓ (PC ✗·모바일 ✓) MG-016 | ⚠️ ✗ (코드 ✓ PRO+) GR-022 | ✗ |
| 구독 조회·변경 요청 | ✗ 대행 SM-012 (SUP-SUB-001 직접 변경 ✓) | ✓ owner만 MA-021·022 | ✗ MG-012 | ✗ | ✗ |
| 모바일 `/app` 사용 | ✗ 리다이렉트 SM-027 | ✓ 겸직 MA-027 | ✓ 겸직 MG-019 | ✓ | — |
| PC 웹 `/admin` 사용 | ✓ | ✓ | ✓ | ✗ 리다이렉트 GR-005 | — |
| 고객 앱 API | ✗ CU-023 | ✗ CU-023 | ✗ | ✗ CU-023 | ✓ |

### 5.2 부정·경계 테스트 (TC-X)

| TC ID | 화면ID | 항목 | 사전조건 | 절차 | 기대결과 | 우선순위 | 자동화 |
|---|---|---|---|---|---|---|---|
| TC-X-01 | SUP-* | 플랫폼 화면·API 차단 | shop01·manager01·groomer01·고객 토큰 | `/admin/platform/overview` 직접 입력, `GET /api/v1/platform/shops` | 화면은 `/admin` 또는 `/app/home` 리다이렉트, API 403 "슈퍼마스터 권한이 필요합니다" | P0 | pytest: test_platform_master.py::test_platform_guard_denies_no_token_and_all_shop_roles |
| TC-X-02 | WEB-CUS-003 · WEB-RES-010 · WEB-PAY-007 | 타 샵 ID 직접 접근 | 샵B 고객·예약·결제 ID 확보(shop02) | shop01(샵A)로 `/admin/customers/{샵B id}`, `GET /api/v1/customers/{id}`, `/reservations/{id}`, `/payments/{id}`, `/link-requests/{id}` | 전부 404(존재 비노출), 목록·검색에도 미포함 | P0 | pytest: test_customer_link_queue.py::test_cross_shop_link_id_returns_404, test_platform_act_as.py::test_act_as_tenant_scope_is_selected_shop_only |
| TC-X-03 | WEB-SYS-001 · SCR-LAW-001 | 미용사 샵 정보 수정 차단 | groomer01 | `PATCH /api/v1/shop`, `PATCH /api/v1/shop/pg-settings` | 403 "권한이 없습니다" | P0 | 수동(API) |
| TC-X-04 | WEB-SAL-* | 매출 API 비인가 | groomer01, 고객 | `GET /api/v1/sales/daily`, `/sales/export` | 미용사 403, 고객 403 | P0 | pytest: test_sales_grants.py |
| TC-X-05 | WEB-COM-001 · WEB-COM-007 | 정지 샵 | 샵B 정지(TC-SM-004) | shop02 기존 세션으로 API 호출, groomer01 select-shop(샵B staff_id) | 기존 access 403 "정지된 샵에는 접근할 수 없습니다", refresh 403, select-shop 403 | P0 | pytest: test_platform_master.py::test_suspend_blocks_login_select_shop_and_existing_staff_token |
| TC-X-06 | SUP-BIZ-005 | 정지 계정·소속 직원(D7) | shop02 계정 정지 | shop02 login/refresh/select-shop, 정지 직전 발급 access로 API, manager02·groomer02 로그인 | shop02 3경로 403 `ACCOUNT_SUSPENDED`. 기존 access·소속 직원 결과는 관측값 기록(정본 문구 "소속 직원 즉시 차단"과 비교) | P0 | pytest: test_platform_business_suspend.py(owner 3경로만) |
| TC-X-07 | (`/invite/:token`) | 만료·위조 초대 | — | 만료 토큰, 임의 문자열, 200자 초과 토큰 | 모두 401 "유효하지 않거나 만료된 초대입니다", 응답 `Cache-Control: no-store` | P0 | pytest: test_staff_invites_shops.py |
| TC-X-08 | WEB-STF-003 | 비활성 소속·owner 보호 | shop01이 groomer01(샵A) 비활성 | groomer01 로그인 → 샵 후보 확인, 기존 샵A 세션 API | 샵A가 후보에서 사라지고 샵B로 바로 진입, 샵A 기존 토큰 401 "비활성 계정입니다". owner 비활성화 409(TC-MA-012). 종료 후 groomer01 샵A 재활성화 | P0 | pytest: test_shop_switch_options.py |
| TC-X-09 | WEB-COM-001 | 소속 없는 계정 | 모든 멤버십 비활성 계정 | 로그인 | 401 "연결된 매장이 없습니다" | P1 | pytest: test_auth_social.py::test_existing_social_identity_without_active_shop_is_rejected(소셜 경로) · 수동 |
| TC-X-10 | (API) | 위조 클레임 | shop01 토큰의 `owned_shop_ids` 조작 불가 확인, act-as 토큰 shop/staff 불일치 | refresh 후 재계산, 불일치 act-as 토큰 | 서버가 DB로 재계산(위조 무시), 불일치 401 "대행 컨텍스트가 유효하지 않습니다" | P0 | pytest: test_multi_shop_sales_scope.py::test_refresh_recomputes_and_ignores_forged_claim, test_platform_act_as.py::test_act_as_claims_fail_closed_on_mismatched_staff_shop_and_account |
| TC-X-11 | WEB-STF-002 | 지점장→지점장 초대(D1) | manager01 | TC-MG-008 API 절차 | 정본 기대 403. 현 코드 결과 기록 | P0 | 수동 |
| TC-X-12 | ShopCreate | 비마스터 샵 생성(D2) | manager01, groomer01 | `POST /api/v1/shops` | 정본 기대 403. 현 코드 결과 기록 | P0 | 수동 |
| TC-X-13 | SCR-LAW-002 | 미용사 사고 등록(D3) | groomer01 샵A | `POST /api/v1/incidents` | 정본 기대 403. 현 코드 결과 기록 | P0 | 수동 |
| TC-X-14 | (API) | CSRF | 로그인 쿠키 보유 | `X-CSRF-Token` 없이 상태 변경 POST | 요청 거부(403 예상, 실제 코드값 기록) | P0 | pytest: test_csrf_protection.py |
| TC-X-15 | APP-RES-002 | 고객 타 매장 요청 | CUST-A(샵A만 LINKED) | body/경로에 샵B shop_id로 예약 신청·이력 조회 | 403 "이 매장에 대한 접근 권한이 없습니다", body의 shop_id로 우회 불가 | P0 | pytest: test_customer_link_gate.py::test_gate_is_not_bypassable_by_shop_id_in_body |
| TC-X-16 | SUP-SYS-001 | 점검 중 계정별 동작 | 점검 ON | 5그룹 각각 로그인·API | 슈퍼마스터만 정상. `/auth/login`·`/auth/refresh`는 통과하나 이후 샵 API 503, 고객 앱은 `/customer/auth/login`부터 503(허용 목록 밖) | P0 | pytest: test_service_maintenance.py |

---

## 6. E2E 시나리오 (계정 릴레이)

각 단계는 해당 계정으로 별도 브라우저 프로필/기기에서 수행하고, 단계마다 스크린샷과 관측값을 남긴다. 한 단계 Fail이면 이후 단계는 ⛔로 표기.

### E2E-01 예약 생애주기: 고객 → 마스터·지점장 → 미용사 → 지점장 → 마스터 (P0)

| # | 계정 | 화면ID | 행위 | 확인 |
|---|---|---|---|---|
| 1 | shop01 | WEB-09 | 샵A 가입코드 발급 | 코드·QR 표시 |
| 2 | manager01 → 고객 CUST-A | SCR-CUS-002 → APP-ONB-002 → APP-ONB-004b | manager01이 "QA고객A"(`010-7777-0001`)·반려동물 "QA콩" 등록 → 고객이 같은 번호로 가입 → 코드 입력 | 홈 카드 PENDING |
| 3 | manager01 | SCR-CUS-012/013 | 연결 요청 승인 | 상세에 메모·마일리지·평문 연락처 없음, LINKED |
| 4 | 고객 | APP-ONB-005 → APP-RES-002 | 기존 반려동물(QA콩) 확인 → 내일 오전 슬롯 신청 | REQUESTED, 대기 탭 표시 |
| 5 | groomer01 | SCR-RES-001(신청 탭) → SCR-RES-016 | 본인 배정으로 확정 | BOOKED, 캘린더 표시 |
| 6 | 고객 | APP-RES-001 | 새로고침 | 확정 탭(푸시는 ⛔) |
| 7 | groomer01 | SCR-RES-013/014/015 | 완료 처리 + 사진 2장 + 알림장 코멘트 발송 | 샵A는 PREMIUM이라 사진 허용, 발송 결과 기록 |
| 8 | manager01 | WEB-NTF-005 | (정책 APPROVAL일 때) 알림장 승인 | 승인 후 고객 앱 APP-JNL-001 표시 |
| 9 | groomer01 | SCR-PAY-001~004 | 결제(마일리지 일부 차감, 카드) | 결제 CONFIRMED, 마일리지 적립 |
| 10 | 고객 | APP-HIST-001 / APP-JNL-002 | 이력·알림장 확인 | 마일리지 잔액·완료 이력·사진(인증 경로) |
| 11 | shop01 | WEB-SAL-001 / WEB-DSH-003 | 오늘 매출 확인 | 9번 결제가 매출·미용사별 실적에 반영 |
| 12 | manager01 | WEB-PAY-008 | 부분 환불 | PARTIALLY_CANCELED, 마일리지 회수 |
| 13 | shop01 | WEB-SAL-001 · WEB-SAL-006 | 순매출·Excel | 환불 차감 순매출, export 일치 |
| 14 | groomer02 | SCR-CUS-001 · SCR-RES-001 | 같은 고객·예약 검색 | 샵B 소속이라 미노출(교차 확인) |

### E2E-02 등급 승격 릴레이: 마스터 → 슈퍼마스터 → 마스터 → 지점장 (P0)

| # | 계정 | 화면ID | 행위 | 확인 |
|---|---|---|---|---|
| 1 | manager02 | WEB-DSH-003 · `/admin/sales` | 매출 접근 | 차단(BASIC) — 기준값 |
| 2 | shop02 | WEB-SUB-001 | 시드 PENDING(BASIC→PRO) 확인 | 대기 표시, 중복 신청 차단 |
| 3 | 슈퍼마스터 | SUP-SUB-002 | 승인 | 대상 업체 공지, 이력 기록(SUP-SUB-003) |
| 4 | shop02 | WEB-SUB-001 · 공지 | 재로그인 | PRO, 직원 한도 5, GRACE 해제 공지(D9) |
| 5 | shop02 | WEB-STF-004 | manager02에 WEB-SAL-001 grant | 저장 |
| 6 | manager02 | WEB-SAL-001 / WEB-SAL-002 | 재로그인 후 접근 | 001 허용·002 차단 |
| 7 | shop02 | WEB-STF-002 | 지점장 초대 시도 | 여전히 403 `TIER_LOCKED`(PREMIUM 전용) |
| 8 | shop02 | WEB-PAY-008 · WEB-SYS-005 | 결제 취소·사고 기록 | PRO 해제 확인 |
| 9 | 슈퍼마스터 | SUP-SUB-001 | BASIC으로 원복 | 대기 신청 정리 |
| 10 | manager02 | WEB-SAL-001 | 재접근 | grant 행 보존·무효(403) |

### E2E-03 초대 → 다중 샵 → 비활성: 마스터 → 미용사 → 마스터 (P1)

| # | 계정 | 화면ID | 행위 | 확인 |
|---|---|---|---|---|
| 1 | shop01 | WEB-COM-008 → WEB-STF-002 | 샵A-2로 전환 후 미용사 초대 링크 생성 | 7일 만료 |
| 2 | groomer02 | `/invite/:token` | 로그인 상태로 수락 | 샵A-2 멤버십 추가 |
| 3 | groomer02 | 샵 선택 | 재로그인 | 샵B·샵A-2 선택지 |
| 4 | groomer02 | SCR-CUS-001 · SCR-LAW-003 | 샵A-2 진입 | 샵A-2 데이터만, PREMIUM이라 사고 기록 열람 가능 / 샵B 전환 시 TIER_LOCKED |
| 5 | shop01 | WEB-STF-003 | groomer02(샵A-2) 비활성 | — |
| 6 | groomer02 | 로그인 | 재로그인 | 샵B로 바로 진입, 샵A-2 토큰 401 |
| 7 | 공통 | — | 같은 링크 재사용 | 401 |

### E2E-04 슈퍼마스터 운영 개입: 슈퍼마스터 → 신규 마스터 → 슈퍼마스터 (P0)

| # | 계정 | 화면ID | 행위 | 확인 |
|---|---|---|---|---|
| 1 | 슈퍼마스터 | SUP-BIZ-006 | 신규 업체(PRO) 등록 | 임시 비밀번호 |
| 2 | 신규 마스터 | WEB-COM-001 → WEB-COM-002 | 최초 로그인 | 비밀번호 변경 강제 → 완료 후 `/admin` |
| 3 | 슈퍼마스터 | WEB-COM-007 | 신규 샵 대행 진입 | 배너 |
| 4 | 슈퍼마스터(대행) | WEB-RES-005 · ShopCreate · `/auth/change-password` | 예약 등록(담당=신규 마스터), 샵 추가·비번 변경 시도 | 예약 성공, 금지 행위 403 |
| 5 | 슈퍼마스터 | `/auth/act-as/exit` → WEB-PLT-001 | 대행 종료 → 샵 정지 | — |
| 6 | 신규 마스터 | WEB-COM-001 | 로그인 | 403 정지 샵 |
| 7 | 슈퍼마스터 | WEB-PLT-001 → SUP-BIZ-005 | 샵 재개 → 계정 정지 → 해제 | 각 단계 로그인 결과 |
| 8 | DB | `platform_action_logs` | 조회 | 3~7 요청 전부 실제 행위자로 기록 |

### E2E-05 점검 모드 전 계정 영향 (P1, 개발 한정)

슈퍼마스터 SUP-SYS-001 ON → shop01(PC)·groomer01(모바일)·고객 앱에서 안내 문구·예상 완료 시각 표시 확인 → 슈퍼마스터는 정상 조회 → OFF 후 3개 화면 새로고침 복구. (TC-SM-025, TC-X-16 결합)

---

## 7. 이전 QA(v2.6) 이슈 회귀

출처: `ppodok_qa_report_v2.6.html/.xlsx`(2026-09-16, dark1514), `ppodok_qa_02_v2.6.html/.xlsx`(2026-09-17, shop02·manager02·groomer02). 코드 기준 사전 판정은 "예상"이며 수행 결과로 확정한다.

| REG ID | v2.6 이슈 | 원 심각도 | 코드 기준 사전 판정 | 재검증 절차 | 연결 TC |
|---|---|---|---|---|---|
| REG-01 | ISSUE-001 매출관리 전체 미구현(WEB-SAL-001~006 플레이스홀더, 사이드바 메뉴 없음) | High | 보고서의 URL(`/sales/daily`·`/sales/service`·`/sales/stylist`)은 현재 라우트에 없음 → 미정의 경로가 `AdminPlaceholder`(path `*`)로 떨어진 것으로 추정 `(확인 필요)`. 실제 경로 `/admin/sales`·`/sales/weekly`·`/sales/monthly`·`/sales/by-service`·`/sales/by-groomer`·`/sales/export`, 6화면 구현·e2e 존재. 09-17 라운드에서 shop02로 정상 확인됨 | shop01로 6화면 확인, 슈퍼마스터는 샵A 대행 후 확인 | TC-MA-015, TC-SM-013 |
| REG-02 | ISSUE-002 마일리지 메뉴 사이드바 미노출 | Med | **여전히 사이드바 그룹에 마일리지 없음**(`AdminLayout.tsx`). 경로: `/admin/mileage`(003)·`/mileage/history`(004)·`/mileage/ledger`(001)·`/mileage/adjustment/:id`(002) | 사이드바·고객 상세 등 진입 경로 확인, 설계서 v2.9 메뉴 구조와 대조 | TC-MA-024 |
| REG-03 | ISSUE-003 WEB-DSH-003 슈퍼마스터 대행 매출 "권한 없음" | Med | 대행은 등급만 통과하면 전 매출 허용(`has_sales_report_access`). v2.6 관측 업체 등급 분포가 BASIC 68·PREMIUM 2 → **등급 게이트 기대 동작 가능성 높음**. 단 문구가 "권한 없음"이라 등급 안내가 아님 → UX 이슈 후보 | PREMIUM(샵A)·BASIC(샵B) 대행 비교 | TC-SM-013 |
| REG-04 | ISSUE-004(Info) 슈퍼마스터 `/app` → `/admin` 리다이렉트로 모바일 미테스트 | Info | `ProtectedRoute` 설계 동작. 모바일은 미용사·겸직 계정으로 수행 | 리다이렉트 목적지 확인 | TC-SM-027, §4.4 |
| REG-05 | WEB-MGR-001 manager02 WEB-DSH-003 "매출 조회 권한이 없어…" | 중간 | 샵B BASIC + grant 0건 → **기대 동작** | grant 0 / grant 부여 / PRO 승격 3단계 비교 | TC-MG-003, E2E-02 |
| REG-06 | WEB-MGR-002 manager02 `/admin/sales` → 대시보드 리다이렉트·메뉴 없음 | 높음 | `salesReportKey` 라우트 가드의 **기대 동작** | 동일 | TC-MG-003·004 |
| REG-07 | WEB-MGR-003 manager02 DSH-001 매출 KPI 미표시 | 낮음 | 대시보드 매출 요약은 WEB-SAL-001 접근권 기준 → **기대 동작** | grant 001 부여 후 KPI 표시 확인 | TC-MG-004 |
| REG-08 | WEB-SCH-001 `/admin/schedule` 미구현 | 낮음 | 존재하지 않는 URL. 일정관리 = `/admin/reservations`(WEB-RES-001, 사이드바 "일정관리") → **URL 오기** | 사이드바 경로로 확인 | TC-MG-017 |
| REG-09 | WEB-BRN-001 `/admin/branch` 미구현 | 낮음 | 존재하지 않는 URL. 지점 = `/admin/shops`(WEB-COM-008), 지점 관리 = `/admin/staff/branches`(WEB-STF-005) | 사이드바 "지점"으로 확인 | TC-MA-006 |
| REG-10 | WEB-MIL-001 `/admin/mileage/logs` 미구현 | 낮음 | 존재하지 않는 URL. 실제 `/admin/mileage/ledger` | 실제 경로 확인 + REG-02 진입 경로 | TC-MA-024 |
| REG-11 | SUP-SUB-002 화면 ID 표기(설계서 대비 확인 필요) | — | 코드 SUP-SUB-002 = 구독 변경 신청 큐(정합). 설계서의 SUP-BIZ-003/004·SUP-SUB-004는 코드 화면 없음 `(확인 필요)` | 화면 ID 표기 대조 | TC-SM-021 |
| REG-12 | groomer02 `/app/reservations` = SCR-RES-003 | — | Phase 27로 주별 = SCR-RES-018, 위저드 005~008 삭제 → SCR-RES-019 | 화면 ID·탭 순서 재확인 | TC-GR-007·008 |
| REG-13 | manager02 PG 설정 접근 "권한 검토 필요" | — | 지점장 허용(`require_role` 지점장·마스터) — 정본 "담당 샵 정보 수정 ✓"과 합치 | 판정 확정 | TC-MG-010 |
| REG-14 | v2.6 미테스트: WEB-COM-002~006, WEB-RES-004·006~015, WEB-SVC-002~004, WEB-CUS-002~014, WEB-PAY-001~005·007~008, WEB-NTF-002~005, WEB-STF-002~005, WEB-SYS-002~007, SUP-TIR-002, SUP-SUB-001·003·005, SUP-BIZ-002·005·006, SUP-STA-002, SUP-NTF-002 | — | 이번 라운드 §4에 포함 | 계정별 시트에서 커버 확인 | §4 전체 |
| REG-15 | 고객 앱 SCR-CX-001~016 미테스트 | Info | 구현 ID는 APP-* 21화면(SCR-CX는 설계 ID, 대응표는 screen-id-map-app.md). SCR-CX-001/003/008/011은 "구현 상이", SCR-CX-007 갤러리 저장 스코프 제외 | APP-* 기준 수행, SCR-CX 대응 기록 | §4.5 |
| REG-16 | 09-29~30 실기기 UAT 수정 결함 5건(운영 가입 500 등) | — | 목록 상세는 22-UAT·260930 기록 `(확인 필요)` | 운영·스테이징에서 가입·로그인 스모크 | TC-MA-001, TC-CU-001·004 |

---

## 8. 범위 외 · 블로커 · 리스크

| # | 항목 | 상태(2026-09-30) | 이번 라운드 영향 | 대응 |
|---|---|---|---|---|
| B1 | SMS(SOLAPI 또는 대체) 계약 | 미계약. 운영 `PHONE_VERIFICATION_DEBUG_MODE=true` 상시(active_risk) | 실 SMS 수신·직원 초대 SMS 수신(23-UAT) ⛔. 인증은 debug_code로만 검증 | 운영에서 임의 번호 인증 가능 → 테스트 번호만 사용, 운영 데이터 오염 금지. 계약 후 DEBUG OFF + 23-UAT 재수행 |
| B2 | Firebase/FCM(Android) | `google-services.json`·EAS 자격 없음, logcat `FirebaseApp failed to initialize` | 예약 확정·거절·완료 푸시 수신, 알림 탭 딥링크(15-UAT #4/5/7/8) ⛔ | 토큰 등록 API만 검증(TC-CU-025). 대장님 Firebase 등록 후 재빌드·재측정 |
| B3 | Play Console 등록(Phase 16) | 대기 | 스토어 설치 경로 검증 불가 | EAS internal 사이드로드 빌드(69dcdb05)로 대체 |
| B4 | PG 실연동(토스) | 미계약, `ManualPaymentGateway` | 실 카드 승인·취소 연동, PG 웹훅 ⛔ | 수기 결제·취소 금액 정합만 검증 |
| B5 | 카카오 알림톡 실채널 | 솔라피 비즈니스 인증·템플릿 승인 전 | 실제 카카오 수신 ⛔ | 발송 이력·상태(SKIPPED_CONSENT/FAILED/SENT 기록)만 확인 |
| B6 | 소셜 로그인 | 실기기 카카오·네이버 검증 완료(09-21) | 개발 환경은 mock | 개발 mock + 실기기 1회 회귀 |
| B7 | 운영 테스트 샵·샵 소속 계정 없음 | 운영에는 고객 UAT 계정만 | 운영 계정 릴레이(E2E) 불가 | 운영은 스모크만. 필요 시 대장님 승인으로 SUP-BIZ-006 테스트 업체 생성 후 종료 시 정지·정리 |
| B8 | 알림장 사진 기기 저장(SCR-CX-007) | 스코프 제외 결정 | — | 범위 외 명시 |
| R1 | 공유 mysql-test 오염 | pytest·e2e 상호오염 미해결(TODO) | 자동화 기준선 흔들림, 수동 TC 데이터 충돌 | 수동 라운드 전 초기화 + 약관 시드, 수동 TC는 고유 접두어 데이터 사용 |
| R2 | 샵B 직원 한도 GRACE(D9) | 로그인 시 유예 시작, 만료 후 job이 계정 비활성화 | 라운드 도중 manager02·groomer 계정 비활성 가능 | P-06 기록, 필요 시 E2E-02로 PRO 승격 또는 시드 재구성 |
| R3 | 시드 예약 날짜 노후화 | 재실행 시 skip | "예정" 예약 부족 | P-09, TC 내 신규 예약 |
| R4 | 정본-코드 불일치 D1~D7 | 기획 확인 전 | Pass/Fail 판정 기준 모호 | 관측값 기록 + ⚠️/Fail을 기획(yong)·대장님 확인 후 확정. 이슈는 "정본 개정" 또는 "코드 수정"으로 분기 |
| R5 | 테스트 계정 비밀번호 `123456` | 운영 정책 위반(개발 전용) | 운영 DB 시드 금지 | 시드 가드(P-01) 확인, 운영 반영 전 제거 |
| R6 | 기존 불안정 테스트 | `test_platform_act_as.py` 마커 테스트(WINDOWS #19), KST 자정 직후 실패 | 자동화 열 신뢰도 | 해당 항목은 수동 확인 병행 |

---

## 9. 일정 · 결과 보고

### 9.1 수행 순서와 예상 소요

| 순서 | 대상 | 이유 | 예상 |
|---|---|---|---|
| 0 | 사전 준비 P-01~P-14 + 자동 회귀 기준선 | 데이터·등급 상태가 모든 판정의 전제 | 0.5일 |
| 1 | 마스터(shop01 → shop02) | 초대·grant·가입코드 등 다른 계정 TC의 선행 데이터 생성 | 1일 |
| 2 | 지점장(manager01 → manager02) | grant·등급 판정, D1·D2 확인 | 0.5일 |
| 3 | 미용사(groomer01 → groomer02) | 모바일·실기기, 오프라인 초안 | 1일 |
| 4 | 고객(CUST-A/B/S, 실기기) | 연결 승인은 지점장·마스터 계정 병행 | 1일 |
| 5 | 슈퍼마스터 | 정지·등급 변경·점검 모드는 다른 계정 상태를 바꾸므로 마지막에 | 0.5일 |
| 6 | 교차(TC-X) + E2E-01~05 | 계정 릴레이 | 1일 |
| 7 | 회귀(REG) + 이슈 정리·보고 | — | 0.5일 |
| | **합계** | | **약 6일(1인)** |

- 파괴적 TC(TC-SM-004·019·020·025, E2E-02·04·05)는 해당 단계 마지막에 수행하고 즉시 원복한다.
- 판정 보류(⚠️) 항목은 매일 종료 시 목록으로 공유한다.

### 9.2 결과 보고 양식 (yong xlsx 구조 계승)

파일명: `ppodok_qa_account_v2.9.xlsx` (+ 동일 내용 html 요약). 시트 구성:

| 시트 | 내용 |
|---|---|
| 테스트 요약 | 일자·환경·빌드(커밋 해시)·설계서 버전, 역할별 요약표(역할 / 계정 / 접근 경로 / 총 TC / ✅ / ❌ / ⚠️ / ⛔ / 비고), 주요 이슈 요약 |
| 슈퍼마스터_yuyunsu87 | §4.1 |
| 마스터_shop01 / 마스터_shop02 | §4.2 (계정별 분리 — v2.6 `샵마스터_shop02` 계승) |
| 지점장_manager01 / 지점장_manager02 | §4.3 |
| 미용사_groomer01 / 미용사_groomer02 | §4.4 |
| 고객_CUST-A 등 | §4.5 |
| 교차검증 | §5.1 매트릭스 실측값 + §5.2 |
| E2E | §6 단계별 결과 |
| 회귀 | §7 REG 판정 |
| 이슈 상세 | 이슈 ID / 심각도 / 역할 / 화면ID / URL·API / 재현 절차 / 기대 / 실제 / 근거(정본·코드) / 권고 / 스크린샷 |

계정별 시트 열(v2.6 계승 + TC ID): `TC ID | 화면 ID | 화면명 | URL | 상태 | 이슈 ID | 관측 내용 | 담당 | 우선순위 | 비고`.
이슈 ID 규칙: `ISSUE-<역할약어>-<번호>`(SM/MA/MG/GR/CU/X), 정본 불일치는 `DOC-<번호>`.

### 9.3 이슈 심각도 기준

| 심각도 | 기준 | 예 |
|---|---|---|
| Critical | 권한 우회·타 테넌트 데이터 노출·개인정보 유출·결제 금액 오류·계정 탈취 가능 | 타 샵 고객 조회, 미용사의 결제 취소 성공 |
| High | 역할의 주 업무 불가, 정본 권한과 반대로 동작(허용/거부 뒤바뀜) | 지점장이 지점장 초대 가능(D1), 가입 500 |
| Medium | 우회 가능한 기능 오류, 잘못된 안내로 오판 유도 | 등급 잠금을 "권한 없음"으로 안내(REG-03) |
| Low | 표시·문구·정렬 | 배지 색, 라벨 |
| Info | 설계·정본 판단 필요, 결함 아님 | 설계 차이 기록, 범위 외 |

yong 표기와 대응: High=높음, Med=중간, Low=낮음.

---

## 참조

| ID | 경로 (저장소 `/home/user/ppodog`) |
|---|---|
| R1 | `docs/account-groups-and-permissions.md` |
| R2 | `docs/test-accounts.md` |
| R3 | `backend/app/db/seed_test_accounts.py` |
| R4 | `docs/screen-id-map.md`, `docs/screen-id-map-admin.md`, `docs/screen-id-map-front.md`, `docs/screen-id-map-app.md` |
| R5 | `docs/test/ppodok_qa_report_v2.6.html`, `docs/test/ppodok_qa_report_v2.6.xlsx`, `docs/test/ppodok_qa_02_v2.6.html`, `docs/test/ppodok_qa_02_v2.6.xlsx` |
| R6 | `backend/app/core/data_scope.py`, `backend/app/core/deps.py`, `backend/app/core/phone_verification.py`, `backend/app/main.py`(플랫폼 감사 미들웨어) |
| R7 | `backend/app/api/v1/auth.py`, `social_auth.py`, `customer_auth.py`, `customer_social_auth.py`, `staff_invites.py`, `staff.py`, `staff_availability.py`, `shops.py`, `shop.py`, `platform.py`, `sales.py`, `subscription.py`, `payments.py`, `incidents.py`, `customers.py`, `customer_links.py`, `customer_shops.py`, `customer_reservations.py`, `customer_history.py`, `customer_pets.py`, `customer_account.py`, `reservations.py`, `services.py` |
| R8 | `backend/app/services/subscriptions.py`(`TIER_FEATURES`, `DEFAULT_TIER_LIMITS`, staff quota GRACE) |
| R9 | `backend/tests/` (§4 자동화 열의 파일) |
| R10 | `frontend/src/shared/auth/ProtectedRoute.tsx`, `frontend/src/shared/layouts/AdminLayout.tsx`, `frontend/src/admin/routes.tsx`, `frontend/src/app/routes.tsx`, `frontend/src/shared/staff/StaffInviteContent.tsx`, `frontend/e2e/{admin,app}/*.spec.ts` |
| R11 | `customer-app/app/`(auth·onboarding·tabs·modals 라우트), `customer-app/app.config.ts`, `customer-app/eas.json` |
| R12 | `.planning/STATE.md`(Deferred Items, Operator Next Steps), `.planning/MILESTONES.md`, `.planning/REQUIREMENTS.md`(RESUX-01~07), `docs/TODO.md` |
| R13 | `docs/multi-tenancy-signup-design.md`, `docs/social-login-setup.md`, `docs/superpowers/plans/2026-08-31-auth-invite-details.md`, `docs/offline_grooming_draft_flow.md` |
