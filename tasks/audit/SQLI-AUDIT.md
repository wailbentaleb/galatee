# Audit SQLi — Galatee

- Date : 2026-10-04
- Branche : `security/sqli-audit`
- Révision de départ : `d8dacbc7b9338a104d58b77c9239c805f8700538`
- Périmètre : injections SQL uniquement
- État : étapes 1 et 2 terminées ; aucune correction applicative effectuée

## Écart de contexte important

Le fork audité `wailbentaleb/galatee` utilise actuellement `node:sqlite` et ne contient ni dépendance `pg`, ni adaptateur PostgreSQL, ni appel direct à PostgreSQL. Les placeholders `?` sont donc ceux attendus par SQLite. Une future branche PostgreSQL devra faire l'objet d'une revue séparée de son adaptateur et de la traduction vers `$1`, `$2`, etc.

## Étape 1 — Cartographie

Fichiers applicatifs contenant des requêtes ou du DDL :

- `backend/analyticsSystem.js`
- `backend/clubSystem.js`
- `backend/customerAuthSystem.js`
- `backend/driverSystem.js`
- `backend/loyaltySystem.js`
- `backend/menuSystem.js`
- `backend/orderSystem.js`
- `backend/reservationSystem.js`
- `backend/seedDemo.js`
- `backend/server.js`

Les tests et scripts locaux ont aussi été recherchés. Ils ne sont pas exposés comme routes HTTP et utilisent des paramètres ou des constantes internes.

## Étape 2 — Revue statique

Verdict de l'étape : aucune requête actuellement atteignable depuis une entrée utilisateur ne concatène cette entrée dans le SQL. Les données issues de `body`, `params` et `query` sont liées avec `?` ou avec les paramètres nommés SQLite.

