# 병원 CRM 기획

## 1. 개요

웹사이트에서 수집된 상담 정보와 회원 데이터를 병원 운영에 활용하기 위한 CRM(Customer Relationship Management) 기능을 관리자 페이지에서 제공합니다.

---

## 2. CRM 기능 범위

### 2.1 웹사이트 CRM

사이트 내에서 관리하는 경량 CRM 기능:

| 기능 | 설명 |
|-----|------|
| 상담 고객 관리 | 웹 상담 신청 고객 정보 |
| 방문 예약 관리 | 예약 현황 및 관리 |
| 고객 이력 관리 | 상담/시술 이력 조회 |
| 메시지 발송 | SMS/이메일 발송 |
| 통계 리포트 | 상담/시술/매출 통계 |

### 2.2 외부 CRM 연동

기존 병원 CRM과 연동이 필요한 경우:

| 시스템 | 연동 방식 |
|-------|----------|
| One-DAS | API 연동 |
| 구름CRM | API 연동 |
| 자체 구축 시스템 | Webhook/API |

---

## 3. 데이터 모델

### 3.1 고객 (Customer)

```sql
-- 웹사이트 회원과 통합
users (
  id: UUID
  email: VARCHAR
  phone: VARCHAR
  name: VARCHAR
  birth_date: DATE

  -- CRM 추가 필드
  customer_type: VARCHAR -- new, consulted, booked, surgery, dormant
  first_visit_date: DATE
  last_visit_date: DATE
  total_consultations: INT
  total_surgeries: INT
  estimated_value: INT -- 예상 매출
  actual_value: INT -- 실제 매출

  -- 마케팅
  source: VARCHAR -- google, instagram, blog, friend
  campaign: VARCHAR
  consent_sms: BOOLEAN
  consent_email: BOOLEAN
)
```

### 3.2 상담 (Consultation)

```sql
consultations (
  id: UUID
  user_id: UUID (FK -> users)
  category: VARCHAR -- 시술 카테고리
  title: VARCHAR
  content: TEXT
  is_public: BOOLEAN

  -- CRM 추가 필드
  status: VARCHAR -- new, contacted, booked, cancelled, completed
  priority: VARCHAR -- low, normal, high
  staff_id: UUID -- 담당 직원
  booked_at: TIMESTAMP -- 예약일시

  -- 상담 결과
  consultation_result: TEXT -- 상담 내용 요약
  surgery_plan: TEXT -- 시술 계획
  estimated_cost: INT -- 예상 비용

  created_at: TIMESTAMP
  updated_at: TIMESTAMP
)
```

### 3.3 방문 예약 (Booking)

```sql
bookings (
  id: UUID
  user_id: UUID (FK -> users)
  consultation_id: UUID (FK -> consultations)

  -- 예약 정보
  booking_date: DATE NOT NULL
  booking_time: TIME NOT NULL
  type: VARCHAR -- consultation, surgery, followup
  status: VARCHAR -- pending, confirmed, completed, cancelled, noshow

  -- 시술 정보
  procedures: JSONB -- ["eye", "nose"]
  expected_duration: INT -- 예상 소요시간(분)

  -- 담당자
  staff_id: UUID (FK -> admin_users)

  -- 비용
  estimated_cost: INT
  actual_cost: INT

  created_at: TIMESTAMP
  updated_at: TIMESTAMP
)
```

### 3.4 시술 기록 (Surgery)

```sql
surgeries (
  id: UUID
  user_id: UUID (FK -> users)
  booking_id: UUID (FK -> bookings)

  -- 시술 정보
  procedures: JSONB -- ["double_eyelid", "ptosis_correction"]
  surgeon_id: UUID (FK -> admin_users)
  surgery_date: DATE
  surgery_time: TIME

  -- 비용
  cost: INT
  payment_method: VARCHAR -- card, cash, transfer

  -- 결과
  satisfaction: INT -- 1-5
  notes: TEXT
  complications: TEXT -- 부작용 등

  created_at: TIMESTAMP
)
```

### 3.5 메시지 발송 기록

