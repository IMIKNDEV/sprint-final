# Simplon Maghreb — Sprint Final
# Semaine 1 — Séance 4 : Introduction aux tests avec Pest (le pont vers le DevOps)

## Objectifs pédagogiques
- Comprendre pourquoi on automatise les tests plutôt que de tout re-cliquer à la main
- Écrire un premier feature test sur un endpoint d'API
- Tester une route protégée (401) et une validation (422)
- Tester un comportement asynchrone (un Job dispatché) sans appeler l'IA réelle

## Objectifs techniques
Pest, feature test, `getJson` / `postJson`, `assertStatus`, `assertJsonStructure`, `assertJsonValidationErrors`, `Sanctum::actingAs`, `Queue::fake`, `assertPushed`

## Table des matières
1. Pourquoi tester automatiquement
2. Pest en deux minutes
3. Tester l'API
4. Tester l'asynchrone
5. Récapitulatif et prochaines étapes

## Déroulé de la séance (Type Découverte, 2h)
- **0–20 min** : Théorie. Le coût des tests manuels, ce qu'est un test automatisé (sections 1 à 2)
- **20–55 min** : LAB 6. Premier feature test sur un endpoint
- **55–90 min** : LAB 7. Tester une route protégée + une validation
- **90–115 min** : LAB 8. Tester un Job dispatché
- **115–120 min** : Récapitulatif et 3 points clés

---

## 1. Pourquoi tester automatiquement

Chaque fois que tu modifies ton code, comment sais-tu que tu n'as rien cassé ailleurs ? Sans tests, tu rouvres Postman, tu rejoues à la main chaque endpoint, à chaque modif. C'est long, et un jour tu oublies d'en vérifier un — et le bug part en production.

Un **test automatisé** est un bout de code qui vérifie ton code à ta place. Tu l'écris une fois, et tu peux le relancer en une commande, autant de fois que tu veux. C'est aussi le **prérequis du DevOps** : la semaine prochaine, une machine (CI) lancera ces tests automatiquement à chaque push. Sans tests, pas de CI qui ait du sens.

## 2. Pest en deux minutes

