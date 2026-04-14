# Kranq

> L'ingénieur vélo dans la poche — app mobile d'ingénierie cycliste et de suivi d'usure pour cyclistes exigeants et mécanos indépendants.

**Statut :** Architecture document only — no implementation yet.

Ce repo contient **la conception architecturale** de Kranq. Le code n'est pas encore écrit. L'objectif de ce document est de formaliser la vision technique et de démontrer une démarche d'architecte sur un contexte mobile-first distinct de [VeloFleet](https://github.com/francisboudo/velofleet).

---

## Pourquoi ce repo existe

Kranq est un projet d'app mobile orienté **ingénierie cycliste**. Il répond à un besoin concret : les cyclistes engagés et les mécanos indépendants manipulent au quotidien des calculs (pression de pneus selon poids et pneu, tension de rayons, géométrie, couples de serrage) et doivent suivre l'usure de composants (chaîne, plaquettes, câbles, cassette) pour anticiper les interventions.

Ce repo formalise :

- La **vision produit** (personas, usages)
- L'**architecture cible** (bounded contexts, sync offline, calcul engine, data)
- Les **choix de stack assumés** (Node.js + NestJS, Python, React Native, PostgreSQL)
- Les **ADRs structurants** (mobile-first, offline-first, calcul engine versionné, découpage micro-services, data layer)

Il sert également de démonstration d'architecture dans un contexte **mobile-first avec hétérogénéité de stacks justifiée** (Node pour le métier, Python pour le calcul et l'IA).

---

## Positionnement produit

### Deux profils utilisateurs distincts

#### Cycliste exigeant (consumer)

Le cycliste route, gravel, VTT ou compétiteur qui veut optimiser ses réglages et suivre l'usure de ses composants sur plusieurs vélos.

Besoins :
- Calculs rapides de pression de pneus (pneu, jante, poids système, type de pratique)
- Enregistrement des sorties pour suivi du kilométrage par composant
- Alertes préventives sur les seuils d'usure
- Historique d'entretien de ses vélos

#### Mécano vélo indépendant (pro)

Le mécano d'atelier indépendant qui gère ses clients et a besoin d'outils de diagnostic et de référence en mobilité.

Besoins :
- Calculs d'ingénierie de terrain (couples, tensions, géométrie)
- Diagnostic d'usure d'un vélo client
- Référence marques/modèles et spécifications techniques
- Historique d'interventions par client et par vélo

### Différenciation avec VeloFleet

| Dimension | VeloFleet | Kranq |
|---|---|---|
| Cible | Loueurs, flottes d'entreprise, ateliers multi-sites | Cyclistes individuels, mécanos indépendants |
| Contexte | SaaS B2B multi-tenant, back-office web | Mobile-first, usage terrain |
| Volumétrie | Flottes de 20 à 1000+ vélos par tenant | Quelques vélos par utilisateur |
| Enjeu technique | Gestion opérationnelle, facturation, multi-site | Calcul d'ingénierie, offline, sync, IA |

Les deux projets partagent un **univers métier commun** (le cyclisme technique) mais **n'adressent pas les mêmes clients** et ne se cannibalisent pas.

---

## Architecture

Voir schéma généré dans `docs/architecture.svg`.

### Vue macro

L'architecture est composée de trois couches distinctes :

1. **Core métier NestJS** — monolithe modulaire portant le domaine CRUD et l'orchestration
2. **Micro-services dédiés** — extractions justifiées par des caractéristiques techniques différentes
3. **Couche data** — activable quand la volumétrie justifie une séparation OLTP / OLAP

### Bounded contexts du Core (NestJS)

| Module | Responsabilité |
|---|---|
| **Wear** | Suivi d'usure des composants, seuils d'alerte, consommation par session |
| **Configuration** | Setup vélo, géométrie, composants, groupset, réglages |
| **Maintenance** | Historique d'interventions, planification, templates d'intervention |
| **Catalog** | Référentiel marques, modèles, composants, spécifications techniques |
| **Profile** | Profil utilisateur (cycliste / mécano), préférences, unités (métrique/impérial) |
| **Session** | Enregistrement de sorties, kilométrage, contextes (route/VTT/gravel) |
| **Sync** | Résolution des conflits, stratégie delta client-serveur, timestamp vectoriel |

### Micro-services extraits

#### Engineering Service (Python / FastAPI)

Le calcul scientifique et les modèles d'IA sont **déportés dans un micro-service Python dédié**. Justifications :

- **Python est le langage de référence pour le calcul scientifique** (NumPy, SciPy, Pandas) et l'IA (scikit-learn, PyTorch, TensorFlow).
- Les formules d'ingénierie cycliste bénéficient de bibliothèques spécialisées (interpolation, régression, optimisation).
- Le profil CPU-bound du calcul justifie un **scaling indépendant** du Core NestJS qui reste I/O-bound.
- **Préparation à une couche IA** : recommandations de pression personnalisées basées sur l'historique utilisateur, analyse prédictive de l'usure, suggestions de maintenance préventive.
- Le Core NestJS reste **stateless** face à ce service : il passe les inputs, reçoit les résultats, persiste en base.