| Fichier | Fonction / lignes | Risque | Pourquoi |
| --- | --- | --- | --- |
| `analyticsSystem.js` | `recordEvent`, `upsertRevenue`, `listRevenue`, lignes 62–115 | Faible | Toutes les valeurs sont transmises avec des placeholders `?`. |
| `analyticsSystem.js` | `getDashboard`, `getSiteStats`, lignes 120–222 | Faible | `groupBy` est contrôlé par `GROUP_BYS` avant de sélectionner une expression SQL parmi quatre constantes. Les dates restent paramétrées. |
| `analyticsSystem.js` | `queryOrderAnalytics`, `queryTraffic`, lignes 288–369 | Faible | Les expressions de période et la colonne de date viennent de constantes internes ; les intervalles utilisent `?`. |
| `analyticsSystem.js` | `queryProductPerformance`, lignes 332–348 | Faible | La clause `IN` contient seulement des placeholders générés depuis une liste constante ; les statuts sont passés séparément. |
| `analyticsSystem.js` | `queryBusiestSlots`, lignes 394–405 | Faible | `ORDER BY` et `LIMIT 8` sont fixes ; les dates sont paramétrées. |
| `clubSystem.js` | `getContent`, lignes 60–66 | Faible | Le fragment conditionnel est choisi entre deux chaînes codées en dur selon un booléen. |
| `clubSystem.js` | mutations Club, lignes 70–105 | Faible | Les contenus et identifiants sont tous liés avec `?`. |
| `customerAuthSystem.js` | requêtes comptes, codes, resets et sessions, lignes 128–545 | Faible | Aucun fragment SQL dynamique ; toutes les entrées email, identifiant, token et profil utilisent des placeholders. |
| `driverSystem.js` | CRUD, sessions et commandes, lignes 67–475 | Faible | Les entrées de route sont liées avec `?`. Le filtre `includeInactive` choisit uniquement une clause fixe. |
| `driverSystem.js` | `getStats`, lignes 476–518 | Faible | Le nom de colonne alterne uniquement entre `updated_at` et `delivered_at`; `status` reste paramétré et les appels internes utilisent seulement `delivered` ou `cancelled`. |
| `driverSystem.js` | abonnements push, lignes 561–580 | Faible | Endpoint, clés et identifiants sont paramétrés. |
| `loyaltySystem.js` | progression et récompenses, lignes 136–274 | Faible | La clause `IN` est construite avec des placeholders pour une liste constante ; identifiants et montants sont liés. |
| `loyaltySystem.js` | `listCustomerProgress`, lignes 277–291 | Faible | La recherche `LIKE` ajoute `%` à la valeur, puis transmet le motif comme paramètre. Le fragment `WHERE` est une constante. |
| `menuSystem.js` | `migrateLegacyItems`, lignes 163–185 | Faible | Le nombre de placeholders `IN` est généré depuis une liste interne ; les identifiants restent liés. |
| `menuSystem.js` | `listPublished`, lignes 200–230 | Faible | La catégorie issue de la requête HTTP est normalisée puis liée avec `?`; seules des clauses fixes sont assemblées. |
| `menuSystem.js` | `listAdmin`, lignes 233–265 | Faible | Le fragment de statut est choisi entre deux chaînes fixes selon un booléen. L'`ORDER BY` est fixe. |
| `menuSystem.js` | créations et mises à jour, lignes 274–420 | Faible | Toutes les valeurs métier sont liées avec des placeholders. |
| `orderSystem.js` | communes, lignes 150–203 | Faible | Le filtre actif choisit une clause fixe ; identifiants, noms et frais sont paramétrés. |
| `orderSystem.js` | `createOrder`, lignes 208–272 | Faible | Les valeurs client et les montants recalculés sont tous transmis avec `?`. |
| `orderSystem.js` | `resolveItems`, lignes 275–329 | Faible | Seul le nombre de placeholders `IN` dépend du nombre d'articles ; les identifiants eux-mêmes sont liés séparément. |
| `orderSystem.js` | `listOrders`, lignes 340–353 | Faible | Les clauses assemblées sont codées en dur ; statut, client et dates restent dans `params`. L'`ORDER BY` est fixe. |
| `reservationSystem.js` | `ensureColumn`, lignes 262–270 | Moyen, non exploitable actuellement | Les noms de table et les définitions DDL sont interpolés et ne peuvent pas être paramétrés. Tous les appels actuels utilisent toutefois des constantes de démarrage, jamais une entrée HTTP. Une whitelist sera nécessaire avant toute réutilisation. |
| `reservationSystem.js` | tables et attribution, lignes 624–698 | Faible | Les clauses optionnelles sont fixes ; toutes les valeurs de réservation utilisent `?`. |
| `reservationSystem.js` | `listReservations`, lignes 701–722 | Faible | Les clauses sont internes et les valeurs utilisent les paramètres nommés `$date`, `$status`, `$customerId`. |
| `reservationSystem.js` | compteurs, lignes 1046–1074 | Faible | Les clauses `IN` et d'exclusion ne contiennent que des placeholders ; les valeurs sont séparées. |
| `seedDemo.js` | génération et nettoyage démo, lignes 194–264 | Faible | Script local non exposé ; les motifs `LIKE` sont liés avec `?`. |
| `server.js` | readiness, ligne 118 | Faible | Requête statique `SELECT 1 AS ok`, sans entrée utilisateur. |

## Cas particuliers vérifiés

- `ORDER BY` : aucune colonne reçue depuis HTTP ; tous les tris sont fixes.
- Noms de table ou de colonne : seul `ensureColumn` les interpole, avec des constantes internes au démarrage.
- `LIMIT` / `OFFSET` : limites fixes (`8`, `500`) ; aucun offset utilisateur.
- `IN (...)` : seuls les placeholders sont générés dynamiquement, jamais les valeurs.
- `LIKE` : les caractères `%` sont ajoutés côté JavaScript, puis le motif est lié comme valeur.
- Templates SQL : aucun `${req.body}`, `${req.params}`, `${req.query}`, header ou valeur dérivée équivalente n'a été trouvé.

## Classement SQLi provisoire

1. **Urgent / Haut** : aucun point confirmé.
2. **Moyen exploitable** : aucun point confirmé.
3. **Moyen latent** : `reservationSystem.ensureColumn`, à verrouiller par whitelist avant toute réutilisation avec des données variables.
4. **Faible / fiable** : toutes les requêtes métier atteignables utilisent des paramètres ou des fragments SQL fixes contrôlés par le code.

## Limites de cette étape

- Revue statique uniquement : aucun payload SQLi n'a encore été envoyé.
- Aucun test contre un serveur distant ou tiers.
- La branche PostgreSQL décrite dans le contexte n'est pas présente sur ce fork et n'est donc pas couverte par ce verdict.

Hors périmètre, en une ligne : le fork audité ne contient pas le runtime PostgreSQL annoncé ; il faudra synchroniser ou fournir la branche correspondante avant de pouvoir l'évaluer.
