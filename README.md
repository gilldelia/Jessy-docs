## Memory

# Memory Service

Microservice de gestion de la m�moire vectorielle pour J.E.S.S.I.

## Endpoints

### POST /memories/upsert

Ajoute un souvenir en m�moire.

**Body :**
```json
{
  "sense": 3,
  "origin": 0,
  "text": "Le texte � m�moriser"
}
```

**Param�tres :**

| Champ | Type | Description |
|-------|------|-------------|
| `sense` | int | Source sensorielle du souvenir |
| `origin` | int | Origine (interne/externe) |
| `text` | string | Contenu textuel � m�moriser |

**Valeurs `sense` (SourceSense) :**

| Valeur | Nom | Description |
|--------|-----|-------------|
| 0 | Smell | Souvenir li� � une odeur |
| 1 | Taste | Souvenir li� � un go�t |
| 2 | Sight | Souvenir visuel |
| 3 | Hearing | Souvenir auditif (parole, son) |
| 4 | Touch | Souvenir tactile |

**Valeurs `origin` (SourceOrigin) :**

| Valeur | Nom | Description |
|--------|-----|-------------|
| 0 | Internal | Pens�e, r�flexion propre � l'IA |
| 1 | External | Information re�ue de l'ext�rieur |

**R�ponse :** `204 No Content`

---

### POST /memories/search

Recherche des souvenirs similaires � une requ�te.

**Body :**
```json
{
  "query": "Texte de recherche",
  "topK": 10
}
```

**Param�tres :**

| Champ | Type | Description |
|-------|------|-------------|
| `query` | string | Texte de la requ�te |
| `topK` | int | Nombre max de r�sultats (d�faut: config) |

**R�ponse :** `200 OK`
```json
{
  "results": [
    {
      "id": "uuid",
      "role": "ouie-externe",
      "text": "Contenu du souvenir",
      "timestamp": "2025-01-15T12:00:00Z",
      "recurrenceScore": 5,
      "lastRecallTimestamp": "2025-01-15T14:00:00Z",
      "hybridScore": 0.85
    }
  ]
}
```

---

### GET /healthz

V�rifie l'�tat du service.

**R�ponse :** `200 OK` avec `"ok"`

---

## Configuration

Fichier `appsettings.json` :

```json
{
  "Memory": {
    "QdrantEndpoint": "http://localhost:6334",
    "QdrantCollection": "memory",
    "QdrantIntuitiveCollection": "memory_intuitive",
    "EmbeddingsEndpoint": "http://localhost:11434/v1/embeddings",
    "EmbeddingsModel": "nomic-embed-text",
    "TopK": 10,
    "RecurrenceWeight": 0.2,
    "ReorgMinutes": 5,
    "LongTermEvictionAgeDays": 365
  }
}
```

## Fonctionnement interne

- **R�organisation automatique** : Un worker en t�che de fond recalcule p�riodiquement les scores d'�viction.
- **Recherche hybride** : Combine similarit� vectorielle + poids de r�currence.
- **Fusion des m�moires** : La recherche interroge la collection principale et intuitive, fusionne et trie par score hybride.




## Racine

# J.E.S.S.I


**J**oint **E**xecution & **S**upervision **S**ystem for **I**ntelligence

**J.E.S.S.I** est une application console .NET 10 qui connecte un modèle LLM local (via Ollama) à une personnalité configurable, avec mémoire à court et long terme gérée par Qdrant.

## Fonctionnalit�s

- **Personnalit� configurable** : d�finis identit�, traits, valeurs, style de communication dans `personality.json`.
- **M�moire contextuelle** :
  - **Court terme** : derniers �changes (max 200 messages, 3 jours) dans `history.json`, fen�tr�s � chaque tour.
  - **Long terme** : messages embed�s dans Qdrant (collection explicite) avec score de r�currence et �viction.
  - **M�moire intuitive** : les r�ponses assistant r�currentes peuvent migrer vers une collection d�di�e avant �viction.
- **LLM local** : API OpenAI-compatible (Ollama) avec mod�le ajustable (`llama3.3:70b` par d�faut).
- **Embeddings** : `nomic-embed-text` pour la recherche vectorielle.
- **RAG** : rappel auto du top-K depuis Qdrant avec scoring hybride (similarit� + r�currence).

## Architecture rapide

- `DTOs/` : mod�les de config, messages, entr�es m�moire.
- `Services/` : appels HTTP LLM (`LlmService`), embeddings (`EmbeddingClient` via `EmbeddingService`), config, stockage conversationnel (`ConversationStorageService` + wrapper `ConversationStorage`).
- `Repositories/` : `QdrantRepository` pour upsert/recherche/�viction.
- `Managers/` : `MemoryBuffer`, `MemoryWindowManager` (fen�trage CT), `MemoryManager` (flush, migration CT?LT, �viction, r�currence).
- `Builders/` : `SystemPromptBuilder` pour le prompt syst�me.
- `Program.cs` : fil conducteur (chargement config/personnalit�, boucle chat, retrieval, sauvegarde, r�org m�moire).

