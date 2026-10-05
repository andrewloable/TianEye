# TianEye

Self-hosted replacement for the Yoosee app and Gwell cloud. A Go server talks to Yoosee cameras on
the LAN and serves an Angular web UI. See README.md for goals and the known camera facts.

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:7510c1e2 -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->

## Stack

- **Backend:** Go, single binary. API over ConnectRPC (connect-go).
- **Frontend:** latest Angular in web/, talking to the backend with connect-es (@connectrpc/connect-web).
- **API contract:** protobuf in proto/tianeye/v1/, generated with buf for both Go and TypeScript.
  Browsers only get unary and server-streaming calls, so design the API around those.
- **Video:** not over ConnectRPC. The camera's RTSP goes to the browser through WebRTC or MSE;
  the transport is still to be chosen (see beads).
- **Deploy:** the Angular build is embedded in the Go binary with go:embed, so it ships as one binary.

## Layout (planned; directories appear as the work lands)

```
cmd/tianeye/        server entrypoint
internal/           discovery, stream probing, recording
proto/tianeye/v1/   .proto API definitions
gen/                generated Go + connect code (committed)
web/                Angular app; generated TS goes in web/src/gen (committed)
spike/              throwaway experiments; give each its own go.mod so root builds and CI skip it
```

## Build & Test

```bash
buf generate                      # after editing proto/
go vet ./... && go test ./...     # backend
cd web && npm ci && npx ng build  # frontend
cd web && npx ng serve            # dev UI; proxy API calls to the Go server
```

## Working with real cameras

- Probe a stream: `ffprobe -rtsp_transport tcp rtsp://admin:<pw>@<ip>:554/onvif1`
- Find open ports: `nmap -p- <ip>`
- Camera IPs, passwords, snapshots and recordings never go in the repo. They belong in
  *.local.yaml, .env, data/ or recordings/, which are all gitignored. Footage shows real places
  and people, so open every image before committing it.
- Use placeholder IPs in tests and docs: 192.0.2.x (TEST-NET-1), never real LAN addresses.

## Reverse engineering (cloud redirection)

A main goal is to redirect the cameras' cloud connections to TianEye with local DNS, emulating
the Gwell servers. The protocol comes from captured camera traffic and the decompiled Yoosee
Android APK.

- re/ is the gitignored workspace for the APK, jadx output and pcaps. Nothing from it gets committed:
  decompiled Yoosee code is proprietary, and captures hold device IDs, passwords and LAN IPs.
- Findings go in docs/ in our own words: hostnames, ports, message formats. Don't paste decompiled code.
- Redact device IDs, passwords and IPs from any capture excerpt that goes into docs or tests.
