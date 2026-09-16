# Script de démo — Entretien Iniwave

Démo live, ~7-8 minutes, à répéter à voix haute au moins 2-3 fois avant le jour J.

## Avant de commencer (la veille + le matin même)

- [ ] Vérifier que `n8n.luluplug.com`, `odoo-iniwave.luluplug.com` et le VPS répondent bien
- [ ] Vérifier le workflow n8n : actif, Error Workflow bien configuré
- [ ] Nettoyer les données de test dans Odoo (garder 2-3 exemples propres, pas 10 doublons)
- [ ] Préparer un **hotspot 4G** en secours si le wifi du bureau est capricieux
- [ ] Avoir un **screen recording** d'un run complet réussi, en secours ultime si la démo live plante (ne jamais improviser à l'aveugle devant le jury — mieux vaut dire "laissez-moi vous montrer une capture" que de galérer 5 minutes)
- [ ] Recharger le laptop à 100%

## 0:00 – 0:30 — Le pitch d'ouverture

> "Je vais vous montrer une démo que j'ai construite spécifiquement pour ce poste — elle reproduit exactement le cas d'usage cité dans votre fiche : Salesforce vers NetSuite. J'ai utilisé HubSpot comme CRM et Odoo comme ERP, parce que NetSuite demande une licence entreprise que je n'ai pas — mais l'architecture et le code sont transposables un pour un."

## 0:30 – 1:30 — Vue d'ensemble du workflow (sans rien exécuter encore)

Ouvrir le workflow n8n, montrer le canvas avec les sticky notes.

> "Le workflow a 13 étapes numérotées. En gros : on détecte un deal signé côté HubSpot, on vérifie deux fois qu'on ne va pas créer de doublon — une fois sur la commande, une fois sur le client — puis on crée le client si besoin, la commande, on la confirme, et on génère la facture. À la fin, une requête SQL directe vérifie l'état de la base."

Pointer rapidement : le nœud de polling (#1-2), les deux IF anti-doublon (#7, #9), le Merge (#9c).

## 1:30 – 3:00 — Action live côté HubSpot

Aller sur HubSpot, créer un nouveau deal en direct devant le jury (nom clair, ex: "Démo Iniwave [date]"), associer un contact avec un nom réel, remplir un montant, **passer directement en Closed Won**.

> "Je viens de simuler un commercial qui vient de signer un contrat."

## 3:00 – 4:30 — Déclencher et observer le workflow

Dans n8n, cliquer sur **"Test workflow"** pour déclencher manuellement (ne pas attendre les 5 minutes du polling en direct).

Narrer pendant que ça tourne :

> "Il va chercher les deals Closed Won récents, récupérer les infos du contact associé, vérifier qu'aucune commande n'existe déjà pour ce deal, créer le client s'il n'existe pas encore, créer la commande, la confirmer, et générer la facture."

Laisser les coches vertes s'allumer nœud par nœud.

## 4:30 – 5:30 — Vérification côté Odoo

Basculer sur `odoo-iniwave.luluplug.com`, montrer :
- Le nouveau client créé dans Contacts
- La commande dans Ventes, avec le bon montant
- La facture liée

> "Tout ce que vous voyez ici a été créé automatiquement, sans que personne ne recopie quoi que ce soit."

## 5:30 – 6:30 — La requête SQL directe

Revenir dans n8n, montrer le résultat du nœud Postgres (#14).

> "Ce dernier nœud interroge directement la base Postgres d'Odoo, en SQL, avec une jointure entre les factures et les clients — sans repasser par l'API. Deux raisons : ça montre que je sais aussi interroger une base directement, pas seulement consommer une API. Et ça me sert de vérification indépendante — je ne me contente pas de croire ce que l'API m'a dit avoir créé, je relis directement dans la base que la facture est bien là, avec le bon client."

## 6:30 – 7:30 — Démonstration de la gestion d'erreur

Deux options selon l'ambiance de l'entretien (juger sur place) :

**Option sûre (recommandée)** : ne pas casser quoi que ce soit en live. Montrer le workflow d'erreur séparé + les messages Slack déjà reçus lors des tests précédents (garder ces messages visibles dans le canal, ne pas les effacer avant l'entretien).

> "Voici le filet de sécurité : si une étape plante — Odoo injoignable, token expiré — un workflow séparé se déclenche automatiquement et m'alerte sur Slack avec le détail de l'erreur, le nœud fautif et un lien direct vers l'exécution."

**Option risquée (si le jury semble vouloir du live et que le timing est bon)** : relancer le workflow sur le **même deal** qu'on vient de traiter — l'anti-doublon va bloquer, montrer que ça s'arrête proprement (pas une vraie erreur technique, mais démontre la logique de garde-fou en direct).

## 7:30 – 8:00 — Clôture

> "Cette démo couvre à peu près tous les points de votre fiche de poste : workflows n8n avec branches et fusions, gestion des déclencheurs, API REST avec authentification, transformation JSON en JavaScript, anti-doublon, gestion d'erreurs avec alerting, SQL direct sur la base, et tout ça auto-hébergé en Docker sur mon propre VPS. Je suis preneur de vos questions, ou si vous voulez, je peux ouvrir le code d'un nœud en particulier."

## Questions probables après la démo

Voir `wiki/projets/projet-iniwave.md` et la fiche de révision entretien pour les réponses préparées sur : Closed Won, HTTP Request, OAuth2, rate limiting, SQL, choix Odoo vs NetSuite.

## Ce qu'il faut assumer si on pose la question (rigueur > bluff)

- Le port Postgres est exposé publiquement sur le VPS pour cette démo — en vrai production, on isolerait n8n et Odoo sur un réseau Docker interne partagé, sans exposer le port du tout.
- Les factures créées restent en statut "draft" (pas auto-validées) — choix assumé de garder un humain dans la boucle avant validation comptable finale, à discuter si demandé.
- Le déclenchement est en polling (toutes les 5 min), pas en webhook — la clé de service HubSpot utilisée ne permet pas nativement les webhooks ; un vrai webhook nécessiterait une app HubSpot plus complète (OAuth2 + app enregistrée).
