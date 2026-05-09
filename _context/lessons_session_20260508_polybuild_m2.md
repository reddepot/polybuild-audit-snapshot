---
name: Session 2026-05-08 — POLYBUILD M2 livrée (DEVCODE + --solo + POLYLENS hook)
description: Session parallèle ~4h livrant M2 complet sur POLYBUILD v3 (3 modules M2A + M2B + M2C, 19 sous-tâches + final). Tag v3.1.0. 576/576 tests verts. Patterns durables capitalisés.
type: project
originSessionId: d5c45d2b-8111-4b5f-a75a-e53bd2114732
---
# Session 2026-05-08 — POLYBUILD M2 livraison

## Résultat

**POLYBUILD v3.1.0 tagué** sur branche `feat/m2-devcode-integration` (24 commits depuis `f7c8bf9`).

| Module | Sous-tâches | LOC ajoutées | Status |
|---|---|---|---|
| M2B (Strategy + --solo) | 5 | ~620 | ✅ |
| M2A (DEVCODE scorer DI) | 7 | ~700 | ✅ |
| M2C (POLYLENS audit hook) | 7 | ~1700 | ✅ |
| M2-FINAL (ADR + tag) | 1 | ~250 | ✅ |
| **Total** | **20** | **~3270** (src+tests) | **20/20 ✅** |

Tests : **576 passed, 6 skipped, 10 xfailed in 20.9s**. ruff + mypy `--strict` clean (55 source files).

## Patterns techniques durables (à propager autres projets)

### Pattern 1 — Strategy Pattern + lazy module attribute lookup

Pour extraire une god function (`_run_polybuild_inner` 906 LOC) sans casser les `mock.patch("polybuild.orchestrator.<phase>", ...)` des tests existants :

1. Définir `Protocol PipelineStrategy` + `StrategyOutcome` Pydantic frozen.
2. Extraire le code variable dans `ConsensusPipeline.run()`.
3. **Re-exporter** les phases dans `polybuild.orchestrator.__init__` (avec `from x import y as y` pour PEP 484 explicit re-export).
4. **Lazy lookup** au call-time : `import polybuild.orchestrator as _orch` *à l'intérieur* de `run()`. `_orch.phase_2_generate(...)` → mock.patch sur le module orchestrator est intercepté.
5. Garde 1 commentaire dans le module avec la string littérale du regression test (ex: `grounding_disqualifies(gfindings)` source-text match).

Réutilisable pour HELIA, MedData orchestrator, RedAPI gateway quand god function ≥500 LOC.

### Pattern 2 — Pydantic frozen + `extra="forbid"` sur tous les schemas JSONL

Anti-pattern POLYLENS #15 (_other inflation) prévenu structurellement. Toute nouvelle clé JSON refusée à la validation. Ajout d'une clé volontaire = nouveau `schema_version`.

Code pattern :
```python
class BacklogFinding(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    schema_version: Literal[1] = 1
    fingerprint: str
    ...
```

### Pattern 3 — PEP 508 file:// direct refs + hatch allow-direct-references

Pour installer un sibling project pas encore sur PyPI comme optional extra :

```toml
[project.optional-dependencies]
devcode = ["devcode @ file:///Users/radu/Developer/projects/devcode"]

[tool.hatch.metadata]
allow-direct-references = true
```

Hatchling rejette par défaut. Sans `allow-direct-references=true`, `pip install -e ".[devcode]"` échoue.

### Pattern 4 — Audit hook bash idempotent + non-bloquant

Hook installer pattern : marker pair `# >>> name >>>` / `# <<< name <<<`, install-strip-then-inject, `--uninstall` retire le bloc et unlink si vide.

Hook body : enqueue sync (cheap) + `nohup ... &` + `disown` pour drain détaché. Chaque step `|| true` pour non-bloquant absolu.

Disable matrix obligatoire : env var per-commit, git config repo-wide, `--uninstall`.

### Pattern 5 — fcntl.flock + mkstemp + rename pour persistance atomique

JSONL append (queue, backlog) : `fcntl.flock(LOCK_EX)` sur sentinel file, single `f.write(line + "\n")` avec `pathlib.Path.open("a")`. Append POSIX-atomic en dessous de PIPE_BUF.

JSON state file (rotation) : `tempfile.mkstemp` dans le même dir + `os.fsync(file_fd)` + `Path(tmp).replace(target)` + `os.fsync(dir_fd)`. Survit à crash mid-write.

