# Simplon Maghreb — Sprint Final
# Semaine 1 — Séance 1 : Remise à niveau API REST (le squelette propre)

## Objectifs pédagogiques
- Comprendre ce qui distingue une API REST propre d'une API qui « marche mais fuit »
- Reconstruire le squelette REST de ThreadForge : routes, controller, réponse JSON
- Valider les entrées avec une Form Request et renvoyer les bons status codes
- Formater les sorties avec une API Resource pour ne jamais exposer de données internes

## Objectifs techniques
API REST, `routes/api.php`, controller d'API (sans Blade), réponse JSON, status codes (200, 201, 422), Form Request, API Resource, fuite de données

## Table des matières
1. Pourquoi une API « qui marche » ne suffit pas
2. Le fil conducteur : ThreadForge, round 2
3. Le squelette REST propre
4. Valider et répondre proprement
5. Récapitulatif et prochaines étapes

## Déroulé de la séance (Type Remise à niveau, 2h)
- **0–25 min** : Théorie. Pourquoi une API propre, l'histoire de la fuite de données, rappel REST (sections 1 à 2)
- **25–75 min** : LAB 1. Routes + Controller + réponse JSON
- **75–115 min** : LAB 2. Form Request + status codes + API Resource
- **115–120 min** : Récapitulatif et 3 points clés

---

## 1. Pourquoi une API « qui marche » ne suffit pas

### 1.1 L'incident : l'API qui montrait trop

Mardi, 14h. Reda est développeur junior. Il a livré la semaine dernière l'API de ThreadForge — l'outil qui transforme des notes de dev en posts X. Son endpoint `GET /api/posts` fonctionne : il renvoie bien les posts générés. Reda est content, c'était son premier projet API.

Jeudi. Un autre développeur, Salma, branche un frontend sur l'API de Reda. En ouvrant les outils réseau du navigateur, elle voit la réponse JSON complète d'un post. Et là, dans les données, il y a : le `user_id` du créateur, l'objet `user` entier avec son **email** et son **mot de passe haché**, les dates internes, et même le `payload_brut` — la réponse brute de l'IA avec les métadonnées internes.

Pourquoi ? Parce que Reda a écrit `return Post::all();`. Eloquent a sérialisé **tout le modèle**, y compris ses relations chargées. L'API « marchait » — mais elle exposait des données qui ne devaient jamais sortir.

Vendredi. Un test de sécurité interne bloque la mise en production. Verdict : fuite de données. L'API doit être reprise avant tout déploiement.

### 1.2 La leçon

Une API n'est pas jugée sur « est-ce qu'elle renvoie les données ». Elle est jugée sur **est-ce qu'elle renvoie exactement les bonnes données, dans le bon format, avec le bon code, et rien d'autre**. Trois disciplines séparent une démo d'une API de production :

- **La validation en entrée** : ne jamais laisser une donnée invalide atteindre la base (sinon erreur SQL exposée, ou données corrompues).
- **Les status codes** : le client doit savoir si ça a marché (200), si c'est créé (201), ou si sa requête est invalide (422) — pas juste « ça a répondu ».
- **Le formatage en sortie** : décider explicitement quels champs sortent, via une API Resource. Jamais `return Model::all()`.

C'est exactement ce qu'on reprend aujourd'hui sur ThreadForge.

## 2. Le fil conducteur : ThreadForge, round 2

Vous connaissez déjà ThreadForge : une API qui transforme un contenu brut (`raw_content`) en post X structuré (`generated_post`), avec des règles de style (`blueprint`). La semaine dernière, l'objectif était de le faire **marcher**. Cette semaine, l'objectif est de le faire **proprement et de le tester** — exactement la différence entre « j'ai codé une feature » et « j'ai livré du logiciel fiable ».

On repart du même domaine, mais on se concentre sur la qualité de l'API, puis (séances suivantes) sur les Jobs, l'IA, et les tests automatisés.

## 3. Le squelette REST propre

Rappel express avant le LAB. Une API REST, c'est :
- des **routes** dans `routes/api.php` (jamais `web.php` pour une API)
- des **controllers** qui renvoient du **JSON**, jamais une vue Blade
- des **verbes HTTP** qui portent le sens : `GET` (lire), `POST` (créer), `PATCH/PUT` (modifier), `DELETE` (supprimer)
- des **status codes** qui disent au client ce qui s'est passé

