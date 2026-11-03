# HANDOFF — Distiller un modèle local pour la CAO pilotée par SolidCraft

> Rédigé le 2026-11-03 à partir d'une session Claude Code (dépôts meowmute et proto3D).
> Ce document est autonome : l'agent suivant n'a pas besoin de relire cette session.

## 1. Contexte et objectif

Chris modélise des pièces à imprimer (Bambu Lab P1S) avec **SolidCraft**, une app de CAO paramétrique pilotée par un serveur MCP (`solidcraft-cli mcp`). Un agent OpenCode doit pouvoir construire une pièce à partir d'une demande en français, par exemple « un tube en S de Ø60, 150 mm de haut, décalé de 120 mm en X et en Y ».

**But** : obtenir un modèle **local** qui fasse ce travail aussi bien que DeepSeek V4.1 Flash, par distillation. DeepSeek joue le professeur : on enregistre ses sessions réussies, puis on fine-tune en LoRA un modèle local, l'élève.

**Pourquoi c'est faisable** : SolidCraft vérifie seul un résultat. `measure` renvoie le volume et la boîte englobante, et `batch`/`execute` acceptent un champ `expect` qui signale tout écart avec les valeurs attendues. Les sessions réussies se trient donc automatiquement, sans relecture humaine.

## 2. Mesures de départ (comparatif du 2026-11-02)

Même prompt, même `AGENTS.md`, SolidCraft sans fenêtre (`--headless`), OpenCode v2.0.20 :

| Modèle | Fournisseur | Résultat | Appels / erreurs | Durée | Coût |
|---|---|---|---|---|---|
| `qwen-next/qwen3.8-flash-next-iq3_s` (MoE ~180B, IQ3_S) | local, `https://llm.dc-tech.work/v1` | échec : affirme à tort qu'aucune commande d'arc n'existe | 6 / 2 | 163 s | ~0 |
| `llm-local/qwen3-8-27b` (Swift-1.5-Qwen3.8-27B-FP8, vLLM TP=2) | local, même URL | même échec, design partiel sauvegardé | 9 / 7 | 222 s | ~0 |
| `openrouter/deepseek/deepseek-v4.1-flash` | OpenRouter | **réussi** : volume conforme à 0,01 %, trajectoire calculée correctement | 14 / 2 | 175 s | 0,027 $ |

Prompt du comparatif, à reprendre tel quel comme premier cas d'évaluation :

```text
Construis avec SolidCraft un tube creux en S pour une gaine d'air. Entrée en bas, centrée à l'origine, axe vertical. Sortie 150 mm au-dessus, décalée de 120 mm en X et de 120 mm en Y, axe vertical aussi. Diamètre intérieur 60 mm, paroi 2 mm. Tout en paramètres. Vérifie le résultat avec measure (volume et boîte englobante attendus calculés à l'avance), puis sauvegarde dans tube.sc dans ce dossier. Ne me pose pas de question : choisis ce qui manque et annonce-le à la fin.
```

Valeurs attendues : anneau de 389,56 mm² (π·(32² − 30²)) balayé sur deux arcs tangents de rayon 75,57 mm et d'angle 97,05° chacun, soit environ 99 730 mm³.

Défauts typiques des modèles locaux, que la distillation doit corriger :
- confondre les **fonctions MCP** (`skill_reference`, `list_commands`…) avec les **commandes moteur** passées dans `execute` ;
- redéclarer `const tools` ou passer par `tools.search` ;
- remplacer un sweep par une extrusion ;
- prendre un diamètre pour un rayon ;
- conclure trop vite qu'une commande n'existe pas, au lieu d'appeler `list_commands`.

## 3. Ce qu'il faut savoir sur la chaîne d'outils

### OpenCode v2 en « code mode »
OpenCode v2 n'expose pas les outils MCP un par un : le modèle écrit du JavaScript dans un outil `execute`. Le flag de config `codemode: false` existe dans le schéma, mais il n'a aucun effet en v2.0.20.

Forme correcte, à respecter **à l'identique** dans les données d'entraînement :

```js
// fonctions : appel direct, avec leurs propres arguments
const ref = await tools["solidcraft"].skill_reference({skill: "solidcraft-modeling"});
const d = await tools["solidcraft"].inspect_design({measure: false, sketches: false});
const doc = await tools["solidcraft"].describe_command({id: "solid.sweep"});
// commandes moteur : uniquement dans execute / batch
return await tools["solidcraft"].batch({verbose: false, commands: [
  {command: "parameters.add", params: {name: "L", expression: "40 mm"}},
  {command: "sketch.create", params: {plane: "XY", name: "Profil"}},
  {command: "sketch.circle.center", params: {center: [0, 0], radius: 5}},
  {command: "sketch.finish", params: {}},
  {command: "solid.extrude", params: {distance: "L", name: "Corps"}}
]});
```

