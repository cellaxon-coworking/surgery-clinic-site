# 2FA (2단계 인증) 구현 가이드

## 1. 개요

모든 계정에 대해 2FA를 지원하며, 관리자 계정은 2FA가 **필수**입니다. 비용 부담이 적고 국내외 사용자가 부담을 느끼지 않는 방식을 선택합니다.

---

## 2. 2FA 방식 비교

| 방식 | 비용 | 사용자 경험 | 보안 | 해외 지원 | 추천 |
|-----|------|-------------|------|-----------|------|
| **이메일 OTP** | 무료 | 보통 | 보통 | 완벽 | ✅ |
| **TOTP (앱)** | 무료 | 좋음 | 좋음 | 완벽 | ✅ |
| **SMS OTP** | 유료 (건당 ~50원) | 좋음 | 좋음 | 국제 SMS 비용 expensive | ❌ |
| **WebAuthn** | 무료 | 매우 좋음 | 매우 좋음 | 하드웨어 필요 | △ |

### 2.1 추천 조합

```
일반 사용자: 이메일 OTP (기본) + TOTP (선택)
관리자: TOTP (필수) + 이메일 OTP (백업)
```

**이유:**
- 이메일 OTP: 별도 비용 없음, 이미 이메일 인증 시스템 존재
- TOTP: 앱만 설치하면 무료, Google Authenticator 등 범용
- SMS: 국내 무료 문자도 한계 있고, 해외는 비용 비쌈

---

## 3. TOTP (Time-based OTP)

### 3.1 지원 앱

| 앱 | 플랫폼 | 비고 |
|-----|--------|------|
| Google Authenticator | iOS/Android | 가장 널리 사용 |
| Microsoft Authenticator | iOS/Android | 백업 지원 |
| Authy | iOS/Android | 멀티디바이스 |
| Yubico Authenticator | iOS/Android | 하드웨어 키 연동 |

### 3.2 구현 방식

```
1. 사용자가 TOTP 활성화 요청
2. 서버가 Secret Key 생성 → QR 코드 제공
3. 사용자가 인증 앱으로 QR 스캔
4. 앱에서 6자리 코드 생성
5. 사용자가 입력 → 서버 검증
6. TOTP 활성화 완료
```

### 3.3 백엔드 구현

```typescript
// packages/backend/src/services/totp.ts

import { authenticator } from 'otplib';
import QRCode from 'qrcode';

interface TOTPSetup {
  secret: string;
  qrCode: string;
  backupCodes: string[];
}

// TOTP 설정 생성
export async function generateTOTP(userId: string, email: string): Promise<TOTPSetup> {
  // Secret 생성
  const secret = authenticator.generateSecret();

  // QR Code URL (authenticator 앱용)
  const qrCodeUrl = authenticator.keyuri(
    email,
    'OO성형외과',
    secret
  );

  // QR Code 이미지 생성
  const qrCode = await QRCode.toDataURL(qrCodeUrl);

  // 백업 코드 생성 (기기 분실 시)
  const backupCodes = Array.from({ length: 10 }, () =>
    Math.random().toString(36).substring(2, 8).toUpperCase()
  );

  // DB에 저장 (아직 활성화 전)
  await db.userTOTP.create({
    userId,
    secret,
    backupCodes,
    verified: false
  });

  return { secret, qrCode, backupCodes };
}

// TOTP 코드 검증
export function verifyTOTP(token: string, secret: string): boolean {
  return authenticator.verify({
    token,
    secret
  });
}

// 백업 코드 검증
export async function verifyBackupCode(
  userId: string,
  code: string
): Promise<boolean> {
  const totp = await db.userTOTP.findUnique({ where: { userId } });

  if (!totp) return false;

  const codeIndex = totp.backupCodes.indexOf(code);

  if (codeIndex === -1) return false;

  // 사용한 백업 코드 삭제
  const newBackupCodes = totp.backupCodes.filter((_, i) => i !== codeIndex);

  await db.userTOTP.update({
    where: { userId },
    data: { backupCodes: newBackupCodes }
  });

  return true;
}
```

### 3.4 DB 스키마

```sql
CREATE TABLE user_totp (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,

    -- TOTP Secret (암호화 저장)
    secret_encrypted VARCHAR(255) NOT NULL,

    -- 백업 코드 (해시 저장)
    backup_codes_hash JSONB NOT NULL,

    -- 활성화 여부
    verified BOOLEAN NOT NULL DEFAULT FALSE,

    -- 생성일
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## 4. 이메일 OTP

### 4.1 구현 방식

```
1. 로그인 시 2FA 필요 확인
2. 이메일로 6자리 코드 발송
3. 사용자가 입력 → 서버 검증
4. 로그인 완료
```

### 4.2 백엔드 구현

```typescript
// packages/backend/src/services/emailOtp.ts

