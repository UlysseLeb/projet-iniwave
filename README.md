# Projet Iniwave — Démo CRM → ERP

Projet construit pour l'entretien présentiel (2 personnes de l'équipe technique / fondateurs) chez **Iniwave**, cabinet d'intégration NetSuite. Objectif : montrer une démo live d'automatisation qui reproduit exactement leur cas d'usage cité en exemple dans la fiche de poste (*"Salesforce vers Netsuite"*), avec des outils réellement construits par moi, pas juste racontés.

## Pourquoi Odoo à la place de NetSuite

NetSuite est un logiciel d'entreprise vendu via un processus commercial (pas de self-service, pas d'essai gratuit accessible en quelques semaines sans budget). **Odoo** est utilisé comme ERP de démonstration : open source, auto-hébergeable, avec les mêmes objets métier qu'un ERP classique (clients, commandes, factures, champs personnalisés). Le principe d'intégration (API REST, authentification, mapping de champs, idempotence) est strictement identique — le code est transférable à NetSuite avec un changement d'endpoints, pas de logique.

**À dire clairement en entretien** (rigueur > bluff) : *"J'ai reproduit l'architecture exacte que vous décrivez, avec un ERP open-source accessible plutôt que NetSuite qui nécessite une licence entreprise. Le pattern d'intégration est identique."*

## Architecture

```
HubSpot (CRM, gratuit)
   │  webhook : deal passé en "Closed Won"
   ▼
n8n (auto-hébergé, Docker)
   │  1. Webhook Trigger (reçoit le deal)
   │  2. Code node (JS) : mapping des champs HubSpot → format Odoo
   │  3. HTTP Request : vérifier si le client existe déjà dans Odoo (idempotence)
   │  4. Switch : client existant ? sinon → créer
   │  5. HTTP Request : créer/mettre à jour la commande (sale.order) dans Odoo
   │  6. Postgres node : requête SQL directe sur la base Odoo (vérification/reporting)
   │  7. Error Trigger : notification (email/Slack) si une étape échoue
   ▼
Odoo (self-hébergé, Docker, Postgres)
   Client (res.partner) → Commande (sale.order) → Facture (account.move)
```

## Correspondance Odoo ↔ NetSuite (à connaître par cœur pour l'entretien)

| Concept métier | Odoo | NetSuite (équivalent) |
|---|---|---|
| Fiche client | `res.partner` | Customer record |
| Commande client | `sale.order` | Sales Order |
| Facture | `account.move` | Invoice |
| Ligne de commande | `sale.order.line` | Sales Order Line |
| Champ personnalisé | Custom field (Studio/XML) | Custom field (record customization) |
| API | JSON-RPC / XML-RPC ou REST (module) | SuiteTalk REST / SOAP |
| Auth | Clé API / OAuth2 (module) | OAuth 2.0 / Token-based |

## Ce que chaque brique démontre (mapping direct avec la fiche de poste)

| Point de la fiche de poste | Élément du projet |
|---|---|
| Conception d'architectures de flux (Salesforce → NetSuite) | Schéma HubSpot → n8n → Odoo, table de correspondance ci-dessus |
| Workflows n8n complexes (loops, merges, If/Else, Switch) | Switch (client existant ou non), Merge (client + lignes de commande) |
| Webhooks entrants/sortants | Webhook Trigger HubSpot → n8n |
| Gestion des erreurs & monitoring | Error Trigger node + alerte, idempotence anti-doublon |
| Credentials & sécurité | Credentials n8n pour HubSpot (OAuth2) et Odoo (clé API), variables d'environnement |
| REST, OAuth2, API Keys, rate limiting | HubSpot OAuth2, Odoo API key, retry/backoff sur le nœud HTTP Request |
| JavaScript pour transformation de données | Code node : aplatissement JSON imbriqué (deal + line items + champs custom) |
| SQL (PostgreSQL) | Nœud Postgres : requête directe sur la base Odoo pour un rapport de vérification |
| ETL — nettoyer/mapper données hétérogènes | Mapping des champs HubSpot (deal) vers les champs Odoo (sale.order) |
| Connaissances CRM/ERP | Table de correspondance Odoo/NetSuite, modèle de données |
| Docker (auto-hébergement n8n) | n8n ET Odoo tournent en Docker |
| Git | Projet versionné, dépôt Git |

## Plan de construction (2-3 semaines)

**Semaine 1 — Infra**
- [x] Scaffold du projet (ce dépôt)
- [ ] Odoo self-hébergé en local (Docker), apps Ventes/Facturation/Contacts activées
- [ ] Compte HubSpot (réutilisation du compte du projet NovaSupply) : objet Deal configuré avec les champs nécessaires
- [ ] App HubSpot privée créée (clé API ou OAuth2) pour le webhook sortant

**Semaine 2 — Workflow n8n**
- [ ] Webhook Trigger + réception d'un vrai payload HubSpot (test réel, capturé et sauvegardé)
- [ ] Code node : mapping JS des champs
- [ ] Intégration API Odoo : recherche client, création client si absent, création commande, création facture
- [ ] Idempotence : ne jamais créer deux fois la même commande
- [ ] Error Trigger + alerte (email ou Slack)
- [ ] Nœud Postgres : requête SQL directe de vérification

**Semaine 3 — Durcissement + répétition**
- [ ] Test de bout en bout avec plusieurs cas (client existant, client nouveau, erreur volontaire)
- [ ] Sauvegarde d'un payload HubSpot réel pour la démo (mode "replay" local, indépendant du wifi le jour J)
- [ ] Script de démo minuté (voir `docs/demo-script.md`)
- [ ] Répétitions à voix haute, chronométrées

## Fiabilité du jour J

Le jour de l'entretien, ne pas dépendre du wifi du bureau pour le webhook HubSpot en direct : le nœud Webhook de n8n sera déclenché avec un **vrai payload HubSpot capturé pendant les tests** (fichier JSON sauvegardé), rejoué localement. La chaîne reste 100% authentique (le payload est réel, capturé sur un vrai événement), seul le déclenchement est manuel pour éviter le risque de démo ratée à cause d'une connexion instable.

## Démarrer l'environnement local

```bash
cp .env.example .env
docker compose up -d
# Odoo : http://localhost:8069
```

## Dossier

- `docker-compose.yml` — stack Odoo + Postgres
- `docs/mapping-netsuite-odoo.md` — détail de la correspondance des objets
- `docs/demo-script.md` — script minuté de la démo live
- `n8n/` — export du workflow n8n (ajouté une fois construit et testé)