Fonctions disponibles : `skill_reference`, `inspect_design`, `list_commands`, `describe_command`, `execute`, `batch`, `measure`, `body_topology`, `check_parameters`, `set_parameter`, `save`, `export`, `screenshot`, `undo`, `redo`, `new_design`, `open`.

### Guide système de référence
Le fichier `AGENTS.md` du dossier `~/Documents/GitHub/my-apps/proto3D/` sert de prompt système à l'élève comme au professeur. Il contient la forme des appels, la méthode SolidCraft, les recettes (tube et sweep, plan à angle) et les règles d'impression P1S. **Le copier dans chaque dossier de génération** pour que professeur et élève voient le même contexte.

### Lancer une session sans fenêtre
Un dossier par session, initialisé en dépôt git (sinon OpenCode ignore le `opencode.json` local) :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {"openrouter": {"options": {"apiKey": "{env:OR_BENCH_KEY}"}}},
  "mcp": {"solidcraft": {"type": "local", "command": ["solidcraft-cli", "mcp", "--headless", "--allow-path", "<dossier>"], "enabled": true}}
}
```

```bash
cd <dossier> && timeout 900 opencode run -m openrouter/deepseek/deepseek-v4.1-flash "$PROMPT"
```

- Clé OpenRouter : secret Kubernetes `openrouter`, clé de donnée `apikey`, namespace `hermes`. Elle est lue au lancement et passée en variable d'environnement : ne jamais l'écrire sur le disque ni l'afficher.
- Les sessions tournent en parallèle sans problème : chaque `opencode run` démarre son propre SolidCraft sans fenêtre.
- **Ne jamais utiliser `--connect 127.0.0.1:27878`** pour la génération : c'est l'app de Chris, avec son design ouvert.

### Récupérer une session
- La base OpenCode se trouve dans `~/.local/share/opencode/opencode.db`, table `session_v2`, avec les colonnes `id`, `directory`, `time_updated`, `cost`…
- Pour exporter : `cd <dossier> && opencode session export <id> > session.json`. Ça donne `{info, messages}`, chaque message ayant `type` (user/…) et `content[]`, avec des parts `text` ou `tool`.
- Une part `tool` contient `name` (`execute`), `state.input.code` (le JS), `state.content[].text` (la sortie) et `state.metadata.error`.
- `opencode session list` renvoie `[]` dans ces dossiers : passer par la base en lecture seule (`sqlite3 -readonly`).

## 4. Design retenu

```
banque de pièces (prompts + attendus)
        │
        ▼
génération professeur (DeepSeek V4.1 Flash, N essais par pièce, SolidCraft sans fenêtre)
        │
        ▼
vérification automatique (mesure finale vs attendu, fichier .sc présent, erreurs)
        │  ne garder que les succès, au plus 1 ou 2 par pièce
        ▼
