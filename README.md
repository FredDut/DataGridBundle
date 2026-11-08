# DataGrid Bundle

`DataGridBundle` est un bundle Symfony qui permet de déclarer, configurer et exposer des grilles de données via une API JSON.

Il fournit notamment :

- un registre de grilles (`GridRegistry`) ;
- une configuration déclarative des colonnes, filtres, tris et actions groupées ;
- des contrôleurs HTTP pour les données, les métadonnées, l’export et les actions groupées ;
- une interface de contrôle d’accès remplaçable par l’application hôte ;
- une implémentation par défaut, `AllowAllGridAccessChecker`, qui autorise toutes les grilles.

## Installation

```bash
composer require freddut/data-grid-bundle
```

Si le bundle n’est pas découvert automatiquement par Symfony Flex, ajoutez-le dans `config/bundles.php` :

```php
return [
    // ...
    FredDut\DataGridBundle\DataGridBundle::class => ['all' => true],
];
```

## Concepts

| Concept | Rôle |
|---|---|
| `GridConfiguration` | Décrit une grille : entité, colonnes, filtres, tri et actions |
| `GridRegistry` | Enregistre les grilles et les retrouve par leur nom |
| `GridController` | Expose les routes HTTP du bundle |
| `GridAccessCheckerInterface` | Décide si l’utilisateur courant peut consulter, exporter ou exécuter une action sur une grille |
| `AllowAllGridAccessChecker` | Implémentation par défaut : toutes les grilles sont accessibles, y compris sans authentification |

## Déclarer une grille

Créez une classe de configuration dans votre application :

```php
<?php

declare(strict_types=1);

namespace App\Grid;

use App\Entity\BlogPost;
use FredDut\DataGridBundle\Configuration\Column;
use FredDut\DataGridBundle\Configuration\GridConfiguration;
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag('data_grid.configuration')]
final readonly class BlogPostGrid
{
    public function __invoke(): GridConfiguration
    {
        return new GridConfiguration(
            name: 'blog_post_simple',
            entityClass: BlogPost::class,
            columns: [
                new Column(name: 'title', label: 'Titre', sortable: true),
                new Column(name: 'author', label: 'Auteur', sortable: true),
                new Column(name: 'createdAt', label: 'Créé le', sortable: true),
            ],
        );
    }
}
```

La grille est ensuite disponible à cette URL :

```text
/backoffice/grids/blog_post_simple/data/native
```

## Routes exposées

Le bundle expose les routes suivantes :

| Action | Méthode | Route |
|---|---|---|
| Données de la grille | `GET` | `/backoffice/grids/{gridName}/data/{engine}` |
| Métadonnées de la grille | `GET` | `/backoffice/grids/{gridName}/metadata` |
| Export | `GET` | `/backoffice/grids/{gridName}/export/{engine}` |
| Action groupée | `POST` | `/backoffice/grids/{gridName}/bulk-action` |

Exemple d’appel pour récupérer les données :

```text
GET /backoffice/grids/blog_post_simple/data/native?page=1&pageSize=25&sorts=[]&filters=[]
```

Paramètres courants :

- `page` : numéro de page ;
- `pageSize` : nombre d’éléments par page ;
- `sorts` : critères de tri ;
- `filters` : filtres appliqués à la grille.

## Contrôle d’accès

Par défaut, le bundle fournit `AllowAllGridAccessChecker`. Toutes les grilles sont donc accessibles, y compris pour les visiteurs non authentifiés.

Cette implémentation est adaptée à un site public ou à une première intégration sans authentification.

### Interface du bundle

```php
namespace FredDut\DataGridBundle\Security;

use FredDut\DataGridBundle\Configuration\GridConfiguration;
use Symfony\Component\Security\Core\User\UserInterface;

interface GridAccessCheckerInterface
{
    public function canView(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool;

    public function canExport(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool;

    public function canBulkAction(
        ?UserInterface $user,
        GridConfiguration $configuration,
        string $action,
        ?array $context = null,
    ): bool;
}
```

Le bundle ne doit jamais imposer l’authentification. Il transmet l’utilisateur courant au checker ; cet utilisateur peut être `null` lorsque la requête est anonyme.

## Grilles publiques et protégées

Pour un site où certaines grilles sont publiques et d’autres nécessitent une authentification ou un rôle, créez votre propre implémentation de `GridAccessCheckerInterface`.

### Exemple : règles par nom de grille