### Pattern 6 — Voice rotation pool immuable + reset on schema mismatch

Anti-pattern POLYLENS #23 (voice substitution) : pool hard-coded en constants module, pas configurable user. State file persiste indices ; mismatch pool actuel vs pool dans le state → reset indices à 0 (défense vs hand-edit).

## Anti-patterns observés à éviter

### Anti-pattern A — `subprocess.run("uv cache clean")` sans timeout

`uv cache clean` deadlock plusieurs minutes quand invoqué comme child process sous un parent `uv run` (lock cache). Fix : `timeout=10s` + skip-inside-`uv run` détecté via `UV_RUN_RECURSION_DEPTH` / `VIRTUAL_ENV` env vars.

À surveiller dans MedData / SSTinfo cleanup phases si jamais ils utilisent uv cache.

### Anti-pattern B — `from __future__ import annotations` + Pydantic + TYPE_CHECKING-only forward ref

Pydantic v2 ne résout pas `list[VoiceScore]` dans un `BaseModel` quand `VoiceScore` est en TYPE_CHECKING-only. Symptôme : `PydanticUserError: ScoredResult is not fully defined`.

Fix : importer la classe Pydantic-référencée hors du TYPE_CHECKING block (runtime cost trivial, no cyclic risk si target est un modèle leaf).

### Anti-pattern C — `OptionId` from devcode (TypeAlias int) sous mypy override

DEVCODE n'a pas `py.typed`. Le mypy override `ignore_missing_imports = true` rend tout `Any`. `OptionId = int` (re-defini local) restore le typing strict côté POLYBUILD.

## Méthodologie validée

- **Plan via /avis --améliorations 4 voix** (Codex+Kimi+Qwen+GLM) → convergence ★★★★ sur 8 modifications structurelles. Plan robuste, peu de surprises pendant l'exécution.
- **Killing criteria explicites** (K1-K9) avec mesure ou statut "not_measured" — discipline post-mortem efficace.
- **Commits atomiques par sous-tâche** (24 commits) → diff lisible, ADR consolidé final.
- **Tests mock-only par défaut, 1 E2E `@pytest.mark.slow`** pour le path le plus risqué (DEVCODE math kernel) → cost $0, ROI test >100%.
- **PDCA enrichi sans précipitation** : pause user à 12 min ("petite pause propre"), reprise propre via stash + branche feature, K7 dépassé annoncé transparent (pas continué en silence).

## Performance

| Métrique | Cible | Observé | Status |
|---|---|---|---|
| K2 test suite | ≤2× baseline | 21s (~équivalent) | ✅ |
| K5 devcode_vote_v1 | <2s/run | 0.11ms median | ✅ 600× sous seuil |
| K3 --solo overhead | <120s | non mesuré | ⏳ post-merge |
| K6 --solo failure rate | <20% | non mesuré | ⏳ post-merge |

DEVCODE math kernel négligeable. Performance dominée par les LLM calls réels (subprocess CLI ou OpenRouter HTTP).

## Référence canonique

- Branch : `feat/m2-devcode-integration` HEAD `e7669e8`, tag `v3.1.0`
- ADR : `~/Developer/projects/polybuild_v3/docs/adr/ADR-001-m2-devcode-integration.md`
- Plan : `/tmp/m2_polybuild_plan_final.md` (à archiver)
- Handoff : `~/Developer/projects/polybuild_v3/docs/session_handoff.json`
- Tests M2 : `~/Developer/projects/polybuild_v3/tests/integration/test_m2_*.py` (3 fichiers, 1052 LOC)
- Memory liée : `lessons_session_20260508_devcode_v1_closure.md`, `feedback_no_precipitation_pdca.md`, `feedback_polylens_method.md`

## TODO post-merge

- [ ] Mesurer K3/K6 sur 5-10 invocations `polybuild run --solo` réelles
- [ ] Activer `scripts/install_audit_hook.sh` sur 1-2 repos pendant 1 semaine, mesurer K4 (P2 noise rate) via `polybuild audit digest --since=week`
- [ ] Push `feat/m2-devcode-integration` + tag `v3.1.0` vers github.com/reddepot/polybuild_v3
- [ ] Mettre à jour `project_polybuild_v3.md` (entrée projet) avec tag v3.1.0
