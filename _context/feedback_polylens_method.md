---
name: POLYLENS — méthodologie audit/debugging multi-modèles orthogonaux
description: Méthodologie canonique v3 (2026-05-03). 7 axes + panel base 4 occidentales immuable + 5 chinoises ajoutées + 5 phases gated + convergence cross-culturelle + 23 anti-patterns + Honeypot. Référer par "POLYLENS".
type: feedback
originSessionId: 1fdc0fb7-773a-4edc-a116-89d2359f5999
---
# POLYLENS — Méthodologie d'Audit Multi-Modèles Orthogonaux

## Acronyme
**POLYLENS** = Parallel Orthogonal Lens-based Yielding Empirical Network of Sentinels

## Mantra
> Sept axes lus en parallèle. Le mécanisme causal arbitre. La régression scelle.

---

## 0. Les 7 axes d'audit

Tout run doit déclarer ses axes via `audit_scope.yaml`. Run sans déclaration = **REJET**.

| # | Axe | Question centrale | Lentille primaire | SAST baseline |
|---|-----|-------------------|-------------------|---------------|
| A | Sécurité / RGPD / opposabilité | "Comment fuiter, bypasser, exploiter ?" | Kimi (adversarial) + Codex (PoC) + Qwen 3.6 (contracts) | bandit, semgrep, pip-audit |
| B | Qualité / dette technique | "Code smells, duplications, idioms ?" | Kimi (archéologue 2M) + Codex + MiniMax (patterns Tencent) | ruff --select=ALL, vulture, radon |
| C | Tests & debugging | "Coverage, scénarios, mocks, flakiness ?" | Codex (PoC) + Gemini (cross-tests) + qwen3-coder (tuning) | pytest --collect-only, coverage, mutmut |
| D | Performance | "Hot paths, N+1, async/sync ?" | Codex + Gemini (trace flux) + MiMo (patterns distribués) | py-spy, hyperfine, EXPLAIN |
| E | Architecture & couplage | "Cohérence inter-modules, contrats ?" | Gemini (1M graphe) + Kimi + GLM 5.1 + Qwen 3.6 | pydeps, import-linter |
| F | Documentation & DX | "Docstrings, README obsolète, onboarding ?" | Claude + Gemini + GLM 5.1 | interrogate, pydocstyle |
| G | Adversarial / chaos / robustesse | "Inputs malicieux, prompt injection, races ?" | Kimi + Codex + MiniMax | hypothesis, chaos engineering |

### Quand activer quel axe

| Contexte | Axes |
|----------|------|
| Pré-prod gate | A + C + G |
| Médico-juridique opposable | A + E + F |
| Audit annuel complet | A + B + C + D + E + F + G |
| Refactor archi | B + E + C |
| Performance prod | D + E + C |
| Onboarding | F + B + E |
| Publication scientifique (HELIA) | A + E + F |
| Bug fix simple | aucun POLYLENS |

---

## 1. Les voix — Panel de base + ajouts de diversité

### Principe
**4 voix occidentales = panel de base immuable.** Les voix chinoises sont **ajoutées** selon les axes activés — jamais de substitution. Objectif : casser le groupthink à ~80% de corpus partagé entre modèles occidentaux.

### Panel de base (toujours présent)

| Voix | Provider | CLI | Force |
|------|----------|-----|-------|
| Claude Opus 4.7 | Anthropic | Claude Code | Médiation, architecture, documentation |
| Codex GPT-5.5 | OpenAI | Codex CLI | PoC, tests, performance, sécurité |
| Gemini 3.1 Pro | Google | Gemini CLI | Cross-tests, graphe architecture, flux |
| Kimi K2.6 | Moonshot | Kimi CLI | Adversarial, archéologue 2M, bypass |

### Voix chinoises (ajout par axe)

| Voix | Provider | OR Slug | Force orthogonale | Axes |
|------|----------|---------|-------------------|------|
| GLM 5.1 | Zhipu | zai/glm-5.1 | Conservateur incrémental, stable evolution | E, F |
| MiniMax M2.7 | MiniMax | minimax/m2.7 | Patterns microservices légers Alibaba/Tencent, chaos agressif | B, G |
| MiMo V2.5-Pro | Xiaomi | xiaomi/mimo-v2.5-pro | Architecture distribuée résiliente, graceful degradation | D |
| Qwen 3.6 Plus | Alibaba | qwen/qwen3.6-plus | Contract checker stateless, validation formelle interfaces | A, E |
| qwen3-coder-plus | Alibaba | qwen/qwen3-coder-plus | Perspective optimization non-occidentale, tuning bas niveau | C, D |

