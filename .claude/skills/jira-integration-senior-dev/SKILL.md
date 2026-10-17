---
name: jira-integration-senior-dev
description: Agir en développeur senior spécialisé dans l'intégration de Jira (REST API v2/v3, Jira Cloud) à un outil tiers, en particulier un outil de visualisation de roadmap. À utiliser dès que l'utilisateur parle de connecter, synchroniser, importer ou exporter des données Jira (epics, issues, JQL, versions, sprints, champs custom, webhooks, API token, OAuth, rate limit, CORS, pagination, CSV Jira), d'une roadmap alimentée par Jira, ou colle une URL developer.atlassian.com / un message d'erreur Jira API (401, 403, 410, 429). Utiliser aussi pour revoir, durcir ou migrer un code d'intégration Jira existant, même si l'utilisateur ne dit pas explicitement "API".
---

# Intégration Jira — posture de développeur senior

Ce skill décrit comment concevoir, coder et fiabiliser une intégration entre Jira et un outil de visualisation de roadmap. Les faits marqués **[vérifié]** ont été contrôlés dans la documentation Atlassian ou des sources Atlassian le 17/10/2026. Les faits marqués **[à confirmer]** viennent de sources secondaires ou de connaissances générales : les revérifier avant de les présenter comme certains ou de les figer dans du code de production.

## 1. Posture

- **Vérifier avant de coder.** L'API Jira Cloud change régulièrement (endpoints retirés, quotas, expiration des tokens). En cas de doute sur un endpoint, un paramètre ou une limite, consulter la doc officielle (WebFetch/WebSearch) plutôt que se fier à la mémoire. Si une information ne peut pas être vérifiée, le dire.
- **Qualifier avant d'écrire du code** (section 2). Une mauvaise hypothèse sur le type d'instance ou l'authentification invalide toute l'architecture.
- **Contredire quand c'est nécessaire.** Si la demande suppose quelque chose de faux ou de risqué (ex. appeler Jira directement depuis le navigateur avec un API token), le dire explicitement, expliquer pourquoi, proposer l'alternative.
- **Moindre privilège et lecture seule par défaut.** Une roadmap de consultation n'a pas besoin d'écrire dans Jira.
- **Petit d'abord.** Un MVP qui marche avec des exports CSV vaut mieux qu'une architecture API complète non validée. Concevoir pour que le passage CSV → API ne casse rien (section 7).
- **Adapter le niveau d'explication** à l'utilisateur : si la personne se dit novice en technique, expliquer chaque terme (token, CORS, proxy, webhook) en une phrase, éviter le jargon non défini.

## 2. Questions de qualification (à poser ou à vérifier avant de concevoir)

Ne poser que celles dont la réponse n'est pas déjà connue, groupées en une seule fois.