jeu SFT (format chat + appels execute, identique à ce que verra l'élève)
        │
        ▼
LoRA sur l'élève (qwen3-8-27b par défaut)
        │
        ▼
évaluation sur des pièces jamais vues : même harnais, comparé à la base et au professeur
```

Décisions préremplies, que l'agent tranche seul :
- **Élève par défaut** : la base de `qwen3-8-27b` (Qwen3.8-27B). Il est dense, sa licence est ouverte, et il est déjà servi en vLLM avec appels d'outils natifs (`--tool-call-parser=qwen3_coder`). Un plus petit modèle (Qwen3.8 ~8B) est l'option de repli si la VRAM d'entraînement manque.
- **Professeur** : `deepseek/deepseek-v4.1-flash` sur OpenRouter. Licence MIT : ses sorties sont réutilisables pour entraîner.
- **Format** : conserver le raisonnement, les appels `execute`, **les erreurs et leurs corrections**, pour que l'élève apprenne à se rattraper. Retirer les sessions qui finissent sur un échec.
- **Langue** : prompts en français, comme les vrais usages.

## 5. Zones de travail

Chaque zone se fait en une session, avec une PR par zone.

### Zone A — Banque de pièces et harnais d'évaluation
- `bench/parts/*.yaml`, une pièce par fichier : `id`, `famille`, `prompt` (FR), `attendu: {volume_mm3, tol, bbox, bodies}`, `split: train|eval`.
- Familles à couvrir, au moins 10 : plaque percée, boîtier creux (shell), support en L, manchon et anneau, tube droit, coude, tube en S (sweep), raccord conique (loft), entretoise et standoff, pièce avec motif circulaire, pièce avec congés et chanfreins, pièce en deux parties emboîtables.
- Volumes attendus calculés de façon analytique, ou vérifiés une fois à la main avec SolidCraft. **Réserver environ 20 % des pièces, dont des familles entières, en `eval`.**
- `scripts/run_session.sh <model> <part.yaml> <outdir>` : crée le dossier, `git init`, `opencode.json`, copie de l'`AGENTS.md`, `opencode run`, export de la session. Avec `--dry-run`, couleurs et préfixes `OK:`/`ERREUR:`.
- `scripts/score.py` : lit l'export et sort `{succès, appels, erreurs, durée, coût, volume_final}`. Succès = dernier `measure` ou `expect` conforme et `.sc` présent.
- Taskfile : `task bench MODEL=... SPLIT=eval`.
- Critère de fin : rejouer le comparatif du §2 reproduit les mêmes conclusions.

### Zone B — Génération des données du professeur
- `task gen N=3 SPLIT=train` : 3 essais par pièce, en parallèle (4 à 6 sessions à la fois), reprise possible après interruption.
- Garder les succès dans `data/raw/<part>/<run>.json`. Journal de coût cumulé, avec arrêt à un plafond (`BUDGET_USD`, 50 $ par défaut).
- Objectif : 500 sessions réussies ou plus, réparties sur toutes les familles.

### Zone C — Construction du jeu SFT
- `scripts/to_sft.py` : export OpenCode vers le format chat avec appels d'outils, au gabarit exact du tokenizer de l'élève. Le prompt système est l'`AGENTS.md` plus le prompt système d'OpenCode, à récupérer dans l'export ou les logs.
- Dédoublonnage, filtre de longueur, découpage train/validation par **famille**, pas par session.

### Zone D — Entraînement LoRA
- QLoRA 4 bits avec Unsloth ou TRL+PEFT.
- **Action réservée à Chris** : libérer les GPU. Le vLLM `qwen3-8-vllm` du namespace `gpu-workloads` occupe deux GPU en TP=2 : l'arrêter est une action de production, donc demander. Sinon, louer un GPU de 48 Go ou plus pour quelques heures.
- Sortie : adaptateur LoRA, et éventuellement un modèle fusionné en FP8 pour vLLM.

### Zone E — Évaluation et service
- `task bench` sur la partie `eval`, avec trois résultats comparés : élève de base, élève LoRA, professeur.
- **Action réservée à Chris** : servir l'adaptateur (vLLM `--enable-lora` ou image dédiée), ce qui passe par le GitOps `continuous-delivery`.

## 6. Hors périmètre et suites possibles
- Renforcement (GRPO) en utilisant la mesure SolidCraft comme récompense, une fois la SFT en place.
- Rendu visuel (`screenshot`) pour juger la forme en plus du volume.

## 7. Vérification
- Zone A : le comparatif du §2 rejoué avec `task bench` donne les mêmes verdicts.
- Zone C : un échantillon de 5 exemples SFT relus à la main ; les appels `execute` ont exactement la forme du §3.
- Zone E : l'élève LoRA fait mieux que l'élève de base sur `eval`, avec un taux de succès rapporté par famille.

## 8. Risques
- **Surapprentissage par famille** : le benchmark BenchCAD montre qu'un petit Qwen fine-tuné généralise mal aux familles nouvelles. La couverture des familles compte plus que le volume de données.
- **Dérive des outils** : une mise à jour de SolidCraft ou d'OpenCode peut changer les noms de commandes ou le format `execute`. Noter les versions dans chaque export (`solidcraft-cli --version`, `opencode --version`).
- **Coût** : environ 0,03 $ par session réussie, plafonné par `BUDGET_USD`.
- **Sécurité** : la clé OpenRouter ne doit apparaître ni dans les données, ni dans les logs, ni dans les commits. Les sessions SolidCraft restent limitées à leur dossier par `--allow-path`.

## 9. Niveau de livraison
Une PR par zone sur ce dépôt. Pas de merge, ni d'arrêt de service GPU, ni de déploiement sans Chris.
