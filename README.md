# -AI-Nexus

AI-Nexus PRD — Production-Ready Revision

1. Executive Summary

AI-Nexus is a unified university operating system merging LMS + MIS into a single role-aware platform for Students, Faculty, and Admins. Built on PostgreSQL + Supabase + TanStack Start, mobile-first with dark mode, and engineered from day one for AI: RAG-grounded study assistant, predictive analytics, and (future) Gemini-powered question-paper generation.

This revision hardens the data model with dedicated students and faculty profile-extension tables, introduces a RAG knowledge base (documents, document_chunks with pgvector), prediction/recommendation tables, and an activity-feed substrate.

2. Problem Statement

Universities run on 5–6 disconnected tools (ERP, Moodle, email, WhatsApp, Excel). Students juggle portals, faculty waste hours on manual entry, admins lack real-time KPIs, and no system uses the rich academic data for intelligence. AI-Nexus collapses these into one product with AI as a first-class layer.

3. User Personas





Aarav — 2nd-year CS student, mobile-only, wants instant attendance %, deadlines, and an AI tutor at midnight.



Dr. Mehta — Assistant Professor, 3 courses / 180 students, wants <60s attendance marking and at-risk dashboards.



Priya — Registrar, 4,000 students / 20 departments, needs bulk onboarding, targeted notices, institution KPIs, and audit trails.

4. User Stories (high-priority)

Student: today view; subject-wise attendance with safe/warning/critical bands; marks + CGPA trend; submit assignments; receive notifications; chat with RAG-grounded AI assistant.
Faculty: bulk/QR attendance; create + grade assignments; CSV mark upload; per-student analytics with at-risk flag; (future) AI question paper generator.
Admin: bulk user import; course/section/timetable management; targeted notice broadcast; institution analytics; full audit log.

5. Feature Breakdown







Module



Student



Faculty



Admin





Auth & Roles



✓



✓



✓ MFA





Dashboard



Personal KPIs



Class load + alerts



Institution KPIs





Attendance



View



Mark/edit/bulk



Reports/thresholds





Marks/CGPA



View + trend



Enter/publish



Override + audit





Timetable



Personal



Personal



Build/assign





Assignments



Submit



Create/grade



Audit





Notices



Inbox



Targeted post



Broadcast





AI Assistant



RAG chat



(Future) QP Gen



Analytics summaries





Predictions



Personal risk



Class risk list



Cohort risk





Activity Feed



Personal



Class events



Global stream

6. Database Schema (PostgreSQL / Supabase + pgvector)

All tables: id uuid PK, created_at, updated_at (triggered). RLS enabled. Explicit GRANT per role.

Identity & Roles





profiles — id (→ auth.users), full_name, avatar_url, phone, dob, gender, address, department_id



user_roles — id, user_id, role enum(student|faculty|admin)  (separate from profiles)



departments — id, name, code, hod_faculty_id

Students (NEW dedicated table)





students — id, user_id (→ profiles), enrollment_no UNIQUE, roll_no, batch (e.g. "2024-28"), semester int, section_id, current_cgpa numeric(3,2), guardian_name, guardian_phone, admission_date, status enum(active|graduated|dropped|suspended)

Faculty (NEW dedicated table)





faculty — id, user_id (→ profiles), employee_id UNIQUE, designation enum(assistant_prof|associate_prof|professor|hod|lecturer), specialization text[], department_id, joining_date date, qualification, experience_years, status enum(active|on_leave|retired)

Academic Structure





courses — id, code UNIQUE, title, description, credits, department_id, semester



sections — id, course_id, section_code, academic_year, faculty_id (→ faculty), capacity



enrollments — id, student_id, section_id, status, UNIQUE(student_id, section_id)



timetable_slots — id, section_id, day_of_week, start_time, end_time, room

Attendance & Assessment





attendance — id, section_id, student_id, date, status enum(present|absent|late|excused), marked_by (→ faculty), UNIQUE(section_id, student_id, date)



