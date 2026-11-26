# AGENTS.md — `research`

Follow the universal Smart Boot Menu agent rules in [`smart-boot-menu/.github/AGENTS.md`](https://github.com/smart-boot-menu/.github/blob/main/AGENTS.md) (locally `../.github/AGENTS.md`).

## Repository Role

`smart-boot-menu/research` is the canonical home for **primary-source ecosystem research, hardware studies, technology evaluations, and empirical validation evidence**.

## Ownership

- `ecosystem/` — Upstream project evaluations, license audits, and Reuse/Wrap/Fork/Study/Avoid recommendations.
- `hardware/` — Physical hardware probes (x86 HP laptop, Apple Silicon M1–M5), GPU/display studies, firmware quirk logs.
- `architecture/` — Comparative technology deep dives backing ADRs.
- `validation/evidence/` — Raw benchmark outputs, QEMU boot timing captures, hardware test logs (redacted of personal serial numbers/MACs/keys).
- `proposals/` — Early exploratory research memos prior to becoming normative specifications.

## Rules for Agents Writing Here

1. **Primary sources only**: Cite exact upstream URLs, verification dates (YYYY-MM-DD), and release tags or commit hashes. Never invent version numbers or claim hardware support that has not been verified.
2. **Distinguish fact from speculation**: Explicitly label sections or table entries as *Verified*, *Experimental / WIP*, or *Unverified / Open Question*.
3. **Never duplicate normative specs here**: Once a design decision or interface is ratified, record the decision in `docs/architecture-decisions/` and the specification in `docs/specifications/`, and link back to the supporting research here.
4. **Preserve knowledge worth retaining**: Whenever any engineering agent discovers a firmware quirk, driver limitation, or benchmark finding, record it here and update `README.md`.
5. **Redact sensitive hardware identifiers**: Never commit ECIDs, UUIDs, serial numbers, MAC addresses, or private keys from physical machines.
