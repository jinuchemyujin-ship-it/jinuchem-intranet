# 지누켐(Jinuchem) 인트라넷 — ERD (Entity Relationship Diagram)

> **버전**: v1.1
> **작성일**: 2026-04-01
> **참고**: Slack/Jira/Confluence/Google Workspace는 외부 SaaS → 자체 DB 불필요, 연동 설정만 저장

---

## 1. ERD 개요

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   사용자      │────<│   부서        │>────│   직급        │
│  (users)     │     │(departments) │     │  (positions) │
└──────┬───────┘     └──────────────┘     └──────────────┘
       │
       ├────────────────────────────────────────────┐
       │                    │                        │
       ▼                    ▼                        ▼
┌──────────────┐  ┌──────────────┐         ┌──────────────┐
│  전자결재     │  │   근태기록    │         │   게시글      │
│ (approvals)  │  │(attendances) │         │   (posts)    │
└──────┬───────┘  └──────┬───────┘         └──────┬───────┘
       │                 │                        │
       ▼                 ▼                        ▼
┌──────────────┐  ┌──────────────┐         ┌──────────────┐
│  결재선       │  │   연차관리    │         │   댓글        │
│(approval_    │  │  (leaves)    │         │  (comments)  │
│  lines)      │  └──────────────┘         └──────────────┘
└──────────────┘
```

---

## 2. 핵심 엔티티 상세

### 2.1 사용자 관리

#### `users` (사용자)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | 사용자 고유 ID |
| `employee_no` | VARCHAR(20) | UNIQUE, NOT NULL | 사번 |
| `name` | VARCHAR(50) | NOT NULL | 이름 |
| `email` | VARCHAR(100) | UNIQUE, NOT NULL | 이메일 (Google Workspace 계정 = 로그인 ID) |
| `password_hash` | VARCHAR(255) | | 비밀번호 해시 (Google SSO 사용 시 NULL) |
| `google_id` | VARCHAR(100) | UNIQUE | Google OAuth sub (SSO 연동) |
| `slack_user_id` | VARCHAR(20) | UNIQUE | Slack 사용자 ID (예: U03C4CT4TQR) |
| `jira_account_id` | VARCHAR(50) | UNIQUE | Jira/Confluence 계정 ID |
| `phone` | VARCHAR(20) | | 연락처 |
| `extension` | VARCHAR(10) | | 내선번호 |
| `department_id` | UUID | FK → departments.id | 소속 부서 |
| `position_id` | UUID | FK → positions.id | 직급 |
| `role` | ENUM | NOT NULL | 'admin', 'executive', 'manager', 'employee' |
| `profile_image` | VARCHAR(500) | | 프로필 이미지 URL |
| `hire_date` | DATE | NOT NULL | 입사일 |
| `birth_date` | DATE | | 생년월일 |
| `status` | ENUM | NOT NULL, DEFAULT 'active' | 'active', 'inactive', 'leave' |
| `annual_leave_total` | DECIMAL(4,1) | DEFAULT 15.0 | 연간 연차 총일수 |
| `annual_leave_used` | DECIMAL(4,1) | DEFAULT 0 | 사용한 연차일수 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | 생성일시 |
| `updated_at` | TIMESTAMP | | 수정일시 |

#### `departments` (부서)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | 부서 고유 ID |
| `name` | VARCHAR(100) | NOT NULL | 부서명 |
| `code` | VARCHAR(20) | UNIQUE | 부서코드 (예: CHEM-R&D) |
| `parent_id` | UUID | FK → departments.id | 상위 부서 (트리 구조) |
| `division` | ENUM | NOT NULL | 'chemical', 'it', 'management' |
| `leader_id` | UUID | FK → users.id | 부서장 |
| `sort_order` | INT | DEFAULT 0 | 정렬 순서 |
| `is_active` | BOOLEAN | DEFAULT TRUE | 활성 여부 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `positions` (직급)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | 직급 고유 ID |
| `name` | VARCHAR(50) | NOT NULL | 직급명 (사원/주임/대리/과장/차장/부장/이사/대표) |
| `level` | INT | NOT NULL | 직급 레벨 (1=사원 ~ 8=대표) |
| `sort_order` | INT | DEFAULT 0 | 정렬 순서 |

---

### 2.2 전자결재

#### `approval_templates` (결재 양식)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | 양식 고유 ID |
| `name` | VARCHAR(100) | NOT NULL | 양식명 (일반기안서, 휴가신청서 등) |
| `code` | VARCHAR(30) | UNIQUE | 양식코드 |
| `category` | ENUM | NOT NULL | 'general', 'leave', 'expense', 'purchase', 'trip', 'education', 'research', 'meeting' |
| `form_schema` | JSON | NOT NULL | 양식 필드 정의 (JSON Schema) |
| `is_active` | BOOLEAN | DEFAULT TRUE | 사용 여부 |
| `sort_order` | INT | DEFAULT 0 | |

#### `approvals` (결재 문서)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | 결재 문서 고유 ID |
| `doc_number` | VARCHAR(30) | UNIQUE, NOT NULL | 문서번호 (APV-2026-0001) |
| `template_id` | UUID | FK → approval_templates.id | 양식 |
| `title` | VARCHAR(200) | NOT NULL | 제목 |
| `content` | JSON | NOT NULL | 양식 데이터 (JSON) |
| `drafter_id` | UUID | FK → users.id, NOT NULL | 기안자 |
| `department_id` | UUID | FK → departments.id | 기안 부서 |
| `status` | ENUM | NOT NULL, DEFAULT 'draft' | 'draft', 'pending', 'in_progress', 'approved', 'rejected', 'withdrawn' |
| `priority` | ENUM | DEFAULT 'normal' | 'urgent', 'high', 'normal', 'low' |
| `current_step` | INT | DEFAULT 0 | 현재 결재 단계 |
| `drafted_at` | TIMESTAMP | | 기안일시 |
| `completed_at` | TIMESTAMP | | 완료일시 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

#### `approval_lines` (결재선)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `approval_id` | UUID | FK → approvals.id, NOT NULL | 결재 문서 |
| `step` | INT | NOT NULL | 결재 단계 (1=검토, 2=합의, 3=승인) |
| `type` | ENUM | NOT NULL | 'review', 'agree', 'approve' |
| `approver_id` | UUID | FK → users.id, NOT NULL | 결재자 |
| `delegate_id` | UUID | FK → users.id | 대리결재자 |
| `status` | ENUM | DEFAULT 'pending' | 'pending', 'approved', 'rejected', 'skipped' |
| `comment` | TEXT | | 의견 |
| `acted_at` | TIMESTAMP | | 처리일시 |

#### `approval_line_templates` (결재선 템플릿)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `name` | VARCHAR(100) | NOT NULL | 템플릿명 |
| `owner_id` | UUID | FK → users.id | 소유자 (개인 템플릿) |
| `is_default` | BOOLEAN | DEFAULT FALSE | 기본 템플릿 여부 |
| `lines` | JSON | NOT NULL | 결재선 정의 [{step, type, approver_id}] |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `approval_attachments` (결재 첨부파일)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `approval_id` | UUID | FK → approvals.id | |
| `file_name` | VARCHAR(255) | NOT NULL | 원본 파일명 |
| `file_path` | VARCHAR(500) | NOT NULL | 저장 경로 |
| `file_size` | BIGINT | | 파일 크기 (bytes) |
| `mime_type` | VARCHAR(100) | | MIME 타입 |
| `uploaded_by` | UUID | FK → users.id | 업로더 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

---

### 2.3 근태관리

#### `attendances` (출퇴근 기록)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | 직원 |
| `date` | DATE | NOT NULL | 근무일 |
| `check_in` | TIMESTAMP | | 출근 시간 |
| `check_out` | TIMESTAMP | | 퇴근 시간 |
| `work_minutes` | INT | DEFAULT 0 | 총 근무시간 (분) |
| `overtime_minutes` | INT | DEFAULT 0 | 연장 근무시간 (분) |
| `status` | ENUM | DEFAULT 'normal' | 'normal', 'late', 'early_leave', 'absent', 'holiday_work', 'remote' |
| `ip_address` | VARCHAR(45) | | 출근 IP |
| `note` | TEXT | | 비고 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| **UNIQUE** | | (user_id, date) | 1인 1일 1레코드 |

#### `attendance_logs` (출퇴근 상세 로그)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `attendance_id` | UUID | FK → attendances.id | |
| `action` | ENUM | NOT NULL | 'check_in', 'check_out', 'out', 'return' |
| `timestamp` | TIMESTAMP | NOT NULL | 기록 시간 |
| `ip_address` | VARCHAR(45) | | IP |
| `note` | TEXT | | 비고 |

#### `leaves` (연차/휴가)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | 신청자 |
| `type` | ENUM | NOT NULL | 'annual', 'half_am', 'half_pm', 'sick', 'special', 'unpaid' |
| `start_date` | DATE | NOT NULL | 시작일 |
| `end_date` | DATE | NOT NULL | 종료일 |
| `days` | DECIMAL(4,1) | NOT NULL | 사용일수 (0.5=반차) |
| `reason` | TEXT | | 사유 |
| `approval_id` | UUID | FK → approvals.id | 연동 결재 문서 |
| `status` | ENUM | DEFAULT 'pending' | 'pending', 'approved', 'rejected', 'cancelled' |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `weekly_work_hours` (주간 근무시간 — 52시간 트래킹)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id | |
| `year` | INT | NOT NULL | 연도 |
| `week` | INT | NOT NULL | 주차 (ISO week) |
| `total_minutes` | INT | DEFAULT 0 | 총 근무시간 (분) |
| `overtime_minutes` | INT | DEFAULT 0 | 연장근무 (분) |
| `alert_level` | ENUM | DEFAULT 'normal' | 'normal', 'warning', 'danger', 'exceeded' |
| **UNIQUE** | | (user_id, year, week) | |

---

### 2.4 게시판

#### `boards` (게시판)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `name` | VARCHAR(100) | NOT NULL | 게시판명 |
| `code` | VARCHAR(30) | UNIQUE | 'notice', 'free', 'anonymous', 'dept_{id}' |
| `type` | ENUM | NOT NULL | 'notice', 'free', 'department', 'anonymous' |
| `department_id` | UUID | FK → departments.id | 부서 게시판일 경우 |
| `description` | TEXT | | 설명 |
| `is_active` | BOOLEAN | DEFAULT TRUE | |
| `sort_order` | INT | DEFAULT 0 | |

#### `posts` (게시글)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `board_id` | UUID | FK → boards.id, NOT NULL | 게시판 |
| `author_id` | UUID | FK → users.id, NOT NULL | 작성자 |
| `title` | VARCHAR(300) | NOT NULL | 제목 |
| `content` | TEXT | NOT NULL | 본문 (HTML) |
| `is_pinned` | BOOLEAN | DEFAULT FALSE | 상단 고정 |
| `is_important` | BOOLEAN | DEFAULT FALSE | 중요 표시 |
| `is_safety` | BOOLEAN | DEFAULT FALSE | 안전 공지 (화학 특화) |
| `category` | VARCHAR(50) | | 카테고리 (선택) |
| `view_count` | INT | DEFAULT 0 | 조회수 |
| `like_count` | INT | DEFAULT 0 | 좋아요 수 |
| `comment_count` | INT | DEFAULT 0 | 댓글 수 |
| `status` | ENUM | DEFAULT 'published' | 'draft', 'published', 'hidden', 'deleted' |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

#### `post_attachments` (게시글 첨부파일)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `post_id` | UUID | FK → posts.id | |
| `file_name` | VARCHAR(255) | NOT NULL | |
| `file_path` | VARCHAR(500) | NOT NULL | |
| `file_size` | BIGINT | | |
| `mime_type` | VARCHAR(100) | | |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `comments` (댓글)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `post_id` | UUID | FK → posts.id, NOT NULL | 게시글 |
| `author_id` | UUID | FK → users.id, NOT NULL | 작성자 |
| `parent_id` | UUID | FK → comments.id | 부모 댓글 (대댓글) |
| `content` | TEXT | NOT NULL | 내용 |
| `is_anonymous` | BOOLEAN | DEFAULT FALSE | 익명 여부 |
| `status` | ENUM | DEFAULT 'active' | 'active', 'deleted' |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

#### `post_likes` (좋아요)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `post_id` | UUID | FK → posts.id | |
| `user_id` | UUID | FK → users.id | |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| **UNIQUE** | | (post_id, user_id) | 중복 방지 |

---

### 2.5 과제/프로젝트 관리

#### `projects` (과제)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `project_no` | VARCHAR(30) | UNIQUE, NOT NULL | 과제번호 |
| `name` | VARCHAR(200) | NOT NULL | 과제명 |
| `type` | ENUM | NOT NULL | 'research', 'development', 'internal' |
| `division` | ENUM | | 'chemical', 'it' |
| `leader_id` | UUID | FK → users.id | 책임연구원 |
| `department_id` | UUID | FK → departments.id | 수행 부서 |
| `start_date` | DATE | | 시작일 |
| `end_date` | DATE | | 종료일 |
| `status` | ENUM | DEFAULT 'planning' | 'planning', 'active', 'paused', 'completed', 'cancelled' |
| `progress` | INT | DEFAULT 0 | 진행률 (0~100) |
| `budget_total` | DECIMAL(15,0) | DEFAULT 0 | 총 예산 (원) |
| `budget_used` | DECIMAL(15,0) | DEFAULT 0 | 집행액 |
| `description` | TEXT | | 과제 설명 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

#### `project_members` (과제 참여인력)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `project_id` | UUID | FK → projects.id | |
| `user_id` | UUID | FK → users.id | |
| `role` | ENUM | DEFAULT 'member' | 'leader', 'co_leader', 'member' |
| `participation_rate` | INT | DEFAULT 100 | 참여율 (%) |
| `joined_at` | DATE | | 참여 시작일 |
| **UNIQUE** | | (project_id, user_id) | |

---

### 2.6 연구비 관리

#### `expense_claims` (연구비 청구)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `claim_number` | VARCHAR(30) | UNIQUE, NOT NULL | 청구번호 (CLM-2026-0001) |
| `project_id` | UUID | FK → projects.id | 과제 |
| `claimer_id` | UUID | FK → users.id, NOT NULL | 청구자 |
| `type` | ENUM | NOT NULL | 'general', 'card', 'payroll', 'refund' |
| `total_amount` | DECIMAL(15,0) | DEFAULT 0 | 총 청구액 |
| `status` | ENUM | DEFAULT 'draft' | 'draft', 'submitted', 'approved', 'rejected', 'paid' |
| `approval_id` | UUID | FK → approvals.id | 연동 결재 문서 |
| `note` | TEXT | | 비고 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

#### `expense_items` (청구 내역 상세)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `claim_id` | UUID | FK → expense_claims.id | |
| `date` | DATE | NOT NULL | 사용일 |
| `category` | VARCHAR(50) | NOT NULL | 비목 (재료비, 여비, 수용비 등) |
| `description` | VARCHAR(300) | NOT NULL | 내용 |
| `vendor` | VARCHAR(100) | | 거래처 |
| `amount` | DECIMAL(15,0) | NOT NULL | 금액 |
| `vat` | DECIMAL(15,0) | DEFAULT 0 | 부가세 |
| `receipt_path` | VARCHAR(500) | | 영수증 파일 경로 |
| `sort_order` | INT | DEFAULT 0 | |

---

### 2.7 외부 도구 연동 (Slack / Jira / Confluence / Google Workspace)

> **참고**: 메신저(Slack), 일정(Google Calendar), 문서(Confluence), 프로젝트(Jira)는
> 외부 SaaS를 사용하므로 해당 데이터를 자체 DB에 저장하지 않는다.
> 연동에 필요한 설정/토큰만 저장한다.

#### `integration_configs` (외부 도구 연동 설정)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `service` | ENUM | NOT NULL, UNIQUE | 'slack', 'jira', 'confluence', 'google' |
| `is_enabled` | BOOLEAN | DEFAULT TRUE | 활성 여부 |
| `config` | JSON | NOT NULL | 연동 설정 (아래 참고) |
| `updated_by` | UUID | FK → users.id | 마지막 수정자 |
| `updated_at` | TIMESTAMP | | |

**config JSON 구조 (서비스별):**
```json
// Slack
{ "workspace_url": "https://jinuchem.slack.com",
  "webhook_url": "https://hooks.slack.com/...",
  "bot_token": "xoxb-...",
  "notification_channel": "#intranet-alerts" }

