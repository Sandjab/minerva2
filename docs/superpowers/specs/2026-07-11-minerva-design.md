# Minerva — Design v1 : extraction d'entités/relations et taxonomies incrémentales

Date : 2026-07-11 · Statut : validé en brainstorming (remplace start.md, dont il rouvre certaines décisions)

## Finalité

Outil **générique** d'extraction de graphes de connaissances depuis un texte via LLM.
Premier corpus : `in/roman.md` (roman français, ~240 Ko). Usages visés : analyse
littéraire/exploration humaine, apprentissage des techniques d'extraction, et à terme
socle pour du RAG. Interface : bibliothèque Python + CLI. Tout en français (prompts,
types, relations, attributs).

## Critère de succès v1

`minerva extract in/roman.md -o graphe.db` produit une base interrogeable (personnages,
attributs, alias, relations fusionnés proprement), **avec** le cycle complet de
taxonomies incrémentales : les nouveautés entrent en statut `candidate` sans jamais
bloquer l'extraction, puis `minerva review` permet de les approuver / renommer /
fusionner / rejeter en propageant au graphe. L'extraction est **reprenable** après
interruption.

## Décisions structurantes

| Sujet | Décision |
|---|---|
| Source de vérité | **Tout SQLite** : une base par corpus contient graphe + taxonomies + état de reprise. Les YAML ne servent que de seed initial et d'export/partage (approche B du brainstorming). |
| Validation humaine | Révision **en lot après coup** (`minerva review`), jamais bloquante pendant l'extraction. |
| Passes LLM | **Paramétrable** : presets `single` / `two` / `three` selon backend/modèle. |
| Reprise | **Incrémentale** : progression persistée par (passe, chunk) ; relancer reprend ; `--restart` repart de zéro. |
| Relations | **2 niveaux** : type → nom canonique (la feuille de taxonomie EST le nom de la relation, ex. `professionnelle → travaille_pour`). Le 3ᵉ niveau (subtype) de start.md est abandonné : extraction LLM plus fiable, review plus légère. |
| Signatures de relations | Contraintes source/cible **stockées et informatives** : injectées dans les prompts, violations signalées dans `review`, jamais bloquantes. |
| Langue | Tout en français. |
| Dépendances | `pydantic`, `anthropic`, `openai`, `pyyaml`. CLI en `argparse`, config en `tomllib` (stdlib). |

## Structure du package

```
minerva/
  model.py        # Entity, Relation, KnowledgeGraph (Pydantic) + fusion
  taxonomy.py     # modèle des taxonomies + cycle de vie candidate/approved
  store.py        # SQLite : graphe + taxonomies + reprise (écriture au fil de l'eau)
  chunking.py     # découpage aux frontières de paragraphes
  extraction.py   # orchestration des passes, fusion incrémentale
  review.py       # révision en lot (approve/rename/merge/reject) + propagation
  config.py       # minerva.toml (tomllib) + surcharges CLI
  cli.py          # argparse, sous-commandes
  llm/            # backends claude-cli / anthropic / openai + fabrique
  prompts/        # un fichier .md par passe
  seeds/          # taxonomies initiales YAML
```

## Modèle de données (Pydantic)

