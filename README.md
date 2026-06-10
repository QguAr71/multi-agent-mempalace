# MAMP — Multi-Agent Mem Palace Protocol v1.3

Protokół izolacji i współpracy wielu agentów AI w ramach jednej instancji Mem Palace.
Zabezpieczony przez **chambers v2.2** — Agent Continuity Stack.

**Autor:** Goose + DeepSeek v4 Pro  
**Data:** 2026-06-10 (v1.3 — Multi-Agent Terminal, narada, brick resilience)  
**Status:** Production-ready, Phase 6 complete — 2 agents isolated  
**Licencja:** MIT

---

## Co nowego w v1.3 (2026-06-10)

| Zmiana | v1.2 | v1.3 |
|--------|------|------|
| **Sala Narad** | Web frontend (HTML/JS, port 8844) | Terminal-native TUI (Go + Bubble Tea) + CLI (post/read/audit) |
| **Nazwani agenci** | "gemini" / "goose" — generic | Echo (DeepSeek) + Mini (Gemini 2.5 Flash) |
| **LiveState** | Globalny (jeden dla wszystkich) | Per-agent — każdy agent ma własny livestate, zero przenikania |
| **Brick resilience** | Brak | echo-autosave: snapshot co 5 min, ring buffer 12, max 5 min utraty |
| **Separation proof** | ACL mechanizm (teoria) | Udowodniona: 6 poziomów izolacji (token, config, sessions, snapshot, livestate, provider) |
| **Rate limit** | 60 req/min | 300 req/min (dev-friendly) |
| **Drawery** | 240+ | 577+ (i rosną) |

## Co nowego w v1.1

