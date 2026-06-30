# Simplon Maghreb — Sprint Final
# Semaine 1 — Séance 2 : De l'appel IA synchrone à la génération asynchrone

## Objectifs pédagogiques
- Faire un appel IA fiable grâce au structured output (un contrat JSON garanti)
- Stocker proprement le résultat de l'IA grâce aux Casts
- Comprendre pourquoi un appel lent ne doit pas rester dans le cycle de la requête
- Déplacer la génération dans un Job pour rendre l'API rapide (réponse 202)

## Objectifs techniques
SDK `laravel/ai`, agent, structured output, `schema()`, Eloquent Casts (`array`, enum), queue, Job, `ShouldQueue`, `dispatch()`, `queue:work`, `QUEUE_CONNECTION=database`, status code 202

## Table des matières
1. Le fil de la séance : ça marche, mais c'est lent
2. Partie 1 — Un appel IA fiable (structured output)
3. Partie 2 — Le problème de la lenteur
4. Partie 3 — La solution : la queue
5. Récapitulatif et prochaines étapes

## Déroulé de la séance (Type Consolidation, 2h30)
- **0–15 min** : Théorie. Le fil de la séance, rappel du structured output (sections 1 à 2)
- **15–75 min** : LAB 3. L'appel IA en synchrone (structured output + Casts)
- **75–80 min** : On observe ensemble : l'API est lente. Pourquoi ? (section 3)
- **80–125 min** : LAB 4. Déplacer la génération dans un Job (le mécanisme de la queue)
- **125–150 min** : LAB 5. Brancher le Job sur l'endpoint réel (réponse 202)
- **150 min** : Récapitulatif et 3 points clés

---

## 1. Le fil de la séance : ça marche, mais c'est lent

Cette séance raconte **une seule histoire**, en trois temps. On ne sépare pas l'IA et la queue dans deux mondes différents — parce que dans ThreadForge, le travail lent qu'on veut mettre dans la queue, **c'est justement l'appel à l'IA**. Les deux ne font qu'un.

Le déroulé logique :
1. On fait d'abord l'appel IA **en synchrone** : l'utilisateur soumet un contenu, l'IA génère un post, on le stocke. Ça marche.
2. On constate un **problème** : la requête a mis 6 à 8 secondes à répondre, parce qu'on a attendu l'IA. Inacceptable pour une API.
3. On applique la **solution** : on déplace ce même appel IA dans un Job. L'API répond tout de suite (`202`), et la génération se fait en arrière-plan.

L'asynchrone n'est donc pas un concept abstrait : c'est la réponse directe à une lenteur que tu auras toi-même ressentie au LAB 3.

## 2. Partie 1 — Un appel IA fiable (structured output)

Vous avez déjà vu l'AI SDK la semaine dernière, donc ici c'est un **rappel consolidé**.

Si tu demandes du JSON à l'IA dans le prompt (« renvoie-moi un hook, des points, un score »), tu joues à la loterie : parfois elle ajoute une phrase avant, oublie un champ, ou met le score en texte. Ton `json_decode` casse.

Le **structured output** impose un **schema** au SDK : la réponse revient garantie, toujours la même forme, les bons types. C'est la différence entre « s'il te plaît, renvoie ça » et « c'est ce format ou rien ».

Les champs en tableau (comme `body_points`) sont stockés en colonnes JSON, et on les manipule comme des tableaux PHP grâce à un **Cast** `array` — sans `json_encode`/`json_decode` manuel.

C'est le LAB 3.

## 3. Partie 2 — Le problème de la lenteur

Après le LAB 3, on observe ensemble dans Postman : la requête de génération a mis plusieurs secondes. Pendant tout ce temps, la requête HTTP est restée **suspendue**, le client a attendu, et si l'IA est lente, ça finit en timeout.

Une API doit répondre **vite** — quelques dizaines de millisecondes — même si le vrai travail prend du temps. Donc on ne fait pas le travail lent pendant la requête. On le met de côté.

