# TP Mini-JARVIS — Tests automatisés en TDD

Oct 22, 2026 · @Michel Cadennes

## Présentation

En une journée, vous allez construire **Mini-JARVIS**, un petit assistant en ligne de commande, entièrement en TDD. Chaque comportement commence par un test qui échoue, puis reçoit le code minimal qui le fait passer, puis un refactoring.

Mini-JARVIS reprend en miniature l'architecture du projet JARVIS : un modèle de langage appelé via Groq, des outils exposés par un serveur MCP, et une boucle d'agent qui relie les deux. Les difficultés de test sont les mêmes que dans le vrai projet : un LLM non déterministe, un serveur externe, des appels réseau payants.

### Objectifs pédagogiques

À la fin de la journée, vous saurez :

- écrire des tests unitaires, fonctionnels et end-to-end avec `pytest` ;
- pratiquer le cycle rouge → vert → refactor ;
- remplacer une dépendance coûteuse (le LLM) par un double de test, grâce à l'injection de dépendance ;
- choisir le bon niveau de test pour chaque comportement ;
- écrire des assertions robustes face à un LLM : vérifier les effets, pas la formulation.

### Prérequis

- Python 3.11 ou plus, `uv` (ou `pip` + `venv`), Git.
- Bases de Python : classes, dataclasses, exceptions, `async`/`await`.
- Une clé API Groq, uniquement pour le test E2E optionnel de la partie 3.

### La démarche : TDD en double boucle

Vous travaillerez de l'extérieur vers l'intérieur :

1. **Boucle externe.** Vous écrivez d'abord un test d'acceptation E2E qui décrit le comportement attendu par l'utilisateur. Il échoue et restera rouge une bonne partie de la journée.
2. **Boucle interne.** Vous construisez les briques une par une, en cycles courts rouge → vert → refactor, avec des tests unitaires puis fonctionnels.
3. **Fin de journée.** Quand toutes les briques sont assemblées, le test d'acceptation passe au vert.

### Règles du jeu

- **Aucune ligne de code de production sans un test rouge qui la justifie.**
- Les signatures, noms d'outils, formats et messages d'erreur de cet énoncé sont **normatifs** : respectez-les à la lettre. La suite de tests de référence du formateur s'en servira pour vérifier votre code.
- Chaque exigence porte un identifiant (`T1`, `P2`, `A4`…). Citez-le dans le nom ou la docstring du test correspondant.
- Committez à chaque étape du cycle, avec un préfixe : `red:`, `green:`, `refactor:`.

## Organisation de la journée

| Horaire | Étape | Niveau de test |
| --- | --- | --- |
| 9h00 – 9h45 | Présentation : pytest et TDD | — |
| 9h45 – 10h15 | Étape 0 : installation, premier test d'acceptation (rouge) | End-to-end |
| 10h15 – 12h30 | Partie 1 : outils, parsing, mémoire | Unitaire |
| 13h30 – 15h30 | Partie 2 : serveur MCP, agent | Unitaire + fonctionnel |
| 15h30 – 16h45 | Partie 3 : client Groq, CLI, acceptation au vert | Unitaire + end-to-end |
| 16h45 – 17h30 | Bonus : changement d'exigence, CI ; bilan | Tous |

### Matériel fourni

Le dépôt de départ ne contient **aucun code de production**, seulement :

- `pyproject.toml` avec les dépendances : `groq`, `mcp`, `pytest`, `pytest-asyncio` (mode `auto`), `pytest-httpserver`, `pytest-mock`, `pytest-cov` ;
- le squelette du paquet `src/mini_jarvis/` : des modules vides et `errors.py`, qui définit `ToolError`, `ParsingError` et `AgentError` ;
- `tests/support/fake_groq.py` : une fixture `fake_groq` qui démarre un faux serveur HTTP compatible avec l'API Groq et rejoue une liste de réponses scriptées ;
- `tests/conftest.py`, qui déclare les marqueurs `e2e` et `groq`.

Installation : `uv sync`, puis `uv run pytest`. Le premier lancement doit indiquer « no tests ran ».

## Architecture et contrats d'interface

Mini-JARVIS se compose de sept modules. L'agent ne connaît le LLM et les outils qu'à travers deux interfaces. C'est ce qui permet de les remplacer par des doubles de test.

&#91;embedded content: architecture de Mini-JARVIS · 12 composants\]

