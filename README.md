# RIMES

Une API REST moderne accompagnée d'une simple interface web pour le
dictionnaire des rimes françaises avec plus de 260 000 mots
français.

> Base de données de 122 000+ mots.
> Cette version : Python/FastAPI, SQLite ou PostgreSQL, prête pour
> Docker, plus une interface web HTML autonome.

---

## Fonctionnalités

- **Recherche de rimes réelle** — saisir un mot, obtenir la liste des
  mots qui riment réellement avec lui, tirés du dictionnaire de
  260 000 mots (comparaison des terminaisons, pas de système de motifs
  rigide).
- **« Voir tout »** — pour les groupes de rimes volumineux, un bouton
  charge l'intégralité des résultats au-delà de l'échantillon initial.
- **Vérification par motifs hérités** — la syntaxe de wildcards de
  l'application VB3 d'origine (`*`, `?`, `!`, `$`, `[BD]`, `[!AE]` …)
  reste prise en charge pour qui souhaite définir ses propres motifs.
- **Ajout de mots** — enrichir le dictionnaire en direct.
- **Statistiques** — nombre de mots, nombre de motifs, longueur moyenne.
- **Interface web** (`rimes.html`) — interface simple, utilisable
  localement dans le navigateur, sans étape de build.
- **Déploiement Docker** — PostgreSQL + API + proxy inverse Nginx en un
  seul ensemble.

---

## Démarrage rapide (développement local)

### Prérequis
- Python 3.12+

### 1. Installer les dépendances
```bash
pip install -r backend/requirements.txt
```

### 2. Préparer la base de données
`rime_data.db`, fournie avec le projet, est déjà remplie (260 352 mots,
14 motifs). Pour la reconstruire :
```bash
python backend/migrate_data.py
```

### 3. Démarrer le serveur
```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8080
```

Important : cette commande doit être exécutée **depuis le dossier
racine `RIMES`** (là où se trouve le sous-dossier `backend`), pas depuis
l'intérieur de `backend/`.

### 4. Ouvrir l'interface web
Double-cliquer simplement sur `rimes.html` pour l'ouvrir dans le
navigateur — elle communique automatiquement avec
`http://127.0.0.1:8080` tant que le serveur tourne.

### 5. Tester
```bash
curl http://localhost:8080/
# {"message": "Welcome to RIMES Rhyme API", "version": "1.0.0"}

curl -X POST http://localhost:8080/rime/rhymes \
  -H "Content-Type: application/json" \
  -d '{"word": "MAISON"}'
```

---

## Documentation de l'API

Une fois le serveur démarré :
- Interface Swagger : `http://localhost:8080/docs`
- Schéma OpenAPI : `http://localhost:8080/openapi.json`

### `POST /rime/rhymes` — recherche de rimes réelle

Trouve les mots du dictionnaire qui riment avec le mot saisi, en
comparant sa terminaison (jusqu'à 6 lettres, raccourcie automatiquement
jusqu'à obtenir suffisamment de résultats). Fonctionne pour **n'importe
quel** mot, sans qu'un motif préexistant soit nécessaire.

```bash
curl -X POST http://localhost:8080/rime/rhymes \
  -H "Content-Type: application/json" \
  -d '{"word": "MAISON"}'
```
```json
{
  "word": "MAISON",
  "rhyme_suffix": "AISON",
  "word_count": 53,
  "words": ["CARGAISON", "COMPARAISON", "LIVRAISON", "RAISON", "SAISON", "..."],
  "truncated": false
}
```

Paramètre optionnel `limit` (par défaut 60, plafonné à 5000) pour
récupérer davantage de résultats en un seul appel — c'est ce
qu'utilise le bouton « +X de plus » de l'interface web :
```bash
curl -X POST http://localhost:8080/rime/rhymes \
  -H "Content-Type: application/json" \
  -d '{"word": "MAISON", "limit": 500}'
```

### `POST /rime/check` — vérification par motifs hérités

Vérifie un mot par rapport aux motifs enregistrés dans `rime_patterns`
(syntaxe de wildcards héritée). Pour chaque motif correspondant,
renvoie en plus un échantillon de mots réels du dictionnaire.

```bash
curl -X POST http://localhost:8080/rime/check \
  -H "Content-Type: application/json" \
  -d '{"word": "NATION"}'
```

> Remarque : les motifs fournis par défaut (`tG=F`, `tG=K`, `XKEY=42`
> …) sont de simples étiquettes de catégorie héritées de l'ancien
> exécutable — ce ne sont **pas** de véritables wildcards exploitables,
> et ils ne renvoient donc généralement aucun résultat. Pour des
> résultats utiles via `/check`, il faut créer ses propres motifs
> wildcard dans `rime_patterns` (ex. `*TION`, `*MENT`, `*[!AE]*`). Pour
> une recherche de rimes classique, utiliser `/rime/rhymes` — c'est la
> voie pratique par défaut.

### `POST /rime/add` — ajouter un mot

```bash
curl -X POST http://localhost:8080/rime/add \
  -H "Content-Type: application/json" \
  -d '{"word": "FRANZ"}'
```
Réponses : `200` en cas de succès, `409` si le mot existe déjà,
`400` si le mot est vide.

### `GET /rime/stats` — statistiques

```bash
curl http://localhost:8080/rime/stats
```
```json
{
  "total_words": {"value": 260352, "description": "Total number of words in dictionary"},
  "total_patterns": {"value": 14, "description": "Number of phonetic patterns"},
  "avg_word_length": {"value": 9, "description": "Average word length"}
}
```

### `GET /` et `GET /health`

Message de bienvenue et vérification de l'état du serveur (ce dernier
est également utilisé par `nginx.conf` pour le proxy inverse Docker).