```sql
messages (
  id: UUID
  recipient_id: UUID (FK -> users)
  recipient_type: VARCHAR -- user, staff

  type: VARCHAR -- sms, email, lms, mms
  status: VARCHAR -- pending, sent, failed

  content: TEXT
  template_id: VARCHAR

  sent_at: TIMESTAMP
  created_at: TIMESTAMP
)
```

---

## 4. CRM 기능 상세

### 4.1 대시보드

```
┌─────────────────────────────────────────────────────────────────┐
│                         CRM 대시보드                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │   신규 상담   │ │   예약 건수   │ │   시술 건수   │ │  예상 매출 │ │
│  │      15     │ │      8      │ │      5      │ │  25,000,000│ │
│  │   ▲ 3      │ │   ▼ 1      │ │      -      │ │     -      │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  주간 상담 추이                                          │   │
│  │  [그래프]                                                │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  시술별 통계                                             │   │
│  │  눈: 40% | 코: 30% | 윤곽: 15% | 가슴: 10% | 기타: 5%    │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  최신 상담                                                │   │
│  │  김○○ | 쌍꺼풀 상담 | 10분 전 | [상담]                  │   │
│  │  이○○ | 코 성형 | 30분 전 | [상담]                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 고객 관리 (Customer Management)

#### 고객 목록

```
┌─────────────────────────────────────────────────────────────────┐
│  고객 관리 > 목록                                                │
├─────────────────────────────────────────────────────────────────┤
│  [검색] [상태▼] [유형▼] [기간▼]            [엑셀 다운로드]      │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 이름  연락처     상태      최상담일   시술  예상매출     │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │ 김○○ 010-xxx... [상담중]  24.03.19   0회   3,000,000    │   │
│  │ 이○○ 010-xxx... [시술완료] 24.03.15   2회   8,500,000    │   │
│  │ 박○○ 010-xxx... [예약]    24.03.20   1회   5,000,000    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  [< 1 2 3 4 5 >]                                               │
└─────────────────────────────────────────────────────────────────┘
```

#### 고객 상세

```
┌─────────────────────────────────────────────────────────────────┐
│  고객 관리 > 김○○                                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐ ┌─────────────────────────────────┐   │
│  │  기본 정보           │ │  통계                          │   │
│  │  이름: 김○○          │ │  총 상담: 3회                 │   │
│  │  연락처: 010-xxx...  │ │  총 시술: 2회                 │   │
│  │  이메일: xxx@...     │ │  총 매출: 8,500,000원          │   │
│  │  생년월일: 1990.05.05│ │  NPS: 8/10                    │   │
│  │  가입일: 2024.01.15  │ │                                 │   │
│  │  경로: 인스타그램     │ │                                 │   │
│  └─────────────────────┘ └─────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  [상담 이력] [시술 이력] [예약 관리] [메시지] [메모]            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  상담 이력                                               │   │
│  │  2024.03.19 | 쌍꺼풀 상담 | [상담중]                    │   │
│  │  2024.03.10 | 코 성형 | [시술완료]                       │   │
│  │  2024.01.20 | 눈매교정 | [시술완료]                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 예약 관리 (Booking Management)

#### 캘린더 뷰

```
┌─────────────────────────────────────────────────────────────────┐
│  예약 관리 > 2024년 3월                                          │
├─────────────────────────────────────────────────────────────────┤
│  [< 3월 >]                                                      │
│  일 월 화 수 목 금 토                                            │
│               1  2  3  4  5                                     │
│   6  7  8  9 10 11 12                                         │
│  13 14 15 16 17 18 19                                         │
│        [09:00 김○○]               [14:00 이○○]                  │
│        [10:00 박○○]               [15:00 최○○]                  │
│  20 21 22 23 24 25 26                                         │
│  27 28 29 30 31                                                │
└─────────────────────────────────────────────────────────────────┘
```

#### 예약 등록

