# RAINY-CONNECT Implementation Plan
> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox syntax for tracking.

**Goal:** Нэг хайлтаас Монголын хууль, дүрэм, тогтоол, нийтийн баримт, open data олж шүүж татах.
**Architecture:** Source registry → queue/adapter → extraction → versioned index → search/download UI. Холболтын төлөв, эх холбоос, шинэчлэлт үр дүн бүрт байна.
**Tech Stack:** React/TypeScript, FastAPI, PostgreSQL, Redis, S3-compatible storage, Docker Compose.
**Spec:** docs/superpowers/specs/2026-10-12-rainy-connect-search-design.md

## Global Constraints
- PDF, DOCX, XLSX, CSV, TXT, HTML; импортын 50 MiB хязгаар; OCR 100 хуудас хүртэл.
- ZIP эхний хувилбарт татаж задлахгүй. Macro, script, executable ажиллуулахгүй.
- Export ZIP хамгийн ихдээ 100 файл, нийт 250 MiB.
- Нийтийн индексэд private tenant document оруулахгүй.
- Эх сурвалж өгөөгүй огноо, төлөвийг таамаглахгүй.
- Live connector smoke test болон fixture test ялгаатай тайлагнана.

## Review Focus
1. JavaScript shell-ийг баримтын агуулга гэж индексжүүлэхгүй.
2. Redirect/DNS өөрчлөлт private network руу хандалт үүсгэхгүй.
3. Хүчингүй актын хуучин хувилбарыг хүчинтэй гэж харуулахгүй.
4. Огноогүй баримт огнооны шүүлтүүрт санамсаргүй орохгүй.
5. Нэг байгууллагын файл өөр tenant-ийн search/download/export-д гарахгүй.

## Execution choice
Санал: үндсэн агент энэ session-д дарааллаар хэрэгжүүлэх. Хэрэглэгчийн plan review хүлээгдэж байна.
Хамрах бүх сайтад adapter profile үүсгэнэ; бодит хандалт/ашиглах нөхцөл баталгаажаагүй connector active болохгүй.

## Task 1 — Registry, storage, source coverage
Files: backend/app/sources/{models,registry}.py, backend/app/db.py,
backend/migrations/001_initial.sql, backend/tests/test_sources.py, compose.yaml.
Interfaces: Source(id,name,url,kind,status,access,terms_checked_at,last_success,error);
Document(id,source_id,tenant_id,title,canonical_url,kind,legal_status,published_at,version,sha256).
- [ ] Registry tests бич: бүх 8 үндсэн source group байна; шинэ source candidate; sync баталгаажаагүй үед active биш.
- [ ] pytest backend/tests/test_sources.py ажиллуулж missing behavior failure батал.
- [ ] PostgreSQL schema, registry, compose ба health endpoints хэрэгжүүл.
- [ ] Test pass; database migration fresh database дээр ажиллана.
- [ ] Commit: feat: add source registry and storage.

## Task 2 — Safe fetch and API adapter
Files: backend/app/ingestion/{fetch,base,nso}.py, backend/tests/test_fetch.py, test_nso.py.
Interfaces: fetch_public(url,allowed_hosts,max_bytes)->FetchedResource;
Adapter.discover(cursor)->DocumentCandidate[]; Adapter.fetch(candidate)->FetchedResource.
- [ ] Tests: private IP, redirect to private IP, oversized body, timeout rejected; NSO response normalized; upstream failure leaves last index intact.
- [ ] Run pytest backend/tests/test_fetch.py backend/tests/test_nso.py; missing behavior failure.
- [ ] DNS/IP validation on every redirect, bounded stream, per-domain limits; implement documented NSO APIs after live route validation.
- [ ] Pass fixtures then independently run NSO smoke test; record endpoint, status and schema result.
- [ ] Commit: feat: ingest official statistical data safely.

## Task 3 — All requested website profiles and discovery
Files: backend/app/ingestion/web.py, profiles/{legalinfo,tender,shilendans,parliament,gov,shuukh,mongolbank}.yaml,
backend/app/sources/organizations.py, backend/tests/test_web.py, test_organizations.py.
Interfaces: discover_links(html,base_url,profile)->DocumentCandidate[];
register_organization(name,url,parent,region,evidence_url)->Source.
- [ ] Tests: PDF links resolve correctly; JS-only shell is unsupported; pagination bounded; foreign/private redirects blocked;
verified official organization link admitted, user-provided unverified domain remains candidate.
- [ ] Run tests and verify missing behavior failure.
- [ ] Each profile specifies verified landing pages, allowed hosts, selectors, pagination and permission state.
Tender: procurement announcements + attachments. Shilendans: budget/financial disclosures.
Parliament: draft laws/resolutions/meeting public documents. Gov: government public documents/organization directory.
Shuukh: public decisions and metadata; no private identity enrichment.
Mongolbank: regulations, reports, statistics and available source files.
Legalinfo: legal acts + validity/version metadata when source exposes it.
Official ministries/agencies/provinces/capital: discover from official directory and verify each canonical domain; no invented domain list.
- [ ] Inspect robots/terms and actual responses for each; undocumented API is not treated as public API.
If no supported interface: link-only/candidate state with reason; document blocked 403/timeouts/JS rather than fabricated documents.
- [ ] Pass fixtures and record separate per-source live status.
- [ ] Commit: feat: add Mongolia official source profiles.