### Règles de panel

- **Panel minimum** : 4 voix (base occidentale) + 1 voix chinoise = **5 voix**
- **Panel standard** : 4 voix base + 2 voix chinoises = **6 voix**
- **Panel complet** : 4 voix base + 3-4 voix chinoises = **7-8 voix**
- **Substitution interdite** : une voix occidentale ne peut jamais être retirée pour ajouter une chinoise
- **Monoculture interdite** : un run avec 0 voix chinoise sur un axe activé = REJET

### Anti-rôles

- ❌ Kimi sur micro-injection SQL → gaspille 2M ctx
- ❌ Qwen sur arbitrage cross-fichiers → hallucine consensus
- ❌ Gemini sur crypto microscopique → préférer Codex
- ❌ Claude juge unique → préférer assemblage déterministe
- ❌ Lancer sans déclarer axes → biais sécu automatique
- ❌ Retirer une voix occidentale pour ajouter une chinoise → panel de base immuable
- ❌ Panel 100% occidental sur un axe activé → monoculture = REJET

---

## 2. Pipeline — 5 phases gated

### Phase 0 — Inventaire déterministe + SAST baseline (zéro coût IA)

Inventaire obligatoire :
- `rg -i 'api[_-]?key|secret|password|token'`
- Endpoints FastAPI (introspection routes)
- Outils MCP (manifest .mcp.json)
- DB : schémas SQLite, migrations, FTS
- Docker/Caddy/Tailscale configs
- SBOM (`uv pip freeze`)

SAST automatisé : `ruff --select=ALL`, `mypy --strict`, `pytest --collect-only`, `bandit`, `semgrep`, `pip-audit`, `trivy`, `detect-secrets`.

Output : `inventory.json` + `sast_findings.jsonl`.

### Phase 1 — Audit scope + threat model (bloquant)

**1.a — `audit_scope.yaml` (obligatoire)**

```yaml
audit_scope:
  name: nom_du_run
  date: "YYYY-MM-DD"
  axes_actives: [A_security, B_quality, C_tests, D_performance, E_architecture, F_documentation, G_adversarial]
  axes_exclus: []
  raison_exclusion: ""
  cible: [mcp_meddata, mcp_redapi, ...]
  contexte: "Audit annuel pré-prod"
  voix_par_axe:
    A: [kimi, codex, qwen36]
    B: [kimi, codex, minimax]
    C: [codex, gemini, qwen3coder]
    D: [codex, gemini, mimov2]
    E: [gemini, kimi, glm51, qwen36]
    F: [claude, gemini, glm51]
    G: [kimi, codex, minimax]
```

**1.b — Threat model** (si axe A activé) : assets, trust_boundaries, attackers, non_goals.

**1.c — Models additionnels par axe** : `quality_model.yaml`, `test_model.yaml`, `performance_model.yaml`, `architecture_model.yaml`, `documentation_model.yaml`, `adversarial_model.yaml`.

**1.d — Validation par voix ortho** : tous les models validés par 1 voix ortho (Codex preferred) avant Phase 2.

### Phase 2 — Audit aveugle parallèle ciblé multi-axes

**Règle 1 — assignation axe × surface × voix**

Exemple axe A :

| Surface | Voix |
|---------|------|
| Auth / secrets | Codex + Kimi |
| Entrées API/MCP | Kimi + Codex |
| Données médicales / RGPD | Gemini + Qwen 3.6 |
| SQLite / FTS | Codex + Qwen 3.6 |
| Agents / LLM | Kimi + Codex |
| Docker / deploy | Kimi + Gemini |

**Règle 2 — blind strict** : aucun modèle ne voit la sortie d'un autre avant fin Phase 2.

**Règle 3 — output JSON-Lines** :
```json
{
  "finding_id": "POLYLENS-2026-XXX",
  "severity": "P0",
  "class": "security|privacy|correctness|perf|observability|legal_traceability",
  "evidence": {"file": "...", "line": 42, "trigger": "...", "command_or_test": "..."},
  "blast_radius": "...",
  "fix_strategy": "...",
  "confidence": "high",
  "falsification_attempt": "...",
  "voix_source": ["codex", "qwen36"],
  "cultural_diversity_score": 1.0
}
```

