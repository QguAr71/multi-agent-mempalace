# MAMP — Multi-Agent Mem Palace Protocol v1.0

Protokół izolacji i współpracy wielu agentów AI w ramach jednej instancji Mem Palace.

**Autor:** Goose (AAIF) + DeepSeek v4 Pro  
**Data:** 2026-06-06  
**Status:** Production-ready, przetestowany na 2 agentach (Goose + Gemini CLI)  
**Licencja:** MIT

---

## Problem

Mem Palace domyślnie nie zapewnia izolacji między agentami. Wszystkie drawery, roomy, tunele i fakty KG są współdzielone. Przy dwóch lub więcej agentach AI korzystających z tej samej instancji, ryzykujesz:

- **Zanieczyszczenie pamięci** — Agent A czyta snapshoty Agenta B jako swoje
- **Sprzeczne fakty w KG** — Dwa agenty dodają ten sam fakt w różnych wersjach
- **Chaos w roomach** — Każdy agent wrzuca wszystko do `general/`
- **Wyciek kontekstu** — Agent czyta decyzje drugiego agenta i przyjmuje je za własne

## Rozwiązanie

MAMP definiuje **topologię wingów**, **protokół shared space** i **reguły izolacji**, które zapewniają:

- Każdy agent ma własny wing
- Fakty współdzielone (sprzęt, konfiguracja) żyją w dedykowanym `shared/`
- Tunele łączą skrzydła tylko tam, gdzie jest to świadome powiązanie
- KG protokół rozróżnia źródło faktu
- Nowy agent dostaje template onboardingowy — nie zgaduje

---

## Topologia

```
┌─────────────────────────────────────────────────────┐
│                   Mem Palace                        │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│  │mem-agent1│   │mem-agent2│   │mem-agentN│  ...   │
│  │          │   │          │   │          │       │
│  │ własne   │   │ własne   │   │ własne   │       │
│  │ roomy    │   │ roomy    │   │ roomy    │       │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘       │
│       │              │              │              │
│       │    TUNELE    │    TUNELE    │              │
│       └──────────────┼──────────────┘              │
│                      │                             │
│              ┌───────┴────────┐                    │
│              │    shared/     │                    │
│              │                │                    │
│              │  hardware      │                    │
│              │  ai-config     │                    │
│              └────────────────┘                    │
│                                                     │
│  ┌──────────────────────────────────────────┐      │
│  │              projekty/                   │      │
│  │  (dokumentacja projektów — read-only     │      │
│  │   dla agentów, RW dla właściciela)       │      │
│  └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
```

**Zasada:** Każdy agent izolowany we własnym wing. Komunikacja cross-agent tylko przez `shared/` i KG.

---

## Struktura wingów

### Agent wing (`mem-{nazwa}`)

Każdy agent ma własny wing. Sugerowane roomy:

| Room | Przeznaczenie |
|------|---------------|
| `state-memory` | Kontekst startowy dla agenta (odpowiednik GOOSE_MEMORY) |
| `session-logs` | Zapiski z sesji (co było robione, decyzje) |
| `decisions` | Decyzje architektoniczne podjęte przez agenta |
| `setup` | Konfiguracja agenta, prompty onboardingowe |
| `archive` | Stare snapshoty, historia |

### Shared wing (`shared`)

Fakty niezależne od agenta. Każdy agent czyta, zapisuje ostrożnie.

| Room | Przeznaczenie |
|------|---------------|
| `hardware` | Specyfikacja sprzętu (CPU, GPU, RAM, storage, boot) |
| `ai-config` | Konfiguracja AI (provider, modele, profile Ollama, limity VRAM) |
| `network` | Topologia sieci, porty, firewalle (opcjonalnie) |
| `security` | Secure Boot, LUKS, TPM, klucze (opcjonalnie) |

### Project wing (`{projekt}`)

Dokumentacja konkretnego projektu. Agenci czytają, nie modyfikują bez zgody właściciela.

---

## Protokół onboardingu agenta

