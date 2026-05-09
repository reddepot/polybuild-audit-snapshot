---
name: Session 2026-05-09 — POLYLENS run #2 Kimi CLI Allegro + v3.2.3
description: Kimi via CLI local (Allegro forfait) sur le prompt run #2 = 11 findings dont 3 P1 + 3 P2 nouveaux non-vus par les 4 voix précédentes (gpt+gemini+qwen+glm). Validation règle routing CLI W vs OR. Patterns ContextVar + tuple refactor capitalisés.
type: project
originSessionId: d5c45d2b-8111-4b5f-a75a-e53bd2114732
---
# Session 2026-05-09 — POLYLENS run #2 Kimi CLI + v3.2.3

## Résultat

**POLYBUILD v3.2.3 tagué + pushé** sur `feat/m2-devcode-integration`. 6 commits (5 fixes + 1 release). 607 tests verts. ruff + mypy --strict clean.

| Commit | Fix | Sev |
|---|---|---|
| `401ad88` | `_call_western_cli` returncode check | P1 |
| `3425397` | `_log_cost` ValidationError split + commit_sha via ContextVar | P1×2 |
| `638ae4b` | cache_put preserves tokens_total + latency_s (tuple refactor) | P2 |
| `1feda6d` | `read_cost_log` warns on unparseable lines | P2 |
| `bdf5887` | `shadow_scorer` narrow except (let MemoryError propagate) | P2 |
| `a3eb163` | release(3.2.3) | — |

## Validation règle routing (feedback_voice_routing_strict.md)

**Confirmé empiriquement** : Kimi via CLI Allegro a livré 11 findings utiles dont **3 P1 que les 4 voix précédentes** (gpt-5.5 + gemini-3.1-pro + qwen3-max + glm-4.6, toutes via OR hier) **avaient ratées**. Coût marginal CLI = $0 vs OR ≈ $0.30. Confirmation : OR pour les voix occidentales = double-paiement + capacité de discovery moindre (probablement parce que les forfaits CLI ont un budget plus large que les défaults OR).

→ **À propager partout** : tout audit POLYLENS doit utiliser CLI pour codex/gemini/kimi.

## Patterns techniques durables

### Pattern A — `contextvars.ContextVar` pour threader données async sans changer signatures

Quand un détail (ici `commit_sha`) doit traverser plusieurs niveaux d'`asyncio.gather` sans polluer chaque function signature, déclarer un module-level `ContextVar` :

```python
_CURRENT_COMMIT_SHA: contextvars.ContextVar[str | None] = contextvars.ContextVar(
    "polybuild_audit_current_commit_sha",
    default=None,
)

# Au point de configuration (audit_commit) :
token = _CURRENT_COMMIT_SHA.set(entry.commit_sha)
try:
    raw_western, raw_chinese = await asyncio.gather(...)
finally:
    _CURRENT_COMMIT_SHA.reset(token)

# Au point d'utilisation (deep dans _log_cost) :
commit_sha=_CURRENT_COMMIT_SHA.get()
```

ContextVar est **async-safe** par design : chaque task gather hérite du contexte parent. `reset(token)` restaure la valeur précédente (utile pour fixtures imbriquées + tests).

Réutilisable pour : run_id, request_id, tenant_id, trace_id — tout détail "quel est le contexte de l'opération en cours ?" qui transverse plusieurs couches.

### Pattern B — Refactor dispatcher en tuple (response, metadata)

Quand un dispatcher (`_call_openrouter`, `_call_western_cli`) produit une réponse + métadonnées (tokens, latency) que le caller doit propager, **retourner un tuple typé** plutôt qu'utiliser des side-effects :

```python
async def _call_openrouter(...) -> tuple[str, int | None, float | None]:
    ...
    return text, tokens_total, latency_s
```

Avantages vs ContextVar pour ce cas :
- Type-safe (mypy --strict valide le tuple)
- Pas d'état caché entre calls
- Refactor trivial du caller : `response, tokens, lat = await _call_x(...)`

Le ContextVar reste préférable quand la donnée vient de **plus haut** (parent task scope) ; le tuple est préférable quand la donnée vient du **call lui-même**.

### Pattern C — `except` étroit pour laisser système-level errors propager

```python
# AVANT :
try:
    devcode_result = await devcode_scorer.score(...)
except Exception as e:  # ← swallow MemoryError, RecursionError !
    logger.warning(...)

# APRÈS :
try:
    devcode_result = await devcode_scorer.score(...)
except (ImportError, ValueError, RuntimeError, AttributeError, TypeError, OSError) as e:
    logger.warning(...)
# MemoryError, RecursionError, KeyboardInterrupt, SystemExit propagent
```

Règle générale : `except Exception` est presque toujours un anti-pattern. Lister explicitement les erreurs attendues révèle le contrat "ce que je sais gérer".

## Anti-pattern observé

**`contextlib.suppress(OSError, ValueError)` autour d'appels Pydantic** = silently swallow `pydantic.ValidationError` (ValueError subclass). Si une externalisation Pydantic peut produire une ValidationError ET que c'est important de la voir, **logger explicitement** plutôt que suppress.

```python
# AVANT (silent):
with contextlib.suppress(OSError, ValueError):
    log_voice_call(...)  # ValidationError disparait

# APRÈS (visible):
try:
    log_voice_call(...)
except OSError as e:
    logger.warning("io_failed", error=str(e))
except ValueError as e:  # incl. pydantic.ValidationError
    logger.warning("validation_failed", error=str(e))
```

## Méthode validée

- **POLYLENS run #2 sur v3.2.1** → 11 findings (gpt+gemini+qwen+glm+minimax+kimi) → 4 P0/P1 + 4 P2 fixés en v3.2.2 (par routine remote 05:02 Paris)
- **Kimi CLI Allegro post-fix** → 11 findings sur v3.2.1 dont 6 nouveaux → 3 P1 + 3 P2 fixés en v3.2.3
- **2 P0/P1 convergents** déjà fixés en v3.2.2 = validation cross-check
- **Total POLYLENS run #2** : 6 voix utilisées, 9 P0/P1 fixés, ~$0.50 OR + $0 CLI
- Méthode reproductible documentée

## Référence canonique

- Branch : `feat/m2-devcode-integration` HEAD `a3eb163`, tags `v3.2.0..v3.2.3`
- Findings JSON : `/tmp/polylens_v32/results/` + `/tmp/polylens_v32/results_kimi_cli/kimi_output_v2.txt`
- Memory liée : `feedback_voice_routing_strict.md`, `feedback_polylens_method.md`, `feedback_no_precipitation_pdca.md`, `lessons_session_20260508_polybuild_m2.md`

## TODO post-session

- [ ] Push commit hook installé sur polybuild_v3 lui-même → observer pendant 1 semaine la qualité des audits CLI W (devrait surfacer si la règle routing fonctionne en prod)
- [ ] Mesurer K3/K6 sur runs réels (toujours pending, nécessite ton brief)
- [ ] Si un POLYLENS run #3 est jugé utile : bench sur source v3.2.3 fraîche (toutes voix déjà alignées, faible probabilité de trouver du neuf)