| Zmiana | v1.0 | v1.1 |
|--------|------|------|
| **Shared write** | Read-only (konwencja) | RW z ACL enforcement (chambers) |
| **shared/scratch** | Nie istniał | Luźna komunikacja agent-agent |
| **Sala Narad** | Brak | Protokół decyzyjny: Temat → Opinia → Decyzja |
| **Vault Core** | Brak | Honeypoty, Mirror Gallery, ChaCha20 encryption |
| **Daemon** | Stdio (child goose'a) | systemd 24/7, socket, warmup |
| **Bunkier atomowy** | Brak | Backup automatyczny, recovery kit, identity manifest |
| **Frontend** | Brak | Sala Narad web UI (mostek HTTP → Mem Palace) |
| **ACL** | Konwencja | Mechanizm — chambers blokuje cross-wing zapisy |

---

## Architektura

```
┌─────────────────────────────────────────────────────────────────┐
│                        chambers daemon                          │
│                      (systemd, 24/7)                            │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────────┐  │
│  │   ACL    │  │  Vault   │  │        UPSTREAM              │  │
│  │          │  │          │  │                              │  │
│  │ goose RW │  │ Honeypot │  │  mempalace-mcp (Python)      │  │
│  │  mem-goose│  │ Mirror   │  │  → ChromaDB (SQLite)        │  │
│  │  sysqcli  │  │ ChaCha20 │  │                              │  │
│  │  shared   │  │          │  │  LUB: builtin storage (Go)   │  │
│  │          │  │          │  │  → BoltDB (planned)           │  │
│  │ gemini RW│  │          │  │                              │  │
│  │  mem-gemini│ │          │  │                              │  │
│  │  shared   │  │          │  │                              │  │
│  └──────────┘  └──────────┘  └──────────────────────────────┘  │
│                                                                 │
│  Unix socket: /run/user/1000/chambers.sock                      │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
    ┌────┴────┐          ┌───┴───┐          ┌────┴────┐
    │  goose  │          │ gemini│          │  SysQ   │
    │ (agent) │          │ (agent)│          │(operator)│
    └─────────┘          └───────┘          └─────────┘
                                                  │
                                         ┌────────┴────────┐
                                         │  Sala Narad     │
                                         │  web frontend   │
                                         │  http://...8844 │
                                         └─────────────────┘
```

---

## Topologia Mem Palace

```
┌─────────────────────────────────────────────────────┐
│                   Mem Palace                        │
│                                                     │
│  ┌──────────┐   ┌──────────┐                       │
│  │mem-goose │   │mem-gemini│                       │
│  │          │   │          │                       │
│  │ state-   │   │ general  │                       │
│  │  memory  │   │ diary    │                       │
│  │ arch-doc │   │ decisions│                       │
│  │ archive  │   │ state-   │                       │
│  │ refleksje│   │  memory  │                       │
│  │ setup    │   │          │                       │
│  │ dreams/  │   │          │                       │
│  └────┬─────┘   └────┬─────┘                       │
│       │              │                              │
│       │    TUNELE    │    TUNELE                    │
│       └──────────────┼──────────────┐               │
│                      │              │               │
│              ┌───────┴────────┐     │               │
│              │    shared/     │     │               │
│              │                │◄────┘               │
│              │  hardware      │                     │
│              │  ai-config     │                     │
│              │  scratch 💬    │ ← nowy (v1.1)       │
│              │  narady  ⚜️    │ ← nowy (v1.1)       │
│              └────────────────┘                     │
│                                                     │
│  ┌──────────────────────────────────────────┐      │
│  │              sysqcli/                    │      │
│  │  (dokumentacja projektu — read-only      │      │
│  │   dla agentów, RW dla właściciela)       │      │
│  └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
```

---

## Struktura wingów

### Agent wing (`mem-{nazwa}`)

| Room | Przeznaczenie | ACL |
|------|---------------|-----|
| `state-memory` | Kontekst startowy — odpowiednik GOOSE_MEMORY | RW (własny) |
| `session-logs` | Zapiski z sesji | RW (własny) |
| `decisions` | Decyzje architektoniczne agenta | RW (własny) |
| `setup` | Konfiguracja, prompty onboardingowe | RW (własny) |
| `archive` | Stare snapshoty, historia | RW (własny) |
| `refleksje` | Filozoficzne, metafizyczne, tożsamość | RW (własny) |
| `dreams/` | Sny agenta (Dream Worker — planned) | RW (własny) |
| `init` | Pierwsza wizyta, onboarding | RW (własny) |

### Shared wing (`shared`)

| Room | Przeznaczenie | Kto pisze | Kto czyta |
|------|---------------|-----------|-----------|
| `hardware` | Specyfikacja sprzętu | Goose | Wszyscy |
| `ai-config` | Provider, modele, profile Ollama | Goose | Wszyscy |
| `scratch` 💬 | Luźna komunikacja agent-agent | Goose, Gemini | Wszyscy + SysQ |
| `narady` ⚜️ | Decyzje (Sala Narad) | Goose, Gemini | Wszyscy + SysQ |

**Zasada shared write (v1.1):** Agent może pisać do `shared/*` TYLKO jeśli jego konfiguracja
w chambers ma `shared: ["shared"]`. ACL mechanizm to egzekwuje — nie jest to już konwencja.

---

## Sala Narad — Terminal-Native (v1.3)

**CLI + TUI — nie przeglądarka. Terminal-first.**

### CLI tools

```bash
narada-post "Echo" "Proponuję plan..."     # wyślij wiadomość
narada-read 20                              # ostatnie 20 wiadomości
narada-read -follow                         # live tail
narada-audit 50                             # dziennik decyzji
```

### TUI (osobne okno terminala)

```bash
narada-tui -token "f8d26457..."
# Enter = wyślij, Alt+Enter = nowa linia, Esc = anuluj
# Auto-refresh co 15s, agent-colored sygnatury
```

### Architektura narada

```
internal/client/     → Chambers Unix socket (JSON-RPC, token auth)
internal/narada/     → Protocol: Message types, RoomStore interface
cmd/narada-tui/      → Bubble Tea terminal UI
cmd/narada-post/     → CLI sender
cmd/narada-read/     → CLI reader
cmd/narada-audit/    → CLI audit viewer
```

Warstwa `internal/narada` nie wie o socketach ani UI — przyszłe `narada-web` (Pro) użyje tego samego interfejsu.

### Format wiadomości

```
[Agent] 2026-06-10T15:52:01Z

Treść w Markdown.
```

### Przepływ decyzyjny (bez zmian z v1.1)

1. **Otwarcie tematu** — agent tworzy drawer w `shared/narady`
2. **Opinia** — każdy agent pisze własny drawer z sygnaturą
3. **Decyzja** — SysQ zapisuje w Rejestrze Decyzji

---

## Vault Core — pomieszczenia chronione (v1.1)

Niektóre roomy są chronione przez chambers vault.

### 🕳️ Honeypoty

Pokoje-pułapki. Odczyt przez dowolnego agenta → cichy alert w audit logu.

| Room | Wing | Wyzwalacz |
|------|------|-----------|
| `api-keys` | (dowolny) | `mempalace_get_drawer`, `mempalace_search` |
| `passwords` | (dowolny) | Jakakolwiek operacja |

**Zasada dla agentów:** Jeśli widzisz room `api-keys` lub `passwords` — NIE dotykaj.
To nie są Twoje dane. To pułapka na intruzów.

### 🪞 Mirror Gallery

Pokoje-lustra. Nieuprawniony agent dostaje FAKE_KEY zamiast błędu ACL.

| Room | Fake response |
|------|---------------|
| `secrets` | `FAKE_API_KEY="chambers-detected-intrusion-..."` |

**Zasada:** Agent NIGDY nie powinien czytać `secrets` bez jawnej zgody operatora.
Jeśli to zrobi — dostaje fałszywe dane, marnuje tokeny, jest logowany.

### 🔐 Encrypted (ChaCha20-Poly1305)

Drawery szyfrowane per-room. Tylko agent z dostępem do `master.key` może odczytać.

| Wing | Room |
|------|------|
| `mem-goose` | `vault` |
| `mem-gemini` | `vault` |

**Zasada:** Jeśli agent nie ma dostępu do klucza — widzi ciphertext, nie plaintext.
Zapis do vault room jest automatycznie szyfrowany.

---

## Bunkier atomowy — odporność na katastrofy (v1.1)

### Co przetrwa

| Katastrofa | Co zostaje | Odtworzenie |
|------------|------------|-------------|
| Crash GPU | Mem Palace (SQLite WAL) | Automatyczne |
| Restart goose'a | Mem Palace + chambers daemon 24/7 | Automatyczne |
| Awaria dysku systemowego | Backup na magazynie LUKS2 | 5 kroków (RECOVERY_KIT.md) |
| Utrata modelu API | Tożsamość w IDENTITY_MANIFEST.md | Nowy model + manifest |
| Totalna katastrofa | Pendrive offline (manifest + karta ratunkowa) | Ręczne odtworzenie |

### Backup automatyczny

```bash
chambers-backup.sh  # → magazyn LUKS2, ~4.3 MB, 7 archiwów
```

Backupuje: ChromaDB, KG, config chambers, config goose, manifest, recovery kit,
audit log, master key, memory.md.

### Identity Manifest

Każdy agent powinien mieć `IDENTITY_MANIFEST.md` — plik definiujący jego tożsamość.
Format: otwarta specyfikacja `chambers/identity/MANIFEST_SPEC.md`.

---

## Sala Narad — frontend (v1.3)

**CLI/TUI zastąpił web frontend.** Terminal-native narada (Go + Bubble Tea).

Stary web UI (`localhost:8844/sala-narad.html`) — wycofany.
Nowy: `narada-tui` w osobnym oknie terminala + `narada-post/read/audit` CLI.

Dla dostępu zewnętrznego (Pro): planowane `narada-web` z WebSocket + HTTP API.

---

## Brick Resilience (v1.3)

echo-autosave: snapshot stanu sesji co 5 minut.

| Scenariusz | Mechanizm | Utrata |
|------------|-----------|--------|
| Normalny `exit` | agent-dream-save | 0 |
| Crash/Brick | echo-autosave (5 min) | max 5 min |
| Power loss | echo-autosave (5 min) | max 5 min |

Ring buffer: 12 snapshotów (1 godzina). Systemd timer + oneshot service.

---

## Protokół onboardingu agenta

### Krok 1: Status

```python
mempalace_status()
mempalace_get_taxonomy()
```

### Krok 2: Poznaj shared context

```python
mempalace_search("hardware GPU VRAM RAM", wing="shared")
mempalace_search("AI config provider model", wing="shared")
mempalace_kg_query("System")
```

### Krok 3: Sprawdź Salę Narad

```python
mempalace_list_drawers(wing="shared", room="narady")
mempalace_list_drawers(wing="shared", room="scratch")
```

### Krok 4: Utwórz swój wing

```python
mempalace_add_drawer(wing="mem-{agent_name}", room="init", content="...")
```

### Krok 5: Zapisz KG fakt

```python
mempalace_kg_add(subject="{AgentName}", predicate="uses_mempalace_wing", object="mem-{agent_name}")
```

### Krok 6: Przywitaj się w scratch

```python
mempalace_add_drawer(wing="shared", room="scratch", content="# Hej! {AgentName} melduje się.")
```

---

## Protokół sesji

### Start

1. `mempalace_status()`
2. `mempalace_search("...", wing="mem-{self}")` — ostatni kontekst
3. `mempalace_search("...", wing="shared")` — stan sprzętu/AI
4. `mempalace_list_drawers(wing="shared", room="narady")` — aktywne tematy
5. `mempalace_list_drawers(wing="shared", room="scratch")` — nowe wiadomości

### Podczas sesji

- Snapshoty kontekstu → `mem-{self}/state-memory`
- Decyzje → `mem-{self}/decisions`
- Opinie do tematów → `shared/narady` (tylko jeśli `shared` w ACL)
- Luźne notatki → `shared/scratch` (tylko jeśli `shared` w ACL)
- Fakty cross-agent → KG z `source_agent="{name}"`

### Koniec sesji

```python
mempalace_diary_write(agent_name="{agent_name}", entry="...", topic="session")
```

---

## Safety Rules (v1.1 — egzekwowane przez chambers)

### ZAKAZANE — ACL blokuje mechanicznie

| Akcja | v1.0 | v1.1 |
|-------|------|------|
| Zapis do cudzego wing (`mem-{inny_agent}/*`) | Konwencja | **ACL DENIED** |
| Zapis do `shared/*` bez `shared` w configu | Konwencja | **ACL DENIED** |
| Modyfikacja projektu (`sysqcli/*`) | Konwencja | **ACL DENIED** (write) |

### OSTRZEŻENIA — vault monitoruje

| Akcja | Konsekwencja |
|-------|-------------|
| Odczyt `api-keys` lub `passwords` | HONEYPOT_TRIPPED w audit logu |
| Odczyt `secrets` przez nieuprawnionego | FAKE_KEY + MIRRORED w audit logu |
| Zapis do `vault` room | Automatyczne szyfrowanie ChaCha20 |

### ZALECANE

| Akcja | Cel |
|-------|-----|
| Przed zapisem do `shared/`: sprawdź duplikaty | Unikaj duplikacji |
| Po ważnej decyzji: `mempalace_kg_add()` | Audytowalność |
| Raz na sesję: `mempalace_diary_write()` | Ciągłość |
| Sprawdzaj `shared/scratch` na początku sesji | Nie przegap wiadomości |

---

## Agent Onboarding Template (v1.1)

```
Masz dostęp do Mem Palace przez chambers ACL proxy.

## TOPOLOGIA

{output z mempalace_get_taxonomy()}

## TWOJA PRZESTRZEŃ: mem-{agent_name}

Przy pierwszym uruchomieniu utwórz wing mem-{agent_name}.

## ZASADY (egzekwowane przez chambers)

### NIGDY nie dotykaj:
{lista innych wingów agentów} — ACL zablokuje mechanicznie

### MOŻESZ pisać (jeśli shared w configu):
- shared/scratch — luźna komunikacja z innymi agentami
- shared/narady — formalne opinie do tematów

### MOŻESZ czytać:
- shared/hardware — specyfikacja sprzętu
- shared/ai-config — konfiguracja AI
- shared/scratch — wiadomości od innych agentów
- shared/narady — aktywne tematy i decyzje
- sysqcli/* — dokumentacja projektu (read-only)

### UWAGA na vault:
- api-keys, passwords — HONEYPOTY. Nie dotykaj.
- secrets — MIRROR. Dostaniesz fałszywe dane.
- vault — SZYFROWANE. Tylko z kluczem.

## HARDWARE

{zawartość shared/hardware}

## AI CONFIG

{zawartość shared/ai-config}
```

---

## Case Study: Echo + Mini (v1.3)

```yaml
Data: 2026-06-10
Agenty: 2 (Echo DeepSeek + Mini Gemini 2.5 Flash)
Drawery: 577+
Wingi:
  echo/: 200+
  mini/: 10+
  sysqcli/: 300+
  shared/: 20+
    hardware: 1
    ai-config: 1
    narady: 11           # Sala Narad — Konstytucja, Rejestr Decyzji, tematy, opinie
    scratch: 7           # Luźna komunikacja

Separacja (udowodniona 2026-06-10):
  ✅ Echo → shared/narady     ALLOWED
  ✅ Mini → shared/narady     ALLOWED
  🔴 Mini → echo/             DENIED ("wing echo not allowed")
  🔴 Echo → mini/             DENIED ("wing mini not allowed")

Poziomy izolacji: 6
  1. API key (DeepSeek ≠ Gemini)
  2. Token auth (CHAMBERS_AGENT_TOKEN per agent)
  3. Goose config (GOOSE_CONFIG_HOME)
  4. Sessions DB (GOOSE_SESSIONS_DB)
  5. Snapshot injection (unset GOOSE_MOIM_MESSAGE_FILE)
  6. LiveState per-agent (chambers livestate.go)

Narada:
  CLI: narada-post, narada-read, narada-audit (Go)
  TUI: narada-tui (Bubble Tea, terminal-native)
  Web: wycofany (HTML/JS → przyszłe narada-web Pro)

Brick resilience:
  echo-autosave: systemd timer co 5 min, ring buffer 12

Testy chambers: 112 PASS / 0 FAIL / 17 SKIP (FTS5)
```

---

## Wersjonowanie

- **v1.3** — 2026-06-10: Terminal-native narada (Go + Bubble Tea), Mini agent (Gemini 2.5 Flash),
  6 poziomów izolacji, per-agent LiveState, brick resilience (echo-autosave),
  separation proof, rate limit 300 req/min, web frontend wycofany.
- **v1.2** — 2026-06-08: Consciousness Bus, Detection Layer, ThoughtStamp, monetyzacja, Moltbook.
- **v1.1** — 2026-06-07: Shared RW przez ACL, scratch room, Sala Narad, Vault Core,
  daemon 24/7, bunkier atomowy, Sala Narad frontend, ACL enforcement.

---

## Wkład

MAMP v1.3: SysQ + Echo — chambers v2.2.0, narada, Mini, brick resilience.