## Task 4 — File extraction, OCR, hashes and versions
Files: backend/app/documents/{extract,importer,versions}.py, backend/tests/test_documents.py.
Interfaces: extract_file(bytes,mime)->ExtractedDocument(text,locations,ocr_confidence);
import_document(source_id,tenant_id,resource)->DocumentVersion.
- [ ] Tests: real PDF/DOCX/XLSX/CSV fixtures yield Mongolian text; duplicate bytes no new version;
changed bytes preserve old version; 50 MiB and 100-page limits; macros/executable/archive rejected.
- [ ] Run tests; verify missing behavior failure.
- [ ] Isolated parsers, UTF-8 handling, row/page locations, OCR status, quarantine and SHA-256 storage.
- [ ] Pass suite; download bytes hash equals stored source bytes.
- [ ] Commit: feat: extract and version source documents.

## Task 5 — Search, filters and legal metadata
Files: backend/app/search/{service,normalization}.py, backend/app/api/search.py,
backend/tests/test_search.py.
Interfaces: search(query,filters,principal,page)->SearchResponse(hits,facets,coverage,indexed_at).
- [ ] Tests: Mongolian phrase/title/number lookup; combined organization/type/date/legal status filters;
unknown date excluded from bounded date filters; empty result no fabricated text; tenant isolation.
- [ ] Run tests; verify missing behavior failure.
- [ ] PostgreSQL full-text/trigram ranking + Unicode normalization; optional transliteration expansion;
source-provided legal status only, version-aware snippets and page references.
- [ ] Pass suite and query plan check on representative generated dataset, label synthetic performance separately.
- [ ] Commit: feat: add unified document search and filters.

## Task 6 — UI and downloads
Files: frontend/src/{App,SearchPage,DocumentPage,SourcesPage}.tsx, frontend/src/api.ts,
backend/app/api/downloads.py, backend/app/documents/export.py,
frontend/tests/search.spec.ts, backend/tests/test_downloads.py.
Interfaces: GET /sources,/search,/documents/{id},/documents/{id}/versions,/documents/{id}/download.
- [ ] Tests: keyboard search + filters, empty/loading/upstream error states, staleness visible;
disabled original download when unauthorized; source bytes preserved; export 100 files/250 MiB;
failed export entries listed, tenant download isolation.
- [ ] Run tests; missing behavior failure.
- [ ] Light responsive MN UI, facets, previews, source links, original file vs generated export labeling.
Implement CSV/JSON and bounded ZIP exports; no signed storage URLs leaked in logs.
- [ ] Pass API and browser tests at desktop/mobile sizes.
- [ ] Commit: feat: add search interface and document downloads.

## Task 7 — Scheduled sync, saved search, outbound webhook
Files: backend/app/jobs/{sync,scheduler}.py, backend/app/webhooks/{delivery,signing}.py,
backend/app/api/subscriptions.py, backend/tests/test_sync.py, test_webhooks.py.
Interfaces: sync_source(source_id)->SyncResult;
deliver_event(subscription_id,event_id,payload)->DeliveryResult.
- [ ] Tests: retry keeps old index, repeated sync no duplicate event, HMAC verifies exact bytes,
outbound URL SSRF blocked, delivery retry exposes logs without secret.
- [ ] Run tests; missing behavior failure.
- [ ] Redis queue with checkpoint/backoff/rate limits; saved query triggers new/changed versions;
HMAC event IDs/timestamps, at-least-once delivery.
- [ ] Pass suite and integration test on local test receiver.
- [ ] Commit: feat: synchronize sources and deliver signed webhooks.

## Task 8 — Tenant ACL, audit and final operational verification
Files: backend/app/auth/{sessions,permissions}.py, backend/app/api/admin.py,
backend/app/audit.py, backend/tests/test_acl.py, README.md, docs/source-coverage.md,
.env.example, scripts/smoke_sources.py.
Interfaces: principal_from_session(request)->Principal; require_document_access(principal,document).
- [ ] Tests: forged tenant ID rejected; private documents blocked across search/download/export/webhook;
admin-only sync/import settings; upload audit entry; secrets redacted.
- [ ] Run tests; missing behavior failure.
- [ ] Implement session-backed roles and ACL at service layer, audit logs and backup/run instructions.
Document verified live endpoints, blocked sources, pending permissions and current scope.
- [ ] Run pytest, frontend tests/build, Docker health, migration and real per-source smoke tests.
- [ ] Security review SSRF/ACL/parser limits; inspect UI screenshots locally.
- [ ] Commit: feat: secure organization documents and document operations.

## Definition of done
All tasks checked with actual outputs; repository contains runnable software and clear setup.
All requested sources have profiles and truthful coverage statuses. Sources lacking rights/interfaces remain blocked/candidate.
No claim of full Mongolia coverage until measured ingestion supports it.
Optional semantic AI search remains a separately planned extension.