// Jira
{ "base_url": "https://jinuchem.atlassian.net",
  "api_token": "...",
  "default_project": "JC" }

// Confluence
{ "base_url": "https://jinuchem.atlassian.net/wiki",
  "api_token": "...",
  "default_space": "JINUCHEM" }

// Google
{ "client_id": "...",
  "client_secret": "...",
  "domain": "jinuchem.com" }
```

#### `user_integrations` (사용자별 연동 토큰)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | |
| `service` | ENUM | NOT NULL | 'slack', 'jira', 'confluence', 'google' |
| `access_token` | TEXT | | OAuth 액세스 토큰 (암호화 저장) |
| `refresh_token` | TEXT | | 리프레시 토큰 (암호화 저장) |
| `token_expires_at` | TIMESTAMP | | 토큰 만료일시 |
| `external_user_id` | VARCHAR(100) | | 외부 서비스 사용자 ID |
| `is_connected` | BOOLEAN | DEFAULT FALSE | 연결 상태 |
| `connected_at` | TIMESTAMP | | 연결일시 |
| **UNIQUE** | | (user_id, service) | 사용자당 서비스 하나 |

---

### 2.9 문서관리

#### `documents` (문서)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `folder_id` | UUID | FK → document_folders.id | 폴더 |
| `title` | VARCHAR(300) | NOT NULL | 문서 제목 |
| `file_name` | VARCHAR(255) | NOT NULL | 파일명 |
| `file_path` | VARCHAR(500) | NOT NULL | 저장 경로 |
| `file_size` | BIGINT | | 크기 |
| `mime_type` | VARCHAR(100) | | |
| `version` | INT | DEFAULT 1 | 버전 |
| `uploaded_by` | UUID | FK → users.id | 업로더 |
| `category` | ENUM | DEFAULT 'general' | 'general', 'msds', 'template', 'tech_doc' |
| `tags` | JSON | | 태그 배열 |
| `is_latest` | BOOLEAN | DEFAULT TRUE | 최신 버전 여부 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `document_folders` (폴더)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `name` | VARCHAR(100) | NOT NULL | 폴더명 |
| `parent_id` | UUID | FK → document_folders.id | 상위 폴더 |
| `department_id` | UUID | FK → departments.id | 소유 부서 |
| `created_by` | UUID | FK → users.id | |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `msds` (물질안전보건자료 — 화학 특화)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `material_name` | VARCHAR(200) | NOT NULL | 물질명 |
| `material_name_en` | VARCHAR(200) | | 영문 물질명 |
| `cas_number` | VARCHAR(30) | | CAS 번호 |
| `hazard_class` | VARCHAR(50) | | 위험등급 |
| `ghs_symbols` | JSON | | GHS 심볼 코드 배열 |
| `document_id` | UUID | FK → documents.id | MSDS 문서 |
| `supplier` | VARCHAR(200) | | 공급업체 |
| `revision_date` | DATE | | 개정일 |
| `version` | VARCHAR(20) | | 버전 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |
| `updated_at` | TIMESTAMP | | |

---

### 2.10 ~~회의관리~~ → Google Calendar + Google Meet로 대체

> 회의실 예약, 회의록 작성 등은 Google Calendar와 Google Meet/Docs로 대체.
> 자체 DB 테이블 불필요.

---

### 2.11 인사·급여

#### `payrolls` (급여)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id | |
| `year` | INT | NOT NULL | 연도 |
| `month` | INT | NOT NULL | 월 |
| `base_salary` | DECIMAL(15,0) | | 기본급 |
| `overtime_pay` | DECIMAL(15,0) | DEFAULT 0 | 연장근무수당 |
| `bonus` | DECIMAL(15,0) | DEFAULT 0 | 상여금 |
| `deductions` | JSON | | 공제 내역 (소득세, 건강보험 등) |
| `total_deduction` | DECIMAL(15,0) | DEFAULT 0 | 총 공제액 |
| `net_pay` | DECIMAL(15,0) | | 실 수령액 |
| `paid_at` | DATE | | 지급일 |
| **UNIQUE** | | (user_id, year, month) | |

---

### 2.12 매출·매입

#### `vendors` (거래처)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `name` | VARCHAR(200) | NOT NULL | 거래처명 |
| `business_no` | VARCHAR(20) | UNIQUE | 사업자번호 |
| `ceo_name` | VARCHAR(50) | | 대표자명 |
| `address` | TEXT | | 주소 |
| `phone` | VARCHAR(20) | | 전화번호 |
| `email` | VARCHAR(100) | | 이메일 |
| `bank_name` | VARCHAR(50) | | 은행명 |
| `account_no` | VARCHAR(50) | | 계좌번호 |
| `type` | ENUM | | 'customer', 'supplier', 'both' |
| `is_active` | BOOLEAN | DEFAULT TRUE | |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `transactions` (매출·매입 거래)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `type` | ENUM | NOT NULL | 'sales', 'purchase' |
| `vendor_id` | UUID | FK → vendors.id | 거래처 |
| `date` | DATE | NOT NULL | 거래일 |
| `amount` | DECIMAL(15,0) | NOT NULL | 공급가액 |
| `vat` | DECIMAL(15,0) | DEFAULT 0 | 부가세 |
| `total` | DECIMAL(15,0) | NOT NULL | 합계 |
| `description` | VARCHAR(300) | | 적요 |
| `invoice_no` | VARCHAR(50) | | 세금계산서 번호 |
| `status` | ENUM | DEFAULT 'pending' | 'pending', 'confirmed', 'paid' |
| `created_by` | UUID | FK → users.id | |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

---

### 2.13 알림

#### `notifications` (알림)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, NOT NULL | 수신자 |
| `type` | ENUM | NOT NULL | 'approval', 'message', 'notice', 'attendance', 'safety', 'leave', 'system' |
| `title` | VARCHAR(200) | NOT NULL | 알림 제목 |
| `content` | TEXT | | 알림 내용 |
| `link` | VARCHAR(500) | | 이동 링크 (페이지 ID) |
| `is_read` | BOOLEAN | DEFAULT FALSE | 읽음 여부 |
| `is_urgent` | BOOLEAN | DEFAULT FALSE | 긴급 여부 |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

#### `notification_settings` (알림 설정)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id, UNIQUE | |
| `approval_enabled` | BOOLEAN | DEFAULT TRUE | 결재 알림 |
| `message_enabled` | BOOLEAN | DEFAULT TRUE | 메시지 알림 |
| `notice_enabled` | BOOLEAN | DEFAULT TRUE | 공지 알림 |
| `attendance_enabled` | BOOLEAN | DEFAULT TRUE | 근태 알림 |
| `browser_notification` | BOOLEAN | DEFAULT FALSE | 브라우저 알림 |

---

### 2.14 시스템

#### `system_settings` (시스템 설정)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `key` | VARCHAR(100) | PK | 설정 키 |
| `value` | TEXT | | 설정 값 |
| `description` | VARCHAR(300) | | 설명 |
| `updated_by` | UUID | FK → users.id | |
| `updated_at` | TIMESTAMP | | |

#### `audit_logs` (감사 로그)
| 컬럼명 | 타입 | 제약조건 | 설명 |
|:---|:---|:---|:---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK → users.id | 행위자 |
| `action` | VARCHAR(50) | NOT NULL | 'login', 'logout', 'create', 'update', 'delete', 'approve', 'reject' |
| `target_type` | VARCHAR(50) | | 대상 엔티티 (approval, post, user 등) |
| `target_id` | UUID | | 대상 ID |
| `details` | JSON | | 상세 변경 내용 |
| `ip_address` | VARCHAR(45) | | IP |
| `created_at` | TIMESTAMP | DEFAULT NOW() | |

---

## 3. 관계 요약 (Relationship Summary)

```
users ─────1:N──── attendances
users ─────1:N──── leaves
users ─────1:N──── approvals (as drafter)
users ─────1:N──── approval_lines (as approver)
users ─────1:N──── posts
users ─────1:N──── comments
users ─────1:N──── messages
users ─────1:N──── notifications
users ─────N:1──── departments
users ─────N:1──── positions
users ─────M:N──── projects (via project_members)
users ─────1:N──── user_integrations (외부 도구 토큰)

