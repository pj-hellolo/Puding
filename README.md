# 🏆 Cocky

소마고 학생들을 위한 교내 AI 코딩 배틀 플랫폼.
AI가 2일마다 문제를 자동 출제하고, 학생들이 익명/실명으로 풀면 AI가 피드백과 점수를 주고,
다양한 기준으로 랭킹을 보여준다.

## 👾 Team

> 김민준 — Design  
> 정성원 — Backend  
> 양지우 — Frontend  
> 임서하 — AI  
> 이겸이 — Project Support  
> 김민욱 — Project Support 

## 기술 스택
- **Frontend**: React + Vite (전역 상태는 Context API)
- **Backend**: FastAPI + PostgreSQL + SQLAlchemy + Alembic
- **AI**: OpenAI API (작업별 모델 분리 — 생성/주간/월간 `gpt-5.4-mini`, 즉시/2일 `gpt-5.4-nano`)
- **채점**: Judge0 API
- **배포**: Vercel (프론트) + Railway (백엔드)

---

## 빠른 시작 (로컬)

### 1. 백엔드

```bash
cd backend
python -m venv .venv
.venv\Scripts\activate          # Windows (mac/linux: source .venv/bin/activate)
pip install -r requirements.txt

copy .env.example .env          # 값 채우기 (DATABASE_URL 필수)
# 키 없이 UI만 둘러볼 경우 OPENAI_API_KEY / JUDGE0_API_KEY 는 비워둬도 됨

python seed.py                  # 데모 데이터 생성 (관리자/학생/샘플 문제/제출)
uvicorn app.main:app --reload   # http://localhost:8000  (문서: /docs)
```

데모 계정:
- 관리자 `admin@somago.hs.kr` / `admin1234`
- 학생 `student1@somago.hs.kr` ~ `student6@somago.hs.kr` / `test1234`

### 2. 프론트엔드

```bash
cd frontend
npm install
copy .env.example .env          # VITE_API_BASE_URL=http://localhost:8000
npm run dev                     # http://localhost:5173
```

---

## 핵심 동작

### 출제 스케줄 (`app/core/scheduler.py`)
| 회차 | 공개 | 마감 |
|------|------|------|
| 1회차 | 월 0시 | 화 24시 |
| 2회차 | 수 0시 | 목 24시 |
| 3회차 | 금 0시 | 토 24시 |
| 일요일 | 문제 없음 (랭킹·피드백 열람) | |

- 매일 **23시**: 다음날 회차 문제 자동 생성 (Python/C/Java × Easy/Normal/Hard 각 1개, 총 9문제)
- 매 분: 마감 10분 전 경고 알림 브로드캐스트
- 화/목/토 0시: 직전 회차 2일 피드백 생성
- 일요일 0시: 주간 피드백 (+ 4번째 일요일이면 월간 피드백)

### AI 문제 생성 3단계 필터 (`app/services/ai_generator.py`)
1. **유사도 검사** — 기존 문제와 80% 이상 유사하면 재생성
2. **정답 검증** — AI가 정답 코드 생성 → Judge0 실행해 예제 통과 확인
3. **난이도 재확인** — AI에게 난이도 되물어 의도한 버전과 일치하는지 확인

→ 3회 재시도 후에도 실패 시 관리자 알림 (`notification.notify_admin_generation_failure`)

### 점수 체계
| 난이도 | 기본 | AI 피드백 | 최대 |
|--------|------|-----------|------|
| Easy | 30.00 | 30.00 | 60.00 |
| Normal | 50.00 | 30.00 | 80.00 |
| Hard | 70.00 | 30.00 | 100.00 |

AI 피드백 = 시간복잡도 효율(10) + 코드 가독성(10) + 풀이 독창성(10).
모든 점수는 소수점 2자리. 같은 문제는 **마지막 제출**만 점수에 반영.

### 랭킹 (`app/services/ranking.py`)
별도 집계 테이블 없이 `submissions.submitted_at` 기준으로 직접 집계.
각 (user, problem)의 마지막 제출만 합산.
- 기간: 2일 / 주간 / 월간
- 분류: 전교 통합 / 학년별 / 반 대항전(평균) / 반 내 개인
- 월간 랭킹은 4번째 주 일요일에만 공개, 그 외에는 잠금 + D-day

---

## 디렉토리
```
soma-battle/
├── backend/   FastAPI (app/api, app/services, app/models, app/core)
└── frontend/  React (src/pages, src/components, src/api, src/store, src/hooks)
```

## Repositories

- **Client** : https://github.com/pj-hellolo/Hellolo-Client
- **Server** : https://github.com/pj-hellolo/Hellolo-Server
- **AI** : https://github.com/pj-hellolo/Hellolo-AI

---

## 구현 메모 / 스펙과의 차이
- **스타일링**: 별도 CSS 프레임워크 없이 CSS 변수 + 유틸리티 클래스(`src/styles/global.css`)로
  디자인 시스템을 구현했다. 컬러 팔레트/타이포/컴포넌트 스타일은 스펙 그대로 반영.
- **전역 상태**: React Context API(`src/store/authStore.jsx`, `rankingStore.jsx`)로 구현.
  호출 시그니처는 `useAuthStore((s) => s.user)`처럼 셀렉터를 그대로 받도록 유지.
- **알림**: 인메모리 큐 + 폴링(`notification.py`, Navbar 30초 폴링)으로 구현.
  실서비스에서는 WebSocket/푸시/이메일로 교체 가능하도록 인터페이스만 통일.
- **인증**: JWT(localStorage) 기반. 익명 토글 시 AI 스타일 닉네임 자동 생성.
- **DB 테이블 생성**: 개발 편의를 위해 startup에서 `create_all`. 운영은 Alembic 사용.
- **뱃지/연속참여**: 언어 마스터·Hard 도전 뱃지는 제출 기록으로 클라이언트 계산.
  연속 참여/랭킹 뱃지는 서버 집계 로직 추가 시 확장 가능.

