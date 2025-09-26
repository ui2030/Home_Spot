# Home_Spot – 부동산 거래 웹 서비스 (Node.js + MySQL)
Admin/Agent/Tenant 멀티 롤을 가진 부동산 거래 플랫폼입니다.
매물 등록·승인·검색·대시보드, 중개사 승인(자격증 업로드) 및 승인/거절 자동 메일(AWS SES), 실시간 1:1 채팅(Socket.IO), 실시간 Chat 봇(Ollama), CSV 내보내기(UTF-8 BOM) 등 운영에 필요한 전 기능 흐름을 구현했습니다.

🔧 시연 영상 링크 :

[웹 서비스 시연]()

## 핵심 기능

### 인증/권한

**ㆍ**bcrypt 해시 저장, httpOnly 세션 + JWT 병행

**ㆍ**로그인 후 역할 기반 라우팅: /admin · /agent · /tenant

**ㆍ**비밀번호 재설정: 토큰 발급 → sha256 해시 저장 → SES 메일 링크

### 중개사 승인

**ㆍ**가입 시 자격증 이미지 업로드 → pending 등록

**ㆍ**관리자 승인/거절 처리 시 자동 안내 메일 발송(AWS SES)

**ㆍ**대기/승인/거절 탭형 대시보드로 상태 가시화

### 매물/이미지

**ㆍ**매물 CRUD + 이미지 최대 N장 업로드, 대표 이미지 지정

**ㆍ**소유권 검증, 목록/삭제 API, 정적 제공 경로 분리(multer)

### 검색/목록

**ㆍ**검색 ?q= + 페이지네이션 page/pageSize
**ㆍ**CSV 내보내기(UTF-8 BOM)로 엑셀 한글 깨짐 방지

### 실시간 채팅

**ㆍ**Socket.IO 기반 사용자–중개사 1:1 상담

**ㆍ**대화 로그 저장, 알림 연동(선택)

### 기술 스택

**ㆍ**Language: JavaScript (Node.js)

**ㆍ**DB: MySQL (mysql2/promise)

**ㆍ**Framework/Libs: Express, multer, bcrypt, JSON Web Token, express-session, Socket.IO, csv-writer(or stream)

**ㆍ**Infra/Others: AWS SES, dotenv, (선택) Nginx/PM2

## 폴더 구조
```css
Home_Spot/
├─ server.js                # 앱 부트스트랩/미들웨어/라우팅 연결
├─ db.js                    # mysql2/promise 커넥션 풀(+ ping)
├─ routes/
│  ├─ auth.js               # 로그인/로그아웃/비번 재설정
│  ├─ agents.js             # 중개사 CRUD/승인 요청
│  ├─ admin.js              # 승인/거절, 대시보드
│  ├─ listings.js           # 매물 CRUD/검색/CSV
│  └─ chat.js               # 채팅 소켓 핸들러
├─ services/                # 비즈니스 로직
├─ dao/                     # DB 접근(프리페어드 쿼리)
├─ public/                  # 정적 제공(이미지 등)
├─ uploads/                 # 업로드 임시/영구 저장소
├─ .env                     # 환경변수 (개발/운영 분리)
└─ README.md
```

## 빠른 실행
### 1) 사전 준비

**ㆍ**Node.js LTS, MySQL

**ㆍ**AWS SES: 발신 주소(검증) & 리전 준비

### 2) 환경 변수 (.env 예시)
```cs
# App
PORT=3000
BASE_URL=http://localhost:3000
SESSION_SECRET=change_me

# DB
DB_HOST=127.0.0.1
DB_USER=homespot
DB_PASS=homespot_pw
DB_NAME=homespot

# AWS SES
AWS_REGION=ap-northeast-2
SES_FROM=you@example.com
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...

# 기타
CSV_BOM=true
JWT_SECRET=change_me_too
```
### 3) 설치 & 실행

**ㆍ**npm i

**ㆍ**npm run start   # 또는 npm run dev

**ㆍ**콘솔 로그 예: [DB] ping ok: { '1': 1 } / Server running at http://localhost:3000