```php
<?php

declare(strict_types=1);

namespace App\Security;

use FredDut\DataGridBundle\Configuration\GridConfiguration;
use FredDut\DataGridBundle\Security\GridAccessCheckerInterface;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
use Symfony\Component\Security\Core\User\UserInterface;

final readonly class GridAccessChecker implements GridAccessCheckerInterface
{
    public function __construct(
        private AuthorizationCheckerInterface $authorizationChecker,
    ) {
    }

    public function canView(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool {
        return match ($configuration->getName()) {
            'blog_post_simple' => true,
            'user_profile' => $user !== null,
            'admin_users' => $user !== null
                && $this->authorizationChecker->isGranted('ROLE_ADMIN'),
            default => false,
        };
    }

    public function canExport(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool {
        return $this->canView($user, $configuration);
    }

    public function canBulkAction(
        ?UserInterface $user,
        GridConfiguration $configuration,
        string $action,
        ?array $context = null,
    ): bool {
        return $this->canView($user, $configuration);
    }
}
```

### Enregistrer le checker

Dans `config/services.yaml` :

```yaml
services:
    FredDut\DataGridBundle\Security\GridAccessCheckerInterface:
        alias: App\Security\GridAccessChecker
```

Cette configuration garantit que le bundle utilise votre implémentation au lieu de `AllowAllGridAccessChecker`.

### Comportement obtenu

| Grille | Visiteur anonyme | Utilisateur connecté | Administrateur |
|---|---:|---:|---:|
| `blog_post_simple` | Autorisé | Autorisé | Autorisé |
| `user_profile` | Refusé | Autorisé | Autorisé |
| `admin_users` | Refusé | Refusé | Autorisé |

## Règles importantes

- Ne forcez pas `UserInterface` dans les contrôleurs : l’utilisateur peut être `null` pour une grille publique.
- Le bundle délègue toutes les décisions de sécurité à `GridAccessCheckerInterface`.
- L’application est responsable des règles métier : rôles, permissions, propriété des données, appartenance à une équipe, etc.
- Pour des règles complexes, utilisez un voter Symfony depuis votre `GridAccessChecker`.

## Exemple avec un voter

```php
public function canView(
    ?UserInterface $user,
    GridConfiguration $configuration,
): bool {
    return $this->authorizationChecker->isGranted(
        'DATA_GRID_VIEW',
        $configuration,
    );
}
```

Votre voter peut alors décider selon :

- le nom de la grille ;
- les rôles de l’utilisateur ;
- les permissions métier ;
- des attributs personnalisés de la configuration ;
- toute règle propre à votre application.

## Implémentation par défaut

Si votre application n’a pas de système d’authentification, ou si toutes les grilles doivent être publiques, utilisez l’implémentation fournie :

```php
namespace FredDut\DataGridBundle\Security;

use FredDut\DataGridBundle\Configuration\GridConfiguration;
use Symfony\Component\Security\Core\User\UserInterface;

final readonly class AllowAllGridAccessChecker implements GridAccessCheckerInterface
{
    public function canView(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool {
        return true;
    }

    public function canExport(
        ?UserInterface $user,
        GridConfiguration $configuration,
    ): bool {
        return true;
    }

    public function canBulkAction(
        ?UserInterface $user,
        GridConfiguration $configuration,
        string $action,
        ?array $context = null,
    ): bool {
        return true;
    }
}
```

## Dépannage

### Une grille publique retourne 401

Vérifiez que le contrôleur ne contient pas de code imposant l’authentification :

```php
// À éviter pour une grille publique
if (!$user instanceof UserInterface) {
    throw new AccessDeniedException('Authentication required.');
}
```

Le contrôleur doit récupérer un utilisateur nullable :

```php
private function getCurrentUser(): ?UserInterface
{
    $user = $this->security->getUser();

    return $user instanceof UserInterface ? $user : null;
}
```

Puis déléguer la décision au checker :

```php
$user = $this->getCurrentUser();

if (!$this->accessChecker->canView($user, $configuration)) {
    throw new AccessDeniedHttpException(sprintf(
        'Access denied to grid "%s".',
        $gridName,
    ));
}
```

### Vérifier le checker utilisé

```bash
php bin/console debug:container GridAccessCheckerInterface
```

Le service résolu doit être votre implémentation applicative si vous avez configuré un alias.

### Vérifier les arguments du contrôleur

```bash
php bin/console debug:container --show-arguments FredDut\DataGridBundle\Controller\GridController
```

L’argument `accessChecker` doit pointer vers :

```text
FredDut\DataGridBundle\Security\AllowAllGridAccessChecker
```

