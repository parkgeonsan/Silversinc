# SilverSync 보안 & 컴플라이언스 가이드

> 개인정보보호, 의료정보 보안, 법적 규제 준수 설계
> 작성일: 2026-02-06 | 버전: v1.0

---

## 1. 관련 법규 분석

### 1.1 개인정보보호법 (PIPA)

SilverSync가 수집하는 데이터의 대부분이 개인정보에 해당하며, 건강/의료 정보는 민감정보로 분류됩니다.

| 원칙 | 적용 사항 | SilverSync 조치 |
|------|----------|----------------|
| **수집 동의** | 개인정보 수집 전 명시적 동의 필요 | 앱 가입 시 동의서 제공, 별도 민감정보 동의 |
| **목적 제한** | 수집 목적 내에서만 이용 | 시설-보호자 정보 동기화 목적으로 한정 |
| **최소 수집** | 필요한 최소한의 정보만 수집 | 선택 항목과 필수 항목 분리 |
| **보유 기간** | 목적 달성 시 지체 없이 파기 | 퇴소 후 3년 보관 → 자동 파기 |
| **파기 의무** | 보유 기간 경과 시 파기 | pg_cron 기반 자동 파기 스케줄 |
| **안전성 확보** | 기술적/관리적 보호조치 | 암호화, RLS, 접근 로그, 보안 교육 |

#### 민감정보 처리 특별 규정

```
민감정보 해당 항목:
├── 건강 상태 (컨디션, 기분, 식욕 기록)
├── 질환 정보 (장기요양 등급, 의료 특이사항)
├── 복약 정보 (처방약, 복약 기록, 사진)
└── 건강 측정 데이터 (혈압, 혈당 등 - Phase 4)

→ 별도의 명시적 동의 필요 (일반 개인정보와 분리)
→ 암호화 저장 필수 (AES-256)
→ 접근 로그 필수 기록
```

#### 개인정보처리방침 필수 항목

1. 개인정보 처리 목적
2. 처리하는 개인정보의 항목
3. 개인정보의 처리 및 보유 기간
4. 개인정보의 제3자 제공 (해당 시)
5. 개인정보처리의 위탁 (Supabase, Google, Naver)
6. 정보주체의 권리·의무 및 행사방법
7. 개인정보의 파기 절차 및 방법
8. 개인정보의 안전성 확보 조치
9. 개인정보 보호책임자

### 1.2 노인장기요양보험법

| 조항 | 의무 | SilverSync 조치 |
|------|------|----------------|
| 시설 기록 의무 | 이용자 서비스 기록 보관 | 일과보고, 복약기록 5년 보관 |
| 이용자 권리 | 서비스 내용 열람/고지 | 보호자 앱에서 전체 이력 조회 가능 |
| 급여 기록 | 급여 관련 데이터 보관 | 급여 계산 데이터 5년 보관 |
| 비밀 보장 | 이용자 정보 비밀 유지 | RLS + 암호화 + 접근 제어 |

### 1.3 의료법

| 항목 | 기준 | SilverSync 적용 |
|------|------|-----------------|
| 건강 정보 | 의료 행위에 의한 정보는 의료법 적용 | 복약 기록은 "투약 보조" 수준 (의료 행위 아님) |
| 처방전 정보 | 의료기관 발급 처방전 | 약 이름/용량만 기록, 처방전 원본은 미보관 |
| 건강 측정 | 자가 측정 데이터 | 의료 행위 아닌 건강 모니터링 |

> **주의**: SilverSync는 의료 행위를 수행하지 않으며, 복약 "보조 기록"과 건강 "모니터링" 수준으로 운영됩니다. 진단, 처방, 치료 행위는 포함되지 않습니다.

### 1.4 위치정보보호법