- `Entity` : `name` (canonique, snake_case sans accents), `type`, `subtype | None`,
  `aliases: list[str]` (occurrences réelles du texte), `attributes: dict[str, list[str]]`
  (multi-valué, dédupliqué à l'insertion, ordre stable).
- `Relation` : `name` (canonique, ex. `travaille_pour`), `type` (ex. `professionnelle`),
  `source` / `target` (noms canoniques d'entités), `attributes: dict[str, list[str]]`.
- `KnowledgeGraph` : entités indexées par nom normalisé + index d'alias, liste de relations.

Le nom canonique est systématiquement normalisé (snake_case, sans accents) ; chaque
occurrence réelle n'est qu'un alias.

### Fusion (déterministe, en code — jamais via LLM)

- Résolution d'une entité : nom normalisé, puis index d'alias.
- Fusion d'entités : union des alias, union des valeurs d'attributs.
- **Raffinement de `autre`** : si le type (ou sous-type) existant est `autre` et que la
  nouvelle extraction propose une valeur spécifique, on raffine ; sinon la première
  valeur spécifique vue est préservée.
- Relations dédupliquées sur `(name, source, target)` ; attributs fusionnés par union.
- Relation citant une entité inconnue : l'entité est créée minimale (`type=autre`,
  taxonomie `candidate`) plutôt que de perdre la relation.

## Persistance SQLite (`store.py`)

Une base = un corpus. Écriture **au fil de l'eau** : chaque chunk traité = une
transaction (fusion + progression). Pas de `save()` global écrasant.

### Taxonomies

```sql
taxonomy(
  id INTEGER PRIMARY KEY,
  kind TEXT NOT NULL,      -- 'entity_type' | 'relation_type' | 'entity_attr' | 'relation_attr'
  parent_id INTEGER REFERENCES taxonomy(id),  -- NULL pour les racines
  name TEXT NOT NULL,      -- snake_case sans accents
  status TEXT NOT NULL,    -- 'seed' | 'candidate' | 'approved' | 'rejected'
  UNIQUE(kind, parent_id, name)
)
relation_signature(
  relation_type_id INTEGER REFERENCES taxonomy(id),
  role TEXT NOT NULL,      -- 'source' | 'target'
  entity_type_id INTEGER REFERENCES taxonomy(id)
)
```

Arbres : entités `type → sous-type` ; relations `type → nom` (2 niveaux). Un attribut
a pour parent le nœud (type ou sous-type) qui le porte : accroché à `personne` il vaut
pour tous ses sous-types, accroché à `personnage_principal` il est spécifique.

### Graphe

Types et noms d'attributs référencés par **FK vers `taxonomy`** — un renommage de
taxonomie est un UPDATE d'une ligne, le graphe suit.

```sql
entities(id PK, name TEXT UNIQUE, type_id → taxonomy, subtype_id → taxonomy NULL)
entity_aliases(entity_id, alias, UNIQUE(entity_id, alias))
entity_attributes(entity_id, attr_id → taxonomy, value TEXT,
                  UNIQUE(entity_id, attr_id, value))
relations(id PK, name_id → taxonomy,          -- feuille : type dérivé via parent
          source_id → entities, target_id → entities,
          UNIQUE(name_id, source_id, target_id))
relation_attributes(relation_id, attr_id → taxonomy, value TEXT,
                    UNIQUE(relation_id, attr_id, value))
```

### Reprise

```sql
runs(id PK, source_path TEXT, source_sha256 TEXT, chunk_size INTEGER,
     passes TEXT, started_at TEXT, finished_at TEXT NULL)
run_progress(run_id, pass_name TEXT, chunk_index INTEGER, status TEXT,
             UNIQUE(run_id, pass_name, chunk_index))
```

Relancer `extract` sur la même source avec la même config (même sha256 + mêmes
paramètres) reprend les chunks non terminés ; `--restart` purge le run et repart.
Une base peut accumuler plusieurs sources (multi-documents) : chaque run est
identifié par sa source.

## Taxonomies : cycle de vie et review

1. Au premier `extract`, les YAML de `minerva/seeds/` initialisent la table
   (`status='seed'`) : types/sous-types d'entités (personne, organisation, lieu,
   animal, objet, autre + sous-types de start.md), types de relations + signatures,
   attributs usuels par type.
2. Pendant l'extraction, tout type/relation/attribut inconnu est inséré en
   `candidate` et utilisé immédiatement.
3. `minerva review graphe.db` présente chaque candidat avec des exemples d'usage
   tirés du graphe (« `metier` : 12 entités, ex. Clémence → employée de banque »).
   Actions et propagation :
   - **approuver** → `status='approved'` ;
   - **renommer** → UPDATE du nœud, le graphe suit par FK ;
   - **fusionner** vers un nœud existant → les références sont re-pointées puis le
     candidat supprimé ;
   - **rejeter** un type/sous-type → les entités re-rattachées au parent ou à `autre` ;
   - **rejeter** un attribut → **suppression des valeurs** du graphe, après affichage
     du nombre de lignes concernées et confirmation explicite.
   Les violations de `relation_signature` observées dans le graphe sont signalées
   (informatif, jamais bloquant).