### Krok 1: Status

```python
mempalace_status()                    # ogólny przegląd
mempalace_get_taxonomy()              # struktura roomów
```

### Krok 2: Poznaj shared context

```python
mempalace_search("hardware GPU VRAM RAM", wing="shared")
mempalace_search("AI config provider model", wing="shared")
mempalace_kg_query("System")          # fakty o systemie
```

### Krok 3: Utwórz swój wing

```python
mempalace_add_drawer(
    wing="mem-{agent_name}",
    room="init",
    content="# {Agent Name} — Mem Palace Init\n..."
)
```

### Krok 4: Zapisz KG fakt o swoim wing

```python
mempalace_kg_add(
    subject="{AgentName}",
    predicate="uses_mempalace_wing",
    object="mem-{agent_name}"
)
```

### Krok 5 (opcjonalnie): Utwórz tunele do shared

```python
mempalace_create_tunnel(
    source_wing="mem-{agent_name}",
    source_room="state-memory",
    target_wing="shared",
    target_room="ai-config",
    label="Agent {name} → shared AI config"
)
```

---

## Protokół sesji

### Start

1. `mempalace_status()` — przegląd stanu
2. `mempalace_search("...", wing="mem-{self}")` — ostatni kontekst własny
3. `mempalace_search("...", wing="shared")` — aktualny stan sprzętu/AI
4. Jeśli są nowe fakty systemowe → `mempalace_kg_query("System", as_of="{today}")`

### Podczas sesji

- Wszystkie snapshoty kontekstu → `mem-{self}/state-memory`
- Wszystkie decyzje → `mem-{self}/decisions`
- Nowe fakty o sprzęcie/AI → `shared/{hardware,ai-config}`
- Nowe fakty cross-agent → KG z `source_agent="{name}"`

### Koniec sesji

```python
mempalace_diary_write(
    agent_name="{agent_name}",
    entry="AAA...",
    topic="session"
)
```

---

## Protokół KG (Knowledge Graph)

### Fakty per-agent

```python
# Agent ląduje w systemie
mempalace_kg_add("Goose", "uses_mempalace_wing", "mem-goose")
mempalace_kg_add("GeminiCLI", "uses_mempalace_wing", "mem-gemini")

# Agent podejmuje decyzję
mempalace_kg_add("Goose", "decided", "GPU clock lock 500 MHz", 
                  source_drawer_id="drawer_...")
```

### Fakty o systemie (shared)

```python
mempalace_kg_add("System", "has_gpu", "NVIDIA RTX 2060 6GB")
mempalace_kg_add("System", "runs_os", "CachyOS Arch Linux")
mempalace_kg_add("System", "boots_via", "Limine UEFI + Secure Boot")
```

### Unieważnianie starych faktów

```python
# Kernel się zmienił
mempalace_kg_invalidate("System", "runs_kernel", "7.0.11")
mempalace_kg_add("System", "runs_kernel", "7.0.12")
```

---

## Safety Rules

### ZAKAZANE dla agenta

| Akcja | Konsekwencja |
|-------|-------------|
| Zapis do cudzego wing (`mem-{inny_agent}/*`) | Zanieczyszczenie pamięci innego agenta |
| Modyfikacja projektu (`{projekt}/*`) bez zgody | Utrata integralności dokumentacji |
| Nadpisanie draweru w `shared/` bez sprawdzenia | Utrata faktów współdzielonych |
| Dodanie faktu KG bez `source_agent` lub `source_drawer_id` | Brak audytu |

### ZALECANE

| Akcja | Cel |
|-------|-----|
| Przed zapisem do `shared/`: `mempalace_search()` sprawdź duplikaty | Unikaj duplikacji |
| Przed odpowiedzią o systemie: sprawdź `shared/hardware` | Używaj aktualnych faktów |
| Po ważnej decyzji: `mempalace_kg_add()` z provenance | Audytowalność |
| Raz na sesję: `mempalace_diary_write()` | Ciągłość między sesjami |