| 항목 | 요건 | SilverSync 조치 |
|------|------|----------------|
| **사업자 신고** | 위치정보를 수집하는 사업자는 신고 필요 | 위치기반서비스사업자 신고 (방통위) |
| **수집 동의** | 위치정보 수집 전 별도 동의 | 운전기사: 근무 중 위치 수집 동의 / 보호자: 차량 위치 조회 동의 |
| **보유 기간** | 서비스 제공 목적 달성 후 즉시 파기 | 차량 위치: 실시간 표시 후 1시간 뒤 삭제 / 운행 기록: 90일 보관 |
| **접근 제한** | 위치정보 접근 권한 최소화 | 관련 보호자만 해당 차량 위치 조회 가능 (RLS) |
| **이용 내역** | 위치정보 이용/제공 내역 통보 | 월 1회 위치정보 이용 내역 앱 내 조회 기능 |

### 1.5 정보통신망법

| 의무 | 내용 | SilverSync 조치 |
|------|------|----------------|
| 기술적 보호조치 | 접근 통제, 암호화, 로그 기록 | Supabase RLS + 암호화 + audit_logs |
| 관리적 보호조치 | 보안 교육, 접근 권한 관리 | 연 2회 보안 교육, 권한 최소화 원칙 |
| 유출 통지 | 유출 인지 후 72시간 내 통지 | 사고 대응 절차 수립 (섹션 7 참조) |

---

## 2. 데이터 분류 및 보호 등급

| 등급 | 분류 | 데이터 예시 | 암호화 | 접근 제어 | 로그 기록 | 보유 기간 |
|------|------|------------|--------|-----------|-----------|-----------|
| **1등급** (최고) | 의료/건강 민감정보 | 복약기록, 건강상태, 질환정보, 의료특이사항 | AES-256 필드 암호화 | RLS + 역할 제한 | 전수 기록 | 퇴소 후 5년 |
| **2등급** (높음) | 개인식별정보 | 이름, 생년월일, 연락처, 주소, 사진 | DB 암호화 (at rest) | RLS | 접근 기록 | 퇴소 후 3년 |
| **3등급** (중간) | 위치정보 | 차량 위치, 픽업 주소, 운행 경로 | 전송 암호화 (TLS) | RLS + 시간 제한 | 접근 기록 | 실시간 후 1시간 삭제 |
| **4등급** (보통) | 서비스 이용정보 | 식단, 프로그램, 공지사항, 채팅 | 전송 암호화 (TLS) | RLS | 선택적 | 서비스 이용 중 |

---

## 3. 기술적 보안 조치

### 3.1 인증 및 접근 제어

#### 인증 흐름

```
[보호자 로그인]
전화번호 입력 → OTP SMS 발송 → OTP 입력 → Supabase Auth 세션 생성 → JWT 토큰 발급
                                                                         │
                                                    Access Token (1시간) ──┤
                                                    Refresh Token (30일) ──┘

[시설 직원 로그인]
이메일/비밀번호 → Supabase Auth 인증 → JWT 토큰 발급
                                        │
                        Access Token (1시간) ──┤
                        Refresh Token (7일) ───┘ ← 보호자보다 짧은 갱신 주기
```

#### 역할 기반 접근 제어 (RBAC)

| 역할 | residents 테이블 | medication_logs | vehicle_locations | messages | facilities |
|------|:---:|:---:|:---:|:---:|:---:|
| system_admin | CRUD (전체) | CRUD (전체) | R (전체) | R (전체) | CRUD (전체) |
| facility_admin | CRUD (자기 시설) | CRUD (자기 시설) | R (자기 시설) | CRUD (자기 시설) | RU (자기 시설) |
| facility_staff | RU (담당) | CRU (담당) | R (자기 시설) | CR (담당 보호자) | R (자기 시설) |
| facility_driver | R (배차) | - | CRU (자기 차량) | - | R (자기 시설) |
| guardian | R (자기 어르신) | R (자기 어르신) | R (자기 어르신 차량) | CR (자기 대화) | R (연결된 시설) |

#### RLS 정책 예시