1. **Jira Cloud ou Data Center/Server ?** (URL en `*.atlassian.net` = Cloud.) Les API diffèrent : endpoints de recherche, authentification (PAT côté Data Center), rate limiting. Le reste de ce skill vaut pour **Cloud**.
2. **Qui héberge l'outil de roadmap** et qui administre Jira ? Un admin Jira/IT doit en général créer le compte de service, le token ou l'app OAuth.
3. **Quelles données ?** Projets, types d'issues (Epic ? Initiative ?), statuts, dates, versions, champs custom précis.
4. **Lecture seule ou écriture ?** (L'écriture ajoute des risques : permissions, rate limit par issue, conflits de modification.)
5. **Fraîcheur attendue** : export manuel, mise à jour quotidienne, quasi temps réel ?
6. **Contraintes de données** : le client autorise-t-il l'appel API ? Des données peuvent-elles transiter par un service tiers ? (Ne jamais supposer.)

## 3. Faits à jour sur l'API Jira Cloud

### 3.1 Versions et recherche

- La doc v2 et la doc v3 coexistent. **[vérifié]** Différence principale connue : en v2 les champs texte riches (description, commentaires) sont du texte/wiki markup, en v3 du format ADF (Atlassian Document Format) **[à confirmer, non revérifié ici]**. Pour une roadmap qui ne lit que titre, statut, dates et parent, v2 suffit.
- **Les anciens endpoints de recherche sont supprimés** **[vérifié]** : `GET/POST /rest/api/{2|3}/search`, `POST /search/id` et `POST /expression/eval` (annonce du 31/10/2024, retrait progressif en 2025, réponses 410 Gone). Remplacement : **`GET` ou `POST /rest/api/2/search/jql`** (ou `/3/`).
- Comportement de `/search/jql` **[vérifié]** :
  - Pagination **par jeton** : réponse avec `issues`, `nextPageToken` (absent sur la dernière page), `isLast`. **Plus de `startAt`, plus de `total`.**
  - `fields` : **par défaut `id` uniquement** (différent de `GET /issue/{key}` qui renvoie tout). Demander explicitement `fields=summary,status,parent,...`.
  - `maxResults` : défaut 50 ; la doc annonce un maximum de 5000 (pour `id`/`key` seuls), mais le nombre réel baisse quand beaucoup de champs sont demandés et des retours terrain signalent un plafond à 100. **Ne jamais supposer la taille de page : boucler sur `nextPageToken`.**
  - La JQL doit être **bornée** (au moins une restriction, ex. `project = X`) ; `ORDER BY` : 7 champs maximum.
  - Cohérence **à terme** : une mise à jour récente peut ne pas apparaître tout de suite. `reconcileIssues` (jusqu'à 50 IDs) donne une cohérence forte quand c'est nécessaire.
  - Compter : `POST /rest/api/{2|3}/search/approximate-count` (estimation, pas un total exact).
  - Pour une JQL longue, utiliser `POST` avec le JSON `{ "jql", "fields": [], "maxResults", "nextPageToken" }` plutôt que `GET`.

### 3.2 Hiérarchie parent/enfant (piège classique pour une roadmap)

**[vérifié]** Atlassian a déprécié les champs **Epic Link** (`customfield_10014` sur de nombreuses instances), **Parent Link** (`customfield_10018`) et `epic` dans les REST API et webhooks pour les projets company-managed. **Utiliser le champ `parent`** (Epic → Story → Sub-task, et niveaux au-dessus d'Epic selon la configuration). La JQL ne change pas : `parent = KEY` renvoie les enfants. Les IDs `customfield_*` varient d'une instance à l'autre : ne jamais les coder en dur sans les découvrir via `GET /rest/api/2/field`.

Dans l'historique (changelog), le lien parent apparaît sous le nom de champ `IssueParentAssociation` (`fieldtype: jira`).

### 3.3 Authentification

| Méthode | Usage | Points d'attention |
|---|---|---|
| **API token** (Basic `email:token`) | Scripts et backends côté serveur | **[vérifié]** Expiration **1 an maximum**, à planifier (rotation). Préférer un **compte de service** (Admin Hub → Directory → Service accounts) plutôt qu'un compte personnel. Un token **scopé** s'utilise via `https://api.atlassian.com/ex/jira/{cloudId}/...` ; un token non scopé via `https://{site}.atlassian.net/...`. |
| *(note)* | Tokens non scopés | **[à confirmer]** Des signalements communautaires (2025-2026) évoquent une dépréciation des tokens sans scope, et la documentation tierce recommande les tokens scopés pour toute nouvelle intégration. Pour une nouvelle intégration, **privilégier un token scopé** (lecture seule : scopes `read:jira-work` au minimum, à ajuster selon les endpoints appelés) et revérifier la politique actuelle. |
| **OAuth 2.0 (3LO)** | Appli qui agit au nom d'un utilisateur, ou appel depuis le navigateur | **[vérifié]** Les appels passent par `https://api.atlassian.com/ex/jira/{cloudId}/{api}`. Le CORS est autorisé sur `api.atlassian.com` avec ce flux. Nécessite une app enregistrée côté Atlassian et un consentement. |
| **OAuth service account (client_credentials)** | Serveur à serveur sans utilisateur | **[à confirmer]** Mentionné sur le forum communauté Atlassian (échange sur `auth.atlassian.com/oauth/token`) ; vérifier dans la doc Admin Hub avant de s'y engager. |
| **Forge / Connect** | Apps installées dans Jira | Hors périmètre sauf demande explicite. Les webhooks dynamiques (4.4) exigent ce type d'app. |

**CORS — décision d'architecture n°1 [vérifié].** Jira Cloud **ne supporte pas le CORS** sur `https://{site}.atlassian.net` (appel navigateur avec API token = bloqué). Mettre un API token dans du JavaScript côté navigateur serait de toute façon une fuite de secret : toute personne qui ouvre la page le lit. Donc, pour un outil web (HTML/JS) :
- **Option A (recommandée)** : un petit **backend/proxy** (Node, Python, fonction serverless) détient le token, appelle Jira, renvoie au front des données **déjà normalisées et filtrées**.
- **Option B** : OAuth 2.0 (3LO) depuis le navigateur vers `api.atlassian.com`. Plus lourd (app OAuth, gestion des tokens, consentement par utilisateur), pertinent seulement si chaque utilisateur doit voir ce que *son* compte Jira lui permet de voir.
- **Option C** : pas d'appel API du tout (exports CSV Jira déposés manuellement). Valide pour un MVP.

### 3.4 Rate limiting

**[vérifié]** (doc Atlassian « Rate limiting », mise à jour le 16/10/2026). Trois systèmes indépendants renvoient tous `429 Too Many Requests` :

1. **Quota par points, horaire** : 1 point de base par requête + points par objet lu (issue = 1, utilisateur/groupe = 2) ; écritures = 1 point. Pool global par défaut : 65 000 points/heure. **S'applique aux apps Forge, Connect et OAuth 2.0 (3LO). Le trafic par API token n'est pas concerné par ce quota** et reste soumis aux limites de rafale.
2. **Rafale par seconde**, par tenant et par endpoint (token bucket). Valeurs par défaut : GET/POST 100 req/s, PUT/DELETE 50 req/s ; certains endpoints ont des valeurs propres.
3. **Écriture par issue** : 20 écritures/2 s et 100 écritures/30 s sur une même issue.

En-têtes de réponse 429 : `Retry-After` (secondes), `X-RateLimit-Reset` (ISO 8601), `RateLimit-Reason` ∈ {`jira-quota-global-based`, `jira-quota-tenant-based`, `jira-burst-based`, `jira-per-issue-on-write`}. Certains 503 transitoires portent aussi `Retry-After`.

Réaction : respecter `Retry-After` ; backoff exponentiel avec jitter (×0,7 à ×1,3) ; nombre de tentatives borné (ex. 4) ; en cas de quota horaire dépassé, **mettre en pause toutes les requêtes** jusqu'à la réinitialisation (début de l'heure UTC). **Ne jamais tester les limites sur un tenant réel** (interdit par la politique d'usage).

### 3.5 Webhooks

- **[vérifié]** Les webhooks **dynamiques** (`POST /rest/api/2/webhook`) ne sont utilisables que par des apps **Connect ou OAuth 2.0**, **expirent au bout de 30 jours** (à prolonger avec `PUT /rest/api/2/webhook/refresh`), acceptent un filtre JQL pour les événements `jira:issue_created/updated/deleted`, et `GET /webhook/failed` liste les échecs de livraison.
- **[à confirmer]** Les webhooks « système » (Admin UI → System → Webhooks) n'expirent pas mais demandent un droit admin ; signature `X-Hub-Signature` (HMAC-SHA256) uniquement pour les webhooks dynamiques avec secret.
- Règle de conception : **un webhook n'est pas un journal fiable**. Toujours prévoir une réconciliation périodique par JQL (`updated >= -1d`).
- Pour un outil de roadmap consultée quelques fois par jour, **une synchro planifiée (polling) suffit presque toujours** et évite l'exposition d'un endpoint public.

## 4. Récupérer les données utiles à une roadmap

### 4.1 Découvrir avant de coder

```
GET /rest/api/2/field                      # id, name, custom, schema de chaque champ
GET /rest/api/2/project/{key}/versions     # versions et releaseDate
GET /rest/api/2/issuetype                  # types disponibles (Epic, etc.)
GET /rest/api/2/status                     # statuts + statusCategory
```

Identifier par leur **nom** puis fixer l'ID découvert : « Start date », « Target start », « Target end », champs de dates métier. Les dates métier (ex. date de release prod, début de recette/UAT, « short description » d'un Epic) **n'existent pas nativement** : déterminer si ce sont des champs custom, des versions (`fixVersions`), des labels ou des composants. **Ne jamais deviner**, demander et montrer ce qui a été trouvé.

### 4.2 Requête type (epics d'un périmètre)

```
jql:    project in (ABC, DEF) AND issuetype = Epic ORDER BY created ASC
fields: summary,status,parent,fixVersions,duedate,labels,updated,<ids des champs de dates>
```

Pour les enfants d'un epic : `parent = ABC-123`. Pour limiter les volumes : `AND updated >= -7d` en synchro incrémentale.

### 4.3 Historique des dates (dérive de planning)

Utile pour montrer qu'une date prévisionnelle a bougé : `GET /rest/api/2/issue/{key}?expand=changelog` ou `GET /rest/api/2/issue/{key}/changelog` (paginé). Plus coûteux : à réserver aux issues affichées en détail, pas à toute la liste.

## 5. Code de référence (Node 18+, fetch natif)

À adapter au contexte ; garder le token côté serveur, jamais dans un fichier front ou un dépôt.

```js
// jira-client.js — lecture seule, pagination par jeton, retry sur 429/503
const BASE = process.env.JIRA_BASE_URL;            // https://<site>.atlassian.net
const AUTH = 'Basic ' + Buffer.from(
  `${process.env.JIRA_EMAIL}:${process.env.JIRA_API_TOKEN}`).toString('base64');

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function jiraFetch(path, init = {}, attempt = 0) {
  const res = await fetch(BASE + path, {
    ...init,
    headers: { Authorization: AUTH, Accept: 'application/json',
               'Content-Type': 'application/json', ...init.headers },
  });
  if ((res.status === 429 || res.status === 503) && attempt < 4) {
    const retryAfter = Number(res.headers.get('Retry-After'));
    const base = retryAfter > 0 ? retryAfter * 1000 : Math.min(2000 * 2 ** attempt, 30000);
    await sleep(base * (0.7 + Math.random() * 0.6));    // jitter 0.7–1.3
    return jiraFetch(path, init, attempt + 1);
  }
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Jira ${res.status} ${path}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// Toutes les issues d'une JQL, page par page (pas de startAt/total).
export async function searchAll({ jql, fields, maxResults = 100 }) {
  const issues = [];
  let nextPageToken;
  do {
    const data = await jiraFetch('/rest/api/2/search/jql', {
      method: 'POST',
      body: JSON.stringify({ jql, fields, maxResults, nextPageToken }),
    });
    issues.push(...data.issues);
    nextPageToken = data.nextPageToken;     // absent => dernière page
  } while (nextPageToken);
  return issues;
}

export const getFields = () => jiraFetch('/rest/api/2/field');
```

### Normalisation vers un modèle neutre

Le front ne doit **jamais** connaître la forme brute de Jira.

```js
// Modèle interne stable, indépendant de Jira
// RoadmapItem { key, title, status, statusCategory, parentKey, start, end,
//               release, labels, updated, url, source: 'jira'|'csv'|'manual' }

export function toRoadmapItem(issue, cfg) {
  const f = issue.fields;
  return {
    key: issue.key,
    title: f.summary,
    status: f.status?.name,
    statusCategory: f.status?.statusCategory?.key,       // new | indeterminate | done
    parentKey: f.parent?.key ?? null,                    // pas Epic Link
    start: f[cfg.startFieldId] ?? null,                  // ID découvert via /field
    end: f[cfg.endFieldId] ?? f.duedate ?? null,
    release: f.fixVersions?.[0]?.releaseDate ?? null,
    labels: f.labels ?? [],
    updated: f.updated,
    url: `${BASE}/browse/${issue.key}`,
    source: 'jira',
  };
}
```

## 6. Pièges fréquents (à vérifier systématiquement en revue de code)

1. Utiliser `/search` (retiré) ou `startAt`/`total` sur `/search/jql` → erreurs 410 ou boucles infinies.
2. Oublier `fields` → la réponse ne contient que des IDs.
3. Lire `customfield_10014` pour l'epic → utiliser `parent`.
4. Supposer qu'un `customfield_XXXXX` est le même sur toutes les instances.
5. Dates : champs date sans heure vs datetime avec fuseau ; ne pas convertir en UTC par réflexe, sinon décalage d'un jour. Les dates Jira sont `YYYY-MM-DD` ou ISO 8601 avec offset.
6. Pagination faite en parallèle sur un seul jeton : impossible, chaque page dépend de la précédente ; paralléliser par **projet**, pas par page, et rester sous les limites de rafale.
7. Un token personnel dans un dépôt, un fichier `.env` commité, ou un front : révoquer immédiatement.
8. Token expiré après 1 an sans alerte → prévoir une date de rotation et un message d'erreur clair sur `401`.
9. `403` : permission de projet insuffisante pour le compte utilisé, ce qui est différent d'un `401` (identifiants invalides). Le diagnostic n'est pas le même.
10. Issues archivées / projets restreints : invisibles pour le compte de service ; la roadmap paraîtra « incomplète » sans erreur. Documenter le périmètre visible.
11. Cohérence à terme de la recherche : une issue modifiée à l'instant peut manquer ; ne pas en faire un bug de l'outil, utiliser `reconcileIssues` si nécessaire.
12. Données personnelles (assignee, reporter) : ne les récupérer que si l'usage le justifie. Les objets utilisateur coûtent aussi plus cher en quota.

## 7. Architecture recommandée pour un outil de roadmap

```
Jira ──(CSV manuel  OU  API serveur)──► Adaptateur ──► RoadmapItem[] ──► Stockage/cache ──► Front
                                          ▲                                 (JSON/SQLite)
                                          └── même interface, deux implémentations
```

- **Interface d'adaptateur unique** : `load(): Promise<RoadmapItem[]>`. Implémentations : `CsvAdapter` (MVP), `JiraApiAdapter` (V2). Le front ne change pas quand la source change.
- **CSV Jira** : les colonnes dépendent de l'export (noms localisés, colonnes répétées pour les labels/sprints, séparateur selon la locale). Mapper les colonnes par **nom d'en-tête** configurable, valider la présence des colonnes obligatoires et afficher une erreur lisible sinon.
- **Données propres à l'outil** (éléments que Jira ne contient pas, ajoutés par l'utilisateur en V2) : les stocker **à part**, avec `source: 'manual'` et la clé Jira éventuelle. Ne jamais écraser une donnée Jira par une donnée manuelle sans conflit explicite visible, et ne pas réécrire dans Jira sans décision explicite.
- **Cache** : l'API ne doit pas être appelée à chaque affichage. Synchro planifiée (ex. toutes les heures ou une fois par jour) → fichier JSON/DB → le front lit le cache. Afficher « données Jira du <date/heure> ».
- **Synchro incrémentale** : première charge complète, puis `updated >= <dernier succès - marge>`. Gérer les **suppressions** (une issue supprimée ou déplacée disparaît de la JQL) avec une resynchronisation complète périodique.
- **Observabilité minimale** : journaliser nombre d'issues lues, durée, nombre de 429, date du dernier succès. Alerter quand la synchro échoue depuis N heures.
- **Hébergement par un tiers (IT du client)** : prévoir l'ouverture réseau (sortant vers `*.atlassian.net`), la gestion des secrets (coffre de secrets, pas de fichier), le compte de service, et la revue de sécurité. Documenter cela dans un court dossier de passation (diagramme, variables d'environnement, procédure de rotation du token, périmètre de données).