assessments — id, section_id, name, type enum(quiz|mid|end|assignment|project), max_marks, weight, scheduled_at



marks — id, assessment_id, student_id, score, remarks, published_at, graded_by

Assignments





assignments — id, section_id, title, description, attachments jsonb, due_at, max_marks, rubric_json



submissions — id, assignment_id, student_id, content, files jsonb, submitted_at, grade, feedback, graded_by

Communications





notices — id, title, body, audience_type enum(global|role|department|section), audience_ref uuid, published_by, published_at, expires_at



notifications — id, user_id, type, title, body, payload_json, read_at

AI Knowledge Base (NEW — RAG)





documents — id, title, source_type enum(syllabus|lecture_notes|textbook|notice|policy), source_uri, course_id (nullable), department_id (nullable), uploaded_by, status enum(pending|processed|failed), metadata jsonb



document_chunks — id, document_id, chunk_index, content text, token_count int, metadata jsonb, UNIQUE(document_id, chunk_index)



embeddings — id, chunk_id (→ document_chunks), model_version text, embedding vector(3072), created_at  (HNSW index on embedding vector_cosine_ops)



ai_conversations — id, user_id, title, context_scope jsonb (course_ids, doc_ids), created_at



ai_messages — id, conversation_id, role enum(user|assistant|system), content, citations_json (chunk_ids + scores), token_usage jsonb

AI Predictions (NEW)





predictions — id, subject_user_id (student), prediction_type enum(at_risk|grade_forecast|dropout_risk|attendance_forecast), score numeric, confidence numeric, features_json, model_version, valid_until, created_at



recommendations — id, user_id, source enum(prediction|ai_assistant|rule_engine), category enum(study_plan|resource|intervention|schedule), title, body, action_url, priority enum(low|med|high), status enum(new|seen|acted|dismissed), expires_at

Activity & Audit (NEW)





activity_logs — id, actor_id (→ profiles), actor_role, action text (e.g. attendance.marked, assignment.submitted, notice.published), entity_type, entity_id, summary text, metadata jsonb, visibility enum(self|section|department|global), created_at  (indexed on (visibility, created_at desc) and (actor_id, created_at desc))



audit_log — id, actor_id, action, entity, entity_id, diff_json, ip, user_agent, created_at  (privileged/admin-only, immutable)

7. Entity Relationship Diagram (Updated)

auth.users ──1:1── profiles ──1:N── user_roles
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    students       faculty         (admin via user_roles only)
        │              │
        │              └──N:1── departments ──1:N── courses ──1:N── sections
        │                                                              │
        └────N:M (enrollments)─────────────────────────────────────────┘
                                                                       │
                                  ┌────────────────┬──────────────┬────┴──────────┐
                                  ▼                ▼              ▼               ▼
                             timetable_slots  attendance   assessments──marks  assignments──submissions

notices ──audience──▶ (role | department | section | global)
profiles ──1:N── notifications
profiles ──1:N── activity_logs

── AI Knowledge Base ──
documents ──1:N── document_chunks ──1:1── embeddings (vector 3072)
profiles ──1:N── ai_conversations ──1:N── ai_messages ──cites──▶ document_chunks

── AI Predictions ──
students ──1:N── predictions
profiles ──1:N── recommendations

8. System Architecture

┌─────────────────────────────────────────────────────────────┐
│ Client (TanStack Start · React 19 · Tailwind v4 · PWA)      │
│  Role-aware routing · Dark mode · Mobile-first              │
└──────────────┬──────────────────────────────────────────────┘
               │ HTTPS
┌──────────────▼──────────────────────────────────────────────┐
│ TanStack Server Functions (Cloudflare Workers / Edge)       │
│  attachSupabaseAuth · requireSupabaseAuth · Zod validation  │
│  Domain services: academic · attendance · assignments ·     │
│  notices · ai · predictions · activity · admin              │
└──┬─────────────────┬──────────────────────┬─────────────────┘
   │                 │                      │