```
┌─────────────────────────────────────────────────────────────────┐
│  예약 등록                                                       │
├─────────────────────────────────────────────────────────────────┤
│  고객 검색: [           ] [검색]                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  김○○ (010-xxxx-xxxx) | [선택]                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  예약일시: [2024-03-20] [09:00 ▼]                              │
│  유형:     ( ) 상담  (○) 시술  ( ) 재진                       │
│  시술:     [☑ 쌍꺼풀] [☑ 눈매교정] [☐ 코성형]                 │
│  담당의:   [김의사 ▼]                                          │
│  예상비용: [3,000,000원]                                       │
│  메모:     [_________________________]                        │
│                                                                 │
│                          [등록] [취소]                         │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 메시지 발송 (Messaging)

#### 템플릿 관리

```typescript
// SMS/이메일 템플릿
const templates = {
  consultationConfirm: {
    name: '상담 예약 확인',
    type: 'sms',
    content: '[OO성형외과] {date} {time} 상담 예약이 완료되었습니다.\n\n변경 시 02-1234-5678로 연락주세요.'
  },
  consultationReminder: {
    name: '상담 전 알림',
    type: 'sms',
    content: '[OO성형외과] 내일 {time} 상담 예약이 있습니다.\n\n시간 변경 시 연락바랍니다.'
  },
  surgeryReminder: {
    name: '시술 전 알림',
    type: 'sms',
    content: '[OO성형외과] {date} 시술 예정입니다.\n\n금식, 약물 복용 중지 등 주의사항 확인바랍니다.'
  },
  followUp: {
    name: '경과 확인',
    type: 'sms',
    content: '[OO성형외과] {days}일 경과 확인입니다.\n\n부작용 등 문의사항 있으시면 연락주세요.'
  }
};
```

#### 대량 발송

```
┌─────────────────────────────────────────────────────────────────┐
│  메시지 발송                                                     │
├─────────────────────────────────────────────────────────────────┤
│  발송 유형: ( ) SMS  (○) LMS  ( ) 이메일                       │
│  수신자:   [테마별] [그룹별] [직접 선택]                        │
│  템플릿:   [상담 예약 확인 ▼]                                  │
│                                                                 │
│  내용:                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ [OO성형외과] {date} {time} 상담 예약이 완료되었습니다.    │   │
│  │                                                         │   │
│  │ 변경 시 02-1234-5678로 연락주세요.                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  예상 발송: 15명                                                │
│  예상 비용: 750원 (건당 50원)                                   │
│                                                                 │
│                            [미리보기] [발송]                    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 통계 리포트 (Analytics)

#### 기간별 매출

```
┌─────────────────────────────────────────────────────────────────┐
│  통계 > 매출 현황                                                │
├─────────────────────────────────────────────────────────────────┤
│  [2024년 3월]                                                   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  일별 매출                                               │   │
│  │  [그래프: 3/1 ~ 3/31 일별 매출 추이]                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  시술별 매출                                             │   │
│  │  눈: ████████ 12,000,000원 (48%)                        │   │
│  │  코: ██████   7,500,000원 (30%)                         │   │
│  │  윤곽: ███     3,000,000원 (12%)                        │   │
│  │  기타: ██     2,500,000원 (10%)                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### 상담 퍼널

```
┌─────────────────────────────────────────────────────────────────┐
│  통계 > 상담 퍼널                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     웹사이트 방문                                               │
│            │                                                   │
│            ▼ 1,000                                             │
│     상담 신청 (100건)                                           │
│            │                                                   │
│            ▼ 80건                                              │
│     상담 연결 (80%)                                             │
│            │                                                   │
│            ▼ 30건                                              │
│     예약 전환 (37.5%)                                           │
│            │                                                   │
│            ▼ 25건                                              │
│     시술 완료 (83.3%)                                           │
│            │                                                   │
│            ▼                                                   │
│     매출: ₩85,000,000                                           │
│                                                                 │
│  전환율: 2.5% (상담 → 시술)                                     │
│  평균 매출: ₩3,400,000                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. API 설계 (CRM)

### 5.1 고객 관리

```typescript
// 고객 목록 조회
GET /admin/crm/customers
Query: status, type, search, page, limit

// 고객 상세 조회
GET /admin/crm/customers/{id}

// 고객 정보 수정
PATCH /admin/crm/customers/{id}

// 고객 메모 작성
POST /admin/crm/customers/{id}/notes

// 고객 태그 추가/삭제
POST /admin/crm/customers/{id}/tags
DELETE /admin/crm/customers/{id}/tags/{tagId}
```

### 5.2 예약 관리