Les flèches indiquent « utilise ». En test, `FakeLLM` remplace `GroqClient`, et le faux serveur HTTP remplace l'API Groq.

| Module | Rôle | Testé surtout en |
| --- | --- | --- |
| `errors.py` (fourni) | `ToolError`, `ParsingError`, `AgentError` | — |
| `tools.py` | Fonctions pures : calcul, conversion d'unités, date | Unitaire |
| `parsing.py` | Lecture et validation des appels d'outils du LLM | Unitaire |
| `memory.py` | Historique de conversation, avec troncature | Unitaire |
| `notes.py` + `mcp_server.py` | Stockage des notes, serveur MCP qui expose les outils | Unitaire + fonctionnel |
| `llm.py` | Interface `LLMClient`, implémentation `GroqClient` | Unitaire (mock) |
| `agent.py` | Boucle : message → LLM → outils → LLM → réponse | Fonctionnel |
| `cli.py` | Commande `jarvis ask "..."` | End-to-end |

### Types partagés

```python
# parsing.py
@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    arguments: dict

# llm.py
@dataclass(frozen=True)
class LLMResponse:
    content: str | None
    tool_calls: list[dict]   # format brut Groq/OpenAI, liste vide si aucun

class LLMClient(Protocol):
    def chat(self, messages: list[dict], tools: list[dict] | None = None) -> LLMResponse: ...

# agent.py
class ToolExecutor(Protocol):
    async def list_tools(self) -> list[dict]: ...          # schémas au format Groq
    async def call(self, name: str, arguments: dict) -> str: ...  # lève ToolError
```

### Format des messages

Les messages suivent le format de l'API Groq, compatible OpenAI. Un appel d'outil renvoyé par le LLM ressemble à ceci. Attention : `arguments` est une **chaîne** JSON, pas un objet.

```json
{"id": "call_1", "type": "function",
 "function": {"name": "add_note", "arguments": "{\"text\": \"acheter du lait\"}"}}
```

Le résultat d'un outil est renvoyé au LLM dans un message de rôle `tool` :

```json
{"role": "tool", "tool_call_id": "call_1", "content": "Note n°1 ajoutée"}
```

### Outils exposés par le serveur MCP

| Outil | Arguments | Résultat (texte) |
| --- | --- | --- |
| `add_note` | `text: str` | `Note n°{id} ajoutée` |
| `list_notes` | aucun | `Aucune note.` ou une ligne par note : `{id}. {texte}` |
| `delete_note` | `id: int` | `Note n°{id} supprimée` |
| `calculate` | `a: float`, `op: str`, `b: float` | le résultat, ex. `42.0` |
| `convert` | `value: float`, `from_unit: str`, `to_unit: str` | le résultat, ex. `5.0` |

### Variables d'environnement

| Variable | Rôle | Défaut |
| --- | --- | --- |
| `GROQ_API_KEY` | Clé API Groq | obligatoire |
| `JARVIS_MODEL` | Modèle Groq compatible avec l'appel d'outils | à choisir dans la console Groq |
| `GROQ_BASE_URL` | URL de l'API, remplacée par le faux serveur en test | URL officielle |
| `JARVIS_NOTES_FILE` | Fichier JSON des notes | `notes.json` |

## Étape 0 : le test d'acceptation (30 min)

Avant toute ligne de code, écrivez dans `tests/e2e/test_acceptance.py` le test qui décrit la réussite de la journée. Il doit échouer, et c'est voulu.

**E1 — Ajouter une note en langage naturel.**

- *Étant donné* un faux serveur Groq scripté pour répondre d'abord par un appel à `add_note` avec `{"text": "acheter du lait"}`, puis par le texte `C'est noté.` ;
- *et* `JARVIS_NOTES_FILE` pointant vers un fichier vide d'un dossier temporaire (`tmp_path`) ;
- *quand* on lance `jarvis ask "Note : acheter du lait"` dans un sous-processus ;
- *alors* le code de sortie vaut 0, la sortie standard contient `C'est noté.` et le fichier de notes contient une note dont le texte est `acheter du lait`.
- *Et* le faux serveur a reçu exactement 2 requêtes ; la seconde contient un message `tool` dont le contenu est `Note n°1 ajoutée`.

**E2 — Clé API absente.** Sans `GROQ_API_KEY`, `jarvis ask "Bonjour"` se termine avec le code 2 et écrit sur la sortie d'erreur : `Erreur : la variable GROQ_API_KEY n'est pas définie.`

Squelette de départ :

```python
import json, os, subprocess, sys
import pytest