ou vers votre propre implémentation si vous l’avez enregistrée.

### Vider le cache

Après modification des services ou du conteneur :

```bash
php bin/console cache:clear
```

## Intégration frontend

La route principale pour afficher une grille est :

```text
GET /backoffice/grids/{gridName}/data/{engine}
```

Exemple :

```text
GET /backoffice/grids/blog_post_simple/data/native?page=1&pageSize=25&sorts=[]&filters=[]
```

### Paramètres de requête

| Paramètre | Type | Description |
|---|---|---|
| `page` | entier | Numéro de la page demandée. Commence à `1`. |
| `pageSize` | entier | Nombre d’éléments par page. |
| `sorts` | JSON | Tableau des tris appliqués. |
| `filters` | JSON | Tableau des filtres appliqués. |

### Exemple de tri

```text
sorts=[{"field":"createdAt","direction":"desc"}]
```

### Exemple de filtre

```text
filters=[{"field":"title","operator":"contains","value":"Symfony"}]
```

Les opérateurs disponibles dépendent du moteur de données et des filtres déclarés dans la `GridConfiguration`.

## Format de réponse

La réponse est un objet JSON contenant les lignes de la page demandée ainsi que les informations de pagination.

```json
{
  "data": [
    {
      "id": 1,
      "title": "Premier article",
      "author": "Jean Dupont",
      "createdAt": "2026-11-05T10:00:00+00:00"
    },
    {
      "id": 2,
      "title": "Deuxième article",
      "author": "Marie Martin",
      "createdAt": "2026-11-04T15:30:00+00:00"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 25,
    "totalItems": 128,
    "totalPages": 6
  }
}
```

### Champs de réponse

| Champ | Description |
|---|---|
| `data` | Tableau des enregistrements de la page courante |
| `pagination.page` | Page actuellement renvoyée |
| `pagination.pageSize` | Nombre maximal d’éléments par page |
| `pagination.totalItems` | Nombre total d’éléments correspondant aux filtres |
| `pagination.totalPages` | Nombre total de pages |

Les noms des propriétés présentes dans chaque élément de `data` correspondent aux colonnes déclarées dans la `GridConfiguration`.

## Exemple JavaScript

```javascript
async function loadGrid(gridName, page = 1, pageSize = 25) {
  const params = new URLSearchParams({
    page: String(page),
    pageSize: String(pageSize),
    sorts: JSON.stringify([]),
    filters: JSON.stringify([]),
  });

  const response = await fetch(
    `/backoffice/grids/${gridName}/data/native?${params.toString()}`,
    {
      headers: {
        Accept: 'application/json',
      },
    },
  );

  if (!response.ok) {
    throw new Error(`Impossible de charger la grille : ${response.status}`);
  }

  return response.json();
}
```

Utilisation :

```javascript
const result = await loadGrid('blog_post_simple');

console.log(result.data);
console.log(result.pagination.totalItems);
```

## Exemple avec tri et filtre

```javascript
const sorts = [
  { field: 'createdAt', direction: 'desc' },
];

const filters = [
  { field: 'title', operator: 'contains', value: 'Symfony' },
];

const params = new URLSearchParams({
  page: '1',
  pageSize: '25',
  sorts: JSON.stringify(sorts),
  filters: JSON.stringify(filters),
});

const response = await fetch(
  `/backoffice/grids/blog_post_simple/data/native?${params.toString()}`,
);

const result = await response.json();
```

## Gestion des erreurs

| Code HTTP | Signification |
|---|---|
| `200` | La grille a été chargée avec succès |
| `400` | Paramètres invalides, filtre ou tri non supporté |
| `401` | Accès refusé car l’utilisateur n’est pas authentifié |
| `403` | Accès refusé par le `GridAccessChecker` |
| `404` | La grille demandée n’existe pas dans le `GridRegistry` |

Pour une grille publique, une requête anonyme doit retourner `200`. Si elle retourne `401`, vérifiez que le contrôleur ne force pas l’authentification avant d’appeler `GridAccessCheckerInterface::canView()`.

## Actions groupées

Les actions groupées permettent d’appliquer une opération à plusieurs enregistrements sélectionnés dans une grille.

La route est :

```text
POST /backoffice/grids/{gridName}/bulk-action
```

Exemple :

```text
POST /backoffice/grids/blog_post_simple/bulk-action
```

### Requête

Le corps de la requête doit être du JSON.

```json
{
  "action": "publish",
  "ids":,[1][2][3]
  "context": {
    "comment": "Publication automatique"
  }
}
```

