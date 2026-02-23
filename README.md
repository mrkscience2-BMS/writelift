[README.md](https://github.com/user-attachments/files/25495745/README.md)
# WriteLift AI — Backend API

Node.js/Express backend for the WriteLift AI student writing platform.

## Quick Start

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- Redis 6+ (optional but recommended)
- OpenAI API key
- Azure AD app registration (for Microsoft SSO)

### Setup

```bash
# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.example .env
# Edit .env with your credentials

# 3. Create and migrate database
createdb writelift_db
npm run migrate

# 4. Seed demo data (for beta testing)
npm run seed

# 5. Start development server
npm run dev
```

Server runs at `http://localhost:4000`

---

## API Reference

### Authentication
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/auth/microsoft` | Get Microsoft OAuth URL |
| GET | `/api/auth/microsoft/callback` | Microsoft SSO callback |
| POST | `/api/auth/microsoft/token` | Exchange MS access token for app JWT |
| GET | `/api/auth/me` | Get current user (requires JWT) |

### Student Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/student/dashboard` | Full student dashboard data |
| GET | `/api/student/skills/growth` | Skill growth over time |
| GET | `/api/student/exercises` | Pending/completed exercises |
| GET | `/api/student/readings` | Leveled readings assigned to student |
| POST | `/api/student/readings/:id/complete` | Complete a reading |
| GET | `/api/student/milestones` | Earned achievements |

### Submissions & Assessment
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/submissions` | Submit writing → triggers AI assessment |
| GET | `/api/submissions/my` | Student's submission history |
| GET | `/api/submissions/:id` | Single submission with assessment |
| POST | `/api/submissions/exercises/:id/respond` | Respond to an exercise |

### Teacher Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/teacher/dashboard` | Teacher overview + alerts |
| GET | `/api/teacher/classes/:id/heatmap` | Skill heatmap for class |
| POST | `/api/teacher/assignments` | Create assignment |
| PATCH | `/api/teacher/assignments/:id` | Update assignment |
| POST | `/api/teacher/classes` | Create class |
| POST | `/api/teacher/classes/:id/enroll` | Enroll student by email |
| GET | `/api/teacher/classes/:id/submissions` | All class submissions |
| GET | `/api/teacher/students/:id` | Individual student detail |

### Admin Endpoints
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/admin/stats` | Platform-wide statistics |
| GET | `/api/admin/users` | Manage users |
| PATCH | `/api/admin/users/:id` | Update user |
| GET/POST | `/api/admin/readings` | Manage reading library |
| GET/POST | `/api/admin/schools` | Manage schools |

### LTI 1.3 (Schoology)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/lti/login` | OIDC login initiation |
| POST | `/api/lti/launch` | LTI launch (JWT callback) |
| GET | `/api/lti/config` | Tool configuration JSON |
| POST | `/api/lti/grades/passback` | Grade passback to Schoology |

---

## AI Assessment Engine

### Skill Areas
Each submission is scored 0–100 across 6 dimensions, mapped to levels 1–5:

| Skill | Description |
|-------|-------------|
| **Organization** | Paragraph structure, flow, transitions |
| **Evidence** | Supporting details, citations, reasoning |
| **Grammar** | Correctness, mechanics, punctuation |
| **Sentence Variety** | Length and structural variation |
| **Vocabulary** | Word choice, precision, richness |
| **Argument Strength** | Thesis, reasoning, counterargument |

### Score → Level Mapping
- L5 (Advanced): 90–100
- L4 (Proficient): 75–89
- L3 (Grade Level): 55–74
- L2 (Approaching): 35–54
- L1 (Below Grade): 0–34

### Adaptive Flow
1. Student submits → AI scores all 6 skill areas
2. Weak micro-skills flagged (e.g., "comma splices", "topic sentences")
3. Up to 3 targeted exercises auto-generated via GPT-4o
4. Student skill profile updated with weighted rolling average
5. Milestones checked and awarded
6. Leveled readings auto-assigned based on adjusted grade level

---

## Database Schema

```
schools ─┬─ users (teachers, students, admins)
         ├─ classes ─┬─ class_enrollments
         │           ├─ assignments ─── submissions ─── assessments
         │           └─ (student data via enrollments)
         └─ (school-level reporting)

users ─┬─ skill_profiles  (rolling avg per student)
       ├─ skill_history   (time-series per submission)
       ├─ exercises       (AI-generated practice tasks)
       ├─ student_readings
       └─ milestones

lti_sessions (ephemeral, for Schoology OIDC flow)
```

---

## Microsoft SSO Setup

1. Register app in [Azure Portal](https://portal.azure.com)
2. Add redirect URI: `https://your-api.com/api/auth/microsoft/callback`
3. Grant permissions: `User.Read`, `openid`, `email`, `profile`
4. Copy **Tenant ID**, **Client ID**, **Client Secret** → `.env`

## Schoology LTI 1.3 Setup

1. In Schoology App Center → Developer → Add LTI Tool
2. **OIDC Login URL**: `https://your-api.com/api/lti/login`
3. **Launch URL**: `https://your-api.com/api/lti/launch`
4. **JWKS URL**: Provide your tool's public key endpoint
5. Copy **Client ID** and **Deployment ID** → `.env`

---

## Environment Variables

See `.env.example` for the full list. Required for beta:

```
DATABASE_URL=postgresql://...
OPENAI_API_KEY=sk-...
JWT_SECRET=<random 64-char string>
AZURE_AD_TENANT_ID=...
AZURE_AD_CLIENT_ID=...
AZURE_AD_CLIENT_SECRET=...
FRONTEND_URL=https://your-frontend.com
```

---

## Beta Testing Notes

- Run `npm run seed` to create demo teacher + 5 demo students
- All demo accounts use Microsoft SSO (no passwords)
- Submit writing via `POST /api/submissions` with a Bearer token
- Teacher heatmap available at `GET /api/teacher/classes/:classId/heatmap`
- Redis is optional — gracefully degrades if unavailable

---

© 2026 WriteLift AI. All rights reserved.
