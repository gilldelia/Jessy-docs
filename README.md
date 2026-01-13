# Jessy - Assistant numérique personnalisé avec IA locale

**Jessy** est une application console .NET 10 qui connecte un modèle LLM local (via Ollama) à une personnalité configurable, avec mémoire à court et long terme gérée par Qdrant.

## Fonctionnalités

- **Personnalité configurable** : définis identité, traits, valeurs, style de communication dans `personality.json`.
- **Mémoire contextuelle** :
  - **Court terme** : derniers échanges (max 200 messages, 3 jours) dans `history.json`, fenêtrés à chaque tour.
  - **Long terme** : messages embedés dans Qdrant (collection explicite) avec score de récurrence et éviction.
  - **Mémoire intuitive** : les réponses assistant récurrentes peuvent migrer vers une collection dédiée avant éviction.
- **LLM local** : API OpenAI-compatible (Ollama) avec modèle ajustable (`llama3.3:70b` par défaut).
- **Embeddings** : `nomic-embed-text` pour la recherche vectorielle.
- **RAG** : rappel auto du top-K depuis Qdrant avec scoring hybride (similarité + récurrence).

## Architecture rapide

- `DTOs/` : modèles de config, messages, entrées mémoire.
- `Services/` : appels HTTP LLM (`LlmService`), embeddings (`EmbeddingClient` via `EmbeddingService`), config, stockage conversationnel (`ConversationStorageService` + wrapper `ConversationStorage`).
- `Repositories/` : `QdrantRepository` pour upsert/recherche/éviction.
- `Managers/` : `MemoryBuffer`, `MemoryWindowManager` (fenêtrage CT), `MemoryManager` (flush, migration CT?LT, éviction, récurrence).
- `Builders/` : `SystemPromptBuilder` pour le prompt système.
- `Program.cs` : fil conducteur (chargement config/personnalité, boucle chat, retrieval, sauvegarde, réorg mémoire).

## Prérequis

1. .NET 10 SDK
2. Ollama (avec les modèles `llama3.3:70b` ou plus léger, et `nomic-embed-text`)
3. Qdrant (Docker ou binaire, port 6333)
4. GPU conseillé pour les grands modèles

## Configuration

### `personality.json`
Définis la personnalité (identité, traits, valeurs, style). Un échantillon est créé automatiquement si absent.

### `config.json`
Paramètres techniques (LLM, embeddings, mémoire). Les champs clé :
- `llm.endpoint`, `llm.model`, `llm.historyFile`
- `embeddings.model`, `embeddings.endpoint`, `embeddings.topK`
- `memory.shortTermMaxMessages`, `memory.shortTermMaxAgeDays`, `memory.qdrantEndpoint`, `memory.qdrantCollection`, `memory.qdrantIntuitiveCollection`, `memory.recurrenceWeight`

#### Score de récurrence (`recurrenceWeight`)
- 0.0 : désactive le boost de récurrence.
- 0.1–0.3 : boost léger (recommandé : 0.2).
- 0.5+ : boost fort, peut remonter des souvenirs moins pertinents.

## Utilisation

1. Démarre Ollama et Qdrant.
2. Lance l'application :
   ```sh
   dotnet run
   ```
3. Saisis tes messages. Entrée vide pour quitter. L'historique court terme est fenêtré et sauvegardé, le long terme est mis à jour lors des réorganisations (buffer flush, migration CT?LT, éviction/mémoire intuitive).

## Fonctionnement détaillé

1. Chargement `personality.json` et `config.json`, construction du prompt système.
2. Boucle chat :
   - Fenêtrage court terme (`MemoryWindowManager`).
   - Embedding de l'entrée, recherche Qdrant (explicite + intuitive), ajout des rappels au contexte.
   - Appel LLM, affichage, sauvegarde historique.
   - Bufferisation des nouveaux messages (en vue de la réorg).
3. Réorganisation mémoire (`MemoryManager`) sur inactivité ou sortie :
   - Flush du buffer vers Qdrant.
   - Migration des anciens/excédents CT vers LT.
   - Recalcule des scores d'éviction.
   - Si un souvenir assistant est très récurrent, migration vers la collection intuitive avant suppression LT.

## Tests

- Projet de tests : `Jessy.Tests` (xUnit).
- Exécution :
  ```sh
  dotnet test Jessy.Tests/Jessy.Tests.csproj
  ```
- Couverture actuelle :
  - `MemoryBuffer` (ajout/clear snapshot).
  - `MemoryWindowManager` (purge par âge et par volume, conservation du système).
  - `MemoryManager` (flush buffer, migration CT?LT, éviction vers mémoire intuitive, no-op sous capacité) via fakes Qdrant/Embeddings/Storage.

## Publication de la documentation (CI ? dépôt public)

- Workflow : `.github/workflows/publish-docs.yml` (déclenché sur push `master` ou manuel).
- Script : `scripts/publish-docs.ps1` génère `docs-out/` (copie `README.md`, `docs/`, samples) et pousse vers le dépôt public.
- Secrets à configurer dans le dépôt privé (Actions ? Repository secrets) :
  - `DOCS_REPO_TOKEN` : PAT avec `contents:write` sur le dépôt public de docs.
  - `PUBLIC_DOCS_REPO_URL` : URL https du dépôt public (ex: `https://github.com/<org>/Jessy-docs.git`).
- Branche cible publique par défaut : `main` (modifie `-Branch` dans le workflow/PS1 si besoin).
- Usage local (sans push) :
  ```sh
  pwsh ./scripts/publish-docs.ps1 -OutputPath docs-out -PublicRepoUrl "https://github.com/<org>/Jessy-docs.git" -Branch main
  ```

## Dépannage

- **LLM** : `curl http://localhost:11434/api/tags` (vérifier Ollama et le modèle).
- **Embeddings** : `ollama pull nomic-embed-text`.
- **Qdrant** : `curl http://localhost:6333/collections`.
- **OOM GPU** : réduire `llm.maxTokens` ou prendre un modèle plus léger.

## Licence

Projet personnel. Utilise et modifie librement.

## Contact

Ouvre une issue ou contacte le mainteneur pour toute question.