```sql
-- 보호자는 자기 어르신의 복약 기록만 조회 가능
CREATE POLICY "guardian_medication_read" ON medication_logs
  FOR SELECT
  TO authenticated
  USING (
    resident_id IN (
      SELECT rg.resident_id
      FROM resident_guardians rg
      JOIN guardians g ON g.id = rg.guardian_id
      WHERE g.user_id = auth.uid()
    )
  );

-- 시설 직원은 자기 시설 어르신의 복약만 기록 가능
CREATE POLICY "staff_medication_insert" ON medication_logs
  FOR INSERT
  TO authenticated
  WITH CHECK (
    resident_id IN (
      SELECT r.id FROM residents r
      JOIN staff s ON s.facility_id = r.facility_id
      WHERE s.user_id = auth.uid()
    )
  );
```

#### 세션 관리

| 항목 | 정책 |
|------|------|
| Access Token 만료 | 1시간 |
| Refresh Token 만료 | 보호자 30일 / 직원 7일 |
| 자동 로그아웃 | 앱 백그라운드 30분 시 재인증 요구 (민감 화면) |
| 다중 기기 | 보호자: 허용 / 직원: 마지막 기기만 활성 |
| 비밀번호 실패 | 5회 연속 실패 시 15분 잠금 |

### 3.2 데이터 암호화

| 계층 | 방식 | 대상 | 구현 |
|------|------|------|------|
| **전송 중** | TLS 1.3 | 모든 API 통신 | Supabase 기본 제공 |
| **저장 시 (DB)** | AES-256 (at rest) | PostgreSQL 전체 | Supabase 기본 제공 |
| **필드 암호화** | AES-256 + Supabase Vault | medical_notes, 건강 기록 | pgcrypto 확장 + Vault |
| **로컬 저장** | AES-256 | 오프라인 캐시 데이터 | flutter_secure_storage / encrypted_shared_preferences |
| **파일 저장** | 서버 측 암호화 | Storage 파일 (사진, 문서) | Supabase Storage 기본 |

#### 필드 레벨 암호화 예시

```sql
-- Supabase Vault를 활용한 민감 필드 암호화
CREATE OR REPLACE FUNCTION encrypt_medical_notes()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.medical_notes IS NOT NULL THEN
    NEW.medical_notes = vault.encrypt(
      NEW.medical_notes,
      vault.get_key('medical_data_key')
    );
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER encrypt_medical_notes_trigger
  BEFORE INSERT OR UPDATE ON residents
  FOR EACH ROW EXECUTE FUNCTION encrypt_medical_notes();
```

### 3.3 네트워크 보안

| 항목 | 조치 | 설정 |
|------|------|------|
| Rate Limiting | API 호출 횟수 제한 | 분당 100회 (인증 API: 분당 10회) |
| DDoS 방어 | Supabase CDN + Rate Limit | Supabase 기본 제공 |
| CORS | 허용 도메인 제한 | 앱 도메인만 허용 |
| API 키 보호 | anon key: 클라이언트용 (RLS 적용) | service_role key: 서버 전용 (Edge Function만) |
| 입력 검증 | SQL Injection, XSS 방지 | Supabase SDK 파라미터 바인딩 + 입력 검증 |

### 3.4 앱 보안

| 항목 | Android | iOS | 구현 |
|------|---------|-----|------|
| 코드 난독화 | ProGuard / R8 | Bitcode | build.gradle / Xcode 설정 |
| 루팅/탈옥 감지 | flutter_jailbreak_detection | 동일 | 감지 시 경고 + 민감 기능 차단 |
| SSL Pinning | OkHttp CertificatePinner | NSAppTransportSecurity | Supabase 인증서 고정 |
| 스크린캡처 방지 | FLAG_SECURE | 없음 (OS 제한) | 복약 사진, 건강 기록 화면 |
| 안전한 저장소 | EncryptedSharedPreferences | Keychain | flutter_secure_storage |
| 디버그 감지 | isDebuggerConnected() | ptrace | Release 빌드에서 차단 |

