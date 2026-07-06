# PI-FACT — Moteur de Facturation CBAO (Vue Globale)

## Présentation du projet

**PI-FACT (Smart Billing)** est le moteur de génération et d'application de règles de facturation de la banque CBAO (Finetech). Il calcule automatiquement les frais de transaction bancaire en évaluant des règles configurables par priorité.

Le système expose une API REST consommée par d'autres services bancaires pour facturer les transactions en temps réel, et fournit une interface de gestion pour les administrateurs (CRUD des règles, critères, actions, simulation, reporting).

---

## Architecture globale

```
┌─────────────────────────────────────────────────────────┐
│                   CBAO Banking Services                  │
│  (services tiers qui appellent /billing/apply-rules)     │
└───────────────────────┬─────────────────────────────────┘
                        │ POST /api/v1/billing/apply-rules
                        │ Bearer: ApiToken (propriétaire)
                        ▼
┌─────────────────────────────────────────────────────────┐
│              Backend — Spring Boot (pi-fact)             │
│  Java 17 / Spring Boot 3.3.1 / Spring Cloud 2023.0.1    │
│  Port: 8080                                              │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │  Moteur de │  │  Gestion des │  │    Dashboard    │  │
│  │ Facturation│  │  règles CRUD │  │    Analytics    │  │
│  └────────────┘  └──────────────┘  └─────────────────┘  │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │PostgreSQL  │  │ Elasticsearch│  │     Consul      │  │
│  │(dev/Oracle │  │ (règles idx) │  │(config+discov.) │  │
│  │ prod)      │  │              │  │                 │  │
│  └────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
                        ▲ REST API /pi-fact/api/v1
                        │ Bearer: Keycloak JWT
┌─────────────────────────────────────────────────────────┐
│            Frontend — Angular 17 SPA (pi-fact-ui)        │
│  Port dev: 4200 | Prod: Nginx Alpine                     │
│  Authentification: Keycloak OAuth2 (realm: cbao-apps)    │
└─────────────────────────────────────────────────────────┘
```

---

## Stack Technique

### Backend
| Composant | Technologie |
|---|---|
| Langage | Java 17 |
| Framework | Spring Boot 3.3.1 |
| Cloud | Spring Cloud 2023.0.1 |
| ORM | Spring Data JPA + Hibernate |
| DB Dev | PostgreSQL |
| DB Prod | Oracle |
| DB Tests | H2 (in-memory) |
| Recherche | Elasticsearch |
| Config/Discovery | Consul |
| Auth | Keycloak OAuth2 (Resource Server) |
| Mapping | MapStruct 1.5.5 |
| Build | Maven |
| API Docs | Springdoc OpenAPI 2.2 (Swagger UI) |

### Frontend
| Composant | Technologie |
|---|---|
| Framework | Angular 17.3.12 |
| UI | PrimeNG 17 + PrimeFlex 3.3 |
| Auth | Keycloak Angular 15 + keycloak-js 25 |
| State | TanStack Angular Query + RxJS |
| Charts | Chart.js + ApexCharts |
| PDF | jsPDF + pdf-lib |
| Dates | Day.js |
| Styles | SCSS + PrimeNG Saga Orange theme |
| Tests | Jasmine + Karma |
| Build | Angular CLI 17 |
| Serve prod | Nginx Alpine (Docker) |

---

## Domaine Métier — Flux de Facturation

```
BillingRequest (transaction)
    │
    ▼
[1] Enrichissement du payload
    ├── Compteur transactions mensuel (TransactionHistory)
    ├── Cumul journalier (PaymentGatewayService) ⚠️ STUB
    ├── Flags promotionnels / jours spéciaux
    └── Champs de la requête (direction, canal, montant, types, etc.)
    │
    ▼
[2] Chargement des règles actives (triées par priorité DESC)
    │
    ▼
[3] Évaluation des règles (première règle où TOUS les critères matchent)
    ├── Vérification conditions temporelles (périodes promo, jours spéciaux)
    └── Évaluation des critères métier (direction, type client, montant, etc.)
    │
    ▼
[4] Application des actions (calcul des frais)
    ├── FIXED_FEE / PERCENTAGE_FEE / SHARED_FEE / EXEMPT
    └── Calcul payeur/payé, frais, devise, compte opération
    │
    ▼
[5] Persistance async (TransactionFee) + MAJ compteur (TransactionHistory)
    │
    ▼
BillingResponse (frais calculés, règle appliquée, ID transaction)
```

---

## Entités Principales

| Entité | Table | Description |
|---|---|---|
| `RuleDefinition` | `pi_fact_rule_definition` | Règle de facturation (priorité, canal, statut, période) |
| `Criteria` | `pi_fact_criteria` | Critère d'évaluation (code, opérateur, valeur) |
| `Action` | `pi_fact_action` | Action de calcul de frais (fixe, %, partagé, exempté) |
| `CriteriaTypeRegistry` | `pi_fact_criteria_type_registry` | Types de critères disponibles |
| `ActionTypeRegistry` | `pi_fact_action_type_registry` | Types d'actions disponibles |
| `TransactionFee` | `pi_fact_transaction_fee` | Historique des frais calculés |
| `TransactionHistory` | `pi_fact_transaction_history` | Compteur mensuel transactions/destinataire |
| `PromotionalPeriods` | `pi_fact_promotional_periods` | Périodes promotionnelles |
| `SpecialDayDefinitions` | `pi_fact_special_day_definitions` | Jours spéciaux (fériés, etc.) |
| `ApiClient` | `api_client` | Clients API externes |
| `ApiToken` | `api_token` | Tokens d'accès pour clients API |