4. `minerva taxonomy export -o taxo.yaml` exporte l'arbre (approuvés + seeds) pour
   partage/versionnage ; `minerva taxonomy import taxo.yaml` seed une autre base.

## Extraction (`chunking.py`, `extraction.py`)

1. **Chunking** aux frontières de paragraphes (`\n\n`), taille max configurable
   (défaut 8 000 caractères).
2. **Pipeline de passes** — presets :
   - `single` : 1 passe (entités + attributs + relations) — défaut pour Claude ;
   - `two` : `entites` puis `relations` (la passe 2 reçoit toutes les entités connues) ;
   - `three` : `entites`, `attributs`, `relations` — petits modèles locaux.
   Une passe = nom + fichier prompt + sous-ensemble du schéma de sortie.
3. **Prompts** : un fichier markdown par passe dans `minerva/prompts/`. Instructions
   stables dans le `system` (cacheable par le prompt caching) ; taxonomie courante
   (approuvés + candidats + signatures), entités connues (liste plafonnée) et chunk
   dans le message user.
4. **Fusion incrémentale** en base après chaque chunk (transaction).

### Schéma de sortie LLM

```json
{
  "entities":  [{"name", "type", "subtype", "aliases": [...],
                 "attributes": [{"name", "value"}]}],
  "relations": [{"name", "type", "source", "target",
                 "attributes": [{"name", "value"}]}]
}
```

Attributs transportés en liste de paires (contrainte `additionalProperties: false`
des structured outputs), convertis en dict côté code.

## Backends LLM (`minerva/llm/`)

Protocole : `LLMBackend.extract(system: str, user: str, schema) -> dict`.

- `AnthropicBackend` : `client.messages.parse(...)` avec schéma Pydantic, modèle par
  défaut `claude-opus-4-8`, `thinking={"type": "adaptive"}`, streaming
  (`messages.stream` + `get_final_message()`), `max_tokens=16000`.
- `OpenAIBackend` : `chat.completions.create(...)` avec `response_format` JSON Schema,
  `base_url` configurable (Ollama `http://localhost:11434/v1`, vLLM, OpenRouter).
- `ClaudeCLIBackend` : `subprocess` sur `claude -p --output-format json` ; schéma imposé
  par le prompt, JSON extrait puis validé par Pydantic, une relance en cas de JSON
  invalide.
- Fabrique `make_backend(provider, model, base_url)`.

## Configuration (`config.py`)

`minerva.toml` optionnel à la racine (lu avec `tomllib`) : provider, model, base_url,
chunk_size, passes. Les flags CLI ont priorité sur le fichier.

## CLI (`cli.py`, argparse)

```
minerva extract roman.md -o graphe.db [--provider claude-cli|anthropic|openai]
        [--model M] [--base-url URL] [--chunk-size N]
        [--passes single|two|three] [--restart]
minerva review graphe.db
minerva show graphe.db [--entity NOM]
minerva export graphe.db -o graphe.json
minerva taxonomy export graphe.db -o taxo.yaml
minerva taxonomy import graphe.db taxo.yaml
minerva status graphe.db
```

## Tests (`tests/`, pytest)

Déterministes, sans appel réseau :

- modèle et fusion — dont raffinement de `autre` et union des attributs multi-valués ;
- chunking (frontières de paragraphes, taille max) ;
- aller-retour SQLite complet : graphe, taxonomies, reprise ;
- orchestration multi-passes avec backend factice, y compris reprise après échec
  simulé au milieu d'un run ;
- review : chaque action (approve/rename/merge/reject) et sa propagation — ex. « un
  renommage de taxonomie doit être visible dans les entités existantes », raison
  d'être des FK ;
- seed/export/import YAML.

Les backends réels ne sont pas testés unitairement (fines couches sur les SDKs).

## Hors périmètre v1 (volontairement)

Provenance fine (offsets dans le texte), résolution de coréférence avancée,
visualisation du graphe, parallélisation des appels LLM, RAG (le schéma SQL
requêtable en pose les bases, rien de plus).
