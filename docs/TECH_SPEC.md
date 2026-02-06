# SilverSync(실버싱크) 기술 스펙 문서

> 노인장기요양 시설 · 보호자 간의 정보 동기화 솔루션
> 작성일: 2026-02-06 | 버전: v1.0

---

## 목차

1. [시스템 아키텍처 개요](#1-시스템-아키텍처-개요)
2. [기술 스택](#2-기술-스택)
3. [사용자 역할 및 권한](#3-사용자-역할-및-권한)
4. [핵심 기능 상세 스펙](#4-핵심-기능-상세-스펙)
5. [데이터베이스 스키마](#5-데이터베이스-스키마)
6. [API 설계](#6-api-설계)
7. [실시간 통신 설계](#7-실시간-통신-설계)
8. [인프라 및 배포](#8-인프라-및-배포)
9. [보안 설계](#9-보안-설계)
10. [단계별 로드맵](#10-단계별-로드맵)

---

## 1. 시스템 아키텍처 개요

```
┌─────────────────────────────────────────────────────────┐
│                    클라이언트 레이어                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ 시설 직원 앱  │  │  보호자 앱    │  │ 관리자 웹     │   │
│  │  (Flutter)   │  │  (Flutter)   │  │  (Flutter Web)│   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                 │            │
└─────────┼─────────────────┼─────────────────┼────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────┐
│                   Supabase 백엔드                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Auth     │  │ Realtime │  │ Storage  │              │
│  │ (인증)   │  │ (실시간)  │  │ (파일)   │              │
│  └──────────┘  └──────────┘  └──────────┘              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │PostgreSQL│  │ Edge     │  │ Push     │              │
│  │ (DB)     │  │ Functions│  │ (FCM)    │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                   외부 서비스 연동                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │네이버 지도│  │ FCM 푸시 │  │ SMS 인증 │              │
│  │   API    │  │  알림    │  │  (NHN)   │              │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

### 핵심 설계 원칙
- **모노레포 구조**: 시설 앱 / 보호자 앱 / 관리자 웹을 하나의 Flutter 프로젝트에서 flavor로 관리
- **오프라인 퍼스트**: 복약 기록 등 핵심 데이터는 로컬 캐시 후 동기화
- **실시간 우선**: 송영 위치, 메시지, 알림은 Supabase Realtime으로 즉시 반영
- **개인정보 보호**: 의료/건강 데이터 암호화, 최소 수집 원칙 (개인정보보호법 + 의료법 준수)

---

## 2. 기술 스택

### 프론트엔드 (모바일 앱)
| 항목 | 기술 | 비고 |
|------|------|------|
| 프레임워크 | **Flutter 3.x** | iOS + Android + Web |
| 상태관리 | **Riverpod 2.x** | Provider 대비 테스트 용이, 코드 생성 지원 |
| 라우팅 | **GoRouter** | 딥링크, 인증 리다이렉트 처리 |
| 로컬 DB | **Hive / Isar** | 오프라인 캐시, 복약 기록 임시 저장 |
| 지도 | **flutter_naver_map** | 네이버 지도 Flutter SDK |
| 푸시 알림 | **firebase_messaging** | FCM 기반 푸시 알림 |
| 카메라/갤러리 | **image_picker** | 어르신 활동 사진 촬영/업로드 |
| 차트 | **fl_chart** | 건강 데이터 시각화 |

### 백엔드
| 항목 | 기술 | 비고 |
|------|------|------|
| BaaS | **Supabase** | Auth, DB, Realtime, Storage, Edge Functions |
| DB | **PostgreSQL 15** | Supabase 내장, RLS(Row Level Security) 활용 |
| 실시간 | **Supabase Realtime** | 위치 추적, 메시지, 알림 실시간 전송 |
| 파일 저장 | **Supabase Storage** | 어르신 사진, 문서, 식단표 이미지 |
| 서버리스 | **Supabase Edge Functions** (Deno) | 복잡 로직, 외부 API 호출, 정산 계산 |
| 인증 | **Supabase Auth** | 이메일/전화번호 인증, 소셜 로그인 |

### 외부 서비스
| 서비스 | 용도 | 비고 |
|--------|------|------|
| **네이버 지도 API** | 송영 차량 위치 표시, 경로 안내 | Mobile Dynamic Map |
| **Firebase Cloud Messaging** | 푸시 알림 | 송영 도착, 복약 알림, 메시지 알림 |
| **NHN Cloud SMS** | 본인 인증, 긴급 알림 | 보호자 전화번호 인증 |
| **카카오 알림톡** | 정기 알림 | 일과 보고, 식단표 발송 (선택) |

### 개발 도구
| 항목 | 기술 |
|------|------|
| 버전 관리 | Git + GitHub |
| CI/CD | GitHub Actions |
| 코드 품질 | flutter_lints, dart analyze |
| 테스트 | flutter_test, integration_test |
| 디자인 | Figma |

---

## 3. 사용자 역할 및 권한

### 역할 정의

| 역할 | 코드 | 설명 | 주요 권한 |
|------|------|------|-----------|
| **시스템 관리자** | `system_admin` | 실버싱크 운영진 | 전체 시스템 관리, 시설 등록 승인 |
| **시설 관리자** | `facility_admin` | 센터장/원장 | 시설 정보 관리, 직원 관리, 전체 어르신 조회 |
| **시설 직원** | `facility_staff` | 사회복지사/요양보호사 | 담당 어르신 관리, 복약/활동/송영 기록 |
| **운전기사** | `facility_driver` | 송영 차량 기사 | 송영 스케줄 확인, 위치 공유, 탑승 체크 |
| **보호자** | `guardian` | 어르신 가족 | 담당 어르신 정보 조회, 소통, 알림 수신 |

### 앱 구분

```
silversync/
├── lib/
│   ├── app/
│   │   ├── facility/      ← 시설용 앱 (관리자 + 직원 + 운전기사)
│   │   ├── guardian/       ← 보호자용 앱
│   │   └── admin/          ← 시스템 관리자 웹
│   ├── core/               ← 공통 모듈 (인증, 네트워크, 모델 등)
│   ├── features/           ← 기능별 모듈
│   └── shared/             ← 공유 위젯, 유틸리티
```

- **시설 앱**: 시설 관리자 + 직원 + 운전기사가 하나의 앱에서 역할별 화면 분기
- **보호자 앱**: 보호자 전용 심플 UI (어르신 상태 조회, 소통 중심)
- **관리자 웹**: Flutter Web으로 시스템 관리자 대시보드

---

## 4. 핵심 기능 상세 스펙

### 4.1 송영 알리미 (Transportation Tracker)

#### 개요
어르신의 시설 등/하원 차량을 실시간 추적하여 보호자에게 알림 제공

#### 기능 상세

| 기능 | 설명 | 사용자 |
|------|------|--------|
| 송영 스케줄 관리 | 요일별/어르신별 픽업 시간·장소 설정 | 시설 관리자 |
| 실시간 위치 공유 | 운전기사 위치를 5초 간격으로 갱신 | 운전기사 → 보호자 |
| 도착 예정 알림 | 픽업 지점 반경 500m 진입 시 푸시 알림 | 보호자 수신 |
| 탑승/하차 체크 | 운전기사가 어르신 탑승/하차 확인 터치 | 운전기사 |
| 탑승 완료 알림 | 어르신 탑승 시 보호자에게 즉시 알림 | 보호자 수신 |
| 경로 히스토리 | 일별 운행 경로 기록 저장 | 시설 관리자 조회 |

#### 기술 구현

```
[운전기사 앱]                    [Supabase]                    [보호자 앱]
    │                              │                              │
    │ 위치 갱신 (5초 간격)          │                              │
    ├──────── INSERT location ─────►│                              │
    │         (lat, lng, timestamp) │                              │
    │                              │◄── Realtime Subscribe ───────┤
    │                              │                              │
    │                              │─── 위치 브로드캐스트 ─────────►│
    │                              │                              │
    │ 탑승 체크                     │                              │
    ├──── UPDATE ride_status ──────►│                              │
    │                              │─── Edge Function: 근접 체크 ──►│ 푸시 알림
    │                              │    (Geofence 500m)            │
```

- **위치 데이터**: `vehicle_locations` 테이블에 5초 간격 INSERT, 1시간 이후 자동 아카이브
- **Geofencing**: Edge Function에서 Haversine 공식으로 거리 계산, 500m 이내 시 FCM 푸시
- **배터리 최적화**: Foreground Service (Android) / Background Location (iOS) 사용, 정확도 조절

---

### 4.2 복약 스케줄러 (Medication Scheduler)

#### 개요
어르신의 복약 일정을 관리하고 복약 여부를 객관적으로 기록·공유

#### 기능 상세

| 기능 | 설명 | 사용자 |
|------|------|--------|
| 복약 일정 등록 | 약 이름, 용량, 시간, 주기 설정 | 시설 직원 |
| 복약 알림 | 복약 시간에 맞춰 직원에게 알림 | 시설 직원 |
| 복약 기록 | 복약 완료/미복약/거부 상태 기록 + 사진 촬영 | 시설 직원 |
| 복약 현황 조회 | 일별/주별 복약 이력 확인 | 보호자 |
| 이상 알림 | 미복약/이중복약 발생 시 보호자에게 자동 알림 | 보호자 수신 |
| 복약 통계 | 월별 복약 준수율 차트 | 시설 관리자, 보호자 |

#### 데이터 흐름

```
1. 시설 직원이 복약 일정 등록 (약 정보 + 시간 + 어르신 연결)
2. 복약 시간 도래 → 직원 앱에 알림 표시
3. 직원이 복약 확인 → 상태(완료/미복약/거부) + 사진 기록
4. Supabase Trigger → 보호자에게 실시간 알림
5. 미복약 시 → Edge Function에서 30분 후 재알림 → 여전히 미처리 시 보호자 알림
```

#### 복약 상태 머신

```
[예정(scheduled)] ──복약완료──► [완료(taken)]
       │                           │
       ├──미복약──► [미복약(missed)] │
       │                           │
       └──거부──► [거부(refused)]   │
                                   │
       모든 상태 ──메모추가──► [해당 상태 + 메모]
```

---

### 4.3 소통 (Communication)

#### 개요
시설↔보호자 간 양방향 메시징 및 공지사항 시스템

#### 기능 상세

| 기능 | 설명 | 사용자 |
|------|------|--------|
| 1:1 채팅 | 직원↔보호자 실시간 메시징 | 양방향 |
| 그룹 공지 | 시설 전체 공지사항 발송 | 시설 관리자 |
| 일과 보고 | 어르신 하루 활동 요약 (사진+텍스트) | 시설 직원 → 보호자 |
| 긴급 연락 | 긴급 상황 시 즉시 알림 (푸시+SMS) | 시설 → 보호자 |
| 상담 요청 | 보호자가 면담/상담 예약 | 보호자 → 시설 |
| 읽음 확인 | 메시지 읽음 상태 표시 | 양방향 |

#### 기술 구현
- **Supabase Realtime**: `messages` 테이블 구독으로 실시간 채팅
- **파일 첨부**: Supabase Storage에 이미지/영상 업로드 후 URL 공유
- **푸시 알림**: 앱 백그라운드 시 FCM으로 새 메시지 알림
- **긴급 알림**: Edge Function에서 FCM + SMS 동시 발송

---

### 4.4 교육 프로그램 (Activity Program)

#### 개요
시설 내 교육/활동 프로그램을 관리하고 보호자에게 참여 현황 공유

#### 기능 상세

| 기능 | 설명 |
|------|------|
| 프로그램 등록 | 프로그램명, 일시, 담당자, 정원 등록 |
| 주간 시간표 | 요일별 프로그램 시간표 (캘린더 뷰) |
| 참여 기록 | 어르신별 프로그램 참여/불참 기록 |
| 활동 사진 | 프로그램 중 촬영 사진 공유 |
| 보호자 조회 | 보호자가 어르신의 프로그램 참여 현황 확인 |

---

### 4.5 식단표 & 가정통신문 (Meal Plan & Newsletter)

#### 개요
일별 식단 정보 제공 및 어르신 일과 요약 보고

#### 기능 상세

| 기능 | 설명 |
|------|------|
| 식단표 등록 | 주간 식단 등록 (아침/점심/간식) |
| 식단 알림 | 매일 오전 보호자에게 당일 식단 자동 발송 |
| 일과 보고서 | 어르신 하루 컨디션, 활동, 특이사항 기록 |
| 가정통신문 | 시설 행사, 공지 등 정기 안내문 |
| 사진 갤러리 | 일별 활동 사진 모음 (보호자 조회) |

---

### 4.6 급여 관리 (Care Fee Management)

#### 개요
장기요양 급여 계산 보조 및 보호자 청구 내역 투명 공개

#### 기능 상세

| 기능 | 설명 |
|------|------|
| 급여 등급 관리 | 어르신별 장기요양 등급 정보 관리 |
| 본인부담금 계산 | 등급별 월 이용료 자동 계산 |
| 청구서 발행 | 월별 청구서 생성 및 보호자 전송 |
| 납부 현황 | 납부 상태 추적 (미납 알림) |

---

## 5. 데이터베이스 스키마

### ERD 개요

```
facilities ─────┐
  │              │
  ├── staff      │
  │              │
  ├── vehicles   │
  │   └── vehicle_locations
  │
  ├── residents ─┼── guardians (M:N → resident_guardians)
  │   │          │
  │   ├── medications ── medication_logs
  │   ├── daily_reports
  │   └── program_participations
  │
  ├── programs
  │
  ├── transportation_schedules
  │   └── ride_logs
  │
  ├── meal_plans
  │
  ├── messages
  │   └── message_reads
  │
  └── announcements
```

### 주요 테이블 정의

#### facilities (시설)
```sql
CREATE TABLE facilities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,                    -- 시설명
  type TEXT NOT NULL,                    -- 시설 유형 (day_care, residential, home_care)
  license_number TEXT,                   -- 시설 인허가 번호
  address TEXT NOT NULL,                 -- 주소
  latitude DOUBLE PRECISION,            -- 위도
  longitude DOUBLE PRECISION,           -- 경도
  phone TEXT,                           -- 대표 전화번호
  capacity INTEGER,                     -- 정원
  subscription_plan TEXT DEFAULT 'basic', -- 구독 플랜
  subscription_status TEXT DEFAULT 'trial', -- 구독 상태
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### staff (직원)
```sql
CREATE TABLE staff (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),  -- Supabase Auth 연결
  facility_id UUID REFERENCES facilities(id),
  name TEXT NOT NULL,
  role TEXT NOT NULL,                    -- facility_admin, facility_staff, facility_driver
  phone TEXT,
  email TEXT,
  position TEXT,                         -- 직책 (센터장, 팀장 등)
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### residents (어르신/이용자)
```sql
CREATE TABLE residents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  facility_id UUID REFERENCES facilities(id),
  name TEXT NOT NULL,                    -- 성명
  birth_date DATE,                      -- 생년월일
  gender TEXT,                          -- 성별
  care_grade TEXT,                      -- 장기요양 등급 (1~5, 인지지원)
  profile_image_url TEXT,               -- 프로필 사진
  medical_notes TEXT,                   -- 의료 특이사항 (암호화 저장)
  emergency_contact TEXT,               -- 긴급 연락처
  admission_date DATE,                  -- 입소일
  status TEXT DEFAULT 'active',         -- active, discharged, deceased
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

#### guardians (보호자)
```sql
CREATE TABLE guardians (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  name TEXT NOT NULL,
  phone TEXT NOT NULL,
  relationship TEXT,                     -- 관계 (자녀, 배우자, 손자녀 등)
  is_primary BOOLEAN DEFAULT false,      -- 주 보호자 여부
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 어르신-보호자 다대다 관계
CREATE TABLE resident_guardians (
  resident_id UUID REFERENCES residents(id),
  guardian_id UUID REFERENCES guardians(id),
  PRIMARY KEY (resident_id, guardian_id)
);
```

#### medications (복약 정보)
```sql
CREATE TABLE medications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  resident_id UUID REFERENCES residents(id),
  name TEXT NOT NULL,                    -- 약 이름
  dosage TEXT,                          -- 용량
  frequency TEXT NOT NULL,              -- 복약 빈도 (daily, twice_daily, etc.)
  times JSONB NOT NULL,                 -- 복약 시간 ["08:00", "18:00"]
  instructions TEXT,                    -- 복약 지침
  prescriber TEXT,                      -- 처방 의사
  start_date DATE NOT NULL,
  end_date DATE,                        -- NULL이면 지속
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### medication_logs (복약 기록)
```sql
CREATE TABLE medication_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  medication_id UUID REFERENCES medications(id),
  resident_id UUID REFERENCES residents(id),
  scheduled_at TIMESTAMPTZ NOT NULL,     -- 예정 시간
  status TEXT NOT NULL,                  -- taken, missed, refused, delayed
  taken_at TIMESTAMPTZ,                  -- 실제 복약 시간
  recorded_by UUID REFERENCES staff(id), -- 기록한 직원
  photo_url TEXT,                        -- 복약 확인 사진
  notes TEXT,                           -- 메모
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### vehicle_locations (차량 위치)
```sql
CREATE TABLE vehicle_locations (
  id BIGSERIAL PRIMARY KEY,
  vehicle_id UUID REFERENCES vehicles(id),
  driver_id UUID REFERENCES staff(id),
  latitude DOUBLE PRECISION NOT NULL,
  longitude DOUBLE PRECISION NOT NULL,
  speed REAL,                           -- 속도 (km/h)
  heading REAL,                         -- 방향
  recorded_at TIMESTAMPTZ DEFAULT now()
);

-- 성능을 위한 파티셔닝 (일별)
-- 1시간 이후 데이터는 vehicle_locations_archive로 이동 (cron)
CREATE INDEX idx_vehicle_locations_vehicle_time
  ON vehicle_locations (vehicle_id, recorded_at DESC);
```

#### transportation_schedules (송영 스케줄)
```sql
CREATE TABLE transportation_schedules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  facility_id UUID REFERENCES facilities(id),
  resident_id UUID REFERENCES residents(id),
  vehicle_id UUID REFERENCES vehicles(id),
  type TEXT NOT NULL,                    -- pickup (등원), dropoff (하원)
  scheduled_time TIME NOT NULL,          -- 예정 시간
  day_of_week INTEGER[],                -- 요일 (1=월 ~ 7=일)
  pickup_address TEXT,                   -- 픽업 주소
  pickup_latitude DOUBLE PRECISION,
  pickup_longitude DOUBLE PRECISION,
  notes TEXT,                           -- 특이사항
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### ride_logs (송영 기록)
```sql
CREATE TABLE ride_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  schedule_id UUID REFERENCES transportation_schedules(id),
  resident_id UUID REFERENCES residents(id),
  driver_id UUID REFERENCES staff(id),
  date DATE NOT NULL,
  status TEXT NOT NULL,                  -- scheduled, in_transit, boarded, arrived, cancelled
  boarded_at TIMESTAMPTZ,              -- 탑승 시간
  arrived_at TIMESTAMPTZ,              -- 도착 시간
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### messages (메시지)
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL,         -- 대화방 ID
  sender_id UUID NOT NULL,               -- 발신자 (staff 또는 guardian)
  sender_type TEXT NOT NULL,             -- staff, guardian
  content TEXT,                          -- 메시지 내용
  message_type TEXT DEFAULT 'text',      -- text, image, file, system
  attachment_url TEXT,                   -- 첨부파일 URL
  is_urgent BOOLEAN DEFAULT false,       -- 긴급 메시지 여부
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE message_reads (
  message_id UUID REFERENCES messages(id),
  reader_id UUID NOT NULL,
  read_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (message_id, reader_id)
);
```

#### daily_reports (일과 보고)
```sql
CREATE TABLE daily_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  resident_id UUID REFERENCES residents(id),
  facility_id UUID REFERENCES facilities(id),
  report_date DATE NOT NULL,
  condition TEXT,                        -- good, fair, poor
  mood TEXT,                            -- happy, neutral, sad, anxious
  appetite TEXT,                        -- good, fair, poor
  activities TEXT[],                     -- 참여한 활동들
  special_notes TEXT,                   -- 특이사항
  photos TEXT[],                        -- 활동 사진 URL 배열
  reported_by UUID REFERENCES staff(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(resident_id, report_date)
);
```

#### programs (교육/활동 프로그램)
```sql
CREATE TABLE programs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  facility_id UUID REFERENCES facilities(id),
  name TEXT NOT NULL,                    -- 프로그램명
  description TEXT,
  category TEXT,                        -- exercise, art, music, cognitive, social
  day_of_week INTEGER[],               -- 요일
  start_time TIME,
  end_time TIME,
  instructor TEXT,                      -- 담당 강사
  capacity INTEGER,                     -- 정원
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE program_participations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  program_id UUID REFERENCES programs(id),
  resident_id UUID REFERENCES residents(id),
  date DATE NOT NULL,
  attended BOOLEAN NOT NULL,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

#### meal_plans (식단표)
```sql
CREATE TABLE meal_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  facility_id UUID REFERENCES facilities(id),
  date DATE NOT NULL,
  meal_type TEXT NOT NULL,              -- breakfast, lunch, snack, dinner
  menu TEXT NOT NULL,                   -- 메뉴 내용
  image_url TEXT,                       -- 식단 사진
  allergens TEXT[],                     -- 알레르기 유발 성분
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(facility_id, date, meal_type)
);
```

### RLS (Row Level Security) 정책

```sql
-- 시설 직원은 자기 시설 데이터만 접근
CREATE POLICY "staff_facility_access" ON residents
  FOR ALL USING (
    facility_id IN (
      SELECT facility_id FROM staff WHERE user_id = auth.uid()
    )
  );

-- 보호자는 자기 어르신 데이터만 접근
CREATE POLICY "guardian_resident_access" ON medication_logs
  FOR SELECT USING (
    resident_id IN (
      SELECT rg.resident_id FROM resident_guardians rg
      JOIN guardians g ON g.id = rg.guardian_id
      WHERE g.user_id = auth.uid()
    )
  );

-- 위치 데이터: 관련 보호자만 실시간 조회
CREATE POLICY "guardian_vehicle_location" ON vehicle_locations
  FOR SELECT USING (
    vehicle_id IN (
      SELECT ts.vehicle_id FROM transportation_schedules ts
      JOIN resident_guardians rg ON rg.resident_id = ts.resident_id
      JOIN guardians g ON g.id = rg.guardian_id
      WHERE g.user_id = auth.uid() AND ts.is_active = true
    )
  );
```

---

## 6. API 설계

### Edge Functions 목록

| Function | Method | 설명 |
|----------|--------|------|
| `check-geofence` | POST | 차량 위치가 픽업 지점 근접 시 알림 트리거 |
| `send-push` | POST | FCM 푸시 알림 통합 발송 |
| `generate-daily-report` | POST | 일과 보고서 자동 생성 (cron 트리거) |
| `calculate-care-fee` | POST | 장기요양 급여 계산 |
| `send-sms` | POST | NHN Cloud SMS 발송 |
| `export-report` | POST | 월간 보고서 PDF 생성 |
| `archive-locations` | POST | 1시간 이상 된 위치 데이터 아카이브 (cron) |

### Supabase Client API 패턴

```dart
// 복약 기록 조회 (보호자 앱)
final logs = await supabase
  .from('medication_logs')
  .select('*, medications(name, dosage)')
  .eq('resident_id', residentId)
  .gte('scheduled_at', today)
  .order('scheduled_at');

// 실시간 차량 위치 구독 (보호자 앱)
supabase
  .from('vehicle_locations')
  .stream(primaryKey: ['id'])
  .eq('vehicle_id', vehicleId)
  .listen((data) {
    // 네이버 지도 마커 업데이트
    updateMarkerPosition(data.last);
  });

// 메시지 실시간 구독
supabase
  .from('messages')
  .stream(primaryKey: ['id'])
  .eq('conversation_id', conversationId)
  .order('created_at')
  .listen((messages) {
    // 채팅 UI 업데이트
  });
```

---

## 7. 실시간 통신 설계

### 실시간 채널 구조

| 채널 | 용도 | 구독자 |
|------|------|--------|
| `vehicle:{vehicleId}` | 차량 위치 실시간 | 관련 보호자 |
| `conversation:{id}` | 채팅 메시지 | 대화 참여자 |
| `facility:{id}:alerts` | 시설 전체 알림 | 해당 시설 직원 |
| `resident:{id}:medication` | 복약 상태 변경 | 보호자 |
| `resident:{id}:status` | 어르신 상태 변경 | 보호자 |

### 푸시 알림 시나리오

| 이벤트 | 수신자 | 우선순위 | 채널 |
|--------|--------|----------|------|
| 송영 차량 출발 | 보호자 | 보통 | FCM |
| 픽업 지점 500m 접근 | 보호자 | 높음 | FCM |
| 어르신 탑승 완료 | 보호자 | 보통 | FCM |
| 복약 완료 | 보호자 | 낮음 | FCM (무음) |
| 미복약 경고 | 보호자 | 높음 | FCM + SMS |
| 긴급 상황 | 보호자 | 긴급 | FCM + SMS + 전화 |
| 새 메시지 | 상대방 | 보통 | FCM |
| 일과 보고 발송 | 보호자 | 낮음 | FCM |

---

## 8. 인프라 및 배포

### Supabase 프로젝트 구성

```
실버싱크 Supabase 프로젝트
├── Database (PostgreSQL)
│   ├── public 스키마 (앱 데이터)
│   └── archive 스키마 (위치 히스토리 등)
├── Auth (사용자 인증)
├── Storage
│   ├── avatars (프로필 사진)
│   ├── medications (복약 확인 사진)
│   ├── activities (활동 사진)
│   ├── meals (식단 사진)
│   └── documents (문서, 가정통신문)
├── Edge Functions
│   ├── check-geofence
│   ├── send-push
│   ├── generate-daily-report
│   └── ...
└── Realtime (실시간 구독)
```

### CI/CD 파이프라인

```
main 브랜치 Push
    │
    ├── flutter analyze + flutter test
    │
    ├── Android Build → Google Play Console (Internal Track)
    │
    ├── iOS Build → TestFlight
    │
    └── Edge Functions Deploy → Supabase
```

### 환경 구분

| 환경 | 용도 | Supabase 프로젝트 |
|------|------|-------------------|
| dev | 개발/테스트 | silversync-dev |
| staging | QA/베타 테스트 | silversync-staging |
| prod | 운영 | silversync-prod |

---

## 9. 보안 설계

### 데이터 보안

| 항목 | 방법 |
|------|------|
| 전송 암호화 | TLS 1.3 (Supabase 기본) |
| 의료 데이터 암호화 | `medical_notes` 필드 AES-256 암호화 저장 |
| 파일 접근 제어 | Supabase Storage 정책 (인증된 사용자만) |
| DB 접근 제어 | RLS 정책 (역할 기반 행 수준 보안) |
| API 키 관리 | 환경변수 분리, anon key는 최소 권한 |

### 개인정보 보호 (개인정보보호법 준수)

| 항목 | 조치 |
|------|------|
| 개인정보 수집 동의 | 앱 가입 시 필수 동의 절차 |
| 목적 외 이용 금지 | RLS로 기술적 접근 제한 |
| 보유 기간 | 퇴소 후 3년 보관 → 자동 파기 |
| 열람/삭제 요청 | 앱 내 개인정보 조회/삭제 기능 |
| 로그 기록 | 개인정보 접근 로그 기록 (audit_logs 테이블) |

### 인증 흐름

```
1. 보호자 가입: 전화번호 SMS 인증 → 시설 초대 코드 입력 → 어르신 연결
2. 시설 직원 가입: 이메일/비밀번호 → 시설 관리자가 승인 → 역할 부여
3. 운전기사: 시설 관리자가 계정 생성 → 임시 비밀번호 발급
```

---

## 10. 단계별 로드맵

### Phase 1: MVP (3개월)
> 핵심 가치 검증 - "시설-보호자 정보 동기화"

#### 목표
- 최소 기능으로 빠르게 시장 반응 확인
- 울산 지역 2~3개 시설 파일럿

#### 포함 기능
| 기능 | 범위 |
|------|------|
| **송영 알리미** | 실시간 위치 추적 + 탑승/하차 알림 (기본) |
| **복약 관리** | 복약 등록 + 기록 + 보호자 조회 (기본) |
| **소통** | 1:1 채팅 + 공지사항 (기본) |
| **인증/가입** | 전화번호 인증 + 초대 코드 |
| **어르신 프로필** | 기본 정보 + 사진 + 장기요양 등급 |

#### 기술 작업
- [ ] Flutter 프로젝트 세팅 (모노레포, flavor 구성)
- [ ] Supabase 프로젝트 생성 + DB 스키마 (핵심 테이블)
- [ ] Supabase Auth 연동 (전화번호 인증)
- [ ] 네이버 지도 연동 + 실시간 위치 추적
- [ ] FCM 푸시 알림 연동
- [ ] 시설 앱: 어르신 관리, 복약 기록, 송영 체크, 채팅
- [ ] 보호자 앱: 위치 조회, 복약 현황, 채팅, 알림 수신
- [ ] RLS 정책 적용
- [ ] 베타 테스트 (울산 2~3개 시설)

#### 산출물
- 시설 앱 (Android APK / iOS TestFlight)
- 보호자 앱 (Android APK / iOS TestFlight)
- Supabase 백엔드 (dev + staging)

---

### Phase 2: 기능 확장 (2개월)
> 사용자 피드백 반영 + 부가 기능

#### 추가 기능
| 기능 | 범위 |
|------|------|
| **교육 프로그램** | 프로그램 등록 + 시간표 + 참여 기록 |
| **식단표** | 주간 식단 등록 + 보호자 조회 |
| **일과 보고** | 하루 요약 보고서 + 사진 갤러리 |
| **가정통신문** | 시설 공지 + 행사 안내 |
| **복약 통계** | 월별 복약 준수율 차트 |
| **송영 히스토리** | 일별 운행 기록 조회 |

#### 기술 작업
- [ ] 교육 프로그램 모듈 개발
- [ ] 식단표/일과 보고 모듈 개발
- [ ] 가정통신문 템플릿 시스템
- [ ] 차트/통계 UI (fl_chart)
- [ ] 오프라인 모드 (Hive/Isar 로컬 캐시)
- [ ] 위치 데이터 아카이브 시스템 (cron)
- [ ] UI/UX 개선 (Phase 1 피드백 반영)

---

### Phase 3: 상용화 (2개월)
> 정식 출시 + 수익 모델 적용

#### 추가 기능
| 기능 | 범위 |
|------|------|
| **급여 관리** | 등급별 급여 계산 + 청구서 |
| **구독 결제** | 시설 구독 플랜 (Basic/Pro) |
| **관리자 웹** | 시스템 관리 대시보드 (Flutter Web) |
| **보고서 내보내기** | 월간 보고서 PDF 생성 |
| **카카오 알림톡** | 정기 알림 (식단, 일과 보고) |

#### 기술 작업
- [ ] 결제 시스템 연동 (토스페이먼츠 or 아임포트)
- [ ] 관리자 웹 대시보드 (Flutter Web)
- [ ] PDF 보고서 생성 (Edge Function)
- [ ] 카카오 알림톡 연동
- [ ] Google Play / App Store 정식 출시
- [ ] 운영 모니터링 (에러 트래킹, 로그)
- [ ] 성능 최적화 + 부하 테스트

---

### Phase 4: 성장 (지속)
> 사용자 확대 + 고급 기능

#### 예정 기능
- 건강 데이터 연동 (혈압, 혈당 등 IoT 기기)
- AI 기반 어르신 컨디션 분석/예측
- 시설 평가 시스템 (보호자 리뷰)
- 다국어 지원 (일본어 우선)
- 방문요양 서비스 확장
- 광고 플랫폼 (노인 관련 제품/서비스)

---

## 부록: 예상 비용 (월간)

| 항목 | MVP 단계 | 상용화 단계 | 비고 |
|------|----------|------------|------|
| Supabase Pro | $25 | $25~$75 | DB, Auth, Storage 포함 |
| 네이버 지도 API | 무료~$50 | $50~$200 | 월 무료 할당량 초과 시 |
| FCM | 무료 | 무료 | Google 무료 |
| NHN SMS | $10~$30 | $50~$100 | 건당 과금 |
| Apple Developer | $8/월 | $8/월 | 연 $99 |
| Google Play | 일회성 $25 | - | |
| **합계** | **~$70/월** | **~$400/월** | |

---

*문서 끝 - SilverSync Tech Spec v1.0*