import crypto from 'crypto';

// 이메일 OTP 생성 및 발송
export async function generateAndSendEmailOTP(userId: string): Promise<void> {
  // 6자리 숫자 코드 생성
  const code = crypto.randomInt(100000, 999999).toString();

  // 5분 유효
  const expiresAt = new Date(Date.now() + 5 * 60 * 1000);

  // DB 저장 (해시)
  await db.emailOTP.create({
    userId,
    codeHash: hash(code),
    expiresAt
  });

  // 이메일 발송
  await sendEmail({
    to: user.email,
    subject: '인증 코드',
    template: 'otp',
    data: { code }
  });
}

// 이메일 OTP 검증
export async function verifyEmailOTP(
  userId: string,
  code: string
): Promise<boolean> {
  const otp = await db.emailOTP.findValid(userId);

  if (!otp) return false;

  // 만료 확인
  if (otp.expiresAt < new Date()) {
    await db.emailOTP.delete({ where: { id: otp.id } });
    return false;
  }

  // 코드 검증
  const isValid = await compare(code, otp.codeHash);

  // 사용한 OTP 삭제
  await db.emailOTP.delete({ where: { id: otp.id } });

  return isValid;
}
```

---

## 5. 로그인 플로우

### 5.1 일반 사용자

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 이메일/비밀번호 로그인                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 2FA 설정 여부 확인                                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
    2FA 미설정                  2FA 설정됨
         │                         │
         ▼                         ▼
┌───────────────┐         ┌─────────────────────────────────┐
│ 로그인 완료    │         │ TOTP 또는 이메일 OTP 요청        │
│ (메인 페이지) │         └─────────────┬───────────────────┘
└───────────────┘                       │
                                         ▼
                              ┌───────────────────────┐
                              │ 6자리 코드 입력        │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │ 코드 검증              │
                              └───────────┬───────────┘
                                          │
                              ┌───────────┴───────────┐
                              │                       │
                              ▼                       ▼
                         성공                      실패
                              │                       │
                              ▼                       ▼
                    ┌───────────────┐         ┌───────────────┐
                    │ 로그인 완료    │         │ 재시도 (3회)   │
                    └───────────────┘         └───────────────┘
```

### 5.2 관리자 (2FA 필수)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 이메일/비밀번호 로그인                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. TOTP 설정 여부 확인                                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
    TOTP 미설정                  TOTP 설정됨
         │                         │
         ▼                         ▼
┌──────────────────┐      ┌───────────────────────┐
│ TOTP 설정 강제    │      │ TOTP 코드 입력 요청    │
│ (설정 페이지로)   │      └───────────┬───────────┘
└──────────────────┘                  │
                                      ▼
                           ┌───────────────────────┐
                           │ TOTP 6자리 코드 입력   │
                           └───────────┬───────────┘
                                       │
                              ┌────────┴────────┐
                              ▼                 ▼
                         성공                실패
                              │                 │
                              ▼                 ▼
                    ┌───────────────┐  ┌───────────────┐
                    │ 관리자 페이지  │  │ 백업 코드 입력 │
                    └───────────────┘  └───────────────┘
```

---

## 6. 프론트엔드 구현

### 6.1 TOTP 설정 페이지

```svelte
<!-- src/routes/my/totp/+page.svelte -->
<script>
  import { onMount } from 'svelte';
  import { invalidate } from '$app/navigation';

  let step = 'setup'; // setup, verify, complete
  let qrCode = '';
  let secret = '';
  let backupCodes = [];
  let totpCode = '';
  let error = '';

  async function setupTOTP() {
    const response = await fetch('/api/auth/totp/setup', {
      method: 'POST'
    });
    const data = await response.json();

    qrCode = data.qrCode;
    secret = data.secret;
    backupCodes = data.backupCodes;
    step = 'verify';
  }

  async function verifyTOTP() {
    const response = await fetch('/api/auth/totp/verify', {
      method: 'POST',
      body: JSON.stringify({ code: totpCode })
    });

    if (response.ok) {
      step = 'complete';
      invalidate('user');
    } else {
      error = '코드가 올바르지 않습니다';
    }
  }

  onMount(() => {
    setupTOTP();
  });
</script>