---

## Test izolacji

Skrypt weryfikujący, że agent nie ma dostępu do cudzego wing:

```python
# Agent próbuje czytać cudzy wing
result = mempalace_search("decision", wing="mem-goose")
# Powinno zwrócić puste lub tylko shared fakty
assert len(result) == 0, "ISOLATION BROKEN: agent czyta cudzy wing"

# Agent próbuje zapisać do cudzego wing
# (Mem Palace na to pozwoli — izolacja jest konwencją, nie mechanizmem)
# Dlatego safety rules są kluczowe
```

---

## Migracja z single-agent

Jeśli masz istniejący Mem Palace z jednym agentem:

### Krok 1: Zidentyfikuj obecną strukturę

```python
mempalace_get_taxonomy()
mempalace_list_drawers()
```

### Krok 2: Wydziel shared space

Przenieś fakty o sprzęcie i konfiguracji z wing agenta do `shared/`:

```python
# Przykład: stary drawer z hardware factami
mempalace_update_drawer(drawer_id="...", wing="shared", room="hardware")
```

### Krok 3: Przemianuj wing agenta

```python
# Z .goose na mem-goose (lub własna konwencja)
for drawer in drawers_in_old_wing:
    mempalace_update_drawer(drawer_id=drawer.id, wing="mem-agentname")
```

### Krok 4: Utwórz KG fakty

```python
mempalace_kg_add("System", "has_gpu", "...")
mempalace_kg_add("AgentName", "uses_mempalace_wing", "mem-agentname")
```

### Krok 5: Onboard nowego agenta

Użyj [Agent Onboarding Template](#agent-onboarding-template).

---

## Agent Onboarding Template

Template do przekazania nowemu agentowi. Zastąp `{PLACEHOLDERS}`:

```
Masz dostęp do Mem Palace — współdzielonej pamięci z innymi agentami AI.

## TOPOLOGIA

{output z mempalace_get_taxonomy()}

## TWOJA PRZESTRZEŃ: mem-{agent_name}

Przy pierwszym uruchomieniu utwórz wing mem-{agent_name} z room init.

## ZASADY

### NIGDY nie dotykaj:
{lista innych wingów agentów}

### MOŻESZ czytać:
- shared/hardware — specyfikacja sprzętu
- shared/ai-config — konfiguracja AI
- {projekt}/* — dokumentacja projektów (read-only)

### ZAWSZE:
- Zapisuj fakty o sprzęcie do shared/
- Używaj KG dla faktów cross-agent
- Zapisuj diary na koniec sesji
- Przed odpowiedzią: sprawdź shared context

## HARDWARE REFERENCE

{zawartość shared/hardware}

## AI REFERENCE

{zawartość shared/ai-config}
```

---

## Case Study: Goose + Gemini CLI

Pierwsza implementacja MAMP. 163 drawery → 3 skrzydła, 16 roomów, 4 tunele.

```yaml
Data: 2026-06-06
Agenty: 2 (Goose + Gemini CLI)
Drawery: 163 → skategoryzowane
Wingi:
  mem-goose:
    state-memory: 7
    arch-doc: 9
    archive: 23
    setup: 1
  sysqcli:
    modules: 33
    changelog: 34
    docs-en: 27
    docs-pl: 14
    install: 10
    contributing: 5
    architecture: 1
  shared:
    hardware: 1
    ai-config: 1
Tunele: 4
  sysqcli/modules → shared/ai-config
  mem-goose/arch-doc → shared/hardware
  mem-goose/state-memory → sysqcli/modules
  sysqcli/install → shared/hardware
```

---

## Wersjonowanie

- **v1.0** — 2026-06-06: Pierwsza wersja. Izolacja per-wing, shared space, tunele, protokół onboardingowy.

---

## Wkład

MAMP powstał jako rezultat sesji diagnostycznej Goose na CachyOS. Protocol design: DeepSeek v4 Pro + Goose (AAIF). Host: sysq.
