
# Luvle - 모바일 웨딩 청첩장 플랫폼

[![Next.js](https://img.shields.io/badge/Next.js-16.1.0-black)](https://nextjs.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.7.12-brightgreen)](https://spring.io/projects/spring-boot)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.1.6-blue)](https://www.typescriptlang.org/)

## 프로젝트 소개

디자이너가 예비 신랑/신부의 모바일 웨딩 청첩장을 제작하고 운영할 수 있는 풀스택 시스템입니다.

### 핵심 기능

- **청첩장 제작**: 커버 이미지, 갤러리, 색상 테마, 배경 음악 등 맞춤 설정
- **관리자 대시보드**: 청첩장 생성/수정, 고객 관리, 하객 통계
- **고객 포털**: 신랑/신부 전용 통계 조회 및 화환 주문 관리
- **외부 서비스 연동**: 카카오맵/내비, 네이버지도, T맵, 카카오톡 공유, SMS 알림

---

## 아키텍처

```
┌─────────────────────────────────────────┐
│  📱 모바일 청첩장        🖥️  관리자 대시보드  │
│  (Next.js)            (Next.js + TS)  │
└──────────────┬──────────────────────────┘
               │ HTTPS / REST API
               ▼
┌──────────────────────────────────────────┐
│        Nginx (Reverse Proxy)            │
│     Blue-Green Deployment Switch        │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│       Spring Boot API Server            │
│  - JWT 인증 (고객/관리자)                  │
│  - JPA/Hibernate ORM                    │
│  - AWS S3 파일 업로드                     │
└──────────────┬───────────┬───────────────┘
               │           │
               ▼           ▼
        ┌─────────┐  ┌─────────┐
        │ MySQL 8 │  │  AWS S3 │
        └─────────┘  └─────────┘
```

---

## 기술 스택

### Backend
- Spring Boot 2.7.12, Java 17
- MySQL 8.0, Spring Data JPA
- Spring Security, JWT
- AWS SDK (S3)

### Frontend
**Mobile Invitation**
- Next.js 16.1.0 (App Router), React 19.0.0
- styled-components, Zustand
- Swiper, Recharts, dayjs

**Admin Dashboard**
- Next.js 16.1.0, TypeScript 5.1.6
- Material-UI, react-hook-form
- Apexcharts, react-color

### Infrastructure
- Docker, Docker Compose
- GitHub Actions (CI/CD)
- AWS EC2, ECR, Lambda, S3
- Nginx, PM2

---

## 주요 기능

### 청첩장 커스터마이징
- 신랑/신부 정보, 결혼식 날짜/장소
- 커버 이미지, 갤러리, 색상 테마
- 배경 음악, 지도 연동
- 계좌번호, 화환 주문

### 관리자 대시보드
- 청첩장 CRUD
- 고객 계정 관리
- 하객 통계 조회
- 화환 주문 관리

### 고객 포털
- 하객 참석 통계
- 화환 주문 내역
- 방명록 조회

### 외부 서비스 연동
- **지도**: 카카오맵, 네이버지도, T맵
- **공유**: 카카오톡
- **알림**: SMS (결혼식 임박 알림)
- **화환**: FlowerBiz API

---

## 프로젝트 구조

```
luvle/
├── backend/                    # Spring Boot API
│   ├── controller/
│   ├── service/
│   ├── repository/
│   ├── entity/
│   └── security/
│
├── mobile-invitation/          # Next.js 모바일 청첩장
│   ├── app/
│   │   ├── [id]/              # 동적 청첩장 페이지
│   │   └── customer/          # 고객 전용 페이지
│   ├── components/
│   ├── store/
│   └── services/
│
├── frontend-admin/             # Next.js 관리자 대시보드
│   ├── app/
│   │   ├── invitations/
│   │   ├── customers/
│   │   └── guests/
│   └── services/
│
├── switch-a-to-b/             # Blue-Green 배포 API
│
├── watcher/                    # AWS Lambda 모니터링
│   └── lambda/
│       ├── wedding-near/      # 결혼식 알림
│       └── health-check/
│
└── .github/workflows/          # CI/CD 파이프라인
```

---

## 배포 전략

### Blue-Green Deployment

무중단 배포를 위한 Blue-Green 전략 사용:

1. Server A(Blue)가 프로덕션 서비스 중
2. 새 버전을 Server B(Green)에 배포
3. Server B 헬스체크 확인
4. Nginx 설정을 Server B로 전환
5. Server A는 롤백용 스탠바이 유지

**장점**
- 무중단 배포
- 즉시 롤백 가능
- 배포 전 안정성 검증

### CI/CD

GitHub Actions를 통한 자동화:
- 환경별(dev, prod-a, prod-b) 독립 워크플로우
- Docker 이미지 빌드 및 AWS ECR 푸시
- SSH를 통한 원격 배포
- 수동 트리거로 배포 통제

### 모니터링

AWS Lambda 기반:
- 결혼식 임박 알림 자동 발송
- 시스템 헬스체크 및 장애 알림

---

## 기술적 특징

- **Blue-Green 무중단 배포**: 프로덕션 서비스 안정성 확보
- **멀티 환경 지원**: Dev, Prod-A, Prod-B 독립 운영
- **서버리스 모니터링**: Lambda 기반 자동화된 알림 시스템
- **한국 시장 특화**: 카카오/네이버/T맵 연동, 화환 주문 API 통합

---

## Technical Highlight

[Technical Highlight 상세 페이지](docs/TECHNICAL_HIGHLIGHT.md)

## 라이선스

본 프로젝트는 비공개 상용 프로젝트입니다.  
이 README는 포트폴리오 목적으로 작성되었습니다.