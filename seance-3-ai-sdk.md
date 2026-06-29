# Simplon Maghreb — Sprint Final
# Semaine 1 — Séance 3 : Consolidation AI SDK (structured output)

## Objectifs pédagogiques
- Comprendre pourquoi le structured output est plus fiable que « demander du JSON » dans le prompt
- Définir un contrat JSON garanti avec le SDK `laravel/ai`
- Brancher l'agent sur la génération de ThreadForge
- Stocker les données de l'IA de façon typée grâce aux Casts

## Objectifs techniques
SDK `laravel/ai`, agent, structured output, `HasStructuredOutput`, `schema()`, contrat JSON, Eloquent Casts (`array`, enum), provider Groq

## Table des matières
1. Le problème : l'IA qui répond « à peu près »
2. Le contrat JSON garanti
3. L'agent de génération de ThreadForge
4. Stocker proprement avec les Casts
5. Récapitulatif et prochaines étapes

## Déroulé de la séance (Type Consolidation, 2h)
- **0–20 min** : Théorie. Texte libre vs contrat garanti, le rôle du schema (sections 1 à 2)
- **20–80 min** : LAB 5. L'agent de génération + le schema
- **80–115 min** : LAB 5 (suite). Les Casts + test du flux complet
- **115–120 min** : Récapitulatif et 3 points clés

---

## 1. Le problème : l'IA qui répond « à peu près »

Si tu demandes à l'IA dans le prompt « renvoie-moi un JSON avec un hook, des points et un score », tu joues à la loterie. Parfois elle ajoute une phrase avant le JSON (« Voici votre post : … »), parfois elle oublie un champ, parfois elle met le score en texte (« 85/100 ») au lieu d'un nombre. Et ton `json_decode` casse.

Pour une API, c'est inacceptable : la donnée qui arrive de l'IA doit avoir une **forme garantie**, toujours la même, pour être enregistrée sans surprise.

## 2. Le contrat JSON garanti

Le **structured output** résout ça. Au lieu de demander gentiment du JSON dans le prompt, on **impose un schéma** au SDK. Le SDK force le modèle à respecter exactement cette forme. La réponse revient garantie : les bons champs, les bons types, à chaque fois.

C'est la différence entre « s'il te plaît, renvoie ça » et « tu n'as pas le choix, c'est ce format ou rien ».

## 3. L'agent de génération de ThreadForge

Notre agent prend le contenu brut et renvoie le contrat suivant :
```json
{
  "hook_propose": "string",
  "body_points": ["string"],
  "technical_readability_score": "integer (0-100)",
  "suggested_hashtags": ["string"],
  "tone_compliance_justification": "string"
}
```
C'est l'objet du LAB 5 : on définit cet agent et son schema.

## 4. Stocker proprement avec les Casts

Les champs comme `body_points` et `suggested_hashtags` sont des tableaux. En base, ils sont stockés en colonnes JSON. Pour les manipuler comme des tableaux PHP **sans `json_encode`/`json_decode` manuel**, on utilise un Cast `array`. Le statut du post, lui, devient un backed enum casté.

## 5. Récapitulatif et prochaines étapes

### 5.1 Les 3 points clés à retenir
1. Le structured output **garantit** la forme de la réponse — pas de texte parasite, pas de champ manquant.
2. Le schema est le **contrat** : c'est lui qui force le modèle, pas le prompt.
3. Les Casts (`array`, enum) suppriment tout `json_encode` manuel et gardent les données typées.

### 5.2 Prochaine séance
Séance 4 : Introduction aux tests avec Pest — vérifier automatiquement que l'API et la génération marchent, sans tout re-cliquer à la main.

---

## 📝 LAB 5 — L'agent de génération + structured output + Casts
**Durée estimée : 90 min**

### Objectif
Définir l'agent `PostGenerator` qui transforme un contenu brut en post structuré garanti, puis stocker le résultat de façon typée grâce aux Casts.

### Contexte
C'est le cœur de ThreadForge. On consolide la partie IA : le contrat JSON, l'agent, et le stockage propre. C'est ce que le Job de la séance 2 appelle.

### Prérequis
- SDK `laravel/ai` installé et provider `groq` configuré (`config/ai.php`)
- `GROQ_API_KEY` dans `.env`
- Les models `RawContent` et `GeneratedPost` en place

### Instructions

**Étape 1 : Créer l'agent**
```bash
php artisan make:agent PostGenerator --structured
```
*(Le flag `--structured` prépare un agent pour le structured output. Il est créé dans `App\Ai\Agents`.)*

**Étape 2 : Définir les instructions (system prompt)**
Dans la méthode `instructions()`, décris le rôle clairement :
> « Tu es un ghostwriter pour la tech community sur X. À partir d'un contenu technique brut (notes de dev, README, article), tu produis un post optimisé : un hook accrocheur, des points clés courts, un score de lisibilité technique, des hashtags pertinents, et une justification du respect du ton. »

**Étape 3 : Définir le schema (le contrat JSON)**
```php
<?php
namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class PostGenerator implements Agent, HasStructuredOutput
{
    use Promptable;

    public function instructions(): string
    {
        return "..."; // ton system prompt de l'étape 2
    }

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
}
```
*(Pour la liste exacte des types disponibles, voir la doc en bas.)*

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

**Étape 5 : Appeler l'agent (test synchrone en Tinker)**
Avant de repasser par le Job, teste l'agent seul en Tinker :
```php
$response = (new \App\Ai\Agents\PostGenerator)->prompt("Mes notes : aujourd'hui j'ai appris les queues Laravel...");
$response['hook_propose'];   // une string
$response['body_points'];    // un tableau
```

**Étape 6 : Vérifier le stockage via le Job**
Relance le flux de la séance 2 (`POST /api/content/repurpose` + worker). Vérifie qu'un `generated_post` est créé avec `body_points` et `suggested_hashtags` en tableaux (grâce au Cast `array`, pas de `json_encode` manuel).

### Critères d'évaluation
- Agent `PostGenerator` avec un `schema()` conforme au contrat JSON
- L'appel renvoie tous les champs garantis (hook, body_points, score, hashtags, justification)
- Casts `array` et enum en place sur `GeneratedPost`
- Le `generated_post` est stocké typé (tableaux exploitables, pas de string JSON brute)

### Livrable
Dans `lab5-notes.md` :
- Capture de l'appel en Tinker montrant la réponse structurée
- Capture du `generated_post` en base avec `body_points` en tableau
- Une phrase : pourquoi le structured output est plus sûr que `json_decode` sur un prompt libre

### En cas de blocage
- **Erreur d'authentification IA** : le provider par défaut n'est pas `groq` dans `config/ai.php`, ou `GROQ_API_KEY` manque.
- **La réponse n'a pas la bonne forme** : revois ton system prompt (instructions trop vagues) et vérifie le `schema()`.
- **`body_points` est une string JSON, pas un tableau** : le Cast `array` n'est pas déclaré sur `GeneratedPost`, ou tu fais encore un `json_encode` manuel quelque part.
- **`Class "PostGenerator" not found`** : vérifie le namespace `App\Ai\Agents`.

### Ressources
- Doc officielle — SDK `laravel/ai` : https://laravel.com/docs/13.x/ai-sdk
- Doc officielle — Eloquent Casting : https://laravel.com/docs/13.x/eloquent-mutators
