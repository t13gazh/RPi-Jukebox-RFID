# Decisions

<!-- Append-only register of architectural and pattern decisions -->

| ID | Decision | Rationale | Date |
|----|----------|-----------|------|
| D001 | Feature-Branch `feature/webui-modernization` for all new work, `develop` stays upstream-sync | Enables clean upstream merges and future PRs back to MiczFlor/RPi-Jukebox-RFID | 2026-03-16 |
| D002 | 4 Pull Requests for upstream contribution, grouped by value tier | Each PR is self-contained and testable. Smaller PRs are reviewer-friendly. PR1 (API) is universally useful even standalone. | 2026-03-16 |
| D003 | Cleaned 12 inherited fork branches (future3, dependabot, legacy fixes) | Not relevant to our work. Reduces noise. upstream/develop tracked separately for sync. | 2026-03-16 |
| D004 | Local-first development — Pi only for final integration test | Only one Pi (production). No test hardware. FastAPI + SvelteKit run on Windows. Mock MPD/RFID/GPIO locally, deploy to Pi when slice is integration-ready. | 2026-03-16 |
| D005 | Streaming features (STRM-*) deferred to future milestone | M001 already 11 slices. Basic stream-URL assignment covered by CARD-06 in S05. RSS subscriptions, podcast catalog, ARD/NDR browser = M002. | 2026-03-16 |
