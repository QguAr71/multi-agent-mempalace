# MAMP Changelog

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