## 4. Partie 3 — La solution : la queue

Une **queue** (file d'attente) est une liste de tâches à faire plus tard. Un **Job** est une de ces tâches. Au lieu d'appeler l'IA pendant la requête, on crée un Job « générer le post » qu'on **dépose dans la queue**, et on répond tout de suite au client : *« reçu, c'est en cours »* (code `202 Accepted`). Un processus séparé, le **worker**, prend les Jobs un par un et les exécute en arrière-plan.

*L'image : tu déposes ta pâte au ferran (le four du quartier), tu repars vaquer à tes occupations, et tu reviens chercher ton pain une fois cuit. Tu ne restes pas planté devant le four à attendre.*

C'est les LAB 4 et LAB 5 : d'abord on comprend le mécanisme du Job, puis on le branche sur l'endpoint réel.

## 5. Récapitulatif et prochaines étapes

### 5.1 Les 3 points clés à retenir
1. Le **structured output** garantit la forme de la réponse de l'IA ; les **Casts** la stockent typée.
2. Un appel lent (l'IA) **ne reste jamais** dans la requête : il part dans un **Job**, l'API répond `202`.
3. Sans **worker** lancé (`queue:work`), rien ne se passe : les Jobs restent en attente. C'est le piège n°1.

### 5.2 Prochaine séance
Séance Tests (Pest) — vérifier automatiquement que cette génération marche, sans tout re-cliquer à la main. On testera notamment que le Job est bien dispatché, sans appeler la vraie IA.

---

## 📝 LAB 3 — L'appel IA en synchrone (structured output + Casts)
**Durée estimée : 60 min**

### Objectif
Faire générer un post par l'IA avec un contrat JSON garanti, et stocker le résultat de façon typée. Pour l'instant, **tout se passe dans la requête** (synchrone) — on verra le problème juste après.

### Contexte
C'est le cœur de ThreadForge. On consolide la partie IA d'abord, avant de s'occuper de la vitesse.

### Prérequis
- SDK `laravel/ai` installé et provider `groq` configuré (`config/ai.php`), `GROQ_API_KEY` dans `.env`
- Les models `RawContent` et `GeneratedPost` en place

### Instructions

**Étape 1 : Créer l'agent**
```bash
php artisan make:agent PostGenerator --structured
```

**Étape 2 : Définir les instructions (system prompt)**
Dans `instructions()`, décris le rôle :
> « Tu es un ghostwriter pour la tech community sur X. À partir d'un contenu technique brut, tu produis un post optimisé : un hook accrocheur, des points clés courts, un score de lisibilité technique, des hashtags pertinents, et une justification du respect du ton. »

**Étape 3 : Définir le schema (le contrat JSON)**
```php
public function schema(JsonSchema $schema): array
{
    return [
        'hook_propose'                  => $schema->string()->required(),
        'body_points'                   => $schema->array()->items($schema->string())->required(),
        'technical_readability_score'   => $schema->integer()->required(),
        'suggested_hashtags'            => $schema->array()->items($schema->string())->required(),
        'tone_compliance_justification' => $schema->string()->required(),
    ];
}
```

**Étape 4 : Ajouter les Casts au model `GeneratedPost`**
```php
use App\Enums\StatutPost;

protected function casts(): array
{
    return [
        'statut'             => StatutPost::class,
        'body_points'        => 'array',
        'suggested_hashtags' => 'array',
        'payload_brut'       => 'array',
    ];
}
```
Crée le backed enum `app/Enums/StatutPost.php` :
```php
enum StatutPost: string
{
    case Draft    = 'draft';
    case Archived = 'archived';
    case Posted   = 'posted';
}
```

**Étape 5 : Générer dans le controller (synchrone, pour l'instant)**
Dans un `Api/ContentController`, méthode `repurpose` — on appelle l'IA **directement** :
```php
public function repurpose(StoreContentRequest $request)
{
    $rawContent = auth()->user()->rawContents()->create([
        'blueprint_id' => $request->validated('blueprint_id'),
        'contenu_brut' => $request->validated('contenu_brut'),
        'statut'       => 'en_attente',
    ]);

    $response = (new PostGenerator)->prompt($rawContent->contenu_brut);

    $rawContent->generatedPost()->create([
        'hook_propose'                  => $response['hook_propose'],
        'body_points'                   => $response['body_points'],
        'technical_readability_score'   => $response['technical_readability_score'],
        'suggested_hashtags'            => $response['suggested_hashtags'],
        'tone_compliance_justification' => $response['tone_compliance_justification'],
        'statut'                        => 'draft',
    ]);

    $rawContent->update(['statut' => 'traite']);

    return response()->json(['message' => 'Post généré.', 'raw_content_id' => $rawContent->id]);
}
```

**Étape 6 : Tester dans Postman**
- `POST /api/content/repurpose` avec un contenu → un `generated_post` est créé, avec `body_points` en tableau (grâce au Cast).
- **Observe le temps de réponse** dans Postman : plusieurs secondes. Garde ce chiffre en tête, on en reparle.

### Critères d'évaluation
- Agent `PostGenerator` avec un `schema()` conforme au contrat
- Casts `array` et enum en place sur `GeneratedPost`
- Le `generated_post` est stocké typé (tableaux exploitables, pas de string JSON brute)
- Tu as noté le temps de réponse (il est lent)

### Livrable
Dans `lab3-notes.md` :
- Capture du `generated_post` en base avec `body_points` en tableau
- Le temps de réponse observé dans Postman

### En cas de blocage
- **Erreur d'authentification IA** : le provider par défaut n'est pas `groq` dans `config/ai.php`, ou `GROQ_API_KEY` manque.
- **`body_points` est une string JSON** : le Cast `array` n'est pas déclaré, ou tu fais un `json_encode` manuel.
- **La réponse n'a pas la bonne forme** : revois ton system prompt et le `schema()`.

### Ressources
- SDK `laravel/ai` : https://laravel.com/docs/13.x/ai-sdk
- Eloquent Casting : https://laravel.com/docs/13.x/eloquent-mutators

---

## 📝 LAB 4 — Déplacer la génération dans un Job (le mécanisme de la queue)
**Durée estimée : 45 min**

### Objectif
Comprendre le cycle d'un Job en sortant l'appel IA de la requête : configurer la queue, créer le Job, le dispatcher, lancer le worker.

### Contexte
Au LAB 3, tu as senti la lenteur. Ici on règle le problème : on déplace **exactement le même appel IA** dans un Job qui tourne en arrière-plan.

### Prérequis
- LAB 3 terminé (la génération synchrone marche)

### Instructions

**Étape 1 : Configurer la queue**
Dans `.env` :
```
QUEUE_CONNECTION=database
```
Dans Laravel 13, la table `jobs` est déjà fournie par défaut. Lance simplement les migrations si ce n'est pas déjà fait :
```bash
php artisan migrate
```

**Étape 2 : Créer le Job**
```bash
php artisan make:job GenererPostJob
```
Déplace la logique de génération du LAB 3 **dans** le `handle()` :
```php
<?php
namespace App\Jobs;

use App\Models\RawContent;
use App\Ai\Agents\PostGenerator;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class GenererPostJob implements ShouldQueue
{
    use Queueable;

    public function __construct(public RawContent $rawContent) {}

    public function handle(): void
    {
        $response = (new PostGenerator)->prompt($this->rawContent->contenu_brut);

        $this->rawContent->generatedPost()->create([
            'hook_propose'                  => $response['hook_propose'],
            'body_points'                   => $response['body_points'],
            'technical_readability_score'   => $response['technical_readability_score'],
            'suggested_hashtags'            => $response['suggested_hashtags'],
            'tone_compliance_justification' => $response['tone_compliance_justification'],
            'statut'                        => 'draft',
        ]);

        $this->rawContent->update(['statut' => 'traite']);
    }
}
```

**Étape 3 : Observer le cycle**
Avant de brancher l'endpoint, teste le dispatch en Tinker :
```php
$rc = \App\Models\RawContent::first();
\App\Jobs\GenererPostJob::dispatch($rc);
```
Regarde la table `jobs` : une ligne est apparue, mais **rien ne se passe** — le statut reste `en_attente`. Le Job attend un worker.

**Étape 4 : Lancer le worker**
```bash
php artisan queue:work
```
Le worker prend le Job, fait la génération, et le `generated_post` apparaît. Le statut passe à `traite`.

### Critères d'évaluation
- `QUEUE_CONNECTION=database` et table `jobs` présente
- `GenererPostJob` contient la logique de génération dans `handle()`
- Sans worker : le Job reste en attente ; avec worker : le post est créé

### Livrable
Dans `lab4-notes.md` :
- Capture de la table `jobs` avec un Job en attente (avant worker)
- Capture du `generated_post` créé après passage du worker

### En cas de blocage
- **Rien ne se passe avec le worker** : `QUEUE_CONNECTION` n'est pas sur `database`, ou tu n'as pas relancé après avoir changé le `.env` (`php artisan config:clear`).
- **`Class GenererPostJob not found`** : vérifie le namespace `App\Jobs`.
- **Une modif du Job n'est pas prise en compte** : `queue:work` garde le code en mémoire, redémarre-le (ou utilise `queue:listen` en développement).

---

## 📝 LAB 5 — Brancher le Job sur l'endpoint réel (réponse 202)
**Durée estimée : 25 min**

### Objectif
Modifier l'endpoint pour qu'il dispatch le Job au lieu de générer en synchrone, et réponde immédiatement `202 Accepted`.

### Contexte
On boucle l'histoire : l'endpoint qui figeait 6 secondes au LAB 3 va maintenant répondre en quelques millisecondes.

### Prérequis
- LAB 4 terminé (le Job fonctionne avec le worker)

### Instructions

**Étape 1 : Alléger le controller**
Remplace l'appel IA synchrone du LAB 3 par un simple `dispatch` :
```php
public function repurpose(StoreContentRequest $request)
{
    $rawContent = auth()->user()->rawContents()->create([
        'blueprint_id' => $request->validated('blueprint_id'),
        'contenu_brut' => $request->validated('contenu_brut'),
        'statut'       => 'en_attente',
    ]);

    GenererPostJob::dispatch($rawContent);

    return response()->json([
        'message'        => 'Contenu reçu, génération en cours.',
        'raw_content_id' => $rawContent->id,
    ], 202);
}
```

**Étape 2 : Tester le flux complet**
- `POST /api/content/repurpose` → réponse **immédiate** `202` (compare le temps avec celui du LAB 3 !)
- Le worker (`queue:work`) traite en arrière-plan
- Après quelques secondes, le `raw_content` passe en `traite` et le `generated_post` apparaît

### Critères d'évaluation
- L'endpoint répond `202` **immédiatement** (plus de page figée)
- Le contrôleur ne fait plus que dispatcher (plus d'appel IA direct)
- La génération se fait bien en arrière-plan via le worker

### Livrable
Dans `lab5-notes.md` :
- Capture de la réponse `202` avec le temps de réponse rapide
- Une phrase : compare le temps de réponse avant (LAB 3) et après (LAB 5)

### En cas de blocage
- **La réponse n'est pas immédiate** : l'appel IA est resté dans le controller. Vérifie que `repurpose` ne fait plus que `dispatch`.
- **Le post n'est jamais créé** : le worker n'est pas lancé (`queue:work`).
- **Erreur sur `$response['...']` dans le Job** : le schema de l'agent ne renvoie pas la bonne structure (revois le LAB 3).