C'est l'objet du LAB 1.

## 4. Valider et répondre proprement

Deux outils Laravel font le gros du travail :
- la **Form Request** valide l'entrée *avant* le controller et renvoie automatiquement un `422` avec les erreurs si c'est invalide ;
- l'**API Resource** décide quels champs sortent dans le JSON — c'est ce qui empêche la fuite de données de l'histoire de Reda.

C'est l'objet du LAB 2.

## 5. Récapitulatif et prochaines étapes

### 5.1 Les 3 points clés à retenir
1. Une API propre **valide en entrée, formate en sortie, et code correctement** — « ça répond » ne suffit pas.
2. `return Model::all()` est une **fuite de données** : on passe toujours par une API Resource.
3. Une entrée invalide doit renvoyer un **422 propre**, jamais une erreur SQL.

### 5.2 Prochaine séance
Séance 2 : Remise à niveau Jobs & Queues — sortir l'appel IA du cycle de la requête pour ne plus figer l'API. Le squelette propre d'aujourd'hui en est le prérequis.

---

## 📝 LAB 1 — Routes API + Controller + réponse JSON
**Durée estimée : 45 min**

### Objectif
Reconstruire le squelette REST de la ressource `blueprints` dans ThreadForge : les routes dans `api.php`, un controller d'API, et des réponses JSON propres pour lister et afficher.

### Contexte
On repart de ThreadForge. Avant de toucher à l'IA ou aux Jobs, on s'assure que le squelette REST de base est solide et compris. On commence par la ressource la plus simple : les `blueprints` (les configs de style).

### Prérequis
- Le projet ThreadForge de la semaine dernière (ou la base fournie)
- `php artisan install:api` déjà exécuté (sinon le lancer)
- Migrations à jour (`php artisan migrate`)

### Instructions

**Étape 1 : Vérifier le fichier de routes API**
Ouvre `routes/api.php`. Sache que toutes les routes ici sont préfixées automatiquement par `/api`.

