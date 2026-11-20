# ReelBroker

> Projet réalisé en 36 h au **Datathon PolyFinances 2026** (78 équipes), en équipe de 4 avec Jonathan Dubois, Marc-Aurele Blanc et Laurent Corbeil.
>
> **Ma contribution :** le backend (moteur de calcul en Python et API FastAPI), le pipeline de 4 agents IA sur Amazon Bedrock et le début de l'interface React.
>
> Dépôt d'origine : [JonathanD-Dev/Reelbroker](https://github.com/JonathanD-Dev/Reelbroker). Cette copie conserve l'historique d'origine, ce qui explique que les commits soient au nom de mon coéquipier.

ReelBroker propose des scénarios de portefeuille fictifs adaptés au profil d'un investisseur et explique chaque résultat, sans jamais recommander d'achat. Principe de base : **le code calcule, le modèle explique.**

## Ce que fait la solution

ReelBroker propose des **scénarios de portefeuille fictifs** adaptés à un profil d'investisseur, et explique chaque résultat. Elle ne recommande aucun achat.

1. **Connaître** : un questionnaire adaptatif (5 questions de compte, 5 de portefeuille) produit un *profil déclaré* : objectif, horizon, tolérance (volonté), capacité (calculée par le code), exclusions. Les réponses manquantes, les hypothèses et les contradictions sont listées, jamais comblées en silence.
2. **Décider** : trois scénarios (plus prudent, aligné sur le profil, plus exposé) sont construits par un optimiseur sous contraintes (diversification maximale, plafond de volatilité, plafond par titre). L'utilisateur peut aussi décrire une idée (« 20 % dans Tesla ») : l'application la confronte au profil, nomme les tensions et montre le seuil qui respecterait les limites.
3. **Apprendre** : le profil *révélé* par les décisions prises dans l'application est montré à côté du profil déclaré. C'est la personne qui décide de mettre à jour sa tolérance ; la capacité, l'horizon et l'objectif ne changent jamais à cause de ses décisions.
4. **Agents et gouvernance** (`alphaforge/agents.py`, Strands Agents sur Amazon Bedrock) : un interprète lit la phrase libre, un analyste lance les calculs (outils = fonctions Python), un explicateur rédige à partir de la trace seulement, puis une **gouvernance en code** vérifie que chaque chiffre existe dans la trace et qu'aucune formule de conseil n'apparaît ; sinon, le texte est refait puis remplacé par un texte produit par le code. Sans identifiants AWS, l'application tourne en mode hors ligne (calculs et contrôles identiques, lecture par mots-clés et texte fixe).
5. **Règle d'or** : *le code calcule, le modèle explique*. Aucun chiffre n'est produit par un modèle de langage.

> **Lancer l'application en local, pas à pas (Windows, macOS, Linux) :** voir [`LANCER-EN-LOCAL.md`](LANCER-EN-LOCAL.md).

## Démarrage rapide (Windows)

Prérequis : Python 3.11 (version utilisée : 3.11.9), Node.js 22.22+ ou 24.15+ (`npm ci` affiche un avertissement EBADENGINE sur les versions antérieures, sans blocage), le jeu de données de départ à `data/raw/datathon2026/Datathon2026/alphaforge_starter/` (fichiers fournis par les organisateurs ; `data/raw/` est exclu de Git).

```
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\streamlit run app.py
.venv\Scripts\python -m pytest -q
```

