# WORKFLOW-SESSION.md
# Session: GV-doc-adr-gaps-20260325
# Date: 2026-03-25

## What changed — governance: DOC-1 missing ADRs added

Six ADR files missing from engx-governance/architecture/decisions/ have been added.
ADR-033 and ADR-034 exist canonically in their service repos (accord/, herald/) —
governance copies added here per the documentation standard.
ADR-035 and ADR-046 were accepted decisions with no document; stubs written from
implementation evidence and ADR cross-references.
ADR-047 and ADR-048 exist canonically in arbiter/ — governance copies added.

## New files
- `architecture/decisions/ADR-033-accord-shared-api-types.md`    — stub, canonical in accord/
- `architecture/decisions/ADR-034-herald-typed-http-client.md`   — stub, canonical in herald/
- `architecture/decisions/ADR-035-engx-deregister.md`            — written from implementation
- `architecture/decisions/ADR-046-conduit-remote-execution.md`   — written from implementation
- `architecture/decisions/ADR-047-arbiter-enforcement-engine.md` — stub, canonical in arbiter/
- `architecture/decisions/ADR-048-arbiter-http-service.md`       — stub, canonical in arbiter/

## Modified files
- (none — governance docs only)

## Apply

cd ~/workspace/projects/engx/engx-governance && \
unzip -o /mnt/c/Users/harsh/Downloads/engx-drop/governance-doc-adr-gaps-20260325.zip -d . && \
ls architecture/decisions/ | sort

## Verify

# Should show all ADRs from 001 to 049 (no gaps except 022 which is reserved)
ls architecture/decisions/ | wc -l
# Expected: 47 files

## Commit

git add architecture/decisions/ WORKFLOW-SESSION.md && \
git commit -m "docs: add missing ADR-033/034/035/046/047/048 to governance decisions (DOC-1)" && \
git tag v2.3.0 && \
git push origin main --tags