**Étape 2 : Créer le controller d'API**
```bash
php artisan make:controller Api/BlueprintController --api
```
*(Le flag `--api` génère un controller sans les méthodes `create` et `edit`, qui n'ont pas de sens en API — pas de formulaire HTML.)*

**Étape 3 : Déclarer les routes**
Dans `routes/api.php`, déclare les routes de la ressource (protégées par `auth:sanctum`, qu'on reverra en détail à la séance auth) :
```php
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/blueprints', [BlueprintController::class, 'index']);
    Route::get('/blueprints/{blueprint}', [BlueprintController::class, 'show']);
});
```

**Étape 4 : Implémenter `index`**
Renvoie les blueprints de l'utilisateur connecté, en JSON :
```php
public function index()
{
    $blueprints = auth()->user()->blueprints()->latest()->get();
    return response()->json($blueprints);
}
```

**Étape 5 : Implémenter `show`**
Renvoie un blueprint précis (route model binding) :
```php
public function show(Blueprint $blueprint)
{
    return response()->json($blueprint);
}
```

**Étape 6 : Tester avec Postman (ou Thunder Client)**
- `GET /api/blueprints` → doit renvoyer un tableau JSON de blueprints
- `GET /api/blueprints/1` → doit renvoyer un seul blueprint
- Observe le code de réponse : `200 OK`

### Critères d'évaluation
- `routes/api.php` contient les deux routes
- `BlueprintController` créé avec `--api`
- `GET /api/blueprints` renvoie du JSON avec un code `200`
- `GET /api/blueprints/{id}` renvoie le bon blueprint

### Livrable
Dans `lab1-notes.md` :
- Capture de la réponse JSON de `GET /api/blueprints` dans Postman (avec le code 200 visible)
- Capture de `GET /api/blueprints/{id}`

### En cas de blocage
- **`Route [login] not defined` ou redirection** : `auth:sanctum` rejette la requête car pas de token. Pour ce LAB, teste avec un token créé en Tinker (`$user->createToken('test')->plainTextToken`) et envoie-le en header `Authorization: Bearer ...` dans Postman. La séance auth détaillera tout ça.
- **Réponse vide `[]`** : l'utilisateur connecté n'a pas de blueprints. Crée-en via Tinker ou un seeder.
- **Erreur 404 sur `/api/blueprints`** : tu as peut-être mis la route dans `web.php` au lieu de `api.php`, ou `install:api` n'a pas été lancé.
- **La relation `blueprints()` n'existe pas** : vérifie qu'elle est bien définie sur le model `User` (`hasMany(Blueprint::class)`).

---

## 📝 LAB 2 — Form Request + status codes + API Resource
**Durée estimée : 40 min**

### Objectif
Sécuriser la création d'un blueprint : valider l'entrée avec une Form Request (422 si invalide), renvoyer un 201 à la création, et formater la sortie avec une API Resource pour ne jamais exposer de champs internes.

### Contexte
C'est la leçon de l'histoire de Reda : on valide en entrée et on formate en sortie. On ajoute la création d'un blueprint, proprement.

### Prérequis
- LAB 1 terminé (controller et routes en place)

### Instructions

**Étape 1 : Créer la Form Request**
```bash
php artisan make:request StoreBlueprintRequest
```
Dans `rules()`, valide les champs du blueprint :
```php
public function authorize(): bool
{
    return true;
}

public function rules(): array
{
    return [
        'nom'            => ['required', 'string', 'max:100'],
        'ton'            => ['required', 'string', 'max:255'],
        'max_hashtags'   => ['required', 'integer', 'min:0', 'max:10'],
        'max_caracteres' => ['required', 'integer', 'min:50', 'max:280'],
    ];
}
```

**Étape 2 : Créer l'API Resource**
```bash
php artisan make:resource BlueprintResource
```
Dans `toArray()`, choisis **explicitement** les champs qui sortent — pas de `user_id`, pas de timestamps internes si tu n'en veux pas :
```php
public function toArray($request): array
{
    return [
        'id'             => $this->id,
        'nom'            => $this->nom,
        'ton'            => $this->ton,
        'max_hashtags'   => $this->max_hashtags,
        'max_caracteres' => $this->max_caracteres,
    ];
}
```

**Étape 3 : Ajouter la route `store`**
```php
Route::post('/blueprints', [BlueprintController::class, 'store']);
```

**Étape 4 : Implémenter `store` avec la Form Request et le 201**
```php
public function store(StoreBlueprintRequest $request)
{
    $blueprint = auth()->user()->blueprints()->create($request->validated());

    return (new BlueprintResource($blueprint))
        ->response()
        ->setStatusCode(201);
}
```

**Étape 5 : Renvoyer les Resources dans `index` et `show`**
Remplace les `response()->json(...)` du LAB 1 par la Resource :
```php
// index
return BlueprintResource::collection(auth()->user()->blueprints()->latest()->get());

// show
return new BlueprintResource($blueprint);
```

**Étape 6 : Tester les trois cas dans Postman**
- `POST /api/blueprints` avec un body valide → `201 Created`, et la réponse ne contient **que** les champs de la Resource
- `POST /api/blueprints` avec un champ manquant (ex. `nom` vide) → `422 Unprocessable Entity` avec les erreurs en JSON
- `GET /api/blueprints` → vérifie qu'aucun champ interne (user_id, timestamps) ne sort

### Critères d'évaluation
- `StoreBlueprintRequest` valide les champs et renvoie `422` si invalide
- `store` renvoie `201` à la création
- `BlueprintResource` utilisée dans `index`, `show` et `store`
- Aucune donnée interne (user_id, mot de passe, etc.) n'apparaît dans le JSON

### Livrable
Dans `lab2-notes.md` :
- Capture d'un `POST` valide → réponse 201 avec la Resource
- Capture d'un `POST` invalide → réponse 422 avec les erreurs
- Une phrase : quel champ interne `Model::all()` aurait exposé, que la Resource bloque

### En cas de blocage
- **Le 422 ne se déclenche pas** : tu as peut-être validé dans le controller au lieu d'utiliser la Form Request en type-hint (`store(StoreBlueprintRequest $request)`).
- **`403 This action is unauthorized`** : la méthode `authorize()` de la Form Request renvoie `false`. Mets `return true;`.
- **Des champs internes sortent encore** : tu renvoies sûrement le modèle directement quelque part, pas la Resource. Vérifie les trois méthodes.
- **`422` mais le message d'erreur n'est pas en JSON** : assure-toi que ta requête Postman envoie le header `Accept: application/json` — sinon Laravel tente une redirection web.