Les dépendances Python sont épinglées dans `requirements.txt` (moteur, Streamlit, Strands sur Bedrock, et `fastapi`, `uvicorn`, `httpx` pour l'API).

Lancer `streamlit run` **depuis la racine du dépôt** : l'application lit ses données et `.env` par chemins relatifs. Tests : 169 tests Python, 336 tests de l'interface (plus 1 échec attendu). Les tests d'application sautent automatiquement si le jeu de départ est absent.

## Lancer la démo

Démo complète : l'interface React (`web/`) appelle l'API FastAPI, qui appelle le moteur. Commandes Windows à lancer depuis la racine du dépôt, sauf indication contraire.

**1. Environnement Python** (une seule fois)

```
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
```

**2. API** (laisser ce terminal ouvert)

```
.venv\Scripts\python scripts\serve_api.py
```

Si le port 8000 est déjà utilisé, lancer `.venv\Scripts\python scripts\serve_api.py --port 8001` (la cible du proxy Vite doit alors être adaptée en conséquence, sinon `/api` ne répond pas).

Vérifier dans le navigateur : http://127.0.0.1:8000/api/health (adapter le port si `--port` est utilisé)

**3. Interface** (second terminal)

```
cd web
npm ci
npm run dev
```

Ouvrir http://localhost:5173. Vite redirige `/api` vers l'API (port 8000). Sans API, l'interface lit `web/public/snapshot.json`. Pour le régénérer, depuis la racine du dépôt (`cd ..` si vous êtes dans `web`) :

```
.venv\Scripts\python scripts\export_snapshot.py
```

**4. Tests**

```
.venv\Scripts\python -m pytest -q
cd web
npx vitest run
npm run build
cd ..
```

**5. AWS (facultatif)**

```
copy .env.example .env
```

Dans `.env`, renseigner `AWS_BEARER_TOKEN_BEDROCK` et `AWS_REGION`, puis relancer l'API. Ne jamais committer `.env`. Sans identifiants, tout fonctionne hors ligne avec des explications déterministes. Détails de vérification : section « Configuration AWS ».

## Architecture

L'interface React (`web/`, Vite et TypeScript) appelle l'API FastAPI (`alphaforge/api.py`, port 8000), qui s'appuie sur le moteur de calcul (`alphaforge/` : données, métriques, optimiseur, questionnaire, décision, apprentissage). Pour la fiche d'un titre, le pipeline à quatre agents (`alphaforge/pipeline/`) enchaîne : **Source Finder** (sources et date d'extraction), **Fact-Checker** (contrôles sur les données), **Data Collector** (série de cours et fenêtre d'analyse) et **Predictor** (trois scénarios hypothétiques favorable, central et défavorable, pas une prévision). Les chiffres viennent toujours du moteur de calcul ; les modèles de langage ne font que rédiger, et la gouvernance en code vérifie le texte. L'application Streamlit (`app.py`, section « Démarrage rapide ») reste disponible.

## Configuration AWS

L'application fonctionne sans AWS (mode hors ligne). Pour activer les agents Bedrock :

1. Compte d'équipe : catalog.workshops.aws/join, code d'accès reçu par courriel, « Open AWS console ». **Le compte expire le lundi 16 novembre en fin de journée et est supprimé avec ses données.**
2. Dans Amazon Bedrock, créer une clé d'API et vérifier que le modèle voulu est activé.
3. Copier `.env.example` vers `.env` et renseigner `AWS_BEARER_TOKEN_BEDROCK`, `AWS_REGION` et, facultativement, `BEDROCK_MODEL_ID`, `BEDROCK_GUARDRAIL_ID`, `BEDROCK_GUARDRAIL_VERSION`.
4. Vérifier : `.venv\Scripts\python scripts\check_aws.py` (un appel minuscule au modèle), puis `.venv\Scripts\python scripts\check_aws.py --agents` (chaîne complète sur un cas d'essai : Robert et Tesla).
5. Relancer l'application : la section « Vérifier une idée » indique « Agents connectés à Amazon Bedrock ».

**Ne jamais publier la clé.** `.env` et `*.key` sont dans `.gitignore`. Le script ne l'affiche pas. Détails et rôle de chaque service : `docs/aws.md`.

## Sources de données

Jeu de départ des organisateurs (`data/raw/datathon2026/Datathon2026/alphaforge_starter/`). Fenêtre : 2021-11-12 au 2026-11-12, extraction le 2026-11-13.

| Source | Contenu | Date d'extraction | Conditions | Limites connues |
|---|---|---|---|---|
| Yahoo Finance via `yfinance` | Prix quotidiens (`adj_close` utilisé), dividendes, fractionnements de 31 actifs | 2026-11-13 | Conditions de Yahoo : https://legal.yahoo.com/us/en/yahoo/terms/otos/index.html | Prix dans la devise de cotation ; aucun alignement de calendrier ; `adj_close` rétro-ajusté (change si on retélécharge) ; lignes crypto = jours UTC ; données non certifiées par la bourse ; CNQ.TO affiche un « fractionnement » 1,021 le 2022-08-22 (probablement une distribution spéciale) |
| Banque du Canada, API Valet (FXUSDCAD) | Taux USD/CAD quotidien | 2026-11-13 | https://www.bankofcanada.ca/terms/ | Aucun taux les jours fériés canadiens (dernier taux reporté) ; taux unique publié vers 16 h 30 HE |
| Banque du Canada, API Valet | Taux directeur, rendements 2 et 10 ans, IPC Canada | 2026-11-13 | https://www.bankofcanada.ca/terms/ | IPC mensuel daté au mois de référence, publié des semaines plus tard (non utilisable avant sa publication) |
| FRED (Réserve fédérale de St. Louis) | Taux des fonds fédéraux, Treasuries 2 et 10 ans, IPC, chômage (États-Unis) | 2026-11-13 | https://fred.stlouisfed.org/legal/ | Valeurs révisées (dernière version), pas telles que publiées |
| Organisateurs (`assets_universe.csv`) | 10 actions canadiennes, 10 américaines, 9 FNB, BTC, ETH | 2026-11-13 | Usage libre dans le datathon | Étiquettes de secteur simplifiées, assignées par les organisateurs |
| **Notre univers étendu** : Yahoo Finance via `yfinance` (`scripts/build_extended_universe.py`, fichiers dans `data/processed/extended/`) | 60 titres de plus : 20 actions canadiennes, 25 américaines, 15 FNB (actions, obligations, immobilier, thématique) ; aucune nouvelle cryptomonnaie. `Adj Close` (`auto_adjust=False`, comme le script des organisateurs), même fenêtre | voir `provenance.json` (`retrieved_at`) | Conditions de Yahoo, comme ci-dessus | **Ce n'est pas une donnée des organisateurs** : c'est notre propre extraction, du même fournisseur. Même biais du survivant (liste choisie à la main en 2026). Seuls les titres à historique complet sont retenus (aucun trou de plus de 7 jours de bourse, aucun prix non positif ; rejets journalisés dans `provenance.json`). Ni cote ESG ni fondamentaux non plus |

Limites qui touchent directement nos résultats :
- **Biais du survivant** : l'univers a été choisi en 2026 parmi des titres aujourd'hui importants ; les chiffres passés sont plus favorables que ce qu'aurait obtenu un choix fait en 2021.
- **Aucune cote ESG, nouvelle ou donnée fondamentale** dans le jeu de départ : le critère « controverses environnementales ou sociales » n'est pas vérifiable avec ces données (l'application l'indique par « donnée insuffisante »).
- **Transformations appliquées** (`alphaforge/data.py`, `load_starter`) : pivot en format large sur `adj_close` ; lignes de fin de semaine des cryptos supprimées (on garde les jours où au moins une bourse est ouverte) ; remplissage par le dernier cours, 3 jours au maximum ; conversion des titres en USD vers le CAD avec le taux de la Banque du Canada (dernier taux reporté). Un rapport de qualité (`alphaforge.quality_report`) est affiché dans l'application.
- **Univers étendu** (`data.load_universe`) : jeu de départ + 60 titres de `data/processed/extended/`, même traitement que `load_starter` (même calendrier, même conversion en CAD avec le taux de la Banque du Canada du jeu de départ). Le jeu de départ reste prioritaire en cas de doublon ; si les fichiers étendus sont absents, l'application retombe silencieusement sur les 31 actifs. Le fil compte alors 91 titres (moins pour un profil qui en exclut). Aucun titre étendu n'est jamais déclaré conforme ESG : étiquettes `esg_unverified` (sociétés) et `esg_unscreened_fund` (fonds, y compris ICLN, thématique énergies propres, non filtré ESG). Régénérer : `.venv\Scripts\python scripts\build_extended_universe.py` (réseau requis ; les valeurs changent légèrement car `Adj Close` est rétro-ajusté).
- Aucun appel réseau n'est nécessaire pour reproduire les résultats : les données sont lues localement.

## Carte du dépôt

| Chemin | Rôle |
|---|---|
| `app.py` | Application Streamlit (questionnaire, profil, options, idée à vérifier, profil révélé, détail) |
| `alphaforge/data.py` | Chargement, rapport de qualité, jeu de départ, conversion de devise, classes d'actifs et plafonds par titre |
| `alphaforge/metrics.py` | Rendement, volatilité, baisse maximale, Sharpe, concentration, contributions au risque |
| `alphaforge/profile.py` | Profil investisseur et table des niveaux de risque (hypothèses) |
| `alphaforge/questionnaire.py` | Questionnaire, profil déclaré, traduction en paramètres du moteur, personas |
| `alphaforge/allocate.py` | Optimiseur sous contraintes |
| `alphaforge/decide.py` | Options à comparer, vérification d'une idée, lecture d'une phrase simple |
| `alphaforge/learn.py` | Profil révélé et comparaison avec le profil déclaré |
| `alphaforge/explain.py` | Piste d'explication en quatre questions (quelles données, pourquoi, limites, ce qui changerait la réponse) |
| `alphaforge/agents.py` | Agents Strands sur Bedrock et contrôle de gouvernance |
| `alphaforge/feed.py` | Flux de découverte : chaque titre une fois, classé (verdict, « à éviter », intérêts, likes, risque du titre, diversification), puis au plus 2 titres d'un même secteur d'affilée |
| `scripts/build_risk_scenarios.py` | Recalcule `data/processed/risk_scenarios.json` (baisses réelles du questionnaire) |
| `scripts/check_aws.py` | Vérifie l'accès Bedrock |
| `scripts/try_step2.py` | Essai rapide de l'étape « Décider » sur les trois personas |
| `tests/` | Tests unitaires et parcours d'application sans navigateur |
| `docs/` | `questionnaire.md` (spécification), `aws.md`, `review.md` (revue critique) ; `docs/given/` : documents des ateliers |
| `data/raw/` | Données brutes des organisateurs (jamais modifiées, hors Git) ; `data/processed/` : résultats calculés |
| `.env.example` | Modèle de configuration AWS |

## Hypothèses (à défendre)

Ce sont **nos** hypothèses, pas des normes. Elles sont dans le code sous forme de constantes lisibles.

| Hypothèse | Valeur | Où |
|---|---|---|
| Niveau de risque retenu | le plus bas de : tolérance, capacité, plafond d'horizon, plafond d'objectif | `profile.py` |
| Budget par niveau 1 à 5 (plafond de volatilité / part maximale d'actifs de croissance) | 4 / 7 / 10 / 14 / 18 % ; 20 / 40 / 60 / 85 / 100 % | `profile.py` (`RISK_BUDGET`) |
| Plafond d'horizon | moins de 1 an : 1 ; moins de 3 : 2 ; moins de 5 : 3 ; moins de 10 : 4 ; sinon 5 | `profile.py` |
| Plafond d'objectif | préserver : 2 ; projet : 3 ; revenu et équilibre : 4 ; croissance : 5 | `profile.py` |
| Capacité à prendre du risque | revenu, coussin de sécurité et besoins de retrait donnent chacun un plafond ; le plus bas l'emporte | `questionnaire.py` (`CAP_*`) |
| Tolérance déclarée -> niveau | faible 1, faible à modérée 2, modérée 3, modérée à élevée 4, élevée 5 ; inconnue : 2 | `questionnaire.py` |
| Plafonds par titre | 10 % par société, 5 % par cryptoactif, 35 % pour un fonds | `data.py`, `profile.py` |
| ZST.TO (obligations à très court terme) | traité comme quasi-liquidités | `data.py` |
| Plancher de volatilité visé | 85 % du plafond (atteint aux 5 niveaux avec l'univers de 91 titres) | `allocate.py` |
| Rétrécissement de la covariance, poids minimal | 10 % vers la diagonale ; 2 % | `metrics.py`, `allocate.py` |
| Profil révélé | au moins 3 décisions, fenêtre des 5 dernières, médiane | `learn.py` |
| Exclusions vérifiables | énergies fossiles (secteur Energy) et cryptoactifs seulement | `questionnaire.py` |

Une revue critique détaillée de ces hypothèses, de leur affichage dans l'interface et des écarts connus est dans `docs/review.md`.

## Reproduire les résultats des personas

Les trois personas de l'énoncé (Amira, Mateo, Robert) sont dans `alphaforge/questionnaire.py` (`PERSONAS`). Les réponses absentes de l'énoncé sont inventées pour la démonstration et marquées comme telles (clé `assumed`).

- **Dans l'application** : lancer `streamlit run app.py`, puis cliquer « Charger Amira », « Charger Mateo » ou « Charger Robert » dans la barre latérale. Pour la scène « idée à la mode », écrire « je veux mettre 20 % dans Tesla » sous « Vérifier une idée » avec Robert chargé, ou utiliser « Explorer à la main ».
- **En ligne de commande** : `set PYTHONPATH=. & .venv\Scripts\python scripts\try_step2.py` (Windows cmd ; en PowerShell : `$env:PYTHONPATH="."; .venv\Scripts\python scripts\try_step2.py`) affiche, pour chaque persona, les trois scénarios (volatilité, baisse maximale, rendement historique, part de croissance) et la vérification de TSLA 20 %, NVDA 5 % et BTC-USD 5 %.
- **Chiffres du questionnaire** (baisses de 14,9 % et 19,4 %) : `set PYTHONPATH=. & .venv\Scripts\python scripts\build_risk_scenarios.py` (réécrit `data/processed/risk_scenarios.json`).
- Niveau de risque retenu observé (exécution du 15 novembre 2026 sur l'univers de 91 actifs) : Amira 4 sur 5 (la capacité, plafonnée par le coussin de sécurité supposé, est la limite active), Mateo 3 sur 5, Robert 2 sur 5 (plafond lié à l'objectif de préservation). Avec Robert, « 20 % dans Tesla » donne le verdict « en conflit » (plafond par titre, volatilité, horizon, objectif) ; le seuil calculé qui respecterait les limites est de 7 % (univers de 91 actifs).

## Ce qui n'est pas terminé

- **Bedrock fonctionne en direct depuis le 15 novembre** (région us-east-1, clé de courte durée de 12 h dans `.env`). Les 91 explications du pipeline sont validées par la gouvernance dès la 1re tentative. Les tests restent hors ligne.
- **Flux de découverte** (`alphaforge/feed.py`) : fini, chaque titre une fois, avec une carte de fin. Tailles observées : Amira 90, Mateo 24, Robert 91 titres. Les 91 explications de cartes sont en cache et validées par la gouvernance.
- **Preuve ESG** : le critère « controverses environnementales ou sociales » (persona Mateo) n'a pas de source de données ; l'application l'indique par « donnée insuffisante » : avec la règle stricte de Mateo, les sociétés individuelles sont exclues faute de preuve, et ESGV est préféré (méthodologie citée depuis la page du fonds chez Vanguard, lue le 2026-11-15, voir `docs/esg-evidence.md`). Aucun titre n'est déclaré conforme.
- **Centres d'intérêt positifs** (IA, techno, crypto, persona Amira) : saisis par la question C6 et utilisés pour le classement du fil et un plancher combiné de 15 % quand c'est faisable.
- **Plancher de volatilité** : avec l'univers de 91 titres, la volatilité estimée atteint 85 % du plafond aux cinq niveaux (vérifié le 2026-11-15 sur le profil de Robert : 3,4 %, 6,0 %, 8,5 %, 11,9 % et 15,3 % pour des plafonds de 4, 7, 10, 14 et 18 %). Avec les 31 titres du jeu de départ seuls, elle restait de 0,2 à 0,7 point sous la cible aux niveaux 1 à 3.
- Ratio de Sharpe calculé avec le taux directeur moyen de la Banque du Canada sur la fenêtre (3,19 %) ; chiffres calculés sur la période d'estimation (pas de test hors échantillon) ; frais, impôts, comptes enregistrés et couverture de change non pris en compte.

Voir `docs/review.md` pour la liste complète, avec fichier, ligne et correctif proposé.

## Utilisation responsable

- **Simulation seulement.** Aucune transaction, aucun ordre, aucun lien avec un compte réel.
- **Ceci est une aide à l'analyse, pas un conseil financier personnalisé** et pas une promesse de rendement.
- **Le rendement passé ne garantit pas le rendement futur.** Toutes les mesures décrivent la période du 12 novembre 2021 au 12 novembre 2026 ; une période sans crise majeure sous-estime le risque.
- **Biais du survivant** : l'univers a été choisi en 2026 ; les chiffres passés sont flatteurs.
- Les scénarios sont des illustrations de compromis entre risque et croissance, pas des recommandations. Une tolérance élevée n'annule jamais une capacité faible, un horizon court ou un objectif de préservation.
- Les agents n'ont accès qu'à des fonctions de calcul ; ils ne peuvent pas produire de chiffre eux-mêmes, et la gouvernance en code rejette les chiffres absents de la trace et les formules de conseil.

# ReelBroker

**English summary.** ReelBroker is a simulation-only, self-explaining investment analysis aid built for the PolyFinances Datathon 2026 ("AlphaForge AI" case by National Bank Direct Brokerage and AWS; the Python package keeps the technical name `alphaforge`). It runs a three-step loop: *Know* the investor (adaptive questionnaire -> declared profile, with tolerance, capacity, horizon and objective kept separate), *Decide* (three fictional portfolio scenarios from a constrained optimiser, plus a check of an idea such as "20 % in Tesla" against the profile), and *Learn* (declared versus revealed risk tolerance, never merged silently). Code does all the arithmetic; language models (Amazon Bedrock via Strands Agents) only read free text and write explanations, and a code-based governance check rejects any number or advice-like wording not backed by the calculation trace. Data: the organisers' public starter pack (Yahoo Finance, Bank of Canada, FRED), 2021-11-12 to 2026-11-12. This is not personalised investment advice; past performance does not predict future returns. Status: the Bedrock path runs live (all 91 card explanations pass governance on the first attempt); see "Ce qui n'est pas terminé".

---