| Champ | Type | Obligatoire | Description |
|---|---|---:|---|
| `action` | string | Oui | Nom de l’action groupée à exécuter |
| `ids` | array | Oui | Identifiants des enregistrements ciblés |
| `context` | object | Non | Données supplémentaires transmises au handler de l’action |

### Exemple JavaScript

```javascript
async function runBulkAction(gridName, action, ids, context = {}) {
  const response = await fetch(
    `/backoffice/grids/${gridName}/bulk-action`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
      },
      body: JSON.stringify({
        action,
        ids,
        context,
      }),
    },
  );

  if (!response.ok) {
    const error = await response.json().catch(() => null);

    throw new Error(
      error?.message ?? `L’action groupée a échoué : ${response.status}`,
    );
  }

  return response.json();
}
```

Utilisation :

```javascript
await runBulkAction(
  'blog_post_simple',
  'publish',
 ,[1][2][3]
  { comment: 'Publication automatique' },
);
```

## Réponse en cas de succès

La réponse confirme que l’action a été exécutée et peut contenir le nombre d’éléments traités.

```json
{
  "success": true,
  "action": "publish",
  "processedCount": 3
}
```

| Champ | Description |
|---|---|
| `success` | Indique que l’action a été exécutée |
| `action` | Nom de l’action exécutée |
| `processedCount` | Nombre d’enregistrements effectivement traités |

## Contrôle d’accès

Avant d’exécuter une action groupée, le bundle appelle :

```php
$this->accessChecker->canBulkAction(
    $user,
    $configuration,
    $action,
    $context,
);
```

L’implémentation par défaut, `AllowAllGridAccessChecker`, autorise toutes les actions.

Pour restreindre certaines actions, implémentez votre propre `GridAccessChecker` :

```php
public function canBulkAction(
    ?UserInterface $user,
    GridConfiguration $configuration,
    string $action,
    ?array $context = null,
): bool {
    if ($configuration->getName() !== 'blog_post_simple') {
        return false;
    }

    if ($action === 'publish') {
        return $user !== null
            && $this->authorizationChecker->isGranted('ROLE_EDITOR');
    }

    return false;
}
```

Dans cet exemple :

- l’action `publish` nécessite un utilisateur connecté avec le rôle `ROLE_EDITOR` ;
- toute autre action groupée sur `blog_post_simple` est refusée ;
- les autres grilles sont refusées par défaut.

## Erreurs possibles

| Code HTTP | Signification |
|---|---|
| `200` | L’action groupée a été exécutée avec succès |
| `400` | Requête invalide : action absente, `ids` absent ou format JSON incorrect |
| `401` | L’utilisateur doit être authentifié pour exécuter l’action |
| `403` | Le `GridAccessChecker` refuse l’action pour cet utilisateur ou cette grille |
| `404` | La grille ou l’action groupée demandée n’existe pas |
| `422` | L’action est syntaxiquement valide, mais les données fournies sont invalides |
| `500` | Une erreur interne est survenue pendant l’exécution |

## Exemple de réponse d’erreur

```json
{
  "error": "Access denied to bulk action \"publish\" for grid \"blog_post_simple\"."
}
```

Pour une grille publique, une action groupée peut néanmoins être protégée : la consultation de la grille et l’exécution d’une action sont deux décisions distinctes. Le bundle utilise donc `canView()` pour lire les données et `canBulkAction()` pour exécuter une action.

# grid


Dans assets/bootstrap.js ou stimulus_bootstrap.js

import { startStimulusApp } from '@symfony/stimulus-bundle';
import NativeGridController from '@freddut/data-grid-bundle/native-grid';

const app = startStimulusApp();

app.register('native-grid', NativeGridController);
const registerLazyController = async (name, modulePath) => {
    if (!document.querySelector(`[data-controller="${name}"]`)) {
        return;
    }

    const { default: controller } = await import(modulePath);

    app.register(name, controller);
};

registerLazyController(
    'datatables-grid',
    '@freddut/data-grid-bundle/datatables-grid',
);

registerLazyController(
    'tabulator-grid',
    '@freddut/data-grid-bundle/tabulator-grid',
);
export default app;


controllers.json
{
    "controllers": {
        "@freddut/data-grid-bundle": {
            "native-grid": {
                "enabled": false
            },
            "datatables-grid": {
                "enabled": false
            },
            "tabulator-grid": {
                "enabled": false
            }
        }
    },
    "entrypoints": []
}

php bin/console cache:clear
php bin/console asset-map:compile