**Statuts de règle :** `DISABLED → SIMULATED → ENABLED`

---

## Authentification

Deux mécanismes coexistent :

| Contexte | Mécanisme | Header |
|---|---|---|
| Interface Admin (UI) | Keycloak JWT (OAuth2 Resource Server) | `Authorization: Bearer <jwt>` |
| Intégrations service-à-service | ApiToken propriétaire (BD) | `Authorization: Bearer <apitoken>` |

- **Keycloak** : `http://localhost:9080` (dev) / `/auths` (prod), realm `cbao-apps`
- **ApiTokenFilter** : intercepte uniquement `/billing/apply-rules`

---

## Endpoints Clés

| Méthode | URL | Auth | Description |
|---|---|---|---|
| `POST` | `/pi-fact/api/v1/billing/apply-rules` | ApiToken | Appliquer les règles (facturation réelle) |
| `POST` | `/pi-fact/api/v1/billing/simulate-rules` | JWT | Simuler les règles (sans impact) |
| `CRUD` | `/pi-fact/api/v1/rule-definitions/**` | JWT | Gestion des règles |
| `CRUD` | `/pi-fact/api/v1/criteria/**` | JWT | Gestion des critères |
| `CRUD` | `/pi-fact/api/v1/actions/**` | JWT | Gestion des actions |
| `GET` | `/pi-fact/api/v1/transaction-fees/**` | JWT | Historique des frais |
| `GET` | `/pi-fact/api/v1/dashboard/**` | JWT | KPIs et analytics |
| `CRUD` | `/pi-fact/api/v1/api-clients/**` | JWT | Gestion des clients API |
| `POST` | `/pi-fact/api/v1/auth/**` | Public | Auth tokens |

---

## Structure des Dossiers

```
facturation/
├── CLAUDE.md              ← Ce fichier
├── DIAGNOSTIC.md          ← Diagnostic Senior Architecte
├── backend/
│   ├── CLAUDE.md          ← Contexte détaillé backend
│   ├── pom.xml
│   └── src/main/java/bank/cbao/smartbilling/
│       ├── controller/    # REST endpoints
│       ├── service/       # Logique métier
│       │   ├── impl/      # Implémentations (BillingServiceImpl ← coeur)
│       │   ├── interfaces/
│       │   └── specification/ # JPA Specifications pour filtrage
│       ├── model/         # Entités JPA
│       ├── repository/    # JPA + Elasticsearch repos
│       ├── dto/           # Data Transfer Objects
│       ├── mapper/        # MapStruct mappers
│       ├── config/        # Security, Constants, ES, ApiTokenFilter
│       └── exception/     # GlobalExceptionHandler
└── frontend/
    ├── CLAUDE.md          ← Contexte détaillé frontend
    ├── src/app/
    │   ├── core/          # Services, guards, interceptors, interfaces
    │   ├── features/      # Modules lazy-loaded (rules, criteria, actions...)
    │   ├── layout/        # Shell (header, sidebar, footer)
    │   └── shared/        # Composants et pipes réutilisables
    └── nginx.conf         # Config prod (SPA routing + proxy API)
```

---

## Commandes

### Backend
```bash
# Dev
cd backend
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev

# Build
./mvnw clean package -DskipTests

# Tests
./mvnw test

# Prérequis dev : PostgreSQL sur 5432, Consul sur 8500 (optionnel), Keycloak sur 9080 (optionnel)
```

### Frontend
```bash
# Dev
cd frontend
npm start                  # Serveur dev sur localhost:4200

# Build
npm run build:prod         # Build production optimisé
npm run build:dev          # Build dev avec source maps

# Tests
npm run test:coverage      # Tests + rapport de couverture
npm run lint               # Linting

# Prérequis : Backend sur localhost:8080, Keycloak sur localhost:9080
```

---

## Variables d'Environnement Clés

### Backend (`.env.prod` ou Consul)
```
SPRING_DATASOURCE_URL=jdbc:oracle:thin:@...
SPRING_DATASOURCE_USERNAME=...
SPRING_DATASOURCE_PASSWORD=...
SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=...
CONSUL_ACL_TOKEN=...          # ⚠️ NE PAS committer en VCS
```

### Frontend (`environment.ts` / `environment.prod.ts`)
```typescript
apiBaseUrl: "/pi-fact/api/v1"
keycloak.url: "/auths"        // Relatif via Nginx en prod
```

---

## Notes Importantes

> **⚠️ STUB non fonctionnel :** `PaymentGatewayService` retourne toujours `20000` pour le cumul journalier. Le critère `DAILY_CUMULATIVE_AMOUNT_OVER_20K` est inopérant jusqu'à implémentation réelle.

> **⚠️ Sécurité :** Voir `DIAGNOSTIC.md` — plusieurs vulnérabilités critiques identifiées, notamment sur `ApiTokenFilter` et le fallback Keycloak.

> **Cycle de statut des règles :** `DISABLED → SIMULATED → ENABLED`. Une règle SIMULATED est évaluée par `/simulate-rules` mais pas par `/apply-rules`.

> **Priorité des règles :** Ordre décroissant (plus haute priorité = évaluée en premier). La première règle dont TOUS les critères matchent est sélectionnée.

> **Tables préfixées :** Toutes les tables métier portent le préfixe `pi_fact_`.