---

## 4. 관리적 보안 조치

### 4.1 개인정보 처리 조직

| 직책 | 역할 | 담당자 |
|------|------|--------|
| 개인정보 보호책임자 (CPO) | 개인정보 보호 총괄 | 대표이사 (초기) |
| 개인정보 취급자 | 시스템 운영, DB 관리 | 개발팀장 |
| 보안 담당자 | 보안 점검, 사고 대응 | CTO / 외부 보안 업체 |

### 4.2 접근 권한 관리

```
접근 권한 원칙: 최소 권한 (Principle of Least Privilege)

├── 개발 환경: 개발자 전원 접근 가능 (테스트 데이터)
├── 스테이징 환경: 개발팀 + QA팀 (마스킹된 실제 데이터)
├── 운영 환경:
│   ├── DB 직접 접근: CTO + DBA (2인)
│   ├── 서버 접근: CTO + DevOps (2인)
│   ├── Supabase Dashboard: 관리자 (3인 이내)
│   └── 모니터링: 전 개발팀
└── 로그 접근: CPO + 보안 담당자
```

### 4.3 수탁업체 관리

| 수탁업체 | 위탁 내용 | 데이터 범위 | 서버 위치 | 보호 조치 |
|---------|----------|------------|-----------|-----------|
| **Supabase** | DB, 인증, 스토리지, 서버리스 | 전체 서비스 데이터 | AWS (서울/도쿄) | SOC 2 Type II, HIPAA |
| **Google (FCM)** | 푸시 알림 발송 | 기기 토큰, 알림 내용 | Google Cloud | ISO 27001 |
| **네이버** | 지도 API | 위치 좌표 | NCP (한국) | ISMS 인증 |
| **NHN** | SMS 발송 | 전화번호, 인증 코드 | NHN Cloud (한국) | ISMS 인증 |

---

## 5. 개인정보 처리 생명주기

```
수집 ──────► 이용 ──────► 보관 ──────► 파기
 │            │            │            │
 ├─ 동의 획득  ├─ 목적 내 이용 ├─ 접근 제어  ├─ 복구 불가 삭제
 ├─ 최소 수집  ├─ 접근 로그   ├─ 암호화    ├─ 로그 기록
 └─ 고지 제공  └─ 제3자 제공  └─ 기간 관리  └─ 파기 확인서
               시 재동의
```

### 단계별 상세

| 단계 | 조치 | 기술적 구현 |
|------|------|------------|
| **수집** | 필수/선택 분리 동의, 수집 목적 고지, 보유기간 명시 | 앱 내 동의 화면 + consent_logs 테이블 |
| **이용** | 수집 목적 내에서만 이용, 접근 권한 제한 | RLS 정책, 역할 기반 접근 제어 |
| **보관** | 암호화 저장, 접근 로그 기록, 보유기간 관리 | AES-256, audit_logs, pg_cron |
| **파기** | 보유기간 경과 시 자동 파기, 파기 로그 기록 | pg_cron + 자동 DELETE + destruction_logs |

### 보유 기간 정책

| 데이터 유형 | 보유 기간 | 파기 방법 | 법적 근거 |
|------------|----------|-----------|-----------|
| 어르신 기본정보 | 퇴소 후 3년 | DB 삭제 + 백업 파기 | 개인정보보호법 |
| 건강/의료 기록 | 퇴소 후 5년 | DB 삭제 + 암호화 키 폐기 | 노인장기요양보험법 |
| 위치 정보 (실시간) | 실시간 표시 후 1시간 | 자동 DELETE | 위치정보보호법 |
| 운행 기록 | 90일 | 자동 DELETE | 내부 정책 |
| 채팅 메시지 | 퇴소 후 1년 | DB 삭제 | 내부 정책 |
| 사진/파일 | 퇴소 후 1년 | Storage 삭제 | 내부 정책 |
| 접근 로그 | 3년 | 자동 DELETE | 정보통신망법 |