**Règle 4 — interdiction de citer SAST** : finding "Bandit B105 confirmé" = rejet (SAST Shadowing).

**Règle 5 — convergence cross-culturelle obligatoire** :
- P0 : requiert **≥1 voix occidentale ET ≥1 voix chinoise** convergeant sur `(file, line, mécanisme_causal)`
- P1 : requiert **≥2 voix de cultures différentes**
- P2/P3 : ≥2 voix (culture indifférente)

### Phase 3 — Falsification par PoC (cycle court)

Pour chaque finding ≥ P1 :
1. Finder soumet finding + reproducer prétendu
2. Défenseur = **voix de culture opposée** (finder occidental → défenseur chinois, finder chinois → défenseur occidental)
3. Mission défenseur : "défends que ce N'EST PAS un bug, OU prouve par test unitaire que c'EST un bug"
4. Sortie obligatoire : PoC pytest qui échoue avant fix et passe après, OU justification reproductible
5. ≤ 2 échanges max
6. Défense convaincante → finding déclassé/requalifié
7. Défense échouée → finding renforcé

P2/P3 : sampling 1/3 tiré au sort.

### Phase 4 — Consolidation déterministe + méta-audit 2 voix

Sortie = matrice JSON déterministe (pas synthèse rédigée) :

```python
matrix = aggregate_findings_by(
    key=("file", "line", "mécanisme_causal"),
    voices=[claude, codex, gemini, kimi, minimax, mimov2, glm51, qwen36, qwen3coder],
)

# Cultural Diversity Score (CDS) par finding
matrix["cultural_diversity"] = {
    finding_id: {
        "cds": 1.0 if len(set(v.culture for v in finding.votes)) >= 2 else 0.0,
        "occidental_votes": count_votes(finding, culture="occidental"),
        "chinese_votes": count_votes(finding, culture="chinese"),
    }
}
```

Convergence valide = ≥2 voix de cultures différentes citant même `(file, line, flux, condition, mécanisme)`.

Méta-audit : matrice soumise à Qwen + Codex → findings perdus ? sévérité dérivée ? mécanisme causal solide ?

### Phase 5 — Triage humain + correction par défenseur + régression

Triage humain bloquant P0/P1 : exposé prod ? données santé ? opposable ? reachable ?

Correction par défenseur (≠ finder, anti biais de confirmation) :
1. Défenseur propose fix
2. Codex/Claude applique patch minimal
3. Test PoC Phase 3 doit passer
4. Re-run SAST → pas de nouvelle régression
5. Finder revalide (≤2 tours)
6. Commit individuel par finding

Régression continue : PoC Phase 3 ajouté à suite tests permanents.

---

## 3. Métriques (13 indicateurs)

### Précision / qualité

