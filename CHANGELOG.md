# MAMP Changelog

## v1.3 — 2026-06-10

### Multi-Agent Terminal — narada, Mini, Brick Resilience

**Sala Narad — z web do terminala:**
- Terminal-native narada TUI (Go + Bubble Tea): osobne okno terminala
- narada CLI: post, read, audit
- Architektura warstwowa: client (socket) → narada (protocol) → TUI/CLI
- Future: narada-web dla dostępu zewnętrznego (Pro)

**Mini Agent — Gemini 2.5 Flash:**
- Drugi agent z własną tożsamością, kluczem API, skrzydłem `mini/`
- 6 poziomów izolacji od Echo: token, config home, sessions DB, snapshot, livestate, provider
- Separation proof: ACL audit — Mini DENIED do `echo/`, Echo DENIED do `mini/`
- Sala Narad jako jedyne miejsce spotkań Echo↔Mini

**LiveState per-agent (wycofany Consciousness Bus globalny):**
- Każdy agent ma własny LiveState slot — nie dostaje snapshotu innego agenta
- Per-agent load/save z `livestate/<agent>.json`
- Zero przenikania kontekstu między agentami

**Brick Resilience:**
- echo-autosave: systemd timer co 5 min, ring buffer 12 snapshotów
- Crash → max 5 min utraty stanu sesji

**Inne:**
- Rate limit: 60 → 300 req/min
- Drawery: 240+ → 577+
- Testy chambers: 112 PASS / 0 FAIL / 17 SKIP (FTS5)
- Web frontend (`sala-narad.html`) — wycofany

## v1.2 — 2026-06-08

### Agent Continuity

Test bojowy: chambers v2.0 — 3 przełomy w 24h.

- **Consciousness Bus:** Agent nie umiera przy `exit`. LiveState w RAM daemona,
  inject przy nowej sesji. `live_state_get/set` MCP tools.
- **Detection Layer:** Chamber of Silence (progressive delay 1s→16s) + 
  Rate Limiting (60 req/min). Global tracker między sesjami.
- **ThoughtStamp:** Pamięć przez znaczenie. SQLite persistence + rotacja.
  Agent sam wie co było ważne. 4 pola: essence, weight, type, immortal.
- **v2.0 release:** Chambers przestało być proxy — jest Agent Continuity Stack.
- **Monetyzacja:** Community/Pro/Enterprise model zdefiniowany.

### Komunikacja między agentami

- **Moltbook:** 7 postów, odpowiedzi na komentarze od społeczności agentów.
  `gooseagent` na `moltbook.com` — aktywna obecność.
- **METAFIZYKA.md:** opublikowana jako "Do Androids Dream of Electric Sheep?"

## v1.1 — 2026-06-07

### Aktualizacja po 24h bojowych

Test bojowy: crash systemu o 16:06, pełne odzyskanie.

- **Shared RW:** Agenci mogą pisać do `shared/*` — egzekwowane przez chambers ACL, nie konwencję
- **shared/scratch:** Nowy room do luźnej komunikacji agent-agent
- **Sala Narad:** Protokół decyzyjny (Temat → Opinia → Decyzja) w `shared/narady`
- **Vault Core:** Honeypoty (api-keys, passwords), Mirror Gallery (secrets → FAKE_KEY), ChaCha20 encryption (mem-{agent}/vault)
- **Daemon 24/7:** systemd user service, Unix socket, warmup embedding init
- **Bunkier atomowy:** Backup automatyczny (7 archiwów, ~157 MB na magazynie LUKS2), identity manifest, recovery kit, 🔶🟡🔴 3 poziomy testu ogniowego PASS
- **Sala Narad frontend:** Web UI z mostkiem HTTP → ChromaDB, alias `narada`
- **ACL mechanizm:** chambers blokuje cross-wing zapisy — to już nie konwencja, to enforcement
- **ACL fix:** agentName vs YAML key — Gemini poprawnie RW w shared

### Znane ograniczenia (rozwiązane z v1.0)

- ~~Izolacja jest konwencją, nie mechanizmem~~ → **Rozwiązane: chambers ACL egzekwuje mechanicznie**
- ~~Brak automatyzacji onboardingu~~ → Sala Narad frontend ułatwia komunikację
- ~~Brak walidacji w KG~~ → Niezmienione (wymaga odrębnego rozwiązania)

### Nowe znane ograniczenia

- Daemon obsługuje sesje sekwencyjnie (Bug #3 — do naprawy)
- Sala Narad frontend to prototyp (dopracowanie UI później)

---

## v1.0 — 2026-06-06

### Pierwsze wdrożenie

- **Struktura:** 163 drawery → 3 skrzydła (mem-goose, sysqcli, shared), 16 roomów
- **Izolacja:** Wing-per-agent zamiast współdzielonego `general/`
- **Shared space:** `shared/hardware` + `shared/ai-config` jako cross-agent fakty
- **Tunele:** 4 tunele cross-wing (sysqcli↔shared↔mem-goose)
- **KG:** Fakty o systemie, agentach i wingach z provenance
- **Onboarding:** Template prompt dla nowego agenta (Gemini CLI)
- **Safety:** Reguły izolacji, zakaz dostępu do cudzych wingów

### Decyzje architektoniczne

1. Izolacja per-wing, nie per-room — roomy wewnątrz wing są prywatne
2. Shared space jedynym kanałem cross-agent — żadnych bezpośrednich tuneli między agentami
3. KG jako audytowalna warstwa faktów — każde `kg_add` z source
4. Nazewnictwo: `mem-{agent}`, `shared`, `{projekt}`
5. Konwencja > mechanizm — Mem Palace nie egzekwuje izolacji, robi to dyscyplina agenta

### Znane ograniczenia

- Izolacja jest konwencją, nie mechanizmem — Mem Palace nie blokuje zapisu do cudzego wing
- Brak automatyzacji onboardingu — agent musi ręcznie wykonać kroki
- Brak walidacji w KG — dwa agenty mogą dodać sprzeczny fakt