## DB 스키마
```sql
-- 0. 관리자 사용자
CREATE TABLE admins(
  admin_id INT AUTO_INCREMENT PRIMARY KEY, 
  username VARCHAR(100) NOT NULL,
  password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 1. 중개사 사용자 (중개사 고유번호, 중개사 아이디, 닉네임, 이메일, 비밀번호,
--                  라이선스 이미지 URL, 승인 상태[대기/승인/반려], 프로필 사진, 생성 시간)
CREATE TABLE agents (
  agent_id        INT AUTO_INCREMENT PRIMARY KEY,                -- PK
  agent_name      VARCHAR(64)  NOT NULL UNIQUE,                  -- 중개사 아이디(로그인/표시)
  nickname        VARCHAR(64) NULL, 
  email           VARCHAR(100) NOT NULL UNIQUE,                  -- 이메일
  password        VARCHAR(200) NOT NULL,                         -- 암호 해시(예: bcrypt)
  license_url     TEXT         NOT NULL,                         -- 자격증 이미지 URL
  license_status  ENUM('pending','approved','rejected')          -- 승인 상태
                  DEFAULT 'pending',               
  profile_url     TEXT NULL,                                     -- 프로필 이미지 URL
  created_at      TIMESTAMP    DEFAULT CURRENT_TIMESTAMP         -- 생성 시각
 
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 2. 사용자 (유저 고유번호, 유저 아이디, 닉네임, 이메일, 비밀번호, 프로필 사진, 생성 시간)
CREATE TABLE users (
  user_id     INT AUTO_INCREMENT PRIMARY KEY,                    -- PK
  user_name    VARCHAR(64)  NOT NULL UNIQUE,                      -- 유저 아이디
  nickname    VARCHAR(64) NULL,
  email       VARCHAR(100) NOT NULL UNIQUE,                      -- 이메일
  password    VARCHAR(200) NOT NULL,                             -- 암호 해시
  profile_url TEXT NULL,                                         -- 프로필 이미지 URL
  created_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP             -- 생성 시각
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 3. 매물 (매물 번호, 중개사 아이디, 건물명, 설명,
--         자산 유형[아파트/오피스텔/연립·다세대/단독],
--         거래 유형[매매/전세/월세/분양권/입주권(=매매성격)],
--         면적[전용, 연면적, 대지권면적, 대지면적], 층, 건축년도,
--         주소지[시, 시군구, 구, 동, 도로명, 번지(본-부)],                  -- [추가: 시/구/동]
--         행정코드[법정동코드(b_code), 행정동코드(h_code)],                  -- [추가]
--         위치좌표[위도(lat), 경도(lng)],                                    -- [추가]
--         파생키[주소키(addr_key), 가격키(price_key_10k)],                    -- [추가]
--         편의[주차여부(has_parking), 반려동물가능(allow_pet)],               -- [추가]
--         금액(만원: 매매/보증금/월세/연세, 실효월세[rent_effective_monthly_10k]), -- [추가]
--         상태[등록/체결/내림], 등록일)
CREATE TABLE listings (
  listing_id      BIGINT AUTO_INCREMENT PRIMARY KEY,
  agent_id        INT NOT NULL,
  title           VARCHAR(200) NOT NULL,
  description     TEXT,

  -- 구분
  property_type   ENUM('apartment','officetel','rowhouse','detached') NOT NULL,
  deal_type       ENUM('sale','jeonse','monthly','presale_right','occupancy_right') NOT NULL,

  -- 주소(번지는 단일 컬럼)
  sigungu         VARCHAR(40)  NOT NULL,
  road_name       VARCHAR(200) NULL,
  lot_no          VARCHAR(32)  NOT NULL,
  si              VARCHAR(20)  NULL,           -- [MERGED]
  gu              VARCHAR(20)  NULL,           -- [MERGED]
  dong            VARCHAR(40)  NULL,           -- [MERGED]
  b_code          CHAR(10)     NULL,           -- [MERGED] 법정동코드
  h_code          CHAR(10)     NULL,           -- [MERGED] 행정동코드

  -- 위치 좌표
  lat             DECIMAL(10,7) NULL,          -- [MERGED]
  lng             DECIMAL(10,7) NULL,          -- [MERGED]

  -- 면적
  area_m2         DECIMAL(10,4) NULL,
  gross_area_m2   DECIMAL(10,4) NULL,
  land_share_m2   DECIMAL(10,4) NULL,
  land_area_m2    DECIMAL(10,4) NULL,
  area_key_m2     DECIMAL(10,4)
                  GENERATED ALWAYS AS (
                    CASE WHEN property_type='detached'
                         THEN COALESCE(land_area_m2, gross_area_m2, area_m2)
                         ELSE area_m2 END
                  ) VIRTUAL,

  -- 기타 특성
  floor           SMALLINT NULL,
  built_year      SMALLINT NULL,

  -- 금액(만원)
  price_sale_10k  INT NULL,
  deposit_10k     INT NULL,
  rent_10k        INT NULL,
  rent_pay_cycle  ENUM('monthly','yearly') NOT NULL DEFAULT 'monthly',
  rent_annual_10k INT NULL,
  rent_effective_monthly_10k INT
                  GENERATED ALWAYS AS (
                    CASE
                      WHEN deal_type='monthly' AND rent_pay_cycle='yearly' AND rent_annual_10k IS NOT NULL
                        THEN CEIL(rent_annual_10k / 12)
                      ELSE rent_10k
                    END
                  ) VIRTUAL,

  -- 파생 키
  addr_key        VARCHAR(255)
                  GENERATED ALWAYS AS (          -- [MERGED]
                    TRIM(CONCAT_WS(' ',
                      COALESCE(si, ''), COALESCE(sigungu, ''), COALESCE(road_name, ''), COALESCE(lot_no, '')
                    ))
                  ) STORED,
  price_key_10k   INT                             -- [MERGED]
                  GENERATED ALWAYS AS (
                    CASE
                      WHEN deal_type IN ('sale','presale_right','occupancy_right') THEN price_sale_10k
                      WHEN deal_type='jeonse' THEN deposit_10k
                      ELSE NULL
                    END
                  ) STORED,

  -- 편의시설
  has_parking     TINYINT(1) NOT NULL DEFAULT 0,  -- [MERGED]
  allow_pet       TINYINT(1) NOT NULL DEFAULT 0,  -- [MERGED]

  status          ENUM('active','sold','removed') DEFAULT 'active',
  created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  -- FK
  CONSTRAINT fk_list_agent FOREIGN KEY (agent_id)
    REFERENCES agents(agent_id) ON DELETE CASCADE,

  -- 거래유형별 금액 유효성 체크
  CONSTRAINT chk_prices_by_dealtype CHECK (
    (deal_type='sale'
      AND price_sale_10k IS NOT NULL
      AND deposit_10k IS NULL
      AND rent_10k IS NULL
      AND rent_annual_10k IS NULL) OR

    (deal_type='jeonse'
      AND deposit_10k IS NOT NULL
      AND rent_10k IS NULL
      AND rent_annual_10k IS NULL) OR

    (deal_type IN ('presale_right','occupancy_right')
      AND price_sale_10k IS NOT NULL
      AND deposit_10k IS NULL
      AND rent_10k IS NULL
      AND rent_annual_10k IS NULL) OR

    (deal_type='monthly' AND (
        (rent_pay_cycle='monthly' AND deposit_10k IS NOT NULL AND rent_10k IS NOT NULL AND rent_annual_10k IS NULL) OR
        (rent_pay_cycle='yearly'  AND deposit_10k IS NOT NULL AND rent_annual_10k IS NOT NULL AND rent_10k IS NULL)
    ))
  ),

  -- 인덱스(모두 sido 제거)
  INDEX ix_list_type       (property_type, deal_type),
  INDEX ix_list_region     (sigungu),
  INDEX ix_list_lot        (sigungu, lot_no),
  INDEX ix_list_price      (price_sale_10k, deposit_10k, rent_10k),
  INDEX ix_list_agent      (agent_id),
  INDEX ix_list_status     (status),
  INDEX ix_list_created    (created_at),
  INDEX ix_list_rent_eff   (rent_effective_monthly_10k),
  INDEX ix_list_codes      (b_code, h_code),      -- [MERGED]
  INDEX ix_list_geo        (lat, lng),            -- [MERGED]
  INDEX ix_list_addr_key   (addr_key),            -- [MERGED]
  INDEX ix_list_price_key  (price_key_10k)        -- [MERGED]
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 4. 매물 이미지 (이미지 번호, 매물 번호, 이미지 URL, 등록일)
CREATE TABLE listing_images (
  image_id   BIGINT AUTO_INCREMENT PRIMARY KEY,                 -- PK
  listing_id BIGINT NOT NULL,                                   -- FK → listings.listing_id
  image_url  TEXT NOT NULL,                                     -- 이미지 URL
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,               -- 등록일
  CONSTRAINT fk_img_listing FOREIGN KEY (listing_id) REFERENCES listings(listing_id) ON DELETE CASCADE,
  INDEX idx_img_listing (listing_id)                            -- 매물별 이미지 조회
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 5. 사용자 이용 로그 [검색, 클릭, 찜, 문의]
CREATE TABLE user_logs (
  log_id      BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id     INT NOT NULL,                                      -- FK → users.user_id
  listing_id  BIGINT NULL,                                       -- FK → listings.listing_id (검색 로그 등은 NULL 가능)
  action_type ENUM('search','click','favorite','inquiry') NOT NULL,
  action_value JSON NULL,                                        -- 예: {"q":"관악구 전세","pos":3} / {"from":"list","section":"top"}
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT fk_logs_user    FOREIGN KEY (user_id)    REFERENCES users(user_id)      ON DELETE CASCADE,
  CONSTRAINT fk_logs_listing FOREIGN KEY (listing_id)  REFERENCES listings(listing_id) ON DELETE SET NULL,

  INDEX idx_logs_user_time (user_id, created_at),                 -- 유저별 타임라인
  INDEX idx_logs_listing   (listing_id),                          -- 매물별 반응
  INDEX idx_logs_action    (action_type)                          -- 액션별 집계
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 6. 알림(찜/조건알림)
CREATE TABLE alerts (
  alert_id      BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id       INT NOT NULL,
  property_type ENUM('apartment','officetel','rowhouse','detached') NULL,
  deal_type     ENUM('sale','jeonse','monthly','presale_right','occupancy_right') NULL,
  min_price_10k INT NULL,
  max_price_10k INT NULL,
  min_area_m2   DECIMAL(10,4) NULL,
  max_area_m2   DECIMAL(10,4) NULL,
  is_active     TINYINT NOT NULL DEFAULT 1,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_alerts_user FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
  INDEX idx_alerts_user_active (user_id, is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 7. 알림이 커버하는 '동(법정동코드)' 목록 (반경을 동 집합으로 확장해 저장)
CREATE TABLE alert_regions (
  alert_id  BIGINT   NOT NULL,
  bjd_code  CHAR(10) NOT NULL,         -- 카카오 b_code 또는 법정동코드
  PRIMARY KEY (alert_id, bjd_code),
  CONSTRAINT fk_alert_regions_alert FOREIGN KEY (alert_id)
    REFERENCES alerts(alert_id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 8. 채팅 방 (채팅 방 번호, 중개사 아이디, 유저 아이디, 생성 시간)
CREATE TABLE chat_rooms (
  room_id    BIGINT AUTO_INCREMENT PRIMARY KEY,                 -- PK
  agent_id   INT NOT NULL,                                      -- FK → agents.agent_id
  user_id    INT NOT NULL,                                      -- FK → users.user_id
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,               -- 생성 시각
  CONSTRAINT fk_room_agent FOREIGN KEY (agent_id) REFERENCES agents(agent_id) ON DELETE CASCADE,
  CONSTRAINT fk_room_user  FOREIGN KEY (user_id)  REFERENCES users(user_id)  ON DELETE CASCADE,
  INDEX idx_room_agent (agent_id),                              -- 중개사별 방 조회
  INDEX idx_room_user  (user_id)                                -- 유저별 방 조회
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 9. 메시지 로그 (메시지 아이디, 방 번호, 발신자 타입[중개사/유저], 내용, 전송 시간)
CREATE TABLE chat_messages (
  message_id   BIGINT AUTO_INCREMENT PRIMARY KEY,               -- PK
  room_id      BIGINT NOT NULL,                                 -- FK → chat_rooms.room_id
  sender_type  ENUM('agent','user') NOT NULL,                   -- 발신자 구분
  message_text TEXT NOT NULL,                                   -- 메시지 내용
  sent_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,             -- 전송 시각
  CONSTRAINT fk_msg_room FOREIGN KEY (room_id) REFERENCES chat_rooms(room_id) ON DELETE CASCADE,
  INDEX idx_msg_room (room_id),                                 -- 방별 메시지 조회
  INDEX idx_msg_time (sent_at)                                  -- 시간순 정렬
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 10. 패스워드 리셋 (패스워드 아이디, 유저 타입,
CREATE TABLE password_resets (
  pwr_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_type ENUM('tenant','agent') NOT NULL,  -- users: tenant, agents: agent
  user_id INT NOT NULL,
  email VARCHAR(100) NOT NULL,
  token_hash CHAR(64) NOT NULL,               -- SHA-256(hex) of token
  expires_at DATETIME NOT NULL,               -- 만료(예: 1시간)
  used TINYINT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  used_at DATETIME NULL,
  INDEX idx_token_hash (token_hash),
  INDEX idx_email (email)
);
```