**Pest** est l'outil de test moderne de Laravel. Un test se lit presque comme une phrase :
```php
it('liste les blueprints', function () {
    // prépare une situation
    // fais une action
    // vérifie le résultat
});
```
Trois temps : on **prépare** (un utilisateur, des données), on **agit** (on appelle l'endpoint), on **vérifie** (le bon code, la bonne structure).

**Comment lancer les tests.** Trois commandes existent, et elles lancent toutes la même suite :
```bash
php artisan test     # le wrapper Laravel — recommandé, cohérent avec les autres commandes artisan
./vendor/bin/pest    # Pest directement — même résultat, affichage propre à Pest
```
Dans ce cours, on utilise **`php artisan test`**. Attention : c'est `php artisan test` (avec un espace), pas `php artisan:test`.

## 3. Tester l'API

On va tester nos endpoints ThreadForge : qu'ils répondent le bon code, la bonne structure JSON, qu'ils rejettent les requêtes sans token, et qu'ils valident bien. C'est l'objet des LAB 6 et 7.

## 4. Tester l'asynchrone

Tester un Job a une subtilité : on ne veut pas appeler la vraie IA (lent, coûteux, non déterministe) à chaque test. On utilise donc des **fakes** : on vérifie que le Job est bien **dispatché**, sans l'exécuter pour de vrai. C'est l'objet du LAB 8.

## 5. Récapitulatif et prochaines étapes

### 5.1 Les 3 points clés à retenir
1. Un test automatisé **vérifie ton code à ta place**, en une commande, autant de fois que tu veux.
2. On teste le **comportement** : le bon code, la bonne structure, le bon rejet — pas l'implémentation.
3. Pour l'asynchrone et l'IA, on **fake** : on vérifie le dispatch sans appeler la vraie IA.

### 5.2 Prochaine séance
Séance 5 : Consolidation — chacun finit avec une API propre, testée, et quelques tests verts. Préparation au DevOps de la semaine 2.

---

## 📝 LAB 6 — Premier feature test sur un endpoint
**Durée estimée : 35 min**

### Objectif
Installer Pest et écrire un premier test qui vérifie que `GET /api/blueprints` répond `200` avec la bonne structure JSON, pour un utilisateur authentifié.

### Contexte
On commence par le test le plus simple : un endpoint de lecture renvoie le bon code et la bonne forme.

### Prérequis
- API ThreadForge fonctionnelle (séances 1 à 3)

### Instructions

**Étape 1 : Installer Pest**
```bash
composer require pestphp/pest pestphp/pest-plugin-laravel --dev --with-all-dependencies
./vendor/bin/pest --init
```
*(Si tu as déjà Pest, saute cette étape. En cas de doute, voir la doc en bas.)*

**Étape 2 : Créer le fichier de test**
```bash
php artisan make:test BlueprintTest --pest
```
Le fichier apparaît dans `tests/Feature/BlueprintTest.php`.

**Étape 3 : Écrire le test**
```php
<?php

use App\Models\User;
use Laravel\Sanctum\Sanctum;

it('liste les blueprints d\'un utilisateur authentifié', function () {
    $user = User::factory()->create();
    Sanctum::actingAs($user);

    $user->blueprints()->create([
        'nom'            => 'Tech éducatif',
        'ton'            => 'professionnel',
        'max_hashtags'   => 2,
        'max_caracteres' => 280,
    ]);

    $response = $this->getJson('/api/blueprints');

    $response->assertStatus(200)
        ->assertJsonStructure([
            'data' => [
                ['id', 'nom', 'ton', 'max_hashtags', 'max_caracteres'],
            ],
        ]);
});
```

**Étape 4 : Lancer le test**
```bash
php artisan test
```
Tu dois voir une ligne verte ✓.

### Critères d'évaluation
- Pest installé et les tests se lancent (`php artisan test`)
- Le test vérifie le code `200` et la structure JSON
- Le test passe (vert)

### Livrable
Dans `lab6-notes.md` :
- Capture du test vert dans le terminal

### En cas de blocage
- **`Sanctum::actingAs` introuvable** : ajoute `use Laravel\Sanctum\Sanctum;` en haut du fichier.
- **`assertJsonStructure` échoue sur `data`** : tes endpoints renvoient des API Resources, qui enveloppent dans `data`. Si tu n'as pas cette enveloppe, retire le niveau `data` de l'assertion.
- **Le test touche ta vraie base** : vérifie que `phpunit.xml` configure une base de test (souvent SQLite en mémoire) et que tu utilises `RefreshDatabase` si besoin.

---

## 📝 LAB 7 — Tester une route protégée (401) et une validation (422)
**Durée estimée : 35 min**

### Objectif
Écrire deux tests qui vérifient les protections de l'API : une requête sans token est rejetée (`401`), et une création invalide est rejetée (`422`).

### Contexte
On teste ce qui doit échouer. Une API robuste rejette proprement — et on veut s'en assurer automatiquement.

### Prérequis
- LAB 6 terminé

### Instructions

**Étape 1 : Tester le rejet sans token (401)**
```php
it('rejette une requête sans token', function () {
    $response = $this->getJson('/api/blueprints');

    $response->assertStatus(401);
});
```

**Étape 2 : Tester la validation (422)**
```php
use App\Models\User;
use Laravel\Sanctum\Sanctum;

it('refuse un blueprint invalide', function () {
    Sanctum::actingAs(User::factory()->create());

    $response = $this->postJson('/api/blueprints', [
        'nom' => '', // requis manquant
    ]);

    $response->assertStatus(422)
        ->assertJsonValidationErrors(['nom']);
});
```

**Étape 3 : Lancer les tests**
```bash
php artisan test
```

### Critères d'évaluation
- Un test vérifie le `401` sans token
- Un test vérifie le `422` avec une entrée invalide et cible le bon champ
- Les deux tests passent

### Livrable
Dans `lab7-notes.md` :
- Capture des tests verts

### En cas de blocage
- **Le test 401 renvoie 200** : ta route n'est peut-être pas protégée par `auth:sanctum`.
- **Le test 422 renvoie 200 ou une erreur SQL** : ta Form Request n'est pas branchée (vérifie le type-hint dans `store`).
- **`assertJsonValidationErrors` échoue** : le nom du champ testé ne correspond pas à la règle de validation.

---

## 📝 LAB 8 — Tester un Job dispatché (sans appeler l'IA)
**Durée estimée : 25 min**

### Objectif
Vérifier que soumettre un contenu **dispatch** bien le Job de génération, sans exécuter la vraie IA.

### Contexte
On teste le comportement asynchrone. On ne veut pas appeler Groq à chaque test — on vérifie juste que le Job part dans la queue.

### Prérequis
- LAB 4 (endpoint de soumission + Job) en place

### Instructions

**Étape 1 : Écrire le test avec `Queue::fake`**
```php
use App\Jobs\TraiterRawContentJob;
use App\Models\User;
use Illuminate\Support\Facades\Queue;
use Laravel\Sanctum\Sanctum;

it('dispatch le job de génération à la soumission', function () {
    Queue::fake();

    $user = User::factory()->create();
    Sanctum::actingAs($user);

    $blueprint = $user->blueprints()->create([
        'nom' => 'Test', 'ton' => 'pro', 'max_hashtags' => 2, 'max_caracteres' => 280,
    ]);

    $response = $this->postJson('/api/content/repurpose', [
        'blueprint_id' => $blueprint->id,
        'contenu_brut' => 'Mes notes de dev du jour.',
    ]);

    $response->assertStatus(202);
    Queue::assertPushed(TraiterRawContentJob::class);
});
```

**Étape 2 : Lancer le test**
```bash
php artisan test
```

**Étape 3 (bonus) : tester le `handle()` avec l'IA fakée**
Pour tester ce que fait le Job *à l'intérieur* sans appeler la vraie IA, le SDK fournit `Agent::fake()`, qui génère des données factices conformes au schema. Voir la doc du SDK pour la syntaxe exacte.

### Critères d'évaluation
- Le test vérifie la réponse `202`
- Le test vérifie que le Job est dispatché (`assertPushed`)
- Le test passe sans appeler la vraie IA

### Livrable
Dans `lab8-notes.md` :
- Capture du test vert
- Une phrase : pourquoi on fake la queue/l'IA au lieu de les exécuter vraiment

### En cas de blocage
- **`assertPushed` échoue** : avec `Queue::fake()`, le Job n'est PAS exécuté, juste enregistré comme dispatché — c'est normal. Vérifie juste que ton controller fait bien `dispatch`.
- **Le test appelle quand même l'IA** : tu n'as pas mis `Queue::fake()` en début de test, donc le Job s'exécute pour de vrai.
- **`202` non reçu** : revois l'endpoint `repurpose` (LAB 4).

### Ressources
- Doc officielle — Tests Laravel 13 : https://laravel.com/docs/13.x/testing
- Doc officielle — Pest : https://pestphp.com/docs
- Guide complet Pest 4 + Laravel 13 (2026, très à jour) : https://hafiz.dev/blog/laravel-pest-4-testing-complete-guide
- Article « Write Better Laravel Tests with Pest » : https://laravelmagazine.com/write-better-laravel-tests-with-pest-php

> Note Laravel 13 : Pest est livré par défaut dans les nouveaux projets. Si `./vendor/bin/pest` fonctionne déjà, tu peux sauter l'installation.
