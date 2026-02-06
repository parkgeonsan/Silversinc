# 🤝 SilverSync (실버싱크)

> **노인장기요양 시설 · 보호자 간의 정보 동기화 솔루션**

[![License](https://img.shields.io/badge/license-Private-red)]()
[![Platform](https://img.shields.io/badge/platform-iOS%20%7C%20Android%20%7C%20Web-blue)]()
[![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B?logo=flutter)]()
[![Supabase](https://img.shields.io/badge/Supabase-Backend-3ECF8E?logo=supabase)]()

---

## 프로젝트 소개

SilverSync는 노인장기요양 시설(주간보호센터, 요양원 등)과 보호자를 실시간으로 연결하는 정보 동기화 플랫폼입니다.

### 해결하는 문제

| 문제 | SilverSync 솔루션 |
|------|-------------------|
| 송영 중 어르신 안전 우려 | 🚐 실시간 차량 위치 추적 + 탑승/하차 알림 |
| 복약 기록 조작 가능한 수기 시스템 | 💊 디지털 복약 기록 + 사진 증빙 + 보호자 실시간 확인 |
| 시설-보호자 간 소통 부재 | 💬 양방향 채팅 + 일과 보고 + 사진 갤러리 |
| 어르신 생활 관련 솔루션 전무 | 📋 통합 생활 관리 (식단, 프로그램, 건강 기록) |

---

## 핵심 기능

### 🚐 송영 알리미
- 네이버 지도 기반 실시간 차량 위치 추적
- 픽업 지점 500m 접근 시 보호자 자동 알림
- 탑승/하차 체크 및 보호자 즉시 알림

### 💊 복약 스케줄러
- 약 복용 일정 관리 및 알림
- 복약 완료/미복약/거부 상태 기록 + 사진 촬영
- 보호자 실시간 복약 현황 조회
- 주간/월간 복약 준수율 통계

### 💬 시설-보호자 소통
- 1:1 실시간 채팅
- 긴급 알림 (푸시 + SMS)
- 공지사항 및 가정통신문

### 📋 일과 보고
- 어르신 하루 컨디션/기분/식욕 기록
- 활동 사진 공유
- 보호자 자동 발송

### 📚 교육 프로그램 관리
- 주간 프로그램 시간표
- 참여 기록 및 보호자 공유

### 🍚 식단표
- 일별 식단 등록 및 보호자 조회

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| **모바일 앱** | Flutter 3.x (iOS + Android) |
| **상태 관리** | Riverpod 2.x |
| **백엔드** | Supabase (Auth, PostgreSQL, Realtime, Storage, Edge Functions) |
| **지도** | 네이버 지도 API |
| **푸시 알림** | Firebase Cloud Messaging (FCM) |
| **SMS** | NHN Cloud SMS |
| **CI/CD** | GitHub Actions |

---

## 프로젝트 구조

```
Silversinc/
├── README.md                          ← 이 파일
├── docs/
│   ├── TECH_SPEC.md                   ← 기술 스펙 (아키텍처, DB, API, 로드맵)
│   ├── ENHANCEMENT_PLAN.md            ← 고도화 방안 (AI/IoT, 글로벌 확장)
│   ├── BUSINESS_MODEL.md              ← 비즈니스 모델 & 수익 전략
│   ├── USER_FLOW.md                   ← 사용자 플로우 & 화면 설계
│   └── SECURITY_COMPLIANCE.md         ← 보안 & 컴플라이언스
├── prototype/
│   ├── style.css                      ← 공통 CSS
│   ├── index.html                     ← 메인 페이지 (프로젝트 소개)
│   ├── facility.html                  ← 시설 직원 앱 프로토타입
│   ├── guardian.html                  ← 보호자 앱 프로토타입
│   ├── login.html                     ← 로그인/온보딩 프로토타입
│   ├── resident-detail.html           ← 어르신 상세 프로필 프로토타입
│   └── admin-dashboard.html           ← 관리자 대시보드 프로토타입
└── (향후 Flutter 프로젝트 추가 예정)
```

---

## HTML 프로토타입

브라우저에서 바로 확인할 수 있는 UI 프로토타입입니다.

| 화면 | 파일 | 설명 |
|------|------|------|
| 메인 | `prototype/index.html` | 프로젝트 소개 + 앱 선택 |
| 시설 앱 | `prototype/facility.html` | 홈, 송영, 복약, 소통, 일과보고 |
| 보호자 앱 | `prototype/guardian.html` | 홈, 송영위치, 복약현황, 일과, 채팅 |
| 로그인 | `prototype/login.html` | 온보딩 플로우 |
| 어르신 상세 | `prototype/resident-detail.html` | 프로필, 복약, 건강, 활동, 보호자 |
| 관리자 | `prototype/admin-dashboard.html` | 시스템 관리 대시보드 |

---

## 문서 가이드

| 문서 | 내용 | 대상 |
|------|------|------|
| [TECH_SPEC.md](docs/TECH_SPEC.md) | 시스템 아키텍처, DB 스키마, API 설계, 단계별 로드맵 | 개발팀 |
| [ENHANCEMENT_PLAN.md](docs/ENHANCEMENT_PLAN.md) | AI/IoT 연동, 데이터 분석, 글로벌 확장, 투자 전략 | 경영진, 투자자 |
| [BUSINESS_MODEL.md](docs/BUSINESS_MODEL.md) | 시장 분석, 구독 플랜, 수익 예측, KPI | 경영진, 투자자 |
| [USER_FLOW.md](docs/USER_FLOW.md) | 사용자 시나리오, 화면 흐름도, 알림 설계 | 기획자, 디자이너 |
| [SECURITY_COMPLIANCE.md](docs/SECURITY_COMPLIANCE.md) | 개인정보보호법, 의료법, 보안 설계 | 법무, 보안팀 |

---

## 로드맵

| Phase | 기간 | 목표 |
|-------|------|------|
| **Phase 1: MVP** | 3개월 | 송영 + 복약 + 소통 (울산 파일럿) |
| **Phase 2: 기능 확장** | 2개월 | 교육, 식단, 일과보고, 통계 |
| **Phase 3: 상용화** | 2개월 | 결제, 관리자 웹, 정식 출시 |
| **Phase 4: AI/IoT** | 6개월 | 컨디션 예측, 건강 디바이스 연동 |
| **Phase 5: 글로벌** | 12개월 | 일본 진출, 다국어 지원 |

---

## 대상 사용자

| 사용자 | 앱 | 주요 기능 |
|--------|-----|----------|
| 시설 관리자 (센터장) | 시설 앱 | 시설 전체 관리, 직원/어르신 관리 |
| 시설 직원 (사회복지사) | 시설 앱 | 복약 기록, 일과 보고, 보호자 소통 |
| 운전기사 | 시설 앱 | 위치 공유, 탑승 체크 |
| 보호자 (가족) | 보호자 앱 | 실시간 알림, 복약 확인, 소통 |
| 시스템 관리자 | 웹 대시보드 | 시설 등록 승인, 통계 |

---

## 연락처

- 프로젝트: SilverSync (실버싱크)
- GitHub: [parkgeonsan/Silversinc](https://github.com/parkgeonsan/Silversinc)

---

*SilverSync - 어르신의 안전하고 행복한 하루를 함께 만듭니다*
