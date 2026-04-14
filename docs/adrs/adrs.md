# Kranq — Architecture Decision Records

Ce document regroupe les décisions architecturales structurantes du projet Kranq, app mobile d'ingénierie cycliste.

> Statut : document d'architecture. Le projet n'a pas encore de code.

---

## ADR-001 — Choix de Node.js et NestJS plutôt que Laravel

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Mon expertise principale est PHP avec Symfony et Laravel (15+ ans). Pour VeloFleet, j'ai choisi Laravel 12 (voir ADR-001 VeloFleet). Pour Kranq, le choix du stack est à nouveau ouvert.

Kranq est fondamentalement **mobile-first**. Le cœur de l'expérience utilisateur est une app React Native utilisée en mobilité (sortie vélo, atelier). Le backend existe principalement pour synchroniser, persister et enrichir les données. La volumétrie est modeste (quelques vélos par utilisateur), la complexité est dans le **calcul métier** et le **fonctionnement offline**.

Dans ce contexte, plusieurs stacks sont envisageables :

1. **Laravel + React Native** — cohérent avec VeloFleet, réutilisation de patterns maîtrisés.
2. **NestJS + React Native** — même langage TypeScript sur les deux couches, patterns proches de Symfony/Laravel.
3. **Python FastAPI + React Native** — écosystème scientifique, bon pour calcul.

### Décision

**NestJS (Node.js + TypeScript) pour le backend, React Native + Expo pour le mobile.**

Justification détaillée :

**Cohérence de stack bout en bout.** Avoir TypeScript sur le backend et le mobile permet de partager les types, les validateurs Zod, les constantes métier via des packages internes. Un `Pressure` défini dans le domaine backend est utilisable tel quel côté mobile. Cette cohérence réduit les bugs d'interface et accélère la vélocité.

**Patterns NestJS familiers.** NestJS reprend la philosophie des frameworks Java/PHP modernes : DI container, modules, decorators, guards, interceptors, pipes. Pour quelqu'un qui maîtrise Symfony ou Laravel, la courbe d'apprentissage est minime. Les concepts DDD s'appliquent identiquement.

**Écosystème Node.js adapté au profil d'usage.** Kranq est I/O bound (appels API, sync, notifications), pas CPU bound. Node.js excelle sur ce profil grâce à son event loop non bloquant. Laravel pourrait faire aussi bien, mais sans avantage décisif.

**Démonstration de polyvalence architecturale.** Pour mon positionnement Tech Lead / Solution Architect, montrer que mes principes (SOLID, DDD, TDD, PSR-like disciplines) sont transposables sur un autre stack renforce mon profil. NestJS partage beaucoup avec l'écosystème PHP moderne, c'est donc un choix défendable sans paraître opportuniste.

**Pas de contre-argument décisif pour Laravel ici.** Laravel aurait été un bon choix, mais sans avantage différenciant pour ce projet spécifique (pas de besoin de Cashier, pas de gros Eloquent magic, pas de Blade). L'argument de cohérence avec VeloFleet est réel mais marginal — les deux projets n'ont pas vocation à partager de code.

### Conséquences

**Positives**

- Typage fort partagé backend/frontend via packages internes.
- Vélocité mobile grâce à la réutilisation des validateurs et types.
- Écosystème Node.js riche pour le sync offline et les outils de calcul.
- Démonstration d'un stack complémentaire dans mon portfolio d'architecte.

**Négatives**

- Perte de la productivité Laravel (Cashier, Horizon UI mature, Telescope, Nova).
- Moins de packages prêts à l'emploi pour certains besoins (auth sociale, facturation).
- Gestion du typage runtime plus verbeuse qu'en PHP 8 avec Eloquent.
- Besoin de discipline accrue sur le découpage modulaire (NestJS laisse plus de liberté que Laravel).

**Mesures d'atténuation**

- Utilisation de Prisma pour compenser la maturité Eloquent.
- Utilisation de BullMQ + Bull Board pour compenser Horizon.
- Structure modulaire stricte dès le départ (équivalent de mon approche VeloFleet).
- Réutilisation de packages maintenus activement (Passport NestJS pour Keycloak, class-validator pour la validation).

### Références

- Documentation NestJS officielle
- *Clean Architecture* — Robert C. Martin (applicable quel que soit le stack)
- Retours d'expérience sur NestJS dans des équipes issues de Symfony/Spring

