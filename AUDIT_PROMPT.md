# AUDIT POLYBUILD v3.2.5 — RED-TEAM RUN #5, GRAND CONTEXTE

Tu auditeras la version 3.2.5. Quatre voix (Grok 4.3 web, Qwen 3.6 Plus 1M, Perplexity Computer / ChatGPT, DeepSeek Expert) ont déjà fait le run #4 sur v3.2.4 et leurs findings convergents sont ADRESSÉS. Le run #5 a un objectif simple : **trouver ce que les 17+ voix précédentes ont raté ou ce que v3.2.5 a accidentellement régressé**. Pas de checklist, pas de redondance.

## Contexte rapide — qui a déjà audité ce code

| Run | Voix | Findings | État |
|---|---|---|---|
| #1 (8 mai, v3.1.x) | 7 | 9 actionnables | tous fixés v3.1.1 |
| #2 (9 mai matin, v3.2.x) | 6 | 9 P0/P1 | fixés v3.2.2 + v3.2.3 |
| #3 (9 mai après-midi, v3.2.3) | 5 | 18+ | fixés v3.2.4 |
| **#4 (9 mai soir, v3.2.4) : 4 voix** | Grok+Qwen+Perplexity+DeepSeek | 4 P1 + 5 P2 + 1 P3 + META | fixés v3.2.5 (que tu audites maintenant) |

## v3.2.5 — fixes appliqués depuis v3.2.4 (NE PAS REDÉCOUVRIR)

- `audit/cache.py` : `cache_stats` + `cache_clear` sous `_CONN_LOCK` ; `os.umask(0o077)` retiré → chmod explicite ; `cache_get_with_metadata` privatisé
- `scoring/shadow_scorer.py` : `diverged` strict (`picked_different` only ; abstain états séparés)
- `models.py` + `orchestrator/solo_pipeline.py` : `VoiceScore.is_solo_stub` + `score=0.0` au lieu de `score=1.0`
- `audit/runner.py` : empty diff au niveau WARNING ; `_call_openrouter` capture tokens en avant pour partial-failure
- `audit/cost_log.py` : `_OPENROUTER_PRICING` synchronisé avec routing.yaml (deepseek-v4-pro, grok-4.20, devstral-2, etc.)
- `security/prompt_sanitizer.py` : raw scan en DEBUG (était WARNING bruyant sur citations légitimes)
- `scoring/__init__.py` : `_load_devcode_scorer` retiré (dead code)
- `audit/_atomic_io.py` : `parent_mode` paramètre ; `safe_write.py` passe 0o755 pour worktree

## Anti-patterns / META déjà flagués mais NON fixés (out-of-scope conscient)

- Strategy Pattern (Grok run #4 P2 META) : duplique 80% du flow consensus/solo — design choice, pas de refactor en 5 sec
- Tests régression `assert "X" in source` (Grok+Codex run #3 P2) : à refactor en property-based, pas immédiat
- 16 profils routing sans test d'intégration (Grok run #3 META) : dette de configuration assumée
- Commentaires "POLYLENS run #N" partout (Perplexity run #4 META) : git/changelog suffirait, mais traçabilité runtime perdue

## Questions précises pour run #5

### Q1 — Yo-yo regressions sur les fixes v3.2.5

Liste les fichiers où v3.2.5 a écrit du code (`git log v3.2.4..HEAD --name-only` ou les commentaires "POLYLENS run #4"). Pour chacun :
- Le fix résout-il vraiment le problème énoncé ?
- Le fix introduit-il un **nouveau bug** que les 4 voix run #4 n'ont pas anticipé ?

Zones nerveuses spécifiques :
- `cache.py` `_CONN_LOCK` → `RLock` (run #3) puis pose en plus `cache_stats`/`cache_clear` (run #5) ; **est-ce que les RLock recursive paths peuvent interagir avec `cache_clear` qui fait DELETE+VACUUM ?**
- `cache.py` retrait `os.umask` : suis-je sûr que sqlite3 crée le fichier .db / .db-wal / .db-shm avant ou après que je puisse chmod ? Y a-t-il une fenêtre TOCTOU ?
- `models.py` `VoiceScore.is_solo_stub: bool = False` ajouté : casse-t-il la rétrocompatibilité Pydantic des JSONL existants ? Les tests qui désérialisent du backlog `extra="forbid"` ?
- `shadow_scorer.py` `diverged` strict : quels logs / dashboards historiques basaient sur `diverged=True` pour les abstain ? Si un consommateur a fait `count("diverged=True")` en run #3 vs run #5, le chiffre va chuter brutalement — est-ce un faux signal de stabilité ?
- `cost_log.py` ajout 5 voix au pricing : les prix sont-ils cohérents avec ce qu'OpenRouter facture vraiment 2026 ? (best-effort, pas crucial)

### Q2 — Ce qui reste objectivement faux dans v3.2.5

Pas de "potentiel issue" sans preuve, pas de "consider" plus de 2 fois. Trouve un truc concret ou dis "rien de critique trouvé".

### Q3 — Incohérences cross-fichier que les 4 voix run #4 n'ont pas vues

Tu as ≥1M tokens. Cherche :
- Une signature qui dérive entre tests et production
- Un schéma Pydantic dont le `schema_version` n'est PAS validé au load
- Un `__all__` qui exporte un symbole supprimé en v3.2.4/v3.2.5
- Un test qui dépend d'un détail d'implémentation supprimé
- Une convention error-handling qui change entre `audit/`, `scoring/`, `security/`, `phases/`
- Une dépendance déclarée mais non importée

### Q4 — META : faut-il continuer les audits ?

Honnêtement, à v3.2.5 et après 17+ voix indépendantes, est-ce qu'on a atteint un point de rendements décroissants ? Si oui, dis-le clairement. Si non, dis ce qui justifie un run #6 et avec quelles voix spécifiquement.

## Format de sortie attendu

Markdown plain.

### FINDINGS [N total — par sévérité]

Pour chaque finding :
**[P0|P1|P2|P3] — Titre court (≤80 chars)**
- **Catégorie** : Q1 yo-yo | Q2 nouveau | Q3 cross-file | Q4 méta
- **Fichier** : `chemin/vers/fichier.py:line`
- **Problème** : 1-3 phrases
- **Preuve** : extrait de code (≤6 lignes)
- **Risque concret** : ce qui casse en prod, condition, qui le voit
- **Fix proposé** : 1 phrase OU patch <10 LOC
- **Pourquoi runs #1-#4 ne l'ont pas vu** : 1 phrase

### NON-FINDINGS — ce qui est fait correctement

3 patterns honnêtement bien faits, calibration utile.

### MÉTA — réponse Q4

Direct. Continue ou stop, et pourquoi.

## Règles dures

- Findings dupliquant les 4 POLYLENS runs précédents = ignorés (cf. liste fixes v3.2.5 ci-dessus + anti-patterns out-of-scope).
- "Potentiel issue" sans preuve = ignoré.
- Mot "consider" >2 fois = répute auteur paresseux.
- Pas de section "next steps" / "general recommendations".
- Si tu trouves zéro chose nouvelle, dis-le clairement — c'est aussi une réponse.

## Commit + tag

```
github.com/reddepot/polybuild_v3 — commit 0a0bd54, tag v3.2.5
```

Tests : 607 passed, 6 skipped, 10 xfailed in 21.84s ; ruff + mypy --strict clean.
