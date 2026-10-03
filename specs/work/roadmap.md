# Overy — Roadmap

> What is still open from the phases. Work goes by need, not by phase number, so a phase here is a bucket for the remaining tail. The closed phases 1–5 (and the cancelled 4) are in `specs/archive/phases/`. What is already in the code — [`../docs/product-architecture.md`](../docs/product-architecture.md).

| # | Phase | What remains | Details |
|---|------|--------------|--------|
| 5 | Production | Desktop signing and notarization. Trigger — the first public distribution. | `specs/archive/phases/05-production/` |
| 6 | Polish / multi-device | Line-by-line outliner with CRDT, Trash UI, Postgres FTS (on PG, `Search` works through ILIKE), custom boards, SSE push. Freshness of an open page is done by polling; push remains. The `contact` trait is out of scope. | `specs/work/phases/06-polish-multi-device/` |
| 7 | Mobile | NativePHP Mobile 3 Air: native iOS and Android from the same app, regular sign-in through Fortify. Not started. | `specs/work/phases/07-mobile/` |
| 8 | Agent loop | Folder mounting (`~/.claude/`, project folders) and the agent loop. The MCP node control plane is done. No CLI: the agent has one surface — MCP. The `agent` trait is out of scope. | `specs/work/phases/08-agent-loop/` |
| 9 | E2E migration | `PassthroughCodec` → `SodiumCodec` (libsodium/WASM), sync v1→v2, a master key and BIP39 recovery, vault decryption on the client. The trigger is not fixed: a compliance requirement or user demand. | `specs/work/phases/09-e2e-migration/` |