## Pr�requis

1. .NET 10 SDK
2. Ollama (avec les mod�les `llama3.3:70b` ou plus l�ger, et `nomic-embed-text`)
3. Qdrant (Docker ou binaire, port 6333)
4. GPU conseill� pour les grands mod�les

## Configuration

### `personality.json`
D�finis la personnalit� (identit�, traits, valeurs, style). Un �chantillon est cr�� automatiquement si absent.

### `config.json`
Param�tres techniques (LLM, embeddings, m�moire). Les champs cl� :
- `llm.endpoint`, `llm.model`, `llm.historyFile`
- `embeddings.model`, `embeddings.endpoint`, `embeddings.topK`
- `memory.shortTermMaxMessages`, `memory.shortTermMaxAgeDays`, `memory.qdrantEndpoint`, `memory.qdrantCollection`, `memory.qdrantIntuitiveCollection`, `memory.recurrenceWeight`

#### Score de r�currence (`recurrenceWeight`)
- 0.0 : d�sactive le boost de r�currence.
- 0.1�0.3 : boost l�ger (recommand� : 0.2).
- 0.5+ : boost fort, peut remonter des souvenirs moins pertinents.

## Utilisation

1. D�marre Ollama et Qdrant.
2. Lance l'application :
   ```sh
   dotnet run
   ```
3. Saisis tes messages. Entr�e vide pour quitter. L'historique court terme est fen�tr� et sauvegard�, le long terme est mis � jour lors des r�organisations (buffer flush, migration CT?LT, �viction/m�moire intuitive).

## Fonctionnement d�taill�

1. Chargement `personality.json` et `config.json`, construction du prompt syst�me.
2. Boucle chat :
   - Fen�trage court terme (`MemoryWindowManager`).
   - Embedding de l'entr�e, recherche Qdrant (explicite + intuitive), ajout des rappels au contexte.
   - Appel LLM, affichage, sauvegarde historique.
   - Bufferisation des nouveaux messages (en vue de la r�org).
3. R�organisation m�moire (`MemoryManager`) sur inactivit� ou sortie :
   - Flush du buffer vers Qdrant.
   - Migration des anciens/exc�dents CT vers LT.
   - Recalcule des scores d'�viction.
   - Si un souvenir assistant est tr�s r�current, migration vers la collection intuitive avant suppression LT.

## Tests

- Projet de tests : `Ame.Tests` et `Memory.Tests` (xUnit).
- Ex�cution :
  ```sh
  dotnet test Ame.Tests/Ame.Tests.csproj
  ```
- Couverture actuelle :
  - `MemoryBuffer` (ajout/clear snapshot).
  - `MemoryWindowManager` (purge par �ge et par volume, conservation du syst�me).
  - `MemoryManager` (flush buffer, migration CT?LT, �viction vers m�moire intuitive, no-op sous capacit�) via fakes Qdrant/Embeddings/Storage.

## Publication de la documentation (CI ? d�p�t public)

- Workflow : `.github/workflows/publish-docs.yml` (d�clench� sur push `master` ou manuel).
- Script : `scripts/publish-docs.ps1` g�n�re `docs-out/` (copie `README.md`, `docs/`, samples) et pousse vers le d�p�t public.
- Secrets � configurer dans le d�p�t priv� (Actions ? Repository secrets) :
  - `DOCS_REPO_TOKEN` : PAT avec `contents:write` sur le d�p�t public de docs.
  - `PUBLIC_DOCS_REPO_URL` : URL https du d�p�t public (ex: `https://github.com/<org>/J.E.S.S.I-docs.git`).
- Branche cible publique par d�faut : `main` (modifie `-Branch` dans le workflow/PS1 si besoin).
- Usage local (sans push) :
  ```sh
  pwsh ./scripts/publish-docs.ps1 -OutputPath docs-out -PublicRepoUrl "https://github.com/<org>/J.E.S.S.I-docs.git" -Branch main
  ```

## D�pannage

- **LLM** : `curl http://localhost:11434/api/tags` (v�rifier Ollama et le mod�le).
- **Embeddings** : `ollama pull nomic-embed-text`.
- **Qdrant** : `curl http://localhost:6333/collections`.
- **OOM GPU** : r�duire `llm.maxTokens` ou prendre un mod�le plus l�ger.

## Licence

Projet personnel. Utilise et modifie librement.

## Contact

Ouvre une issue ou contacte le mainteneur pour toute question.




