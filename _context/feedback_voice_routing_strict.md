---
name: Routing voix CLI vs OpenRouter — règle stricte
description: Pour les voix occidentales (Codex GPT-5.5, Gemini 3.1 Pro, Kimi K2.6), TOUJOURS utiliser le CLI local — jamais OpenRouter. OR uniquement pour les voix CN sans CLI.
type: feedback
originSessionId: d5c45d2b-8111-4b5f-a75a-e53bd2114732
---
# Routing voix — règle stricte (user 2026-05-08)

**Règle** : pour TOUT appel multi-voix (POLYLENS, /avis, audit), router selon ce tableau :

| Voix | Provider | Source à utiliser | $ marginal |
|------|----------|-------------------|------------|
| Claude Opus/Sonnet/Haiku | Anthropic | Session courante OU `claude` CLI | $0 (Max) |
| Codex GPT-5.5 / GPT-5.5-Pro | OpenAI | `codex` CLI (`-m gpt-5.5`) | $0 (ChatGPT Pro) |
| Gemini 3.1 Pro / Flash | Google | `gemini` CLI (`-m gemini-3.1-pro-preview`) | $0 (Pro illimité depuis 02/05) |
| Kimi K2.6 | Moonshot | `kimi` CLI (`--quiet --yolo --thinking`) | **$0 (Allegro maintenant — upgrade depuis Allegretto, plus de capacité)** |
| GLM 5.1 / 4.6 | Zhipu | OpenRouter `z-ai/glm-5.1` (préférer 5.1) | pay-per-token |
| Qwen 3.6 Plus / qwen3-coder | Alibaba | OpenRouter `qwen/qwen3-max` ou `qwen/qwen3-coder` | pay-per-token |
| MiniMax M2.7 | MiniMax | OpenRouter `minimax/minimax-m2.7` | pay-per-token |
| MiMo V2.5-Pro | Xiaomi | OpenRouter `xiaomi/mimo-v2.5-pro` | pay-per-token |
| DeepSeek V4-Pro | DeepSeek | OpenRouter `deepseek/deepseek-v4-pro` | pay-per-token |
| Grok 4.20 | xAI | OpenRouter `x-ai/grok-4.20` | pay-per-token |

**Why** : les abonnements Western (Claude Max, ChatGPT Pro, Gemini Pro, Kimi Allegro) sont déjà payés ; passer par OpenRouter pour ces 4 fournisseurs = double-paiement inutile (~$0.50-2/audit). Les voix CN n'ont pas de CLI → OR seul chemin.

**How to apply** :
- Pour POLYBUILD audit subsystem : `default_voice_caller` doit déjà router CLI pour W et OR pour CN. Vérifier que les ID utilisés dans rotation.py correspondent (e.g. `codex-gpt-5.5` → CLI, `qwen/qwen3-max` → OR).
- Pour scripts d'audit ad-hoc (POLYLENS, /avis) : invoquer codex/gemini/kimi via subprocess CLI, pas via openrouter_call.py.
- Pour OpenRouter (uniquement CN) : continuer d'utiliser `~/.claude/projects/-Users-radu/scripts/openrouter_call.py`.

**Anti-pattern** : utiliser openrouter_call.py pour Codex/Gemini/Kimi par "automatisme" (erreur observée 2026-05-08 fin de session POLYLENS run #2 : 3 voix × ~30K tokens × pricing OR ≈ $0.30 inutile).

**Updates 2026-05-08** :
- Kimi : Allegretto → **Allegro** (bigger forfait, plus de capacité quotidienne).
- GLM : préférer `z-ai/glm-5.1` (et non plus `4.6`) — version récente.