---

## Syntaxe des wildcards (héritée, pour `/rime/check`)

| Caractère  | Signification                       |
|------------|--------------------------------------|
| `*`        | une ou plusieurs lettres             |
| `?`        | exactement une lettre                |
| `!`        | pas une voyelle (ni `AEIOUY`)        |
| `$`        | uniquement `B` ou `D`                |
| `[!AE]`    | exclut `A` et `E`                    |
| `[BD]`     | uniquement `B` ou `D`                |

---

## Exécuter avec Docker

```bash
docker-compose up --build
```

- PostgreSQL, l'API et le proxy inverse Nginx démarrent ensemble
- API accessible via Nginx sur les ports 80/443
- Migration vers PostgreSQL :
  ```bash
  set DATABASE_URL=postgresql://rimeuser:rimepass@localhost:5432/rime_data
  python backend/migrate_postgres.py
  ```

```bash
docker-compose down
```

---

## Développement

### Lancer les tests
```bash
pytest backend/tests/ -v
```

### Avec SQLite en mémoire (tests rapides)
```bash
python run_local.py --mode test
python run_local.py --mode migrate
python run_local.py --mode server
```

---

## Structure du projet

```
RIMES/
├── backend/
│   ├── main.py                  # Application FastAPI, CORS, /health
│   ├── models.py                # Modèles ORM SQLAlchemy
│   ├── database.py              # Moteur DB / session
│   ├── api/
│   │   └── routes.py            # /rime/rhymes, /check, /add, /stats
│   ├── regex_converter.py       # Wildcards héritées → regex
│   ├── migrate_data.py          # Migration SQLite
│   ├── migrate_postgres.py      # Migration PostgreSQL
│   ├── simulate_migration.py    # Simulation en mémoire
│   ├── Dockerfile
│   ├── requirements.txt
│   └── tests/
├── rimes.html                   # Interface web autonome
├── docker-compose.yml
├── nginx.conf
├── init-scripts/01-init-schema.sql
├── extracted_words.txt          # 260 352+ mots français (données source)
├── rime_data.db                 # Base SQLite déjà remplie, prête à l'emploi
└── README.md
```

---

## Limitations connues

- Les entrées `rime_patterns` fournies par défaut (`tG=F`, `XKEY=42` …)
  sont des étiquettes de catégorie héritées sans véritable
  signification wildcard — utiliser `/rime/rhymes` plutôt que `/check`
  pour une recherche de rimes classique.
- `/rime/rhymes` s'appuie sur un cache en mémoire de la liste des mots,
  qui met environ 1 à 2 secondes à se charger lors du premier appel
  après le démarrage du serveur, puis se met à jour automatiquement à
  chaque ajout de mot.
- La détection des rimes repose sur la comparaison des terminaisons
  orthographiques, pas sur une phonétique réelle (API/IPA) — pour une
  liste de mots français, c'est en pratique une bonne approximation,
  mais ce n'est pas un modèle de rime linguistiquement exact.

---

## Licence

Licence MIT

- La licence MIT autorise l'utilisation, la modification et la
  redistribution libres