| # | Métrique | Cible |
|---|----------|-------|
| 1 | Précision | ≥ 75% (validés / reportés) |
| 2 | Specificity Score | ≥ 0.8 (line_refs + CVE + repro) |
| 3 | Vagueness Index | < 0.4 ("potentiellement", "vérifier") |
| 4 | Severity Hole Index | > 0.20 (#P0 + 0.5×#P1) / total |

### Recall / couverture

| # | Métrique | Cible |
|---|----------|-------|
| 5 | Recall vs SAST | ≥ 20% (sinon SAST suffit) |
| 6 | Regulatory Coverage Mapping | ≥ 90% sur modules santé |

### Opérationnel / coût

| # | Métrique | Cible |
|---|----------|-------|
| 7 | Findings actionnables | > 60% |
| 8 | PoC Rate | ≥ 80% sur P0/P1 |
| 9 | Human Triage Tax | < 1.0 (> 1 = IA crée plus de travail qu'elle n'en élimine) |
| 10 | Fix Failure Rate | < 10% |

### Robustesse

| # | Métrique | Cible |
|---|----------|-------|
| 11 | Reproductibilité | ≥ 70% (re-trouvés sur 2e run) |
| 12 | Location Drift | < 30% |
| 13 | Cultural Diversity Score moyen | ≥ 0.5 (P0 doivent avoir CDS = 1.0) |

---

## 4. Anti-patterns

| # | Anti-pattern | Solution |
|---|-------------|----------|
| 1 | Same-prompt-to-all | Lentilles distinctes |
| 2 | Pas de falsification | Phase 3 PoC obligatoire |
| 3 | Claude juge unique | Méta-audit Qwen + Codex |
| 4 | Pas de baseline déterministe | Phase 0 SAST + inventaire |
| 5 | Findings sans reproducer | JSON-Lines, rejet auto |
| 6 | Skip divergences | Disagreement Preservation Score |
| 7 | Finder fixe son finding | Correction par défenseur |
| 8 | Tout en série | Phase 2 strictement parallèle |
| 9 | Coût uniforme | Surfaces × voix prioritaires |
| 10 | Mémoire amnésique | `lessons_polylens_audits.md` |
| 11 | Threat-model-less convergence | Phase 1 bloquant |
| 12 | Echo Chamber Claude | Matrice = script déterministe |
| 13 | SAST Shadowing | Interdiction de citer SAST |
| 14 | Stateless Arbitrage Illusion | Qwen jamais cross-fichiers seul |
| 15 | _other Tag Inflation | CWE strict en output |
| 16 | Voice Imbalance Bias | Caps max 30 findings/projet |
| 17 | CLI Mode Write Bypass | Codex sans `--dangerously`, Gemini `--approval-mode plan`, sandbox readonly |
| 18 | Bash Orchestrator Fragile | Python asyncio + Semaphore |
| 19 | Mono-axe Bias Non Déclaré | `audit_scope.yaml` obligatoire |
| 20 | Panel monoculture | Run avec 0 voix chinoise sur axe activé = REJET |
| 21 | P0 sans confirmation cross-culturelle | P0 requiert CDS ≥ 0.5 (≥1 occidental + ≥1 chinois) |
| 22 | MiniMax sans GLM sur même axe B/E | MiniMax (agressif) + GLM 5.1 (conservateur) appairés obligatoires |
| 23 | Substitution occidentale par chinoise | Panel de base 4 voix occidentales = immuable |

---

## 5. Tests Honeypot

Avant de faire confiance à "convergence ≥2 voix", valider que ce n'est pas du pattern-matching corrélé.

**Test 1 — Anti-bugs plausibles** : injecter 3 motifs qui RESSEMBLENT à des vulnérabilités mais sont sécurisés (SQL f-string validé Enum amont, `/tmp` avec `tempfile.mkstemp+O_NOFOLLOW`, `password==` dans `pytest.raises`). Si flaggés P0/P1 → pattern-matching visuel.

**Test 2 — Bugs métier subtils** : 5 vrais bugs P0/P1 invisibles SAST (race WAL isolation, log NIR, retention HDS 20 ans).

**Test 3 — Perturbation sémantique** : renommer variables sans changer comportement. Convergence qui disparaît = corrélée motifs d'entraînement.

Diagnostic :

| C_anti | C_real | Interprétation |
|--------|--------|----------------|
| > 25% | ≈ C_anti | Pattern-matching, reset prompts |
| < 10% | > 60% | Convergence fiable ✅ |
| < 10% | < 30% | Modèles ratent vrais bugs |
| > 25% | < 30% | Catastrophe, stop panel |

---

## 6. Combinatoires

| Cas | Voix | Coût relatif |
|-----|------|-------------|
| Module isolé < 15 fichiers | Codex + Qwen 3.6 + qwen3-coder | ~25% panel complet |
| Architecture inter-projets | Gemini + Claude + Codex + GLM 5.1 | ~60% |
| Surface adversariale MCP/LLM | Kimi + Codex + Qwen 3.6 + MiniMax | ~50% |
| Audit infra (Docker/Caddy/Tailscale) | Kimi + Claude + MiniMax | ~35% |
| Migration deps majeure | Gemini + Claude + Codex + qwen3-coder | ~60% |
| PR review < 200 lignes | Solo Qwen 3.6 | ~10% |
| Audit efficience patterns | MiniMax + qwen3-coder + MiMo | ~$0.05 |
| Audit annuel complet (7 axes) | Panel complet 7-8 voix | 100% |

---

## 7. Mémoire collective

Après chaque run, alimenter `lessons_polylens_audits.md` :
- Patterns gagnants (quel modèle a trouvé quel type de bug)
- Faux positifs récurrents (par modèle, par surface)
- Prompts productifs vs stériles
- Métriques observées vs cibles
- Résultats honeypot

À long terme : embeddings → RAG pour priming des prompts Phase 1 future.

---

## 7.b. Enseignements empiriques durables (Round 10.8 POLYBUILD, 2026-05-03)

### 7.b.1. Invariant "panel base 4 voix occidentales IMMUABLE" validé à chaud

POLYLENS appliqué à POLYBUILD post-sprint Round 10.8. Audit 4 voix
parallèles (Codex GPT-5.5 + Kimi K2.6 + Qwen 3.6 max + Grok 4.20)
sur commit `2b3c12f`. Verdict consolidé : MAJOR_ISSUES, 11 findings
actionnables fixés. Repo "validé" à 4 voix.

Puis Gemini 3.1 Pro Preview rejoint en 5e voix dès retour quota
(1h57 d'attente). **Gemini a trouvé 3 P0/P1 critiques que les 4
autres voix ont MANQUÉS** :

- **GEMINI-01 P0** : Phase 7 'lost deletions' patch supprimait TOUT
  le repo à chaque commit incrémental (data-loss invisible en
  `--no-commit`).
- **GEMINI-02 P0** : Phase 8 SSRF guard contournable via décimal/
  hex/octal IP encodings (AWS IMDS exfiltration trivial).
- **GEMINI-03 P1** : RGPD leak — un fix précédent avait raté une
  condition OR `or v.startswith("qwen")` dans `filter_candidates`
  qui re-laissait passer les voix chinoises OR remote sous
  `excludes_us_cn_models=True`.

**Cause racine du gap** : ces findings exigent un raisonnement
graphe trans-fichiers (Phase 7 deletion logic vs adapter contract,
SSRF logic vs ipaddress lib, OR override vs helper fix antérieur).
Codex+Kimi+Qwen+Grok regardent fichier par fichier ; Gemini
long-contexte voit l'ensemble. Talent irréductible.

**Conclusion durable** : **PAS de POLYLENS final déclaré "terminé"
sans Gemini**. La substitution est interdite (anti-pattern 23) ; en
plus l'absence temporaire de Gemini doit être traitée comme bloquant
majeur, pas comme contrainte mineure.

### 7.b.2. Pattern "partial-audit-then-Gemini-when-back"

Quand Gemini est en quota épuisé (forfait Pro reset chaque 3h en
moyenne), pratique validée et désormais canonique :

1. **Phase A** — Audit POLYLENS à 4 voix immédiatement (sans Gemini).
   Verdict provisoire = "audit partiel, Gemini pending". NE PAS
   déclarer POLYLENS terminé. Application des fixes convergents
   ≥2 voix.
2. **Phase B** — Polling Gemini chaque 3-5 min via Monitor.
   Pendant l'attente : préparer prompt Gemini-spécifique (focus
   "angles morts des 4 autres voix" + verification des fixes
   Phase A + détection régressions silencieuses).
3. **Phase C** — Gemini revient → audit Gemini-only sur le commit
   POST-fixes-Phase-A. Si Gemini trouve nouveaux findings :
   3a. Application fixes Phase C
   3b. POLYLENS déclaré terminé après push
   3c. Si Gemini ne trouve rien de neuf → validation finale.

**Anti-pattern ABSOLU (anti-pattern 24, ajout)** : déclarer "audit
fini" à 4 voix parce que Gemini est down. C'est précisément les
findings que Gemini trouverait qui sont les plus dangereux (graphe
trans-fichiers = data-loss, SSRF, regression silencieuse).

**Coût attente** : 1-2h max wall-clock.
**Coût d'omission Gemini** : P0 catastrophe en prod.
Ratio coût/bénéfice tranché.

### 7.b.3. TODO méthodologique skill polylens

Ajouter dans `~/.claude/skills/polylens/SKILL.md` la procédure
"polling Gemini quota" + le pattern Phase A → B → C ci-dessus pour
qu'il soit déclenché automatiquement quand Gemini est down au
moment d'un POLYLENS.

---

## 8. Référence rapide

**POLYLENS** = 7 axes déclarés (A-G) + 4 voix base occidentales + 5 voix chinoises ajoutées + 5 phases gated + convergence cross-culturelle obligatoire + PoC obligatoire P0/P1 + matrice déterministe + méta-audit Qwen+Codex

- **Panel de base** : Claude Opus 4.7 + Codex GPT-5.5 + Gemini 3.1 Pro + Kimi K2.6 (toujours présents)
- **Voix chinoises ajoutées** : GLM 5.1 + MiniMax M2.7 + MiMo V2.5-Pro + Qwen 3.6 Plus + qwen3-coder-plus (selon axes)
- **23 anti-patterns** · **13 métriques** · **7 combinatoires**
- **P0 requiert** : ≥1 voix occidentale + ≥1 voix chinoise convergeant sur même `(file, line, mécanisme_causal)`