```typescript
// 예약 목록 조회 (캘린더/리스트)
GET /admin/crm/bookings
Query: start, end, status

// 예약 등록
POST /admin/crm/bookings

// 예약 수정
PATCH /admin/crm/bookings/{id}

// 예약 취소
POST /admin/crm/bookings/{id}/cancel

// 예약 시간대 조회 (빈 시간)
GET /admin/crm/bookings/available-slots
Query: date, procedure
```

### 5.3 시술 기록

```typescript
// 시술 등록
POST /admin/crm/surgeries

// 시술 수정
PATCH /admin/crm/surgeries/{id}

// 시술 사진 업로드
POST /admin/crm/surgeries/{id}/photos
```

### 5.4 메시지

```typescript
// 템플릿 목록
GET /admin/crm/messages/templates

// 템플릿 등록/수정
POST /admin/crm/messages/templates
PATCH /admin/crm/messages/templates/{id}

// 메시지 발송
POST /admin/crm/messages/send

// 발송 내역
GET /admin/crm/messages/history
```

---

## 6. 외부 CRM 연동

### 6.1 Webhook 방식

웹사이트 데이터 변화를 외부 CRM에 전송:

```typescript
// packages/backend/src/services/crm/webhook.ts
export async function sendToCRM(event: CRMEvent) {
  const webhookUrl = env.CRM_WEBHOOK_URL;

  await fetch(webhookUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${env.CRM_API_KEY}`
    },
    body: JSON.stringify({
      event: event.type,
      data: event.data,
      timestamp: new Date().toISOString()
    })
  });
}

// 이벤트 종류
type CRMEvent =
  | { type: 'consultation.created', data: Consultation }
  | { type: 'booking.created', data: Booking }
  | { type: 'surgery.completed', data: Surgery }
  | { type: 'customer.updated', data: User };
```

### 6.2 API 방식

외부 CRM 데이터를 조회:

```typescript
// 외부 CRM 정보 조회
GET /admin/crm/external/customer/:id

// 동기화
POST /admin/crm/sync
Body: { type: 'customer' | 'booking' | 'surgery', id: string }
```

---

## 7. 보안 고려사항

### 7.1 접근 권한

| 역할 | 조회 | 등록 | 수정 | 삭제 | 통계 |
|-----|------|------|------|------|------|
| 원장 | O | O | O | O | O |
| 의사 | O | O | O | X | O |
| 상담사 | O | O | O | X | O |
| 직원 | O | O | 자기것만 | X | X |

### 7.2 데이터 마스킹

- 전화번호: 010-****-****
- 이메일: a***@gmail.com
- 주소: 서울시 강남구 ****

```typescript
function maskPhone(phone: string): string {
  return phone.replace(/(\d{3})-\d{4}-(\d{4})/, '$1-****-$2');
}
```

### 7.3 감사 로그

모든 CRM 활동 로그 기록:

```sql
CREATE TABLE crm_audit_logs (
    id UUID PRIMARY KEY,
    admin_id UUID REFERENCES admin_users(id),
    action VARCHAR(100), -- customer.view, booking.create, etc.
    target_type VARCHAR(50),
    target_id UUID,
    details JSONB,
    ip_address VARCHAR(50),
    created_at TIMESTAMP
);
```

---

## 8. 개발 일정

### Phase 1: 기본 CRM (2주)
- [ ] 고객 관리 기능
- [ ] 상담 이력 관리
- [ ] 기본 통계

### Phase 2: 예약 관리 (1주)
- [ ] 예약 캘린더
- [ ] 예약 등록/수정
- [ ] 알림 기능

### Phase 3: 메시지 발송 (1주)
- [ ] 템플릿 관리
- [ ] SMS 발송 연동
- [ ] 이메일 발송

### Phase 4: 시술 관리 (1주)
- [ ] 시술 기록
- [ ] 사진 관리
- [ ] 만족도 조사

### Phase 5: 통계 리포트 (1주)
- [ ] 매출 통계
- [ ] 상담 퍼널
- [ ] 엑셀 내보내기

### Phase 6: 외부 연동 (1주)
- [ ] Webhook 구현
- [ ] 외부 CRM API 연동
- [ ] 데이터 동기화