---

## ADR-002 — Mobile-first avec React Native et Expo, pas de web

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Kranq est un outil utilisé en mobilité : sur le terrain pour un cycliste (avant une sortie, pendant une réparation de bord de route), dans un atelier pour un mécano (pendant un diagnostic, une intervention). Le desktop n'a pas de valeur ajoutée claire dans ces contextes.

Trois options existaient :

1. **Mobile natif pur** (Swift + Kotlin) — meilleure intégration OS, coût de développement doublé.
2. **PWA** — web responsive, coût unique, mais limites fortes en offline et notifications push iOS.
3. **React Native + Expo** — app native cross-platform, partage de code iOS/Android, écosystème mature.

### Décision

**React Native avec Expo (SDK 50+)** pour les deux apps (cycliste et mécano).

Pas d'app web prévue. Pas d'app desktop. Si un besoin de back-office émerge (administration du catalogue, modération), un back-office web minimal sera ajouté ultérieurement, mais ce n'est pas le périmètre initial.

### Conséquences

**Positives**

- Un seul codebase mobile pour iOS et Android.
- Expo gère la majorité de la complexité de build et de distribution (EAS Build, OTA updates).
- Écosystème mature : navigation (Expo Router), notifications (expo-notifications), storage (expo-sqlite), caméra, GPS.
- Vélocité élevée, adaptée à un projet solo.
- TypeScript partagé avec le backend NestJS.

**Négatives**

- Performances légèrement inférieures au natif pur, acceptables pour ce profil d'app.
- Dépendance à Expo (même si éjection possible via prebuild).
- Courbe d'apprentissage React Native pour certains comportements natifs (caméra, GPS, background tasks).

**Mesures d'atténuation**

- Rester sur Expo Managed Workflow en phase 1 pour vélocité.
- Basculer sur Expo Prebuild si un besoin natif spécifique l'exige (BLE, capteurs non standards).
- Benchmarks de performance sur les écrans critiques (liste composants, calcul engine).

### Références

- Documentation Expo officielle
- Retours de Shopify, Discord, Coinbase sur React Native à grande échelle

---

## ADR-003 — Stratégie offline-first avec SQLite local et sync delta

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

L'app doit être pleinement fonctionnelle hors ligne :

- Un cycliste en sortie VTT en montagne sans réseau doit pouvoir consulter ses composants, calculer une pression de pneus, logger une sortie.
- Un mécano en atelier dans une cave ou dans un local mal couvert doit pouvoir diagnostiquer un vélo, consulter l'historique d'un client, valider une intervention.

L'approche "online-first avec fallback cache" ne suffit pas. Il faut une architecture offline-first véritable, où toute écriture se fait d'abord localement, puis se synchronise avec le serveur dès que la connectivité est disponible.

### Décision

**Architecture offline-first avec SQLite local et synchronisation delta.**

Composants clés :

**Stockage local** : SQLite via `expo-sqlite`. Toutes les entités métier (vélos, composants, sessions, interventions) sont persistées localement. AsyncStorage pour les préférences simples.

**Écriture** : toute mutation est écrite d'abord localement avec un statut `pending` et un timestamp. Puis publiée dans une file d'attente de sync qui tente l'envoi au serveur quand le réseau est disponible.

**Lecture** : toute lecture se fait sur le cache local. Les sync descendantes enrichissent le cache, mais l'app ne dépend jamais d'un appel réseau pour afficher une donnée.

**Synchronisation delta** : chaque entité porte un `version` (timestamp vectoriel ou Lamport clock) et un `lastSyncedAt`. La sync échange uniquement les deltas depuis la dernière sync réussie. Format JSON léger.

