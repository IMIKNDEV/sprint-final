# Simplon Maghreb — Sprint Final
# Semaine 1 — Séance 2 : Remise à niveau Jobs & Queues (ne plus figer l'API)

## Objectifs pédagogiques
- Comprendre pourquoi un traitement lent ne doit jamais rester dans le cycle de la requête
- Maîtriser le cycle complet d'un Job : créer → dispatch → worker
- Rendre la génération de ThreadForge asynchrone (réponse immédiate `202 Accepted`)
- Gérer le statut d'un traitement du début à la fin

## Objectifs techniques
Queue, Job, `ShouldQueue`, `dispatch()`, `queue:work` vs `queue:listen`, `QUEUE_CONNECTION=database`, table `jobs`, status code `202`, gestion de statut

## Table des matières
1. Le problème : l'API qui fige 8 secondes
2. La solution : la queue (file d'attente)
3. Le cycle d'un Job
4. Rendre ThreadForge asynchrone
5. Récapitulatif et prochaines étapes

## Déroulé de la séance (Type Remise à niveau, 2h)
- **0–20 min** : Théorie. Le problème de la requête bloquée, la queue (file d'attente), le cycle d'un Job (sections 1 à 3)
- **20–70 min** : LAB 3. Le cycle d'un Job, version simple
- **70–115 min** : LAB 4. Rendre la génération asynchrone
- **115–120 min** : Récapitulatif et 3 points clés

---

## 1. Le problème : l'API qui fige 8 secondes

Imagine un créateur qui soumet un contenu à ThreadForge. Le serveur appelle l'IA pour générer le post. L'IA met 6 à 8 secondes à répondre. Pendant tout ce temps, la requête HTTP **reste suspendue** : la barre de chargement tourne, le navigateur attend, et si l'IA est lente, la requête finit en timeout.

C'est inacceptable pour une API. Une API doit répondre **vite** — en quelques dizaines de millisecondes — même si le vrai travail prend plus de temps. La solution : on ne fait pas le travail lent pendant la requête. On le **met de côté** pour le traiter en arrière-plan.

## 2. La solution : la queue (file d'attente)

Une **queue** (file d'attente) est une liste de tâches à faire plus tard. Un **Job** est une de ces tâches. Au lieu d'appeler l'IA pendant la requête, on crée un Job « générer le post » qu'on **dépose dans la queue**, et on répond immédiatement au client : *« reçu, c'est en cours de traitement »* (code `202 Accepted`).

Un processus séparé, le **worker**, prend les Jobs de la queue un par un et les exécute en arrière-plan. Le client n'attend plus.

*L'image : tu déposes ta pâte au ferran (le four du quartier), tu repars vaquer à tes occupations, et tu reviens chercher ton pain une fois cuit. Tu ne restes pas planté devant le four à attendre.*

## 3. Le cycle d'un Job

Quatre étapes, qu'on va pratiquer au LAB 3 :
1. **Configurer la queue** : `QUEUE_CONNECTION=database` + une table `jobs`.
2. **Créer le Job** : une classe dans `app/Jobs/` avec la logique dans `handle()`.
3. **Dispatcher** : `MonJob::dispatch($data)` dépose le Job dans la queue et rend la main aussitôt.
4. **Lancer le worker** : `php artisan queue:work` exécute les Jobs en attente.

## 4. Rendre ThreadForge asynchrone

Au LAB 4, on applique ça au vrai cas : la génération d'un post. Le créateur soumet son contenu → on crée le `raw_content` en statut `en_attente`, on dispatch le Job, et on répond `202` immédiatement. Le worker fait la génération en arrière-plan et passe le statut à `traite`.

## 5. Récapitulatif et prochaines étapes

### 5.1 Les 3 points clés à retenir
1. Un traitement lent **ne reste jamais** dans la requête : il part dans un Job.
2. L'API répond tout de suite (`202 Accepted`), le worker travaille en arrière-plan.
3. Sans worker lancé (`queue:work`), **rien ne se passe** : les Jobs restent en attente. C'est le piège n°1.

### 5.2 Prochaine séance
Séance 3 : Consolidation de l'AI SDK — le contrat JSON (structured output) que le Job utilise pour générer un post fiable.

---

## 📝 LAB 3 — Le cycle d'un Job (version simple)
**Durée estimée : 45 min**

### Objectif
Comprendre et pratiquer le cycle complet d'un Job sur un cas minimal : configurer la queue, créer un Job qui simule un travail lent, le dispatcher, et lancer le worker.

### Contexte
Avant de brancher l'IA, on isole le mécanisme de la queue sur un Job tout simple, pour bien voir le « avant / après » du traitement asynchrone.

### Prérequis
- Projet ThreadForge avec la table `raw_contents`

### Instructions

**Étape 1 : Configurer la queue**
Dans `.env` :
```
QUEUE_CONNECTION=database
```
Dans Laravel 13, la table `jobs` est déjà fournie par défaut (sa migration est livrée à l'installation). Il suffit donc de lancer les migrations si ce n'est pas déjà fait :
```bash
php artisan migrate
```

**Étape 2 : Créer un Job simple**
```bash
php artisan make:job TraiterRawContentJob
```
Dans `handle()`, simule un travail lent puis change le statut :
```php
<?php
namespace App\Jobs;

use App\Models\RawContent;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class TraiterRawContentJob implements ShouldQueue
{
    use Queueable;

    public function __construct(public RawContent $rawContent) {}

    public function handle(): void
    {
        sleep(5); // simule un traitement lent (l'IA viendra au LAB 4)
        $this->rawContent->update(['statut' => 'traite']);
    }
}
```

**Étape 3 : Dispatcher le Job**
Quelque part où tu crées un `raw_content` (un controller de test ou Tinker), crée-le en `en_attente` puis dispatch :
```php
$rawContent = auth()->user()->rawContents()->create([
    'blueprint_id' => $blueprintId,
    'contenu_brut' => $texte,
    'statut'       => 'en_attente',
]);

TraiterRawContentJob::dispatch($rawContent);
```

**Étape 4 : Observer SANS worker**
Crée un `raw_content`. Regarde en base : son `statut` reste `en_attente`, et une ligne apparaît dans la table `jobs`. **Rien ne se passe** tant que le worker ne tourne pas.

**Étape 5 : Lancer le worker**
```bash
php artisan queue:work
```
Le worker prend le Job, attend 5 secondes (le `sleep`), puis passe le statut à `traite`. Recharge la base : le statut a changé.

*(En développement, `queue:listen` recharge ton code à chaque Job — pratique tant que tu modifies le Job. `queue:work` est plus rapide mais doit être redémarré après un changement de code.)*

### Critères d'évaluation
- `QUEUE_CONNECTION=database` et table `jobs` présente
- `TraiterRawContentJob` créé et dispatché
- Sans worker : le statut reste `en_attente`, un Job apparaît en table
- Avec worker : le statut passe à `traite` après ~5 s

### Livrable
Dans `lab3-notes.md` :
- Capture de la table `jobs` avec un Job en attente (avant worker)
- Capture montrant le statut passé à `traite` (après worker)

### En cas de blocage
- **Rien ne se passe même avec le worker** : `QUEUE_CONNECTION` n'est pas sur `database`, ou la table `jobs` n'existe pas, ou tu n'as pas relancé le serveur après avoir changé le `.env` (`php artisan config:clear`).
- **`Class TraiterRawContentJob not found`** : vérifie le namespace `App\Jobs`.
- **Le Job s'exécute mais le statut ne change pas** : vérifie que `RawContent` a `statut` dans son `$fillable`.

---

## 📝 LAB 4 — Rendre la génération asynchrone
**Durée estimée : 45 min**

### Objectif
Remplacer le travail simulé du LAB 3 par la vraie génération de post (appel IA), déclenchée par l'endpoint de soumission, avec une réponse immédiate `202 Accepted`.

### Contexte
On branche le mécanisme de queue sur le vrai cas de ThreadForge : soumettre un contenu déclenche une génération en arrière-plan. L'API répond tout de suite.

### Prérequis
- LAB 3 terminé (queue et worker fonctionnels)
- L'agent de génération de la semaine dernière *(repris en détail à la séance 3 — ici on se concentre sur la queue)*

### Instructions

**Étape 1 : Transformer le Job en générateur**
Dans `TraiterRawContentJob` (ou un nouveau `GenererPostJob`), remplace le `sleep` par la génération réelle :
```php
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
```
*(`PostGenerator` est l'agent détaillé à la séance 3. Si le tien marchait la semaine dernière, réutilise-le.)*

**Étape 2 : Créer l'endpoint de soumission**
Dans un `Api/ContentController`, la méthode `repurpose` crée le `raw_content`, dispatch le Job, et répond `202` :
```php
public function repurpose(StoreContentRequest $request)
{
    $rawContent = auth()->user()->rawContents()->create([
        'blueprint_id' => $request->validated('blueprint_id'),
        'contenu_brut' => $request->validated('contenu_brut'),
        'statut'       => 'en_attente',
    ]);

    TraiterRawContentJob::dispatch($rawContent);

    return response()->json([
        'message' => 'Contenu reçu, génération en cours.',
        'raw_content_id' => $rawContent->id,
    ], 202);
}
```

**Étape 3 : Déclarer la route**
```php
Route::middleware('auth:sanctum')->post('/content/repurpose', [ContentController::class, 'repurpose']);
```

**Étape 4 : Tester le flux complet**
- `POST /api/content/repurpose` avec un contenu → réponse **immédiate** `202 Accepted` (la page ne fige pas)
- Le worker (`queue:work`) traite en arrière-plan
- Après quelques secondes, le `raw_content` passe en `traite` et un `generated_post` apparaît

### Critères d'évaluation
- L'endpoint répond `202` immédiatement (pas de page figée)
- Le Job crée bien un `generated_post` lié au `raw_content`
- Le statut passe de `en_attente` à `traite`

### Livrable
Dans `lab4-notes.md` :
- Capture de la réponse `202` (avec le temps de réponse rapide visible dans Postman)
- Capture du `generated_post` créé en base après passage du worker

### En cas de blocage
- **La réponse n'est pas immédiate** : l'appel IA est resté dans le controller au lieu d'être dans le Job. Vérifie que `repurpose` ne fait que dispatcher.
- **Le `generated_post` n'est pas créé** : vérifie la relation `generatedPost()` (hasOne) sur `RawContent` et les `$fillable` de `GeneratedPost`.
- **Erreur sur `$response['...']`** : l'agent ne renvoie pas la bonne structure → c'est l'objet de la séance 3 (structured output).
- **Une modif du Job n'est pas prise en compte** : `queue:work` garde le code en mémoire, redémarre-le (ou utilise `queue:listen`).
