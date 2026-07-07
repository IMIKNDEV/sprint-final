# LABs Séance 1 - Subscription Azure, Budget et Sécurisation

**Sprint DevOps (ThreadForge Part 2) - Semaine 1 - Séance 1**
**Durée totale estimée : 75 min (LAB 1 : 20 min · LAB 2 : 25 min · LAB 3 : 30 min)**

## Ce que vous aurez à la fin

Une Subscription Azure active qui hébergera ThreadForge en production, un budget avec alertes email pour ne jamais avoir de facture surprise, et un compte de travail sécurisé (MFA) distinct du compte propriétaire.

## Prérequis (pour les 3 LABs)

- Email personnel valide (pas de boîte temporaire) + numéro de téléphone mobile (vérification SMS)
- Carte bancaire Visa/Mastercard avec paiement international activé (vérification d'identité uniquement, **aucune facturation automatique**). Cartes prépayées et virtuelles souvent refusées
- Smartphone avec **Microsoft Authenticator** installé
- Alternative sans carte : **Azure for Students** (100 $, 12 mois) si vous avez une adresse email étudiante valide

---

## LAB 1 - Créer la Subscription Azure (20 min)

**Objectif** : créer une Subscription via le Free Trial (200 $ de crédits, 30 jours) et se repérer dans le Portal.

### Étapes

1. **Créer le compte** : navigateur en navigation privée → `https://azure.microsoft.com/free` → **Start free**. Se connecter avec un compte Microsoft existant ou en créer un (**Create one!**).
2. **Vérifications** : renseigner nom, adresse, pays (Maroc) → vérifier le téléphone par SMS → renseigner la carte bancaire → cocher « I agree... » → **Sign up**.
   > Microsoft prélève 1 $ de vérification, remboursé sous 3-5 jours. À la fin des 30 jours, rien n'est facturé : les services sont suspendus tant que vous ne basculez pas explicitement en Pay-As-You-Go.
3. **Identifier la Subscription** : sur `portal.azure.com`, chercher `Subscriptions` → cliquer sur votre Subscription (« Azure subscription 1 » ou « Free Trial ») → noter dans votre gestionnaire de mots de passe :
   - **Subscription ID** (GUID)
   - **Tenant ID** (dans **Properties**)
   - **Status** : Active · **Offer** : Free Trial
4. **Vérifier les crédits** : menu latéral → **Cost Management + Billing** → **Credit balance** → 200,00 $ (ou 100 $ en Students).
5. **Renommer** : page de la Subscription → **Rename** → `sprint-devops-simplon` → **Save**.
6. **Repérer les services clés** dans la barre de recherche (ils reviendront pendant le sprint) : **Virtual machines** (séance 2, hébergera ThreadForge), **Resource groups**, **Microsoft Entra ID** (LAB 3), **Cost Management + Billing** (LAB 2).
7. **Tester la reconnexion** : Sign out → Sign in sur `portal.azure.com` → vérifier que vous retrouvez votre Subscription.

### Validation LAB 1

- [ ] Subscription active, renommée `sprint-devops-simplon`
- [ ] Subscription ID + Tenant ID notés en lieu sûr
- [ ] Crédits visibles (200 $ ou 100 $)
- [ ] Reconnexion réussie

---

## LAB 2 - Budget et alertes de facturation (25 min)

**Objectif** : poser le garde-fou financier **avant** de créer la moindre ressource. Budget mensuel de 20 $, alertes email à 5 $ et 20 $.

### Étapes

1. **Ouvrir Budgets** : Portal → chercher `Cost Management` → **Cost Management + Billing** → menu gauche, section **Monitoring** → **Budgets**.
2. **Vérifier le scope** (badge en haut de page) : il doit pointer sur `sprint-devops-simplon`. Sinon, cliquer le scope et sélectionner votre Subscription.
3. **Créer le budget** : **+ Add** →

   | Champ | Valeur |
   | --- | --- |
   | Name | `budget-mensuel-sprint` |
   | Reset period | `Monthly` |
   | Amount | `20` |

   → **Next**.
4. **Configurer les alertes** : cocher deux seuils, type **Actual**, avec votre email en destinataire :

   | Type | Seuil | Alerte à |
   | --- | --- | --- |
   | Actual | 25 % | 5 $ |
   | Actual | 100 % | 20 $ |

   → **Create**.
   > **Actual** = dépense réelle atteinte (ce qu'on veut). **Forecasted** = dépense prévue (plus proactif, plus de faux positifs).
5. **Vérifier** : le budget apparaît dans la liste, Amount 20 $, Spend to date ~0 $, statut OK. Ajouter `azure-noreply@microsoft.com` à vos contacts pour ne pas rater les alertes.
6. **Repérer le filet Free Trial** : **Subscriptions** → votre Subscription → **Overview** → bandeau « Your remaining $200.00 of free credit expires in X days ». Quand le crédit atteint 0 $ ou les 30 jours, les services payants sont **suspendus** (pas supprimés) — aucune facture possible sans bascule manuelle en Pay-As-You-Go.
7. **Jeter un œil à Cost Analysis** (Cost Management → **Cost analysis**, granularité Daily) : c'est l'écran que vous consulterez à chaque séance pour surveiller la consommation.

**La règle du sprint** : avant de créer une ressource Azure, je m'assure que mon budget est actif.

### Validation LAB 2

- [ ] Budget `budget-mensuel-sprint` créé au scope Subscription, 20 $/mois
- [ ] Deux alertes Actual (5 $ et 20 $) avec email renseigné
- [ ] Bandeau de crédit Free Trial repéré dans Overview

---

## LAB 3 - Utilisateur Entra ID et MFA (30 min)

**Objectif** : ne plus jamais travailler au quotidien avec le compte propriétaire. MFA partout, et un utilisateur `devops-<prénom>` pour l'usage courant.

**Pourquoi** : le compte propriétaire peut fermer la Subscription et changer le moyen de paiement. Une session volée = tout est perdu. ThreadForge manipulera des comptes utilisateurs et des tokens : le réflexe commence ici.

### Étapes

1. **MFA sur le compte propriétaire** : Portal → avatar en haut à droite → **My Microsoft account** → **Security** → **Manage how I sign in** → activer **Two-step verification** → méthode **App** (Microsoft Authenticator) → scanner le QR code → confirmer avec le code à 6 chiffres.
   > **IMPORTANT** : sauvegarder les **codes de récupération** dans un gestionnaire de mots de passe.
2. **Valider le MFA** : Sign out → Sign in sur `portal.azure.com` → le code Authenticator est maintenant demandé.
3. **Créer l'utilisateur** : chercher `Microsoft Entra ID` → **Users** → **+ New user** → **Create new user** :

   | Champ | Valeur |
   | --- | --- |
   | User principal name | `devops-<prénom>` (complété en `...@<tenant>.onmicrosoft.com`) |
   | Display name | `DevOps <Prénom>` |
   | Password | décocher Auto-generate, saisir un mot de passe robuste (12+ car., maj/min/chiffres/symboles) |

   → **Review + create** → **Create**.
4. **Donner le rôle Owner** : **Subscriptions** → `sprint-devops-simplon` → **Access control (IAM)** → **+ Add** → **Add role assignment** :
   - Onglet **Role** : sous-onglet **Privileged administrator roles** (⚠️ pas la barre de recherche, qui ne liste que les Job function roles) → **Owner** → **Next**
   - Onglet **Members** : **+ Select members** → `devops-<prénom>` → **Select**
   - Onglet **Conditions** : choisir **Allow user to assign all roles except privileged administrator roles (Recommended)** (⚠️ sans ce choix, le bouton final reste grisé)
   - **Review + assign**
   > En production réelle, Owner est trop large (on préfère Contributor + gestion d'accès séparée). Pour ce sprint : Owner simplifie.
5. **Première connexion en `devops-<prénom>`** : Sign out → Sign in avec `devops-<prénom>@<tenant>.onmicrosoft.com` (le Primary domain est visible dans Entra ID → Overview) :
   - Azure force le **changement de mot de passe** au premier login : le faire et le noter
   - Azure demande de **configurer le MFA** : Microsoft Authenticator, nouvelle entrée dans l'app (distincte du compte propriétaire)
6. **Vérifier les droits** : toujours en `devops-<prénom>` → **Subscriptions** → la Subscription est visible → **Access control (IAM)** → **Role assignments** → `devops-<prénom>` listé en Owner.
7. **Tester en créant une ressource** : chercher `Resource groups` → **+ Create** → nom `test-droits`, région `France Central` → **Create**. Si le RG se crée, vos droits fonctionnent. **Le supprimer immédiatement** (Delete resource group, taper le nom pour confirmer).

### Validation LAB 3

- [ ] MFA actif sur le propriétaire + codes de récupération sauvegardés
- [ ] Utilisateur `devops-<prénom>` créé, rôle Owner au scope Subscription
- [ ] MFA actif sur `devops-<prénom>`, connexion réussie
- [ ] RG `test-droits` créé puis supprimé

### La discipline à partir de maintenant

Tout le sprint se fait connecté en **`devops-<prénom>`**. Le compte propriétaire ne sert plus qu'aux actions exceptionnelles : changer le moyen de paiement, basculer en Pay-As-You-Go, récupérer l'accès si `devops-<prénom>` est perdu, fermer la Subscription en fin de sprint.

---

## Livrable (un seul fichier : `labs-azure-notes.md`)

1. Capture du Portal avec la Subscription `sprint-devops-simplon` affichée
2. Capture de **Credit balance** (200 $ ou 100 $)
3. Subscription ID et Tenant ID (partiellement masqués)
4. Capture du budget et de ses 2 alertes
5. Capture du bandeau de crédit Free Trial (Overview)
6. Capture de **Entra ID > Users** montrant `devops-<prénom>`
7. Capture de **Access control (IAM)** montrant le rôle Owner sur `devops-<prénom>`
8. Capture du Portal connecté en tant que `devops-<prénom>` (avatar en haut à droite)

---

## En cas de blocage

| Problème | Solution |
| --- | --- |
| Carte bancaire refusée | Carte au nom du titulaire, Visa/Mastercard internationale. Prépayées et virtuelles souvent refusées. Sinon : Azure for Students |
| Microsoft demande une vérification supplémentaire | Procédure anti-fraude : uploader passeport ou CIN, délai 24-48h |
| Email déjà utilisé | Vérifier sur `account.microsoft.com` si un compte existe déjà, sinon utiliser un autre email |
| Pas de 200 $ visibles | Vérifier que l'Offer est bien **Free Trial** et pas Pay-As-You-Go |
| Le 1 $ apparaît sur le relevé | Normal, remboursé sous 3-5 jours |
| « Access denied » sur Cost Management | Vérifier que vous êtes Owner (Subscriptions > Access control (IAM)) |
| Pas de bouton « + Add » sur Budgets | Scope mal sélectionné : se positionner sur la Subscription |
| « You don't have permission to create users » | Entra ID > Roles and administrators : attribuer **User Administrator** à votre compte |
| MFA refuse les codes | Horloge du téléphone décalée : activer l'heure automatique |
| L'utilisateur Entra ID n'a accès à rien | Le role assignment met parfois 1-2 min à se propager. Sinon refaire l'étape 4 |
| Authenticator demande un compte pro | Choisir « **Other account** » dans l'app |
| Mot de passe Entra ID rejeté | 12+ caractères, pas de pattern évident (`Password123!` est refusé) |

## Discipline coût

Surveillez votre **Credit balance** au moins une fois par séance. En Free Trial, deux compteurs tournent : **30 jours** et **200 $** — le premier épuisé suspend les services payants.