---

## 6. 동의 관리

### 6.1 동의 항목 분류

| 구분 | 항목 | 필수/선택 | 동의 거부 시 |
|------|------|----------|-------------|
| **서비스 이용 약관** | 서비스 전체 이용 조건 | 필수 | 서비스 이용 불가 |
| **개인정보 수집·이용** | 이름, 연락처, 생년월일 등 | 필수 | 서비스 이용 불가 |
| **민감정보 처리** | 건강상태, 복약기록, 질환정보 | 필수 | 건강 관련 기능 이용 불가 |
| **위치정보 수집** | 차량 위치 추적, 운행 기록 | 필수 (운전기사) / 선택 (보호자) | 송영 알리미 이용 불가 |
| **마케팅 수신** | 프로모션, 이벤트 알림 | 선택 | 마케팅 알림 미수신 |
| **제3자 제공** | 수탁업체 데이터 처리 | 필수 | 서비스 이용 불가 |

### 6.2 동의 관리 시스템

```sql
CREATE TABLE consent_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  consent_type TEXT NOT NULL,          -- terms, privacy, sensitive, location, marketing
  consent_version TEXT NOT NULL,       -- 약관 버전 (v1.0, v1.1 등)
  is_agreed BOOLEAN NOT NULL,
  agreed_at TIMESTAMPTZ,
  withdrawn_at TIMESTAMPTZ,            -- 철회 시 기록
  ip_address TEXT,
  device_info TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### 6.3 동의 철회 절차

```
보호자 앱 > 설정 > 개인정보 관리 > 동의 철회
    │
    ├── 선택 동의 (마케팅 등): 즉시 철회 가능
    │
    ├── 필수 동의 (개인정보 수집 등):
    │   → 철회 = 회원 탈퇴 안내
    │   → 탈퇴 전 데이터 다운로드 기회 제공 (30일)
    │   → 탈퇴 시 개인정보 즉시 삭제 (법적 보관 의무 제외)
    │
    └── 민감정보 동의:
        → 철회 시 건강/복약 관련 기능 비활성화
        → 기존 기록은 보유기간까지 보관 후 파기
```

---

## 7. 사고 대응 계획

### 7.1 개인정보 유출 사고 대응 절차

```
사고 인지 (0시간)
    │
    ├── 1단계: 초기 대응 (0~2시간)
    │   ├── 유출 범위 파악
    │   ├── 추가 유출 차단 (접근 차단, 키 교체 등)
    │   ├── 사고 대응팀 소집 (CPO + CTO + 법무)
    │   └── 증거 보전 (로그 스냅샷)
    │
    ├── 2단계: 통지 (24~72시간)
    │   ├── 정보주체 통지 (72시간 이내)
    │   │   ├── 유출 항목
    │   │   ├── 유출 시점/경위
    │   │   ├── 피해 최소화 방안
    │   │   └── 담당부서 연락처
    │   ├── 개인정보보호위원회 신고
    │   └── 1,000명 이상 시 한국인터넷진흥원(KISA) 신고
    │
    ├── 3단계: 피해 구제 (1~2주)
    │   ├── 피해자 지원 (비밀번호 변경 안내 등)
    │   ├── 추가 피해 모니터링
    │   └── 보상 방안 수립 (필요 시)
    │
    └── 4단계: 재발 방지 (1~3개월)
        ├── 원인 분석 보고서
        ├── 보안 취약점 보완
        ├── 보안 교육 강화
        └── 대응 절차 개선
