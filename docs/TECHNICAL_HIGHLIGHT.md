# 기술적 하이라이트

## 1. 크로스 플랫폼 호환성

### iOS/Android 클립보드 처리

구형 iOS 웹뷰에서는 Clipboard API가 지원되지 않아 폴백 로직을 구현했습니다.

```javascript
function fallbackCopyTextToClipboard(text) {
  const el = document.createElement('textarea');
  el.value = text;
  el.style.position = 'fixed';
  el.style.opacity = '0';
  
  document.body.appendChild(el);
  el.focus();
  el.select();
  
  const success = document.execCommand('copy');
  document.body.removeChild(el);
  
  if (success) alert('복사되었습니다.');
}

function copy(text) {
  if (navigator?.clipboard) {
    navigator.clipboard.writeText(text)
      .catch(() => fallbackCopyTextToClipboard(text));
  } else {
    fallbackCopyTextToClipboard(text);
  }
}
```

### 네이티브 지도 앱 연동

디바이스를 감지하여 모바일에서는 네이티브 앱을, PC에서는 웹을 실행합니다.

```javascript
const handleMap = () => {
  const device = checkMobile(); // 'ios', 'android', 'pc'
  
  if (type === "kakao") {
    if (device === "ios" || device === "android") {
      window.open(`kakaomap://place?id=${config.kakaoPlaceId}`);
    } else {
      window.open(config.weddingMapLink);
    }
  }
}
```

---

## 2. 아키텍처 설계

### Blue-Green 무중단 배포

결혼식 날짜가 정해진 서비스 특성상 배포 중 서비스 중단이 불가능하여 Blue-Green 배포를 구현했습니다.

```
GitHub Actions → Docker Build → ECR
                                  ↓
                          ┌──────────────┐
                          │ Nginx Proxy  │
                          └──────┬───────┘
                                 │
                      ┌──────────┴──────────┐
                      ↓                     ↓
                 Server A              Server B
                 [Active]              [Standby]
```

**Switch API 구현**:

```typescript
async function switchServer(target: 'A' | 'B') {
  // 1. Nginx 설정 백업
  await execPromise(`sudo cp ${NGINX_CONF} ${NGINX_BACKUP}`);
  
  // 2. 새 설정 적용
  const newConfig = target === 'A' ? NGINX_A_CONF : NGINX_B_CONF;
  await execPromise(`sudo cp ${newConfig} ${NGINX_CONF}`);
  
  // 3. 설정 검증
  const { stderr } = await execPromise('sudo nginx -t');
  if (stderr && !stderr.includes('successful')) {
    await execPromise(`sudo cp ${NGINX_BACKUP} ${NGINX_CONF}`);
    throw new Error('Config validation failed');
  }
  
  // 4. Reload
  await execPromise('sudo systemctl reload nginx');
}
```

**배포 프로세스**:
1. 새 버전을 Standby 서버에 배포
2. 헬스체크 통과 확인
3. Switch API로 트래픽 전환
4. 기존 Active 서버는 Standby로 전환 (롤백 가능)

### AWS Lambda 자동 알림

결혼식 3시간 전 직원 알림을 자동화했습니다.

```javascript
async function sendSMS(phoneNumbers, message) {
  const dateTime = new Date().toISOString();
  const salt = crypto.randomBytes(16).toString('hex');
  const signature = generateSignature(apiSecret, dateTime, salt);
  
  const messageType = Buffer.byteLength(message, 'utf-8') <= 90 ? 'SMS' : 'LMS';
  
  await axios.post(
    'https://api.solapi.com/messages/v4/send-many/detail',
    {
      messages: phoneNumbers.map(phone => ({
        to: phone,
        from: senderPhone,
        text: message,
        type: messageType
      }))
    }
  );
}
```

### 멀티 테넌트 아키텍처

각 커플의 데이터를 격리하면서 공통 코드를 재사용합니다.

```java
@Entity
public class WeddingInvitationConfig {
    @Id
    private Long id;  // Tenant ID
    
    @OneToMany(mappedBy = "config")
    private List<InvitationPhoto> photos;
}

@Entity
public class GuestListLog {
    @Id
    private Long id;
    
    @Column(name = "wedding_invitation_config_id")
    private Long weddingInvitationConfigId;  // 데이터 격리
}
```

**JWT 기반 접근 제어**:

```java
@GetMapping("/list")
public ResponseEntity<Page<GuestListLog>> getGuestList(
    @RequestHeader("Authorization") String token,
    Pageable pageable
) {
    Long customerId = jwtService.getCustomerIdFromToken(token);
    Customer customer = customerService.findById(customerId);
    Long configId = customer.getWeddingInvitationConfigId();
    
    // Foreign Key로 본인 데이터만 조회
    Page<GuestListLog> guests = guestService.findByConfigId(configId, pageable);
    return ResponseEntity.ok(guests);
}
```

### Request Counter 기반 로딩 관리

Axios Interceptor에서 진행 중인 요청을 카운팅하여 로딩 상태를 자동 관리합니다.

```typescript
let loadingCount = 0;
let loadingTimeout: NodeJS.Timeout;

// Request Interceptor
axiosInstance.interceptors.request.use((config) => {
  loadingCount++;
  
  if (loadingTimeout) clearTimeout(loadingTimeout);
  
  // 100ms 디바운스로 깜빡임 방지
  loadingTimeout = setTimeout(() => {
    if (loadingCount > 0) {
      updateLoadingState(true);
    }
  }, 100);
  
  return config;
});

// Response Interceptor
axiosInstance.interceptors.response.use(
  (response) => {
    loadingCount = Math.max(0, loadingCount - 1);
    if (loadingCount === 0) {
      updateLoadingState(false);
    }
    return response;
  }
);
```

---

## 3. DevOps

### 환경별 Docker Compose

- `docker-compose.yml`: 로컬 개발
- `docker-compose-dev.yml`: Dev 서버
- `docker-compose-prod.yml`: Prod A
- `docker-compose-prod-b.yml`: Prod B

### CI/CD 자동화

```yaml
name: Deploy to EC2

on:
  workflow_dispatch:
    inputs:
      branch:
        required: true
        default: 'prod'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Build and Push Docker image
      - name: Deploy to EC2
```

---

## 주요 성과

- 무중단 배포로 서비스 안정성 확보
- Lambda 기반 자동 알림으로 운영 효율화
- 멀티 테넌트 아키텍처로 확장성 확보
- Request Counter 패턴으로 로딩 상태 중앙 관리