## 사용 흐름

**1.** 사용자 가입/로그인 → 역할에 따라 /admin · /agent · /tenant 이동

**2.** 중개사: 자격증 이미지 업로드 → pending 등록

**3.** 관리자: 승인/거절 처리 → SES 메일 자동 발송

**4.** 중개사: 매물 등록/이미지 업로드(대표 설정)

**5.** 사용자: 검색(q) + 페이지네이션으로 매물 탐색 → 상세 보기

**6.** 필요 시 CSV 내보내기(UTF-8 BOM)로 운영 리포트 생성

**7.** 사용자–중개사 실시간 1:1 채팅 (문의/상담)

## 설계 포인트

**ㆍ**계층 분리: Router / Service / DAO로 분리, 입력 검증은 미들웨어 공통 처리

**ㆍ**인증 전략 병행: 웹은 세션, API/비동기는 JWT → 역할 클레임으로 접근 제어

**ㆍ**파일 스토리지 분리: multer 업로드 경로와 정적 제공 경로 분리, 삭제 시 파일·DB 동기화

**ㆍ**검색 성능: 인덱스 가능한 컬럼 설계 + 페이지네이션으로 응답 안정화

**ㆍ**CSV 호환성: UTF-8 BOM + 적절한 Content-Type으로 엑셀 한글 깨짐 방지