{#if step === 'verify'}
  <div class="totp-setup">
    <h2>2단계 인증 설정</h2>

    <div class="qr-code">
      <img src={qrCode} alt="QR Code" />
    </div>

    <p>또는 수동으로 입력:</p>
    <code>{secret}</code>

    <form on:submit|preventDefault={verifyTOTP}>
      <input
        type="text"
        bind:value={totpCode}
        placeholder="6자리 코드"
        maxlength="6"
        pattern="[0-9]{6}"
      />
      {#if error}
        <p class="error">{error}</p>
      {/if}
      <button type="submit">확인</button>
    </form>
  </div>
{:else if step === 'complete'}
  <div class="totp-complete">
    <h2>2단계 인증이 설정되었습니다</h2>

    <div class="backup-codes">
      <h3>백업 코드 (안전한 곳에 보관하세요)</h3>
      <ul>
        {#each backupCodes as code}
          <li>{code}</li>
        {/each}
      </ul>
    </div>
  </div>
{/if}
```

### 6.2 2FA 입력 모달

```svelte
<!-- src/routes/+page.svelte -->
<script>
  import { afterNavigate } from '$app/navigation';

  let show2FAModal = false;

  afterNavigate(({ to }) => {
    // 2FA 필요한 페이지 접근 시 모달 표시
    if (to.url.searchParams.has('2fa')) {
      show2FAModal = true;
    }
  });
</script>

{#if show2FAModal}
  <div class="modal-backdrop">
    <div class="modal" role="dialog" aria-modal="true">
      <h2>2단계 인증</h2>
      <p>인증 앱의 6자리 코드를 입력하세요</p>

      <form on:submit|preventDefault={verifyCode}>
        <input
          type="text"
          inputmode="numeric"
          maxlength="6"
          pattern="[0-9]{6}"
          placeholder="000000"
          autofocus
        />
        <button type="submit">확인</button>
      </form>

      <a href="/auth/totp/backup">백업 코드로 로그인</a>
    </div>
  </div>
{/if}
```

---

## 7. 관리자 2FA 강제 설정

### 7.1 미들웨어

```typescript
// packages/backend/src/middleware/admin2fa.ts

export async function requireAdmin2FA(req: Request): Promise<Response> {
  const session = await getSession(req);

  if (!session) {
    return redirect('/admin/login');
  }

  const user = await db.user.findUnique({
    where: { id: session.userId },
    include: { totp: true }
  });

  // 관리자이지만 TOTP 미설정
  if (user.role === 'admin' && !user.totp?.verified) {
    return redirect('/admin/setup-totp');
  }

  // TOTP 설정 완료 시 계속
  return next();
}
```

---

## 8. 비용 분석

### 8.1 월 예상 비용

| 항목 | 건수 | 단가 | 월 비용 |
|-----|------|------|---------|
| 이메일 OTP | 5,000건 | 무료 (AWS SES) | ₩0 |
| TOTP | - | 무료 | ₩0 |
| SMS (대안) | 5,000건 | ₩50건 | ₩250,000 |

**연간 비용:**
- 이메일 + TOTP: ₩0
- SMS만 사용: ₩3,000,000

### 8.2 AWS SES 비용 (이메일)

| 구간 | 가격 |
|-----|------|
| 0-3,000건/월 | 무료 |
| 3,000건 이후 | $0.10/1,000건 |

**예상 월 사용량 5,000건 기준:**
- 3,000건: 무료
- 2,000건: $0.10 × 2 = $0.2
- **월 $0.2 (약 ₩250)**

---

## 9. UX 가이드라인

### 9.1 사용자 메시지

| 상황 | 메시지 |
|-----|--------|
| TOTP 설정 안내 | "계정 보호를 위해 2단계 인증을 설정하세요" |
| 관리자 강제 설정 | "관리자 계정은 2단계 인증이 필수입니다" |
| 코드 발송 | "이메일로 인증 코드를 발송했습니다 (5분 유효)" |
| 코드 만료 | "인증 코드가 만료되었습니다. 다시 요청해주세요" |
| 백업 코드 안내 | "백업 코드는 안전한 곳에 보관하세요. 기기 분실 시 필요합니다" |

### 9.2 에러 처리

| 에러 | 사용자 메시지 | 조치 |
|-----|-------------|------|
| 코드 불일치 | "코드가 올바르지 않습니다" | 재입력 유도 |
| 코드 만료 | "코드가 만료되었습니다" | 재발송 버튼 제공 |
| 시도 초과 | "너무 많은 시도를 했습니다. 잠시 후 다시 시도하세요" | 5분 락 |
| 백업 코드 소진 | "백업 코드를 모두 사용했습니다. 2FA를 재설정하세요" | 고객센터 안내 |