```

### 7.2 사고 유형별 대응

| 유형 | 즉시 조치 | 통지 범위 |
|------|----------|----------|
| DB 유출 | DB 접근 키 교체, IP 차단, RLS 점검 | 전체 사용자 |
| API 키 노출 | 키 재발급, 이전 키 무효화 | 영향받는 사용자 |
| 앱 취약점 | 긴급 패치 배포, 강제 업데이트 | 전체 사용자 |
| 내부자 유출 | 계정 잠금, 접근 로그 분석, 법적 조치 | 영향받는 사용자 |
| 수탁업체 사고 | 수탁업체 사고 대응 확인, 데이터 영향 파악 | 영향받는 사용자 |

---

## 8. 감사 및 모니터링

### 8.1 접근 로그 시스템

```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  user_id UUID,
  action TEXT NOT NULL,                -- SELECT, INSERT, UPDATE, DELETE
  table_name TEXT NOT NULL,
  record_id TEXT,
  old_values JSONB,                    -- 변경 전 값 (UPDATE/DELETE)
  new_values JSONB,                    -- 변경 후 값 (INSERT/UPDATE)
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 민감 테이블 자동 감사 로그 트리거
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_logs (user_id, action, table_name, record_id, old_values, new_values)
  VALUES (
    auth.uid(),
    TG_OP,
    TG_TABLE_NAME,
    COALESCE(NEW.id::text, OLD.id::text),
    CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN to_jsonb(OLD) END,
    CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN to_jsonb(NEW) END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- 민감 테이블에 트리거 적용
CREATE TRIGGER audit_residents AFTER INSERT OR UPDATE OR DELETE ON residents
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
CREATE TRIGGER audit_medication_logs AFTER INSERT OR UPDATE OR DELETE ON medication_logs
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
CREATE TRIGGER audit_health_records AFTER INSERT OR UPDATE OR DELETE ON health_records
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### 8.2 이상 접근 탐지

| 탐지 규칙 | 조건 | 조치 |
|----------|------|------|
| 대량 조회 | 1분 내 100건 이상 조회 | 자동 차단 + 관리자 알림 |
| 비정상 시간 접근 | 02:00~06:00 접근 (직원) | 관리자 알림 |
| 타 시설 데이터 접근 시도 | RLS 위반 접근 시도 | 로그 기록 + 반복 시 차단 |
| 다중 기기 동시 접근 | 직원 계정 3대 이상 | 전체 세션 종료 + 재인증 |
| 연속 인증 실패 | 5회 연속 실패 | 15분 잠금 + 관리자 알림 |

### 8.3 정기 보안 점검

| 점검 항목 | 주기 | 담당 | 내용 |
|----------|------|------|------|
| RLS 정책 점검 | 월 1회 | 개발팀 | 모든 RLS 정책 테스트 |
| 접근 권한 검토 | 분기 1회 | CPO | 불필요한 권한 회수 |
| 취약점 점검 | 분기 1회 | 외부 업체 | OWASP Top 10 점검 |
| 모의 침투 테스트 | 연 1회 | 외부 업체 | 전체 시스템 침투 테스트 |
| 개인정보 영향평가 | 연 1회 | CPO + 법무 | 법적 컴플라이언스 검토 |
| 수탁업체 점검 | 연 1회 | CPO | 수탁업체 보안 수준 확인 |

---

## 부록: 보안 체크리스트

### 출시 전 필수 점검

- [ ] 개인정보처리방침 작성 및 앱 내 게시
- [ ] 서비스 이용약관 작성
- [ ] 위치기반서비스사업자 신고 (방통위)
- [ ] 개인정보 수집 동의 화면 구현 (필수/선택 분리)
- [ ] 민감정보 별도 동의 화면 구현
- [ ] RLS 정책 전체 테이블 적용 완료
- [ ] 민감 필드 암호화 적용 (medical_notes 등)
- [ ] audit_logs 트리거 적용 완료
- [ ] SSL Pinning 적용
- [ ] 코드 난독화 적용 (Release 빌드)
- [ ] 자동 파기 스케줄 설정 (pg_cron)
- [ ] 사고 대응 연락망 구성
- [ ] 수탁업체 계약서 검토 (개인정보 위탁)
- [ ] 앱 보안 점검 (OWASP Mobile Top 10)

---

*문서 끝 - SilverSync Security & Compliance Guide v1.0*