pytestmark = pytest.mark.e2e

def run_jarvis(*args, env):
    return subprocess.run(
        [sys.executable, "-m", "mini_jarvis.cli", *args],
        capture_output=True, text=True, env=env, timeout=30,
    )

def test_E1_ajouter_une_note(fake_groq, tmp_path):
    fake_groq.script([
        fake_groq.tool_call("add_note", {"text": "acheter du lait"}),
        fake_groq.text("C'est noté."),
    ])
    notes_file = tmp_path / "notes.json"
    env = {**os.environ, "GROQ_API_KEY": "test",
           "GROQ_BASE_URL": fake_groq.url, "JARVIS_NOTES_FILE": str(notes_file)}
    result = run_jarvis("ask", "Note : acheter du lait", env=env)
    ...  # à vous d'écrire les assertions
```

Lancez `uv run pytest -m e2e` : vous devez voir deux échecs. Committez avec `red: tests d'acceptation E1 et E2`.

## Partie 1 : les briques pures, en tests unitaires (2h15)

Ces modules n'ont aucune dépendance externe : c'est le terrain idéal pour des cycles TDD très courts, de 2 à 5 minutes chacun. Traitez les exigences dans l'ordre, une à la fois. Utilisez `@pytest.mark.parametrize` dès que plusieurs exemples illustrent la même règle.

### 1.1 `tools.py`

**T1 — Opérations de base.** `calculate(a: float, op: str, b: float) -> float` applique `+`, `-`, `*` ou `/`.

| Appel | Résultat |
| --- | --- |
| `calculate(2, "+", 3)` | `5.0` |
| `calculate(10, "-", 4)` | `6.0` |
| `calculate(6, "*", 7)` | `42.0` |
| `calculate(7, "/", 2)` | `3.5` |

**T2 — Erreurs de calcul.** `calculate(1, "/", 0)` lève `ToolError("Division par zéro")`. `calculate(1, "%", 2)` lève `ToolError("Opérateur inconnu : %")`.

**T3 — Conversion de longueurs.** `convert(value, from_unit, to_unit) -> float` accepte `km`, `m`, `cm` et `mi` (1 mi = 1 609,344 m). Exemples : `convert(5, "km", "m")` → `5000.0` ; `convert(1, "mi", "km")` → `1.609344`. Comparez les flottants avec `pytest.approx`.

**T4 — Conversion de températures.** Unités `C` et `F`. Exemples : `convert(100, "C", "F")` → `212.0` ; `convert(-40, "F", "C")` → `-40.0`.

**T5 — Erreurs de conversion.** Une unité inconnue lève `ToolError("Unité inconnue : kg")`. Deux unités de familles différentes lèvent `ToolError("Unités incompatibles : km -> C")`. Convertir une unité vers elle-même renvoie la valeur inchangée.

**T6 — Date en français, avec horloge injectée.** `now_text(clock: Callable[[], datetime]) -> str`. Avec une horloge qui renvoie le 22 octobre 2026 à 14h05, le résultat est `Nous sommes le jeudi 22 octobre 2026, il est 14h05.` Le 1er du mois s'écrit `1er`. Les minutes ont toujours deux chiffres. N'utilisez pas `locale` : les noms de jours et de mois sont écrits dans le code.

### 1.2 `parsing.py`

**P1 — Lecture d'un appel d'outil.** `parse_tool_call(raw: dict) -> ToolCall` transforme l'exemple du format des messages en `ToolCall(id="call_1", name="add_note", arguments={"text": "acheter du lait"})`.

**P2 — Arguments vides.** Une chaîne `arguments` vide ou valant `"{}"` donne `arguments={}`.

**P3 — Appels malformés.** Ces cas lèvent tous une `ParsingError` :

- JSON invalide → `Arguments JSON invalides pour add_note` ;
- JSON valide mais qui n'est pas un objet (ex. `"[1, 2]"`) → `Les arguments de add_note doivent être un objet` ;
- clé `function` ou `name` absente → `Appel d'outil sans nom`.

**P4 — Validation par schéma.** `validate_arguments(call: ToolCall, schema: dict) -> None` vérifie les arguments contre le `inputSchema` JSON Schema de l'outil :