**ㆍ**운영 분리: dotenv로 개발/운영 환경 변수 분리, 커넥션 풀 사용

## 문제 해결 사례

**ㆍ**SES 서명 불일치: 리전·자격 증명 재설정, 발신자 검증으로 메일 발송 정상화

**ㆍ**CSV 한글 깨짐: BOM 추가 및 헤더 지정으로 엑셀 호환 확보

**ㆍ**세션 보안: httpOnly + SameSite, 비번 재설정 단발성 토큰 + 만료 시간 적용

**ㆍ**검색 지연: LIKE 남용 구간 파악 → 키 컬럼 인덱싱/전처리로 지연 감소

## 예시 코드
```css
// db.js — 커넥션 풀 + 연결 확인
import mysql from 'mysql2/promise';
export const pool = mysql.createPool({ host: process.env.DB_HOST, user: process.env.DB_USER,
  password: process.env.DB_PASS, database: process.env.DB_NAME, waitForConnections: true });
const [[ping]] = await pool.query('SELECT 1 AS `1`');
console.log('[DB] ping ok:', ping);

// routes/admin.js — 승인/거절
router.post('/admin/agents/decision', async (req, res) => {
  const { agent_id, decision } = req.body;                  // approve | reject
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    const status = decision === 'approve' ? 'approved' : 'rejected';
    const [r] = await conn.execute(
      "UPDATE agents SET status=?, reviewed_at=NOW() WHERE agent_id=? AND status='pending'",
      [status, agent_id]
    );
    if (!r.affectedRows) throw new Error('Invalid state');
    await conn.commit();
    // 커밋 후 SES 메일 발송(요약)
    res.json({ ok: true, status });
  } catch (e) {
    try { await conn.rollback(); } catch {}
    res.status(500).json({ ok: false, msg: e.message });
  } finally { conn.release(); }
});
```
## 기여

이슈·PR 환영합니다. 버그/개선 제안은 Issues에 등록해 주세요.

## 라이선스 & 연락처

**ㆍ**🔧 라이선스: MIT 등 원하는 라이선스로 LICENSE 추가

**ㆍ**🔧 Maintainer: Cheonui Kim / kimcjsdml@gmail.com

**ㆍ**저장소: https://github.com/ui2030/Home_Spot

## 참고

**ㆍ** README에 있는 시연 영상은 (YouTube) 링크가 포함되어 있습니다.