Communication : REST synchrone (latence acceptable sur calcul, simplicité d'intégration) avec événements async via RabbitMQ pour les entraînements et évaluations longs.

#### Notifications Service (Node.js ou Go)

Le service de notifications est **déporté** dès la conception. Justifications :

- Profil d'usage très différent du Core : très haute volumétrie de messages courts, pas de logique métier complexe.
- Besoin de **scaling horizontal indépendant** lors des pics (campagnes, alertes de seuil massives).
- Isolation des **intégrations tierces** (Expo Push, SendGrid, Twilio) qui évoluent et peuvent échouer sans impacter le métier.
- Event-driven naturel : consomme des events du Core et dispatche sans dépendance inverse.

Communication : purement asynchrone via RabbitMQ. Le Core ne "sait" pas qu'un service de notifications existe — il publie des events et c'est le service qui s'y abonne.

### Couche Data (activée selon volumétrie)

Quand la volumétrie et la qualification des données le justifient (seuil à définir, probablement autour de 10k+ utilisateurs actifs et plusieurs millions de sessions/mesures), une couche data distincte est activée :

- **ETL/ELT** : ingestion CDC depuis PostgreSQL vers le warehouse, via Airbyte ou dbt.
- **Data Warehouse** : BigQuery (managed, scaling automatique) ou ClickHouse (auto-hébergé, très performant sur time series).
- **BI / Analytics** : Metabase ou Superset pour les dashboards internes et les rapports utilisateurs premium.
- **ML Engineering** : entraînement des modèles IA du service Engineering sur les données du warehouse, déploiement des modèles via MLFlow ou BentoML.

Cette couche est **prévue dès la conception** pour éviter la dette data, mais **n'est pas activée en phase MVP**. Elle est déclenchée par des signaux concrets : volumétrie de sessions dépassant les capacités OLTP, besoins d'analyse historique, premières demandes IA nécessitant un dataset d'entraînement conséquent.

---

## Stack technique assumé

### Core backend

| Composant | Technologie | Pourquoi |
|---|---|---|
| Langage | TypeScript strict | Typage fort, écosystème mature |
| Framework | NestJS | Patterns DI, modules, decorators — proche philosophie Symfony/Laravel |
| ORM | Prisma | Typage bout en bout, migrations déclaratives |
| Base de données | PostgreSQL 16 | Relationnel solide, JSONB pour specs composants |
| Cache | Redis | Cache calculs, sessions, queues |
| Event bus | RabbitMQ | Communication inter-services découplée |
| Auth | Keycloak OAuth2/OIDC | Cohérence avec mon écosystème (VeloFleet) |
| Tests | Jest, Supertest | Standards Node/NestJS |
| Observabilité | OpenTelemetry, Pino | Standards cloud-native |

### Engineering Service

| Composant | Technologie | Pourquoi |
|---|---|---|
| Langage | Python 3.12+ | Référence calcul scientifique et IA |
| Framework API | FastAPI | Typage Pydantic, async, OpenAPI natif |
| Calcul | NumPy, SciPy | Bibliothèques standards ingénierie |
| ML (phase 2) | scikit-learn, PyTorch | Modèles prédictifs, recommandations |
| Serveur ML (phase 2) | BentoML ou TorchServe | Déploiement et versioning de modèles |
| Tests | Pytest | Standard Python |

### Notifications Service

| Composant | Technologie | Pourquoi |
|---|---|---|
| Langage | Node.js (ou Go si besoin perf) | Léger, scaling horizontal aisé |
| Framework | Fastify | Minimaliste, très performant |
| Queue | RabbitMQ (consumer) | Découplage natif |
| Intégrations | Expo Push SDK, SendGrid, Twilio | Standards du marché |

### Frontend mobile

| Composant | Technologie |
|---|---|
| Framework | React Native + Expo (SDK 50+) |
| Langage | TypeScript strict |
| Navigation | Expo Router (file-based) |
| State serveur | TanStack Query v5 |
| State client | Zustand |
| Storage local | SQLite (expo-sqlite) + AsyncStorage |
| Forms | React Hook Form + Zod |
| UI | Tamagui ou NativeWind (Tailwind) |
| Tests | Jest + React Native Testing Library |
| E2E | Maestro |

### Couche data (phase d'activation)

| Composant | Technologie |
|---|---|
| CDC / ingestion | Airbyte ou Debezium |
| Transformation | dbt |
| Data Warehouse | BigQuery ou ClickHouse |
| BI | Metabase ou Superset |
| ML Ops | MLflow, DVC, BentoML |

### Infrastructure

| Composant | Technologie |
|---|---|
| Déploiement backend | Docker, Google Cloud Run |
| CDN / Assets | Google Cloud Storage + Cloudflare |
| CI/CD | GitHub Actions |
| Build mobile | EAS Build (Expo Application Services) |
| Distribution | TestFlight, Google Play Internal Testing |
| Monitoring | Grafana Cloud + Sentry |

---

## Pourquoi cette hétérogénéité de stacks ?

Le choix d'un stack hétérogène (Node.js pour le Core, Python pour l'Engineering, Node/Go pour les Notifications) n'est pas un caprice technique mais une **décision architecturale motivée par les caractéristiques de chaque domaine** :