- un champ de `required` absent → `ParsingError("Argument manquant : text")` ;
- un type incorrect → `ParsingError("Type invalide pour b : number attendu")` ;
- types à gérer : `string`, `number` (accepte `int` et `float`), `integer`, `boolean`. Piège : en Python, `True` est un `int`. Un booléen n'est donc ni un `number` ni un `integer` valide.

### 1.3 `memory.py`

`Conversation(system_prompt: str, max_tokens: int, count_tokens: Callable[[str], int] = default_count)`. Par défaut, `default_count` compte les mots (`len(text.split())`).

**M1 — Message système en tête.** `messages()` renvoie toujours d'abord `{"role": "system", "content": system_prompt}`, suivi des messages ajoutés par `add(message: dict)`, dans l'ordre.

**M2 — Rôles autorisés.** `add` accepte les rôles `user`, `assistant` et `tool`. Tout autre rôle lève `ValueError("Rôle invalide : system")`.

**M3 — Copie défensive.** Modifier la liste renvoyée par `messages()` ne modifie pas la conversation.

**M4 — Troncature.** Si le total des tokens (système compris) dépasse `max_tokens`, `messages()` retire les messages les plus anciens jusqu'à repasser sous la limite. Un `content` à `None` compte pour 0. Exemple, avec le compteur par défaut, `max_tokens=10` et un système de 2 mots : après trois messages de 4 mots, seuls les deux derniers sont renvoyés.

**M5 — Ce qui n'est jamais retiré.** Le message système et le dernier message ajouté sont toujours conservés, même si la limite est dépassée.

> Point d'étape : `uv run pytest --cov=mini_jarvis` doit afficher au moins 95 % de couverture sur ces trois modules. Les tests d'acceptation sont toujours rouges.

## Partie 2 : serveur MCP et agent, en tests fonctionnels (2h)

Vous assemblez maintenant plusieurs briques. Un test fonctionnel vérifie un scénario complet à travers plusieurs modules réels, mais sans réseau ni LLM. Les vérifications portent sur l'état final (le contenu du stockage, les messages échangés), pas sur des détails internes.

### 2.1 Stockage des notes : `notes.py` (unitaire)

`NoteStore(path: Path)`. Le fichier JSON a la forme `{"next_id": 3, "notes": [{"id": 1, "text": "..."}]}`.

**N1 — Ajout.** `add(text) -> int` retire les espaces en début et en fin, enregistre la note et renvoie son identifiant. Le premier vaut 1. Un fichier absent équivaut à un stockage vide.

**N2 — Persistance.** Une seconde instance de `NoteStore` ouverte sur le même fichier voit les notes ajoutées par la première.

**N3 — Suppression.** `delete(id)` retire la note. Un identifiant inconnu lève `ToolError("Note n°7 introuvable")`. Les identifiants ne sont **jamais réutilisés** : après suppression de la note 2, la note suivante reçoit le numéro 3.

**N4 — Texte vide.** Un texte vide ou ne contenant que des espaces lève `ToolError("La note est vide")`.

Utilisez la fixture `tmp_path` : aucun test ne doit écrire dans le dossier du projet.

### 2.2 Le serveur MCP : `mcp_server.py` (fonctionnel)

Le serveur est construit avec `FastMCP` par une fonction `build_server(store: NoteStore) -> FastMCP`, pour pouvoir lui injecter un stockage de test. `python -m mini_jarvis.mcp_server` le lance en stdio avec le fichier `JARVIS_NOTES_FILE`.

Les tests se connectent au serveur avec un vrai client MCP : soit en mémoire, avec `create_connected_server_and_client_session` du module `mcp.shared.memory`, soit en stdio en lançant le module dans un sous-processus. Écrivez une fixture asynchrone `mcp_session` qui fournit une session connectée.

**S1 — Catalogue.** `list_tools()` renvoie exactement les 5 outils du tableau des contrats. Le schéma de `add_note` déclare `text` obligatoire et de type `string`.

**S2 — Scénario de notes.** Ajouter « acheter du lait », puis « appeler le garagiste », puis supprimer la note 1 : `list_notes` renvoie alors `2. appeler le garagiste`.

**S3 — Erreurs d'outil.** `delete_note` avec `id=7` donne un résultat marqué en erreur (`isError` vrai) dont le texte contient `Note n°7 introuvable`. Le serveur reste utilisable ensuite.

**S4 — Calcul via MCP.** `calculate` avec `{"a": 6, "op": "*", "b": 7}` renvoie le texte `42.0`.

### 2.3 L'adaptateur MCP : `MCPToolExecutor` (unitaire + fonctionnel)

