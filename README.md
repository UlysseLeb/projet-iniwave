# Intégration CRM → ERP (HubSpot → n8n → Odoo)

Pipeline d'automatisation qui synchronise automatiquement un CRM et un ERP : dès qu'une vente est conclue côté commercial, le client, la commande puis la facture sont créés côté gestion, sans ressaisie manuelle. C'est le même schéma d'intégration que l'on retrouve derrière n'importe quel connecteur CRM/ERP d'entreprise (Salesforce → NetSuite, HubSpot → SAP, etc.), construit et testé de bout en bout avec des outils accessibles.

## Le cas d'usage

Une entreprise type gère ses ventes dans un CRM et sa comptabilité/logistique dans un ERP séparé. Sans automatisation, chaque vente conclue implique une double saisie manuelle (le commercial dans le CRM, l'administratif dans l'ERP), source d'erreurs et de délais. Ce projet élimine cette étape : la détection d'une vente conclue déclenche automatiquement la création de la fiche client, de la commande puis de la facture, avec une protection anti-doublon et une alerte automatique en cas d'échec.

## Pourquoi Odoo plutôt que Salesforce/NetSuite

HubSpot et Odoo sont des outils accessibles (gratuits ou open source), utilisés ici comme représentants d'un CRM et d'un ERP d'entreprise. Le pattern d'intégration (authentification API, mapping de champs, idempotence, gestion d'erreurs) est strictement identique à celui qu'on retrouverait avec des outils propriétaires comme Salesforce ou NetSuite : seuls les endpoints et le format d'authentification changent, pas la logique.

## Architecture

```
HubSpot (CRM)
   │  polling toutes les 5 minutes
   ▼
n8n (auto-hébergé, Docker + Traefik + HTTPS, VPS)
   │  1. Recherche des deals "Closed Won" (HubSpot, API REST)
   │  2. Garde-fou : aucun résultat → arrêt propre (évite de traiter le mauvais enregistrement)
   │  3. Récupération du détail du deal + du contact associé
   │  4. Authentification Odoo (JSON-RPC)
   │  5. Anti-doublon commande (recherche par référence externe = ID du deal)
   │  6. Anti-doublon / réutilisation client (recherche par email)
   │  7. Création du client si nouveau
   │  8. Création de la commande (sale.order)
   │  9. Confirmation de la commande
   │ 10. Création de la facture (account.move)
   │ 11. Vérification indépendante par requête SQL directe sur la base (Postgres)
   ▼
Odoo (ERP, self-hébergé, Docker, Postgres)
   Client (res.partner) → Commande (sale.order) → Facture (account.move)

En parallèle : workflow d'erreurs dédié (Error Trigger), qui surveille chaque étape
ci-dessus et notifie automatiquement un canal Slack en cas d'échec, avec le nœud
fautif et un lien direct vers l'exécution concernée.
```

## Pourquoi une requête SQL directe sur la base (dernière étape du flux)

Le dernier nœud du workflow interroge directement la base Postgres d'Odoo, en dehors de son API, pour deux raisons :

1. **Démontrer une compétence complémentaire** — savoir parler directement à une base de données, pas seulement consommer une API REST/JSON-RPC.
2. **Vérifier indépendamment ce qui a été créé** — l'API répond « facture créée avec succès » en parlant d'elle-même ; la requête SQL relit directement la table pour confirmer que la facture existe bien, avec le bon client rattaché. Un contrôle qui ne dépend pas de ce que le système affirme sur lui-même.

## Défis techniques réels rencontrés

Le pipeline a été testé en conditions réelles (création de vrais deals HubSpot, exécutions réelles observées de bout en bout), ce qui a fait remonter plusieurs bugs de production, pas seulement des cas d'école :

- **Échec silencieux sur recherche vide** : quand aucun deal "Closed Won" ne correspondait, l'étape suivante continuait avec un identifiant vide, ce que l'API interprétait comme une demande différente (liste générale au lieu d'un enregistrement précis) — le workflow traitait alors un enregistrement au hasard comme s'il était valide. Corrigé par un garde-fou qui force un arrêt propre et explicite plutôt qu'un comportement indéfini.
- **Anti-doublon à deux niveaux** : un deal ne doit jamais générer deux commandes, et un client ne doit jamais être dupliqué s'il existe déjà (recherche par référence externe et par email avant toute création).
- **Idempotence des identifiants** : la référence externe utilisée pour la déduplication est l'identifiant du deal côté CRM, pas un identifiant interne, pour rester stable même si le workflow est rejoué.

## Ce que chaque brique démontre

| Compétence | Élément du projet |
|---|---|
| Conception d'architectures de flux CRM → ERP | Schéma HubSpot → n8n → Odoo, table de correspondance ci-dessous |
| Workflows n8n complexes (branches, fusions, conditions) | Anti-doublon commande/client, fusion des branches client neuf/existant |
| Détection d'événements et polling | Recherche périodique des deals conclus côté CRM |
| Gestion des erreurs & monitoring | Workflow d'erreurs séparé, alerte Slack automatique, garde-fou sur les cas limites |
| Authentification API & sécurité | Authentification par clé API (Odoo) et par token (HubSpot), secrets exclus du dépôt |
| REST et RPC | API REST HubSpot, API JSON-RPC Odoo (deux paradigmes différents dans un même projet) |
| SQL (PostgreSQL) | Requête directe sur la base Odoo pour vérification indépendante |
| Connaissances CRM/ERP | Table de correspondance Odoo/NetSuite, modèle de données client-commande-facture |
| Docker & déploiement | n8n et Odoo auto-hébergés en Docker sur VPS, HTTPS via Traefik |
| Débogage en conditions réelles | Bugs de production identifiés et corrigés à partir de vraies exécutions, pas de cas simulés |

## Correspondance Odoo ↔ NetSuite

| Concept métier | Odoo | NetSuite (équivalent) |
|---|---|---|
| Fiche client | `res.partner` | Customer record |
| Commande client | `sale.order` | Sales Order |
| Facture | `account.move` | Invoice |
| Ligne de commande | `sale.order.line` | Sales Order Line |
| Champ personnalisé | Custom field (Studio/XML) | Custom field (record customization) |
| API | JSON-RPC / XML-RPC | SuiteTalk REST / SOAP |
| Auth | Clé API / OAuth2 (module) | OAuth 2.0 / Token-based |

## Démarrer l'environnement local

```bash
cp .env.example .env
docker compose up -d
# Odoo : http://localhost:8069
```

## Dossier

- `docker-compose.yml` — stack Odoo + Postgres
- `docs/mapping-netsuite-odoo.md` — détail de la correspondance des objets
- `docs/schema-infra.html` — schéma visuel de l'infrastructure
- `docs/demo-script.md` — script de démonstration détaillé du pipeline, étape par étape
- `docs/quiz-comprehension.html` — quiz d'auto-évaluation sur l'architecture (22 questions)