departments ─1:N── departments (self, parent-child)
departments ─1:N── boards (department boards)

approvals ──1:N─── approval_lines
approvals ──1:N─── approval_attachments
approvals ──1:1─── leaves (optional)
approvals ──1:1─── expense_claims (optional)

projects ───1:N─── expense_claims
projects ───1:N─── project_members

expense_claims ─1:N── expense_items

documents ──N:1─── document_folders
msds ───────1:1─── documents

vendors ────1:N─── transactions

## 제거된 테이블 (외부 SaaS 대체)
# chat_rooms, chat_members, messages → Slack
# events, event_attendees → Google Calendar
# meeting_rooms, meeting_reservations, meeting_minutes → Google Calendar + Meet
```

---

## 4. 인덱스 권장

| 테이블 | 인덱스 | 용도 |
|:---|:---|:---|
| `users` | (department_id), (email) | 부서별 조회, 로그인 |
| `attendances` | (user_id, date) | 출퇴근 조회 |
| `weekly_work_hours` | (user_id, year, week) | 52시간 조회 |
| `approvals` | (drafter_id, status), (status, created_at) | 기안자별, 상태별 조회 |
| `approval_lines` | (approver_id, status) | 결재 대기함 |
| `posts` | (board_id, created_at), (author_id) | 게시판별, 작성자별 |
| `notifications` | (user_id, is_read, created_at) | 알림 목록 |
| `msds` | (material_name), (cas_number) | MSDS 검색 |
| `user_integrations` | (user_id, service) | 사용자별 연동 조회 |
| `audit_logs` | (user_id, created_at), (target_type, target_id) | 로그 조회 |
