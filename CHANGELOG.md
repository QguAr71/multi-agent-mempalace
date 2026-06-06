# MAMP Changelog

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