`MCPToolExecutor(session)` implémente `ToolExecutor` au-dessus d'une session MCP.

**X1 — Conversion des schémas.** `mcp_tool_to_groq(tool) -> dict` transforme un outil MCP en `{"type": "function", "function": {"name": ..., "description": ..., "parameters": <inputSchema>}}`.

**X2 — Appel.** `call(name, arguments)` renvoie le texte du résultat. Si le résultat est en erreur, il lève `ToolError` avec ce texte.

### 2.4 L'agent : `agent.py` (fonctionnel)

`Agent(llm: LLMClient, tools: ToolExecutor, conversation: Conversation, max_steps: int = 5)`, avec une méthode `async ask(text: str) -> str`.

C'est ici que vous aurez besoin d'un **double du LLM**. Ne l'écrivez pas à l'avance : laissez le premier test de l'agent vous y obliger. Il s'agit d'un `FakeLLM` qui rejoue des `LLMResponse` scriptées et enregistre les `messages` et `tools` reçus, pour qu'on puisse les vérifier. Pour les outils, combinez ce `FakeLLM` avec un `MCPToolExecutor` branché sur le vrai serveur MCP de la partie 2.2.

**A1 — Réponse directe.** Si le LLM répond sans appel d'outil, `ask` renvoie son `content`. La conversation contient alors le message `user` puis le message `assistant`.

**A2 — Le LLM reçoit les outils.** Chaque appel au LLM reçoit l'historique complet (`conversation.messages()`) et la liste des outils au format Groq.

**A3 — Boucle d'outils.** Si le LLM demande un ou plusieurs outils, l'agent :

1. ajoute le message `assistant` contenant les `tool_calls` ;
2. pour chaque appel, dans l'ordre, l'analyse (P1), valide ses arguments (P4), l'exécute, puis ajoute un message `tool` avec le `tool_call_id` correspondant ;
3. rappelle le LLM.

Scénario de référence : le LLM demande `add_note` puis répond `C'est noté.` → `ask` renvoie `C'est noté.` et le stockage contient la note.

**A4 — Les erreurs ne cassent pas l'agent.** Une `ToolError`, une `ParsingError` ou un outil inconnu donne un message `tool` dont le contenu commence par ` ERREUR :  ` suivi du message d'erreur. Le LLM est ensuite rappelé normalement, pour qu'il puisse se corriger.

**A5 — Garde-fou anti-boucle.** Après `max_steps` appels au LLM sans réponse finale, `ask` renvoie `Désolé, je n'ai pas pu terminer cette demande.`

**A6 — LLM indisponible.** Si `llm.chat` lève une exception, `ask` lève `AgentError("Le service de langage est indisponible.")`, avec l'exception d'origine chaînée (`raise ... from ...`).

## Partie 3 : client Groq, CLI et tests end-to-end (1h15)

Il reste à brancher le vrai LLM et la ligne de commande. À la fin de cette partie, les tests d'acceptation E1 et E2 de l'étape 0 doivent passer au vert.

### 3.1 `GroqClient` (unitaire, avec un mock)

`GroqClient(model: str, client)` : le client du SDK Groq (`groq.Groq`) est **injecté**. En test, remplacez-le par un `Mock` (`mocker.Mock()`), sans aucun appel réseau. Une fonction `GroqClient.from_env()` construit le vrai client à partir des variables d'environnement.

**G1 — Requête.** `chat(messages, tools)` appelle une seule fois `client.chat.completions.create` avec `model`, `messages`, `tools` et `temperature=0`. Si `tools` est vide ou `None`, le paramètre `tools` n'est pas transmis.

**G2 — Réponse.** La réponse du SDK est convertie en `LLMResponse`. Le `content` est repris tel quel. Les `tool_calls` sont convertis en dictionnaires au format brut (voir le format des messages), ou en liste vide s'il n'y en a pas.

**G3 — Configuration.** `from_env()` lève `AgentError("la variable GROQ_API_KEY n'est pas définie.")` si la clé manque. Elle utilise `GROQ_BASE_URL` si la variable est définie.

### 3.2 La CLI : `cli.py`