┌──▼──────────┐  ┌───▼──────────────┐  ┌────▼────────────────┐
│ Supabase    │  │ Lovable AI       │  │ Background jobs     │
│ Postgres +  │  │ Gateway          │  │ (pg_cron / webhooks)│
│ pgvector +  │  │  - Gemini chat   │  │  - chunk + embed    │
│ Auth +      │  │  - embeddings    │  │  - run predictions  │
│ Storage +   │  │  - (future) QP   │  │  - digest notices   │
│ Realtime    │  │    generation    │  └─────────────────────┘
└─────────────┘  └──────────────────┘

9. Folder Structure

src/
├─ routes/                       # TanStack file-based routes
│  ├─ __root.tsx
│  ├─ index.tsx                  # Landing
│  ├─ auth.tsx
│  ├─ _authenticated/            # Auth gate (integration-managed)
│  │  ├─ route.tsx
│  │  ├─ student/
│  │  │  ├─ dashboard.tsx
│  │  │  ├─ attendance.tsx
│  │  │  ├─ marks.tsx
│  │  │  ├─ timetable.tsx
│  │  │  ├─ assignments.tsx
│  │  │  ├─ assignments.$id.tsx
│  │  │  ├─ notices.tsx
│  │  │  ├─ ai-assistant.tsx
│  │  │  └─ profile.tsx
│  │  ├─ faculty/
│  │  │  ├─ dashboard.tsx
│  │  │  ├─ classes.tsx
│  │  │  ├─ classes.$sectionId.attendance.tsx
│  │  │  ├─ classes.$sectionId.assignments.tsx
│  │  │  ├─ classes.$sectionId.marks.tsx
│  │  │  └─ classes.$sectionId.analytics.tsx
│  │  └─ admin/
│  │     ├─ dashboard.tsx
│  │     ├─ students.tsx
│  │     ├─ faculty.tsx
│  │     ├─ departments.tsx
│  │     ├─ courses.tsx
│  │     ├─ sections.tsx
│  │     ├─ notices.tsx
│  │     └─ analytics.tsx
│  └─ api/
│     └─ public/
│        ├─ webhooks.ai.ts       # AI job callbacks
│        └─ cron.predictions.ts  # pg_cron trigger
│
├─ components/
│  ├─ ui/                        # shadcn primitives
│  ├─ layout/                    # AppShell, Sidebar, BottomNav, CommandPalette
│  ├─ student/                   # AttendanceRing, CGPACard, AssignmentList…
│  ├─ faculty/                   # AttendanceMarker, MarksGrid, AtRiskTable…
│  ├─ admin/                     # UserTable, BulkImportDialog, KPIBoard…
│  ├─ ai/                        # ChatThread, CitationCard, SuggestionChip
│  └─ feed/                      # ActivityItem, FeedStream
│
├─ features/                     # Domain logic (client-safe)
│  ├─ academic/
│  ├─ attendance/
│  ├─ assignments/
│  ├─ ai/                        # RAG client helpers
│  ├─ predictions/
│  └─ activity/
│
├─ lib/
│  ├─ utils.ts
│  ├─ *.functions.ts             # createServerFn endpoints (client-reachable)
│  └─ *.server.ts                # server-only helpers (admin client, embeddings)
│
├─ integrations/
│  ├─ supabase/                  # client, client.server, auth-middleware, auth-attacher, types
│  └─ ai/                        # gateway wrapper, embedder, chunker, retriever
│
├─ hooks/                        # useRole, useRealtime, useDebounce…
├─ styles.css                    # design system tokens
└─ start.ts / server.ts / router.tsx

10. Component Architecture

Layered, presentation-first:

┌─────────────────────────────────────────────────────────┐
│ Pages (routes/)                                         │
│   • Compose layout + feature widgets                    │
│   • Loaders call server fns via TanStack Query          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ Feature Widgets (components/{role,ai,feed}/)            │
│   • Self-contained UI for one domain concern            │
│   • Read via useSuspenseQuery, mutate via useMutation   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ UI Primitives (components/ui/) + Layout shells          │
│   • shadcn-based, themed via design tokens only         │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ Hooks + Feature Clients (features/, hooks/)             │
│   • Query keys, optimistic updates, realtime channels   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ Server Functions (lib/*.functions.ts)                   │
│   • Zod input · requireSupabaseAuth · has_role check    │
│   • Delegates to *.server.ts services                   │
└─────────────────────────────────────────────────────────┘

Cross-cutting: AppShell (sidebar + bottom nav), RoleGuard, CommandPalette, NotificationBell, AIFab (floating AI button on student routes), ActivityFeed panel.

11. API Structure (Server Functions + Public Routes)

Internal calls use createServerFn (RPC). External callers use /api/public/* routes with signature verification.

lib/academic.functions.ts
  listDepartments, getDepartment, createDepartment(admin)
  listCourses, getCourse, upsertCourse(admin)
  listSections, getSection, assignFacultyToSection(admin)

lib/students.functions.ts
  getMyStudentProfile          (student)
  listStudents(filters)        (admin, faculty-scoped)
  importStudentsCsv            (admin)
  updateStudent                (admin)
  getStudentTranscript         (student-self | admin | faculty-of-section)

lib/faculty.functions.ts
  getMyFacultyProfile          (faculty)
  listFaculty(filters)         (admin)
  upsertFaculty                (admin)
  getFacultyLoad               (admin)

lib/attendance.functions.ts
  getMyAttendance              (student)
  getSectionAttendance         (faculty)
  markAttendance(bulk)         (faculty)
  attendanceReport(scope)      (admin)

lib/assessments.functions.ts
  listMyMarks                  (student)
  getMyCgpa                    (student)
  createAssessment             (faculty)
  publishMarks(bulkCsv)        (faculty)

lib/assignments.functions.ts
  listMyAssignments            (student)
  submitAssignment             (student)
  listSectionAssignments       (faculty)
  gradeSubmission              (faculty)

lib/timetable.functions.ts
  getMyTimetable               (student | faculty)
  upsertSlot                   (admin)

lib/notices.functions.ts
  listMyNotices                (all roles)
  publishNotice(audience)      (faculty | admin)

lib/notifications.functions.ts
  listMyNotifications, markRead, markAllRead

lib/ai.functions.ts
  startConversation, postMessage   → calls Gateway chat
  retrieveContext(query, scope)    → vector search over embeddings
  listMyConversations
  (future) generateQuestionPaper(syllabusRef, blooms) (faculty)

lib/predictions.functions.ts
  getMyRiskScore               (student)
  listSectionAtRisk            (faculty)
  listRecommendations          (all roles)
  dismissRecommendation

lib/activity.functions.ts
  listFeed(scope, cursor)
  emitActivity (internal, called by services)

lib/admin.functions.ts
  institutionKpis, departmentKpis, courseKpis
  bulkImportUsers, auditLogQuery

routes/api/public/
  cron.predictions.ts          # pg_cron → recompute predictions
  cron.embeddings.ts           # pg_cron → process pending documents
  webhooks.ai.ts               # async AI job callbacks (HMAC-verified)

Every protected function: .middleware([requireSupabaseAuth]) + role check via has_role(). Every mutation: emits an activity_logs row and (for admin/destructive ops) an audit_log row.

12. User Journey Diagrams

Student — daily check-in

Open app ─▶ Auth gate ─▶ /student/dashboard
   │             │              │
   │             │              ├─ Today's classes (timetable)
   │             │              ├─ Attendance ring (per subject)
   │             │              ├─ Pending assignments (top 3)
   │             │              ├─ Latest notices
   │             │              └─ Recommendation card (from predictions)
   │             │
   │             └─▶ AI FAB ─▶ /student/ai-assistant
   │                              └─ retrieveContext(course scope)
   │                                  └─ Gemini answer + citations
   │
   └─▶ Notification ─▶ deep-link (assignment / mark / notice)

Faculty — mark attendance

/faculty/dashboard ─▶ Today's class card ─▶ /classes/:id/attendance
                                                │
                                                ├─ Roster preload (enrollments)
                                                ├─ Bulk "all present" toggle
                                                ├─ QR mode (optional)
                                                ├─ Save → markAttendance(bulk)
                                                │     ├─ writes attendance rows
                                                │     ├─ emits activity_logs
                                                │     └─ triggers prediction job
                                                └─ Confirmation + undo (30s)

Admin — onboard a cohort

/admin/students ─▶ Bulk Import ─▶ Upload CSV
                                     │
                                     ├─ Validate (Zod schema)
                                     ├─ Preview diff
                                     ├─ Confirm
                                     │     ├─ supabaseAdmin.auth.admin.createUser ×N
                                     │     ├─ profiles + students rows
                                     │     ├─ user_roles = 'student'
                                     │     ├─ enrollments to default sections
                                     │     └─ audit_log entry
                                     └─ Email invites sent

AI Study Assistant — RAG flow

User question ─▶ ai.postMessage(conv_id, text)
                    │
                    ├─ embed(query)  via Lovable Gateway
                    ├─ vector search documents/embeddings (cosine)
                    │     scope = student's enrolled courses
                    ├─ assemble prompt (system + top-k chunks + history)
                    ├─ chat completion via Gemini
                    ├─ persist ai_messages (with citations)
                    └─ stream tokens to client

Prediction pipeline (background)

pg_cron (nightly) ─▶ /api/public/cron.predictions
                          │
                          ├─ for each active student:
                          │     gather features (attendance %, marks trend, submission rate)
                          │     run model (heuristic v1 → ML later)
                          │     upsert predictions row
                          │     if score > threshold:
                          │         insert recommendation (intervention)
                          │         notify advisor
                          └─ emit activity_logs (visibility=department)

13. Security Requirements





Supabase Auth (email/password + Google via Lovable broker).



Roles in user_roles; has_role(uid, role) SECURITY DEFINER for RLS.



RLS on every public table; default-deny; policies scoped to auth.uid() + role + section/department ownership.



Admin destructive ops require recent re-auth + audit_log entry.



Storage buckets per scope (assignments, documents, avatars) with signed URLs.



AI: server-only LOVABLE_API_KEY, per-user rate limit, PII redaction before embedding, citations required on every assistant message.



Webhooks: HMAC signature verification, timing-safe compare.



Embeddings use pgvector with HNSW; model_version stored to allow re-embedding on model change.

14. Development Roadmap





Phase 0 (W1): Design system, dark mode, auth, role routing, profiles, departments, students/faculty tables, RLS baseline.



Phase 1 — MVP (W2–4): Student dashboard, attendance/marks/timetable view, notices, notifications; faculty attendance + marks entry; admin CRUD + bulk import; activity feed.



Phase 2 (W5–6): Assignments end-to-end, faculty class analytics, admin institution analytics, audit log UI.



Phase 3 — AI Layer (W7–8): Documents + chunking + embeddings pipeline; RAG study assistant; recommendation surfacing.



Phase 4 — Predictive (W9–10): Heuristic at-risk model → ML model; advisor alerts; cohort dashboards.



Phase 5 — Future AI: Question paper generator, auto-summarizer, voice attendance, smart timetable solver.

15. MVP Scope (ship first)

Auth + roles · profiles + students + faculty tables · departments/courses/sections/enrollments · timetable · attendance · marks + CGPA view · assignments view (submit in P2) · notices · notifications · admin user/course management · activity feed · mobile dark UI.

Out of MVP: AI chat, predictions, QP generator, payments, parent/library/hostel modules.

16. Future AI Features

RAG Study Assistant · Question Paper Generator (Bloom's-balanced) · At-Risk & Dropout Prediction · Auto Weekly Digest · Smart Timetable Solver · Voice/Image Attendance · Personalized Study Plans from recommendations.

17. Success Metrics

80% student DAU within 30 days · attendance marking <60s · 90% reduction in admin report time · AI assistant CSAT ≥4.2/5 and ≥3 sessions/student/week by Phase 3 · ≥70% precision on at-risk predictions by Phase 4.