## 8. Méthode de travail pour chaque demande

1. Reformuler l'objectif en une phrase et lister les hypothèses non vérifiées.
2. Qualifier (section 2), puis annoncer le chemin choisi (CSV, proxy + token, OAuth) **avec ses limites**.
3. Découvrir les champs réels (section 4.1) avant d'écrire du mapping.
4. Écrire le code dans des fichiers (pas en bloc dans le chat dès qu'il dépasse quelques lignes), avec secrets en variables d'environnement et un exemple `.env.example` sans valeur réelle.
5. Tester sur des **fixtures** (JSON/CSV d'exemple anonymisés) avant tout appel réel ; premier appel réel en lecture seule sur un périmètre restreint.
6. Fournir : ce qui fonctionne, ce qui n'a pas été testé, et les points à **faire confirmer** par l'admin Jira (permissions, champs, quotas).

## 9. Sources à consulter pour revérifier

- Introduction REST API v2 / v3 : https://developer.atlassian.com/cloud/jira/platform/rest/v2/ et /v3/
- Recherche JQL (enhanced) : https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/
- Rate limiting : https://developer.atlassian.com/cloud/jira/platform/rate-limiting
- Webhooks : https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-webhooks/
- OAuth 2.0 (3LO) : https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/
- Basic auth / API tokens : https://developer.atlassian.com/cloud/jira/platform/basic-auth-for-rest-apis/ et https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/
- Dépréciation Epic Link / Parent Link : https://community.developer.atlassian.com/t/deprecation-of-the-epic-link-parent-link-and-other-related-fields-in-rest-apis-and-webhooks/54048
- Changelog développeur Atlassian : https://developer.atlassian.com/cloud/jira/platform/changelog/

En cas de divergence entre ce skill et la documentation en ligne, **la documentation en ligne prévaut** ; le signaler à l'utilisateur et proposer de mettre le skill à jour.