**Résolution de conflits** : stratégie **last-write-wins** par défaut, avec alertes UX si conflit détecté sur un champ critique (kilométrage, seuil d'usure). Pour les quelques entités à consistance forte (session enregistrée), la version serveur prime.

**Sync opportuniste** : déclenchée au lancement de l'app, après chaque écriture locale si réseau disponible, et périodiquement en background task.

### Conséquences

**Positives**

- Expérience utilisateur fluide même sans réseau.
- Protection contre les pertes de données liées à des coupures.
- Latence perçue minimale (pas de wait réseau sur les actions utilisateur).
- Adapté à l'usage terrain cycliste et mécano.

**Négatives**

- Complexité architecturale significativement plus élevée qu'un modèle online-only.
- Risques de conflits nécessitant une stratégie de résolution claire.
- Volumétrie locale à surveiller (SQLite a ses limites sur très gros volumes).
- Tests spécifiques nécessaires pour valider les scénarios offline et les merges.

**Mesures d'atténuation**

- Module `Sync` dédié, isolé des bounded contexts métier.
- Logging détaillé des conflits et rejeux possibles via interface mécano.
- Quotas sur les données locales (archivage des sessions anciennes, éviction des blobs lourds type photos).
- Suite de tests E2E Maestro qui couvrent explicitement les scénarios offline → online.

### Références

- CRDT et Operational Transformation (pour inspiration, pas d'implémentation CRDT prévue en phase 1)
- *Designing Data-Intensive Applications* — Martin Kleppmann, chapitres sur réplication
- PouchDB, Realm, WatermelonDB (études comparatives)

---

## ADR-004 — Calcul engine versionné avec résultats persistés

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Kranq propose des calculs d'ingénierie cycliste (pression de pneus, tension de rayons, couples de serrage, géométrie). Ces calculs reposent sur des **formules scientifiques** qui évoluent :

- De nouvelles études remplacent d'anciennes formules (exemple : les modèles de pression de pneus ont été refondus par Silca et Zipp en 2020-2023).
- Les préférences utilisateur peuvent changer (système métrique vs impérial).
- Les formules peuvent dépendre de paramètres contextuels (type de pratique, surface, météo).

Deux pièges à éviter :

1. **Calcul à la volée non reproductible** : si je consulte un ancien calcul après évolution de la formule, je ne retrouve pas le même résultat — perte de confiance utilisateur.
2. **Formule figée sans évolution** : si on fige la formule initiale, on ne peut jamais améliorer les recommandations — perte de valeur produit.

### Décision

**Calcul engine versionné avec résultats persistés.**

Concrètement :

**Versioning des formules** : chaque formule est identifiée par un nom et une version (`pressure.v2`, `spoke_tension.v1`). Les versions antérieures restent disponibles et testables.

**Résultats persistés** : chaque calcul effectué par un utilisateur est stocké avec :
- L'identifiant de la formule utilisée (`pressure.v2`)
- Les inputs complets
- Le résultat obtenu
- Un timestamp

**Recomputation explicite** : si une nouvelle version de formule sort, l'utilisateur peut **choisir** de recalculer ses anciens résultats. Le résultat historique reste accessible en parallèle.

**Formules en TypeScript pur** : le module `Engineering` contient les formules en code TypeScript, sans dépendance framework. Chaque formule est une fonction pure avec signature typée. Elles sont testables en pur TypeScript et partageables entre backend et mobile.

**Documentation de chaque formule** : chaque formule est accompagnée de sa source scientifique, de ses hypothèses et de ses limites d'application.

### Conséquences

**Positives**

- Reproductibilité totale des calculs historiques.
- Évolution libre des formules sans rupture de confiance utilisateur.
- Testabilité parfaite (fonctions pures, tests unitaires exhaustifs).
- Partage facile des formules entre backend et mobile via un package TypeScript.
- Documentation scientifique crédibilise le produit.

**Négatives**

- Volumétrie de stockage accrue (chaque calcul persisté).
- Complexité UX pour présenter les versions (comment afficher un résultat "obsolète" ?).
- Maintenance des anciennes versions de formules dans le code.

**Mesures d'atténuation**

- Archivage des résultats très anciens (plus d'un an) avec compression.
- UX : afficher uniquement la dernière version par défaut, avec lien "voir les versions précédentes".
- Déprécation explicite d'une formule avec période de grâce avant suppression complète.

### Références

- Silca Professional Tire Pressure Calculator (inspiration produit)
- Zipp Tire Pressure Guide (inspiration produit)
- *Software Engineering at Google* — chapitre sur le versioning de services

---

## ADR-005 — Deux profils utilisateurs sur un backend partagé

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Kranq adresse deux personas distincts avec des besoins différents :

- **Cycliste** : usage personnel, 1 à 5 vélos max, focus calcul et suivi perso.
- **Mécano indépendant** : usage professionnel, plusieurs dizaines de clients, focus diagnostic et historique multi-clients.

Question : construire **une seule app** ou **deux apps distinctes** ?

### Décision

**Un backend partagé, deux expériences mobiles distinctes au niveau UI/navigation, avec un profil utilisateur qui détermine l'UX au runtime.**

Concrètement :

**Backend unifié** : un seul NestJS, un seul schéma de base. Les entités métier (vélo, composant, session, intervention) sont communes. Un champ `profileType` sur l'utilisateur détermine son rôle.

**Permissions et vues séparées** : le module `Profile` expose des permissions qui varient selon le rôle. Un mécano voit et édite les données de ses clients ; un cycliste voit et édite uniquement les siennes.

**Mobile : un seul codebase, deux parcours** : l'app React Native détecte le profil après login et affiche le parcours cycliste ou le parcours mécano. Écrans partagés (calculs, catalogue), écrans spécifiques (gestion clients pour mécano, carnet d'entraînement pour cycliste).

**Pas d'app "hybride"** : un utilisateur est cycliste OU mécano pour un compte donné. S'il est les deux, il a deux comptes. Cela évite des cas UX complexes (mécano qui voit ses vélos perso mélangés avec ceux de clients).

### Conséquences

**Positives**

- Un seul backend à maintenir et à déployer.
- Un seul codebase mobile, cohérence UX.
- Possibilité de migrer un cycliste vers un compte mécano (et inversement) sans réécrire.
- Économies de coût de développement par rapport à deux apps séparées.

**Négatives**

- Risque de dilution UX (un écran qui essaie de servir deux usages finit par mal servir les deux).
- Nécessité d'une discipline stricte dans la UX : chaque écran doit être pensé pour un seul profil à la fois.
- Tests à couvrir pour les deux parcours.

**Mesures d'atténuation**

- Design system commun, mais composants de navigation et écrans d'accueil strictement séparés.
- Tests E2E qui valident chaque parcours indépendamment.
- Feature flags pour activer/désactiver des écrans selon le profil.
- Review UX régulière avec des utilisateurs des deux profils.

### Références

- *Designing for the Mind* — patterns de personnalisation par persona
- Retours d'expérience Strava (pratiquant) vs Strava for Coaches


---

## ADR-006 — Engineering Service extrait en Python pour calcul scientifique et IA

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

L'ADR-004 a établi que le moteur de calcul serait versionné avec résultats persistés. La question du **langage d'implémentation** de ce moteur reste ouverte.

Deux options :

1. **Implémenter les calculs dans le Core NestJS** — cohérence de stack, partage de types avec le mobile, simplicité opérationnelle.
2. **Extraire le calcul dans un micro-service Python dédié** — langage de référence du calcul scientifique et de l'IA.

Le choix doit tenir compte de la feuille de route produit qui prévoit à terme :

- Recommandations personnalisées de pression de pneus basées sur l'historique
- Analyse prédictive de l'usure des composants selon le profil d'usage
- Suggestions de maintenance préventive
- Potentielle intégration de données météo, profils GPX, données Strava

Tous ces cas d'usage relèvent du **machine learning**, domaine dominé par Python.

### Décision

**Extraction de l'Engineering en micro-service Python dédié (FastAPI), dès la conception**, même si la phase 1 n'utilise que des formules déterministes simples.

Justifications :

**Python est l'écosystème de référence pour le calcul scientifique et l'IA.** NumPy, SciPy, Pandas pour le calcul numérique. scikit-learn, PyTorch, TensorFlow pour l'IA. Tenter de répliquer ces capacités en TypeScript/Node.js est contre-productif — les bibliothèques équivalentes existent mais sont moins matures et moins documentées.

**Profil de charge différent du Core.** Le Core NestJS est I/O-bound (appels base de données, sync, orchestration). L'Engineering Service sera CPU-bound pour les calculs complexes et l'inférence ML. Les deux profils justifient un **scaling indépendant** et des caractéristiques d'instances différentes (le Core en petites instances nombreuses, l'Engineering en instances plus lourdes avec potentiellement GPU pour l'inférence).

**Préparation à l'IA sans sur-engineering phase 1.** En extrayant le service dès le départ, l'évolution vers des modèles ML se fait sans refonte architecturale. Les interfaces (REST + events RabbitMQ) restent stables, seule l'implémentation interne évolue.

**Isolation des dépendances scientifiques.** Les bibliothèques Python de calcul ont leurs propres contraintes de version, de compatibilité CPU/GPU, de packaging (wheels pré-compilés). Les isoler dans un conteneur dédié évite que ces contraintes polluent le Core NestJS.

### Conséquences

**Positives**

- Accès aux bibliothèques scientifiques et IA de référence.
- Scaling indépendant du Core, optimisé pour CPU/GPU.
- Évolution naturelle vers l'IA sans refonte.
- Isolation des dépendances lourdes (NumPy, PyTorch pèsent des centaines de Mo).
- Démonstration de polyglot architecture justifiée par le contexte.

**Négatives**

- Hétérogénéité de stack (Node + Python) avec coût cognitif.
- Deux pipelines CI/CD à maintenir.
- Duplication potentielle de modèles métier entre Core et Engineering.
- Latence réseau ajoutée sur les calculs (REST synchrone).

**Mesures d'atténuation**

- Contrat d'API strict via **OpenAPI** généré par FastAPI, importé par le Core NestJS pour typage client.
- Modèles métier partagés via **JSON Schema** ou via un service registry en cas de besoin.
- Cache Redis agressif sur les calculs stables (même inputs = même output cacheable).
- Monitoring latence spécifique sur les appels Core → Engineering, avec SLO clair.
- Fallback local dans le mobile pour les formules simples (permet de fonctionner offline).
- Un seul langage de tests unitaires par service, pas de partage forcé.

### Références

- *Microservices Patterns* — Chris Richardson, chapitres sur le polyglot
- Retours d'expérience Spotify (Python pour ML, Scala/Java pour data pipeline)
- FastAPI + Pydantic comme standard moderne pour les APIs Python typées

---

## ADR-007 — Notifications Service déporté dès la conception

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Kranq doit envoyer plusieurs types de notifications :

- **Alertes d'usure** : un composant a franchi un seuil, nécessite une attention.
- **Rappels de maintenance** : une intervention programmée approche.
- **Notifications de sync** : un conflit a été détecté lors d'une synchronisation.
- **Communications produit** : mises à jour, nouveautés, tips.

Canaux : **push mobile (Expo)**, **email**, éventuellement **SMS** pour les mécanos pros.

Deux options :

1. **Gérer les notifications dans le Core NestJS** — module Notifications intégré, pas de réseau inter-services.
2. **Extraire les Notifications en micro-service dédié** — isolation, scaling, résilience.

### Décision

**Extraction du service Notifications dès la phase 1**, même si la volumétrie initiale est modeste.

Stack : Node.js (Fastify) ou Go selon les besoins de performance. Communication exclusivement **asynchrone via RabbitMQ**.

Principe : le Core publie des events métier (`ComponentWearThresholdReached`, `MaintenanceDue`, `SyncConflictDetected`). Le service Notifications **s'abonne** à ces events et décide quoi envoyer, à qui, par quel canal, selon les préférences utilisateur.

Le Core **ne connaît pas** l'existence du service Notifications. Il publie des events métier, point. C'est le service Notifications qui porte la connaissance des canaux, templates, préférences, rate limiting.

### Conséquences

**Positives**

- Découplage fort : le métier n'est pas pollué par la logique d'envoi.
- Scaling horizontal indépendant (pics de campagnes, alertes massives).
- Isolation des intégrations tierces (Expo, SendGrid, Twilio) qui peuvent être instables.
- Résilience : une panne du service Notifications n'affecte pas le métier, les events s'accumulent en file.
- Facilité d'ajout de nouveaux canaux (webhook, push web) sans toucher au Core.
- Préparation au scénario multi-produit : si plus tard VeloFleet et Kranq partagent ce service, c'est déjà prêt.

**Négatives**

- Un service de plus à déployer, monitorer, maintenir.
- Complexité opérationnelle accrue vs une solution intégrée.
- Latence ajoutée (event bus, traitement asynchrone) — mais acceptable pour des notifications.
- Gestion des échecs d'envoi distribuée (dead letter queue, rejeu).

**Mesures d'atténuation**

- Démarrer avec un service minimal : un seul handler qui gère push Expo + email via SendGrid.
- Monitoring spécifique : taux de livraison, latence end-to-end, échecs par canal.
- Dead letter queue RabbitMQ avec alertes si remplissage.
- Contrats d'events stricts et versionnés (`ComponentWearThresholdReached.v1`).
- Idempotence garantie côté consumer (un même event rejoué ne produit pas de doublons).
- Templates de notification externalisés (Markdown ou composants Expo) pour édition sans redéploiement.

### Références

- *Building Event-Driven Microservices* — Adam Bellemare
- Documentation Expo Push Service
- Patterns RabbitMQ pour notifications (fan-out, topic exchange)

---

## ADR-008 — Couche data warehouse activée selon seuils de volumétrie

**Statut** : Accepté
**Date** : 2026-04
**Auteur** : Francis Boudo

### Contexte

Kranq est conçu pour évoluer vers des fonctionnalités basées sur la donnée :

- Analytics utilisateurs (dashboards dans l'app : "votre chaîne a parcouru 3 200 km ce mois")
- Recommandations personnalisées (modèles IA sur l'Engineering Service)
- Rapports agrégés pour les mécanos (tendances d'usure sur leur parc clients)
- Éventuellement des insights communautaires (pression moyenne pour un pneu X chez les utilisateurs de poids Y)

Ces fonctionnalités imposent à terme des **charges analytiques lourdes** incompatibles avec la base OLTP (PostgreSQL transactionnelle) :

- Requêtes agrégées sur historiques longs
- Jointures massives pour l'entraînement de modèles
- Lecture intensive qui entrerait en concurrence avec le transactionnel

Deux approches :

1. **Tout en OLTP** — acceptable en phase early, dégradé dès que la volumétrie augmente.
2. **Couche OLAP dédiée dès le départ** — over-engineering en phase MVP, coûte en complexité et en infra.

### Décision

**Prévoir l'architecture data dès la conception, mais n'activer la couche OLAP que sur signaux concrets de volumétrie.**

**Architecture cible documentée** :

- **CDC** (Change Data Capture) depuis PostgreSQL via Debezium ou Airbyte, vers le warehouse.
- **Data Warehouse** : BigQuery (managed, scaling automatique, pay-per-query) en première option, ClickHouse (auto-hébergé, très performant sur time series) en alternative si la volumétrie ou le coût le justifient.
- **Transformation** : dbt pour modéliser les datamarts (dimensional modeling, star schema), versionnage SQL des transformations.
- **BI** : Metabase (open source, simple) pour démarrer, évolutif vers Superset si besoins plus avancés.
- **ML Engineering** : alimentation des modèles IA de l'Engineering Service via le warehouse (datasets d'entraînement), déploiement via MLflow ou BentoML.

**Seuils d'activation** (à calibrer) :

- **Volumétrie** : plus de 10k utilisateurs actifs mensuels OU plus de 10M de lignes dans les tables sessions/measures.
- **Usage analytique** : premier besoin fonctionnel nécessitant des agrégations complexes récurrentes.
- **IA** : premier modèle ML nécessitant un dataset d'entraînement > 100k observations.

Le premier seuil atteint déclenche la mise en place de la couche data. Avant, toutes les analyses se font en PostgreSQL direct, avec index adaptés et read replicas si nécessaire.

### Conséquences

**Positives**

- Architecture évolutive sans refonte au moment de scaler.
- Séparation claire OLTP vs OLAP : le transactionnel reste performant même avec beaucoup de lectures analytiques.
- Préparation naturelle à l'IA via le warehouse comme source d'entraînement.
- Coût maîtrisé : on ne paie pas la couche data tant qu'elle n'est pas utile.
- Documentation existante facilitera la mise en œuvre au moment venu.

**Négatives**

- La décision "quand activer" reste à prendre, avec un risque d'activation tardive si les signaux sont mal lus.
- Phase de transition délicate : migration progressive des analyses OLTP → OLAP.
- Coût d'ingestion CDC à surveiller (bande passante, licences Airbyte cloud le cas échéant).

**Mesures d'atténuation**

- Dashboards de monitoring sur la volumétrie PostgreSQL avec seuils d'alerte (taille tables, durée des requêtes analytiques).
- Revue trimestrielle des besoins analytiques pour identifier le moment optimal d'activation.
- Modélisation dimensionnelle préparée en amont (document de design des datamarts) pour que l'implémentation soit rapide le moment venu.
- Tests de charge réguliers sur PostgreSQL pour valider les limites actuelles.

### Références

- *The Data Warehouse Toolkit* — Ralph Kimball, référence du dimensional modeling
- *Designing Data-Intensive Applications* — Martin Kleppmann
- Documentation dbt et BigQuery
- Patterns d'évolution architecturale des SaaS (Segment, Notion, Linear)
