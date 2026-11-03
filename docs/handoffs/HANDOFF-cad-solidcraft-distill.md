# HANDOFF — Un modèle local pour la CAO pilotée par SolidCraft

> Rédigé le 2026-11-03 à partir d'une session Claude Code (dépôts meowmute et proto3D).
> Ce document est autonome : l'agent suivant n'a pas besoin de relire cette session.

## 1. Contexte et objectif

Chris modélise des pièces à imprimer (Bambu Lab P1S) avec **SolidCraft**, une app de CAO paramétrique open source en Rust, pilotée par un serveur MCP (`solidcraft-cli mcp`). Le code est sur [tangency-labs/solidcraft](https://github.com/tangency-labs/solidcraft). Un agent doit pouvoir construire une pièce à partir d'une demande en français, par exemple « un tube en S de Ø60, 150 mm de haut, décalé de 120 mm en X et en Y ».

**But** : un modèle **local** qui construit ces pièces de façon fiable, y compris les pièces difficiles (sweep, loft, coque, révolution, sketch entièrement contraint).

**Démarche en deux temps** :
1. **Mesurer d'abord.** La communauté SolidCraft a déjà un banc de test et un classement des modèles locaux (§2). Une partie des échecs observés chez Chris vient probablement d'OpenCode v2, pas du modèle (§3).
2. **Distiller ensuite**, seulement là où les modèles locaux restent faibles. DeepSeek V4.1 Flash joue le professeur ; on enregistre ses sessions réussies, puis on entraîne en LoRA un élève local.

**Pourquoi la distillation est faisable** : SolidCraft vérifie seul un résultat (volume, boîte englobante, paramètres). Le banc Pilot (§2) contrôle chaque tâche automatiquement et enregistre la trace complète. Les sessions réussies se trient donc sans relecture humaine.

## 2. État de l'art (au 2026-11-03)

### Hugging Face : aucun modèle ne pilote SolidCraft
Les modèles de CAO publiés écrivent un script **CadQuery** d'un seul coup, alors qu'il faut piloter l'outil étape par étape. On ne peut pas les réutiliser tels quels :
- [ADSKAILab/Zero-To-CAD-Qwen3-VL-2B](https://huggingface.co/ADSKAILab/Zero-To-CAD-Qwen3-VL-2B) : à partir d'images ; jeu de données [Zero-To-CAD-1m](https://huggingface.co/datasets/ADSKAILab/Zero-To-CAD-1m) (980 000 programmes), utile comme banque de formes ;
- [gudo7208/CAD-Coder](https://huggingface.co/gudo7208/CAD-Coder) : Qwen2.5-7B, SFT puis GRPO avec une récompense sur la distance de Chamfer. C'est la recette à imiter pour un éventuel renforcement ;
- [Mhaquehaque/berkelium-qwen3-32b-cadquery-v1](https://huggingface.co/Mhaquehaque/berkelium-qwen3-32b-cadquery-v1) (licence non commerciale) et [yuvit-batra/qwen2.5-coder-7b-cadquery-gguf](https://huggingface.co/yuvit-batra/qwen2.5-coder-7b-cadquery-gguf).

### Communauté SolidCraft : un classement et un banc de test prêts
- **Issue [tangency-labs/solidcraft#4851](https://github.com/tangency-labs/solidcraft/issues/4851)** : un test standard des modèles locaux via l'assistant intégré à SolidCraft, avec appels d'outils natifs. Sur le test v1 (11 tâches simples) :

  | Modèle | v1 |
  |---|---|
  | Muse Glimmer 30B (UD-Q3_K_XL) | 11/11 |
  | **Qwen3.8 27B (UD-Q4_K_XL)** | **11/11**, recommandé au-delà de 16 Go de VRAM |
  | gemma4:26b (MoE) | 11/11 |
  | gemma4 7,5B | 11/11 |
  | gpt-oss:20b | 10/11 |
  | qwen3-coder:30b | 9/11 |
  | qwen3:8b | 7/11 |
  | qwen2.5-coder:7b | 0/11 (écrit ses appels en texte) |

  **Test v2 (12 tâches plus dures)** : les meilleurs modèles locaux plafonnent à 6 ou 7 sur 12. Aucun ne réussit l'axe de révolution, la coque, la liaison pivot ou le sketch entièrement contraint. **C'est là que la distillation a de la valeur.**
- **Le dépôt [dutchman-cad/pilot](https://github.com/dutchman-cad/pilot)** (Apache-2.0), « Helps small local language models drive SolidCraft », apporte deux choses :
  - un relais `dutchman-pilot model --upstream <url>` qui rattrape les appels écrits en texte et propose des formulaires de pièces ;
  - **`bench/`, un banc de test sans fenêtre avec vérification automatique**, en Python standard. Il compte 9 séries : `v1` (11 tâches), `v2` (12), `v3` (40), `multiturn`, `recovery`, `constraints`, `joints`, `units`, `field`. Chaque tâche fournit son prompt, sa règle de réussite et, au besoin, un design de départ.

  Fonctionnement de `bench/run.py`, constaté dans le code :
  - point d'accès compatible OpenAI, Ollama ou Anthropic (`--endpoint`, `--endpoint-kind`) ;
  - clé fournie par `BENCH_API_KEY`, jamais écrite dans les résultats ;
  - `--reps N`, `--through-pilot PATH` ;
  - **chaque ligne de `results.jsonl` contient `transcript`**, la liste des appels avec outil, arguments et résultat ;
  - `bench/report.py` compare deux séries de résultats.

```sh
python3 bench/run.py --set v2 --model MODEL --endpoint URL \
  --solidcraft "$(command -v solidcraft-cli)" --reps 3 --out results/v2-MODEL
python3 bench/report.py results/v2-A/results.jsonl results/v2-B/results.jsonl
```

### Mesures maison du 2026-11-02 (OpenCode v2.0.20, en mode code)
Même prompt, même `AGENTS.md` (celui de `~/Documents/GitHub/my-apps/proto3D/AGENTS.md`), SolidCraft sans fenêtre :

| Modèle | Fournisseur | Résultat | Appels / erreurs | Durée | Coût |
|---|---|---|---|---|---|
| `qwen3.8-flash-next-iq3_s` (MoE ~180B) | local, `https://llm.dc-tech.work/v1` | échec | 6 / 2 | 163 s | ~0 |
| `qwen3-8-27b` (Swift-1.5-Qwen3.8-27B-FP8, vLLM TP=2) | local, même URL | échec | 9 / 7 | 222 s | ~0 |
| `deepseek/deepseek-v4.1-flash` | OpenRouter | **réussi** (volume à 0,01 %) | 14 / 2 | 175 s | 0,027 $ |

Prompt (à ajouter comme tâche maison) : « Construis avec SolidCraft un tube creux en S pour une gaine d'air. Entrée en bas, centrée à l'origine, axe vertical. Sortie 150 mm au-dessus, décalée de 120 mm en X et de 120 mm en Y, axe vertical aussi. Diamètre intérieur 60 mm, paroi 2 mm. Tout en paramètres. Vérifie le résultat avec measure (volume et boîte englobante attendus calculés à l'avance), puis sauvegarde dans tube.sc dans ce dossier. Ne me pose pas de question : choisis ce qui manque et annonce-le à la fin. » Volume attendu : environ 99 730 mm³ (anneau de 389,56 mm² sur deux arcs de R 75,57 mm et 97,05° chacun).

## 3. Cause probable des échecs locaux : le « code mode » d'OpenCode v2

OpenCode v2 n'expose pas les outils MCP un par un : le modèle écrit du JavaScript dans un outil `execute`, sous la forme `tools["solidcraft"].batch({...})`. Le flag `codemode: false` existe dans le schéma de config, mais il n'a aucun effet en v2.0.20. Les erreurs observées viennent toutes de ce format :
- appeler une fonction MCP comme une commande moteur ;
- `const tools = …`, qui casse la portée ;
- passer par `tools.search` ;
- conclure qu'une commande n'existe pas au lieu d'appeler `list_commands`.

La communauté obtient 11/11 avec Qwen3.8 27B en **appels d'outils natifs**. **Hypothèse à vérifier en premier (zone A)** : le `qwen3-8-27b` de Chris, servi en vLLM avec `--tool-call-parser=qwen3_coder`, réussit nettement mieux sur le banc Pilot qu'avec OpenCode.

Conséquence pour la distillation : **produire les données au format d'appels d'outils natifs** (`tool_calls` OpenAI), celui de `bench/run.py`, de l'assistant SolidCraft et de Claude Code. Le mode code d'OpenCode reste un cas particulier, à ne traiter qu'en dernier si Chris y tient.

## 4. Design retenu

```
Zone A : banc Pilot + état des lieux (élève de base vs DeepSeek), y compris les tâches maison
        │   l'élève de base suffit-il ? → si oui : brancher, fin
        ▼
Zone B : génération professeur (DeepSeek via bench/run.py, --reps 3, toutes les séries + tâches maison)
        │   ne garder que les transcripts réussis
        ▼
Zone C : jeu SFT au format tool_calls natif, découpé par famille
        ▼
Zone D : LoRA (élève 8B d'abord, 27B en repli)
        ▼
Zone E : réévaluation avec le même banc et les mêmes séries + mise en service (actions Chris)
```

Décisions préremplies, que l'agent tranche seul :
- **Banc** : réutiliser `dutchman-cad/pilot/bench` (clone ou sous-module, version figée par commit) ; ne pas réécrire de banc.
- **Professeur** : `deepseek/deepseek-v4.1-flash` sur OpenRouter (`https://openrouter.ai/api/v1`). Licence MIT, donc ses sorties sont réutilisables pour l'entraînement.
- **Élèves candidats** : d'abord un ~8B (Qwen3.8 8B, ou gemma4 7,5B qui passe 11/11 en v1), entraînable en QLoRA sur une seule RTX 3090 sans arrêter le vLLM. **Repli : Qwen3.8 27B**, si le 8B reste nettement derrière.
- **Format** : appels d'outils natifs. Garder les erreurs suivies d'une correction réussie, retirer les sessions qui finissent sur un échec.
- **Calculs géométriques** : l'`AGENTS.md` et les données doivent pousser à calculer (rayon, angle, volume attendu) dans un outil ou en code, pas « de tête ». C'est le point faible des petits modèles.

## 5. Zones de travail

Chaque zone se fait en une session, avec une PR par zone.

### Zone A — Banc et état des lieux
1. Intégrer `dutchman-cad/pilot` à une version figée, puis valider le banc : `--verify-reference`, `--do-nothing`, `--negatives` sur `v1` et `v2`, plus `python3 -m unittest discover -s bench/tests -v`.
2. Ajouter une série maison `sets/dc-tech/`, au même format que les séries Pilot (`tasks.json`, règles de contrôle, `start/`) : le tube en S du §2, un manchon, un coude, un raccord conique (loft), un boîtier à coque, une plaque à motif de trous. Prompts en français.
3. Mesurer en `--reps 3` sur `v1`, `v2` et `dc-tech` :
   - `qwen3-8-27b` en direct sur le vLLM de Chris ;
   - le même modèle à travers Pilot (`--through-pilot`) ;
   - DeepSeek V4.1 Flash.
4. Taskfile : `task bench MODEL=... SET=... ENDPOINT=...`.
5. **Livrable** : `docs/baseline.md` avec les scores par série et par tâche, et une conclusion sur l'hypothèse du §3.

**Point d'arrêt** : si `qwen3-8-27b` en natif réussit la plupart des tâches de `v2` et `dc-tech`, la suite (zones B à E) devient facultative. Le dire dans le bilan, avec la recommandation de brancher ce modèle en appels natifs, par exemple via l'assistant intégré à SolidCraft ou Claude Code.

### Zone B — Génération des données du professeur
- `bench/run.py` avec DeepSeek en `--reps 3` sur les séries `v2`, `v3`, `multiturn`, `recovery`, `constraints`, `joints` et `dc-tech`. Les transcripts sont dans `results.jsonl`.
- Ajouter des variantes générées : mêmes familles, autres cotes. On peut aussi puiser des formes dans Zero-To-CAD-1m et les reformuler en demandes.
- Plafond de coût `BUDGET_USD` (50 $ par défaut), avec un cumul suivi d'après les réponses OpenRouter.
- Objectif : au moins 500 transcripts réussis, couvrant toutes les familles.

### Zone C — Jeu SFT
- `scripts/to_sft.py` : `results.jsonl` vers le format chat avec `tool_calls` natifs. Il faut inclure le message système (les `instructions` du serveur MCP, que `run.py` place en `role: system`) et les schémas d'outils.
- Dédoublonnage, filtre de longueur, découpage train/validation **par famille**. Garder des familles entières hors entraînement pour mesurer la généralisation.

### Zone D — Entraînement LoRA
- QLoRA 4 bits (Unsloth ou TRL+PEFT), d'abord sur l'élève 8B.
- 8B : une RTX 3090, sans toucher au vLLM. 27B : **action réservée à Chris** (arrêter le vLLM `qwen3-8-vllm` du namespace `gpu-workloads`, qui tient deux GPU en TP=2), ou location d'un GPU de 48 Go ou plus.

### Zone E — Évaluation et service
- Même banc, mêmes séries, comparé à l'élève de base et au professeur, avec un score par famille, y compris les familles gardées hors entraînement.
- **Action réservée à Chris** : servir l'adaptateur (vLLM `--enable-lora` ou image dédiée) via le GitOps `continuous-delivery`.

## 6. Hors périmètre et suites possibles
- Renforcement (GRPO) avec les contrôles du banc comme récompense, sur le modèle de CAD-Coder.
- Proposer en amont la série `dc-tech` et les résultats sur l'issue #4851 ou à Pilot. **Action réservée à Chris** : rien ne se publie sans son accord.

## 7. Vérification
- Zone A : le banc passe ses validations (`--verify-reference`, `--negatives`). Le comparatif du §2 rejoué en format natif donne un résultat explicable.
- Zone C : 5 exemples SFT relus à la main, avec des appels conformes aux schémas d'outils SolidCraft.
- Zone E : l'élève entraîné fait mieux que l'élève de base sur `v2` et `dc-tech`, y compris sur les familles gardées hors entraînement.

## 8. Risques
- **Surapprentissage par famille** : la littérature (BenchCAD) montre qu'un petit modèle fine-tuné généralise mal aux familles nouvelles. Couvrir large compte plus que la quantité de données.
- **Dérive des versions** : noter dans chaque résultat les versions de `solidcraft-cli`, de Pilot (commit) et du modèle. Un changement de commandes SolidCraft peut rendre les données obsolètes.
- **Coût** : environ 0,03 $ par session, plafonné par `BUDGET_USD`.
- **Secrets** : la clé OpenRouter (secret `openrouter`, clé de donnée `apikey`, namespace `hermes`) passe uniquement par `BENCH_API_KEY`. Elle ne doit apparaître ni dans les données, ni dans les logs, ni dans les commits.
- **App de Chris** : ne jamais utiliser `--connect 127.0.0.1:27878`, qui pilote son app ouverte. Le banc démarre son propre SolidCraft sans fenêtre.

## 9. Niveau de livraison
Une PR par zone sur ce dépôt. Pas de merge, ni d'arrêt de service GPU, ni de publication externe, ni de déploiement sans Chris.