- **Core NestJS** : domaine métier classique (CRUD, orchestration, sync). TypeScript partagé avec le mobile pour la vélocité.
- **Engineering Python** : calcul scientifique, accès aux bibliothèques numériques de référence, préparation IA. Profil CPU-bound, scaling dédié.
- **Notifications Node/Go** : I/O massif, intégrations tierces, scaling horizontal. Isolation des dépendances fragiles.

Cette hétérogénéité est **assumée et justifiée**, pas subie. Elle démontre que l'architecture répond aux besoins de chaque domaine plutôt que d'appliquer un stack unique par confort.

---

## Principes d'ingénierie

Les mêmes principes que VeloFleet sont appliqués, adaptés à chaque stack :

- **SOLID** — DI systématique, interfaces injectées, single responsibility
- **KISS / YAGNI** — pas d'abstractions prématurées, extraction de services uniquement quand justifiée
- **DDD pragmatique** — bounded contexts isolés, langage ubiquitaire cycliste
- **TDD** — tests en premier sur la logique métier, domaine testable sans framework
- **Standards** — ESLint strict, Black/Ruff côté Python, Conventional Commits, PRs reviewées

---

## Défis techniques structurants

### 1. Offline-first avec sync bidirectionnelle

L'app doit être pleinement fonctionnelle hors ligne. Toute écriture est d'abord locale, puis synchronisée avec stratégie de résolution de conflits basée sur des timestamps vectoriels. Voir ADR-003.

### 2. Calcul engine versionné

Les formules évoluent. Le moteur de calcul est versionné pour garantir la reproductibilité des résultats historiques. Voir ADR-004.

### 3. Hétérogénéité Node + Python

Cohabitation de stacks maîtrisée via des contrats d'API explicites (OpenAPI), un event bus commun (RabbitMQ), et une observabilité unifiée (OpenTelemetry). Voir ADR-006.

### 4. Scalabilité différenciée

Chaque composant a son propre profil de charge et ses propres métriques. Scaling indépendant via Kubernetes/Cloud Run. Voir ADR-007.

### 5. Préparation IA sans sur-engineering

L'Engineering Service est conçu dès le départ pour accueillir des modèles IA, mais ces modèles ne seront entraînés qu'une fois un dataset conséquent disponible (phase 3+). Voir ADR-008.

---

## ADRs

| ADR | Sujet |
|---|---|
| ADR-001 | Choix Node.js + NestJS pour le Core plutôt que Laravel |
| ADR-002 | Mobile-first avec React Native + Expo, pas de web |
| ADR-003 | Stratégie offline-first avec SQLite local et sync delta |
| ADR-004 | Calcul engine versionné avec résultats persistés |
| ADR-005 | Deux profils utilisateurs sur un backend partagé |
| ADR-006 | Engineering Service extrait en Python (calcul scientifique et IA) |
| ADR-007 | Notifications Service déporté dès la conception |
| ADR-008 | Couche data warehouse activée selon seuils de volumétrie |

Voir `docs/adr/`.

---

## Roadmap conceptuelle

| Phase | Contenu |
|---|---|
| **Phase 0 — Actuelle** | Architecture document — pas de code |
| **Phase 1 — MVP cycliste** | Core NestJS + Engineering Service basique (formules v1) + mobile cycliste |
| **Phase 2 — Extension mécano** | Profil mécano, multi-clients, notifications déportées |
| **Phase 3 — Data & IA** | Couche warehouse activée, premiers modèles ML sur Engineering Service |
| **Phase 4 — Intégrations** | Strava, Garmin, Wahoo, communauté |

**Calendrier** : aucun engagement. Le développement effectif démarrera après la stabilisation de VeloFleet en production.

---

## À propos de l'auteur

**Francis Boudo** — Tech Lead Backend & Solution Architect, cycliste engagé (environ 15 000 km/an).

Kranq est un projet conceptuel qui combine mes deux centres d'intérêt : l'architecture logicielle et le cyclisme technique. Il complète mon autre projet [VeloFleet](https://github.com/francisboudo/velofleet) (gestion de flotte B2B) dans un univers cohérent mais avec un positionnement produit distinct.

- **LinkedIn** : [linkedin.com/in/francis-boudo](https://linkedin.com/in/francis-boudo)
- **VeloFleet** : [github.com/francisboudo/velofleet](https://github.com/francisboudo/velofleet)
- **Email** : fboudo@gmail.com

---

## Licence

Documentation et architecture sous licence [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — libre réutilisation avec attribution.
Marque, identité et code futur : propriétaire.