`python -m mini_jarvis.cli ask "<texte>"` (alias `jarvis ask`) assemble tous les éléments : `GroqClient.from_env()`, la session MCP en stdio sur `python -m mini_jarvis.mcp_server` (en lui transmettant l'environnement), puis `MCPToolExecutor`, `Conversation` et `Agent`.

**C1 — Succès.** La réponse de l'agent s'affiche sur la sortie standard, suivie d'un retour à la ligne. Code de sortie : 0.

**C2 — Configuration invalide.** La CLI affiche ` Erreur :  ` suivi du message de G3 sur la sortie d'erreur. Code de sortie : 2.

**C3 — LLM indisponible.** Une `AgentError` affiche ` Erreur :  ` suivi de son message sur la sortie d'erreur. Code de sortie : 1. Aucune trace d'exception n'est visible.

**C4 — Usage.** Sans argument, ou avec une sous-commande inconnue, la CLI affiche l'aide d'`argparse` et sort avec le code 2.

C1 et C2 sont couverts par les tests d'acceptation. Pour C3, ajoutez un test E2E dont le faux serveur Groq renvoie une erreur HTTP 500.

> Le test d'acceptation E1 doit maintenant être vert. Prenez le temps de relancer **toute** la suite : `uv run pytest`.

### 3.3 Le vrai Groq (E2E optionnel)

**E3 — Avec un vrai modèle.** Ce test est marqué `@pytest.mark.groq` et sauté automatiquement si `GROQ_API_KEY` n'est pas définie (`pytest.mark.skipif`). Il lance `jarvis ask "Ajoute une note : rappeler le garagiste"` contre l'API réelle.

La réponse d'un LLM n'est pas reproductible : **n'assertionnez jamais sur le texte affiché**. Vérifiez les effets : le code de sortie vaut 0, et le fichier de notes contient une note dont le texte contient `garagiste`, sans tenir compte de la casse.

Lancez-le explicitement : `uv run pytest -m groq`. Configurez `pytest` pour que la commande par défaut l'exclue (`addopts = "-m 'not groq'"`).

À discuter en groupe :

- Pourquoi ce test peut-il échouer alors que votre code est correct ?
- Que coûte-t-il, en temps, en argent et en quota d'API ?
- Combien de tests de ce type faut-il dans le vrai projet JARVIS, et quand les lancer ?

## Bonus, livrables et évaluation

### Bonus 1 : le client change d'avis (45 min)

Le formateur annonce cette nouvelle exigence vers 16h45. Traitez-la en TDD, en commençant par modifier ou ajouter les tests. Toute la suite doit rester verte.

**B1 — Échéances.** `add_note` accepte un argument optionnel `due` (date ISO `AAAA-MM-JJ`). Une date invalide lève `ToolError("Date invalide : 2026-13-45")`. `list_notes` affiche alors `{id}. {texte} (pour le {due})`.

**B2 — Notes en retard.** `list_notes` accepte `overdue_only: bool = False`. Avec `True`, seules les notes dont l'échéance est passée sont listées. La date du jour vient d'une horloge **injectée** dans `NoteStore`, comme pour T6.

Observez combien de tests existants cassent, et pourquoi. Des tests qui vérifient des comportements plutôt que des détails d'implémentation cassent moins.

### Bonus 2 : intégration continue

Ajoutez un workflow GitHub Actions qui lance `uv run pytest --cov=mini_jarvis --cov-fail-under=90` à chaque push. Ajoutez un second job, déclenché à la main (`workflow_dispatch`), qui lance `pytest -m groq` avec la clé stockée dans les secrets du dépôt.

### Livrables

- Le dépôt Git, avec un historique qui montre les cycles `red:` / `green:` / `refactor:`.
- Une suite de tests verte, hors marqueur `groq`, avec au moins 90 % de couverture.
- Un fichier `REPONSES.md` qui répond aux trois questions de la partie 3.3, et à celle-ci : « Pour chaque module du vrai JARVIS, à quel niveau de test le couvririez-vous en priorité, et pourquoi ? »

### Critères d'évaluation

| Critère | Ce qui est attendu | Points |
| --- | --- | --- |
| Démarche TDD | Historique de commits cohérent, tests écrits avant le code | 5 |
| Conformité aux contrats | La suite de référence du formateur passe sur votre code | 5 |
| Qualité des tests | Noms explicites, tests indépendants, `parametrize` et fixtures bien utilisés, pas d'assertion sur le texte du LLM | 5 |
| Choix des niveaux | Chaque exigence testée au bon niveau, doubles de test justifiés | 3 |
| Réflexion (`REPONSES.md`) | Arguments précis, liés au projet JARVIS | 2 |
| **Total** |  | **20** |
