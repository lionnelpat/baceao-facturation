# DIAGNOSTIC COMPLET — PI-FACT Smart Billing
**Réalisé par :** Senior Dev & Architecte
**Date :** 2026-03-24
**Version analysée :** Backend 1.0.0 / Frontend 1.1.0

---

## Résumé Exécutif

PI-FACT est un moteur de facturation bancaire **structurellement solide** avec une architecture claire (Spring Boot + Angular, séparation des couches, DTOs, MapStruct, JPA Specifications). Cependant, l'analyse révèle **10 vulnérabilités de sécurité** dont 2 critiques permettant un contournement d'authentification, **4 bugs logiques** dont 1 avec impact financier direct (double facturation), et des problèmes de performance structurels. Un déploiement en production dans l'état actuel expose la banque à des risques significatifs.

---

## Tableau de Bord des Issues

| Catégorie | Critique 🔴 | Important 🟠 | Amélioration 🟡 |
|---|---|---|---|
| Sécurité | 4 | 4 | 2 |
| Bugs Logiques | 3 | 1 | — |
| Performance | — | 4 | 2 |
| Architecture | — | 3 | 2 |
| Code Quality | — | 1 | 5 |
| Fonctionnel/UX | — | 1 | 4 |
| **TOTAL** | **7** | **14** | **15** |

---

## 🔴 CRITIQUE — Sécurité

---

### [SEC-01] ApiTokenFilter — Authentication Bypass Complet

**Fichier :** `backend/src/main/java/bank/cbao/smartbilling/config/ApiTokenFilter.java:31-48`
**Sévérité :** CRITIQUE — Exploitation immédiate possible

**Problème :**
```java
// CODE ACTUEL — VULNÉRABLE
protected void doFilterInternal(...) {
    if (!isApplyRulesEndpoint(request)) {
        filterChain.doFilter(request, response);  // Route normale OK
        return;
    }
    try {
        filterChain.doFilter(request, response);   // ← BUG: exécute la requête AVANT validation
        String token = extractToken(request);      // ← jamais atteint si exception levée plus haut
        authenticateWithToken(token);              // ← inutile, requête déjà traitée
        filterChain.doFilter(request, response);   // ← double exécution de la chaîne
    } catch (BusinessException e) { ... }
}
```

**Impact :** L'endpoint `POST /billing/apply-rules` traite n'importe quelle requête sans token. N'importe quel système peut déclencher des facturations réelles sans credentials. La validation du token intervient APRÈS que la requête a déjà été traitée (ou est morte-née car la chaîne a déjà été exécutée).

**Correction :**
```java
// CODE CORRIGÉ
protected void doFilterInternal(...) {
    if (!isApplyRulesEndpoint(request)) {
        filterChain.doFilter(request, response);
        return;
    }
    try {
        String token = extractToken(request);      // 1. Extraire d'abord
        authenticateWithToken(token);              // 2. Valider ensuite
        filterChain.doFilter(request, response);   // 3. Traiter seulement si valide
    } catch (BusinessException e) {
        sendErrorResponse(response, HttpStatus.BAD_REQUEST, e.getMessage());
    } catch (BadCredentialsException e) {
        sendErrorResponse(response, HttpStatus.UNAUTHORIZED, e.getMessage());
    }
}
```

---

### [SEC-02] SecurityConfig — Désactivation Totale de la Sécurité sur Fallback Keycloak

**Fichier :** `backend/.../config/SecurityConfig.java:99-103`
**Sévérité :** CRITIQUE — Risque en production

**Problème :**
```java
if (securityEnabled && isKeycloakAccessible()) {
    // Config OAuth2...
} else {
    logger.warn("⚠️ OAuth2 désactivé - Accès libre autorisé");
    http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll()); // ← TOUS les endpoints ouverts
}
```

Si Keycloak est temporairement inaccessible au démarrage (redémarrage, réseau instable, maintenance), le service démarre en mode **totalement non sécurisé** sans que l'opérateur ne s'en aperçoive forcément (simple WARN dans les logs).

**Impact :** Fenêtre de temps où toutes les APIs sont exposées sans authentification — endpoints d'administration, gestion des règles, consultation des transactions.

**Correction :** En production, fail-fast si Keycloak est inaccessible. En dev, utiliser un profil explicite `@Profile("dev")`.

```java
if (securityEnabled && isKeycloakAccessible()) {
    http.oauth2ResourceServer(...);
} else if (!securityEnabled) {
    // Profil dev uniquement — logguer très clairement
    log.warn("MODE DEV — Sécurité désactivée intentionnellement");
    http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
} else {
    // Keycloak inaccessible en prod : BLOQUER tout accès
    log.error("CRITIQUE : Keycloak inaccessible au démarrage — refus de démarrer sans sécurité");
    throw new IllegalStateException("Keycloak inaccessible. Démarrage refusé.");
}
```

---

### [SEC-03] Clé Secrète JWT Hardcodée dans le Code Source

**Fichier :** `backend/.../config/SecurityConfig.java:169`
**Sévérité :** CRITIQUE

```java
String secretKey = "mySecretKeyForDevelopmentOnlyNotForProduction12345";
```

Cette clé est dans le dépôt VCS. Quiconque y accède peut forger des tokens JWT valides dès que le fallback est activé (cf. SEC-02).

**Correction :** Externaliser dans une variable d'environnement `${JWT_FALLBACK_SECRET}` ou supprimer le fallback en production.

---

### [SEC-04] Tokens Consul ACL Committés en VCS

**Fichiers :**
- `backend/src/main/resources/config/application-prod.yml:11` → `acl-token: ed16b7ac-0250-5066-5cd4-e6f612dcad38`
- `backend/src/main/resources/config/application-dev.yml:17` → même token

**Sévérité :** CRITIQUE — Credentials en clair dans le dépôt

**Correction :**
```yaml
# application.yml
spring.cloud.consul.discovery.acl-token: ${CONSUL_ACL_TOKEN}
spring.cloud.consul.config.acl-token: ${CONSUL_ACL_TOKEN}
```
Régénérer immédiatement le token Consul compromis.

---

### [SEC-05] Client Secret Keycloak Exposé dans le Frontend

**Fichier :** `frontend/src/environments/environment.ts:16`
**Sévérité :** IMPORTANT

```typescript
clientSecret: 'uoUs3tFxSGhhjtbRK5JKQFaknAzjkSlf',
grantType: 'password'  // OAuth 2.1 ROPC — déprécié
```

- Le secret est commité en VCS et inclus dans le bundle JavaScript distribué.
- Le flux **Resource Owner Password Credentials (ROPC)** est déprécié depuis OAuth 2.1 car il expose les credentials utilisateur à l'application cliente.

**Correction :** Migrer vers **Authorization Code + PKCE** (standard actuel pour les SPAs). Pas de client_secret nécessaire avec PKCE.

---

### [SEC-06] CORS Wildcard + Credentials en Production

**Fichier :** `backend/.../config/SecurityConfig.java:190-198`
**Sévérité :** IMPORTANT

```java
config.setAllowedOriginPatterns(List.of("*")); // Wildcard
config.setAllowCredentials(true);               // Avec credentials
```

Bien que les navigateurs bloquent `*` + `credentials` selon la spec CORS, cette configuration expose le serveur à des attaques si le comportement évolue ou si les requêtes viennent de clients non-navigateur.

**Correction :**
```java
// Production
config.setAllowedOrigins(List.of(
    "https://facturation.cbao.com",
    "https://admin.cbao.com"
));
```

---

### [SEC-07] Swagger UI et Actuator Publics sans Authentification

**Fichier :** `SecurityConfig.java:74-83`
**Sévérité :** IMPORTANT

`/swagger-ui/**`, `/v3/api-docs/**`, `/actuator/**` sont `permitAll()` sans restriction d'environnement.

**Impact :** En production, n'importe qui peut :
- Explorer toute l'API documentée (Swagger)
- Lire les métriques, l'état de santé, les beans Spring (Actuator)

**Correction :** Désactiver ou protéger ces endpoints en profil production.

```yaml
# application-prod.yml
springdoc.swagger-ui.enabled: false
management.endpoints.web.exposure.include: health,info
management.endpoint.health.show-details: never
```

---

### [SEC-08] JWT Stocké en localStorage (Frontend)

**Fichier :** `frontend/src/app/core/services/auth/`
**Sévérité :** IMPORTANT

Stocker les tokens JWT en `localStorage` les expose aux attaques XSS : un script injecté peut lire et exfiltrer les tokens.

**Correction :** Utiliser des cookies `HttpOnly; Secure; SameSite=Strict` (gérés côté serveur) ou conserver les tokens en mémoire seulement (avec re-auth transparente via refresh token en cookie HttpOnly).

---

### [SEC-09] Données Bancaires Sensibles dans les Logs

**Fichier :** `BillingServiceImpl.java:168`
**Sévérité :** IMPORTANT

```java
log.info("Evaluation payload pour règle {}: {}", rule.getCode(), convertPayloadToJson(payload));
```

Le payload contient : numéros de compte (NCP), alias, montants, codes pays, types de clients. Ces données sont loguées au niveau **INFO** à chaque appel de facturation, donc dans tous les fichiers de logs.

**Impact :** Violation potentielle RGPD/réglementations bancaires locales. Données exploitables si les logs sont accédés par des tiers.

**Correction :** Descendre à `DEBUG` + masquer les champs sensibles (NCP, alias, montant).

---

### [SEC-10] Numéro de Compte Bancaire Hardcodé

**Fichier :** `Constants.java:27`

```java
public static final String OPERATION_ACCOUNT_DEFAULT = "BF1610000103600000010111";
```

Ce numéro ressemble à un vrai compte pivot CBAO hardcodé dans le source. Doit être externalisé en configuration ou base de données.

---

## 🔴 CRITIQUE — Bugs Logiques

---

### [BUG-01] Race Condition — `transactionId` Toujours Null dans la Réponse

**Fichier :** `BillingServiceImpl.java:110-146`
**Sévérité :** CRITIQUE — Bug fonctionnel constant

**Problème :**
```java
// saveTransactionFee est @Async → exécuté dans un autre thread
@Async
protected void saveTransactionFee(..., BillingResponseDTO response, ...) {
    TransactionFee saved = transactionFeeRepository.save(transactionFee);
    response.setTransactionId(saved.getId());  // ← modifie response en async
}

// Dans processBilling :
saveTransactionFee(request, response, selectedRule, payload, isSimulated); // async, non bloquant
// ...
return response;  // ← retourné AVANT que saveTransactionFee ait fini → transactionId = null
```

La méthode `processBilling` retourne la réponse immédiatement. Le save asynchrone n'a pas encore eu lieu. `transactionId` est **toujours null** dans la réponse API retournée aux clients.

**Correction :** Séparer la sauvegarde de la mise à jour de la réponse :
```java
// Option 1 : sauvegarde synchrone pour obtenir l'ID, puis async pour les opérations secondaires
TransactionFee saved = transactionFeeRepository.save(buildTransactionFee(...));
response.setTransactionId(saved.getId());
asyncPostProcessing(request, saved, isSimulated); // historique, notifications, etc.
```

---

### [BUG-02] PaymentGatewayService — Stub Non Fonctionnel en Production

**Sévérité :** CRITIQUE — Règles métier non fonctionnelles

Le service qui calcule le cumul journalier des transactions retourne toujours `20000` :
```java
// PaymentGatewayService.java
public BigDecimal getDailyCumulativeAmount(String recipientNcp) {
    return BigDecimal.valueOf(20000); // TODO: implémenter l'appel réel
}
```

**Impact :** Le critère `DAILY_CUMULATIVE_AMOUNT_OVER_20K` est évalué avec une valeur fictive constante. Toutes les règles basées sur ce critère prennent des décisions de facturation incorrectes en production.

**Correction :** Implémenter l'appel réel : agréger les `TransactionFee` du jour pour ce recipient, ou appeler le service de paiement CBAO.

---

### [BUG-03] Absence d'Idempotence sur `/apply-rules`

**Sévérité :** CRITIQUE — Double facturation possible

Aucun mécanisme d'idempotency key. En cas de :
- Timeout réseau suivi d'un retry
- Défaillance partielle côté appelant
- Bug dans le système appelant

La même transaction sera facturée plusieurs fois, créant des doublons dans `TransactionFee` et dans `TransactionHistory`.

**Correction :** Ajouter un header `Idempotency-Key` :
```java
// BillingController.java
@PostMapping("/apply-rules")
public BillingResponseDTO applyRules(
    @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
    @RequestBody BillingRequestDTO request) {
    return billingService.applyRules(request, idempotencyKey);
}
```
Stocker les clés d'idempotence en Redis ou en base avec TTL 24h.

---

### [BUG-04] `simulateRules` Sans `@Transactional` mais avec Persistance

**Fichier :** `BillingServiceImpl.java:59-61`
**Sévérité :** IMPORTANT

```java
@Override
public BillingResponseDTO simulateRules(BillingRequestDTO request) {
    return processBilling(request, true); // Pas @Transactional
}
```

`processBilling` appelle `saveTransactionFee` (persistance en DB). Sans `@Transactional`, si une exception survient après le save, il n'y a pas de rollback. Des entrées de simulation orphelines peuvent s'accumuler.

---

## 🟠 IMPORTANT — Performance

---

### [PERF-01] Chargement de Toutes les Règles en Mémoire à Chaque Appel

**Fichier :** `BillingServiceImpl.java:72-74`
**Impact :** Dégradation O(n) avec le nombre de règles

```java
List<RuleDefinition> activeRules = ruleDefinitionRepository.findActiveRules(Instant.now());
// Charge TOUTES les règles actives, avec leurs relations (critères, actions, etc.)
```

Avec 500 règles, chaque appel `/apply-rules` charge, sérialise et évalue en mémoire des centaines d'objets JPA avec leurs relations ManyToMany. Sous charge, cela génère une pression GC importante.

**Corrections possibles :**
1. **Cache en mémoire** : `@Cacheable("active-rules")` avec invalidation sur modification (Caffeine, TTL 1min)
2. **Elasticsearch** : Utiliser l'index ES existant pour la recherche des règles applicables
3. **Pré-filtrage DB** : Filtrer par canal/direction en DB avant de charger en mémoire

---

### [PERF-02] Problème N+1 sur les Relations ManyToMany

**Sévérité :** IMPORTANT

Le chargement d'une `RuleDefinition` avec ses `Criteria`, `Actions`, `PromotionalPeriods` et `SpecialDayDefinitions` génère potentiellement 4+ requêtes SQL par règle si le fetch type est LAZY sans JOIN FETCH.

**Exemple de ce qui se passe sans optimisation (100 règles = ~500 requêtes) :**
```
SELECT * FROM pi_fact_rule_definition WHERE ...
SELECT * FROM pi_fact_criteria WHERE rule_id = 1
SELECT * FROM pi_fact_action WHERE rule_id = 1
SELECT * FROM pi_fact_criteria WHERE rule_id = 2
...
```

**Correction :**
```java
// RuleDefinitionRepository.java
@Query("""
    SELECT DISTINCT r FROM RuleDefinition r
    LEFT JOIN FETCH r.criteria
    LEFT JOIN FETCH r.actions
    LEFT JOIN FETCH r.promotionalPeriods
    LEFT JOIN FETCH r.specialDayDefinitions
    WHERE r.status IN ('ENABLED') AND r.startDate <= :now AND r.endDate >= :now
""")
List<RuleDefinition> findActiveRulesWithRelations(@Param("now") Instant now);
```

---

### [PERF-03] Double Appel HTTP Synchrone à Keycloak au Démarrage

**Fichier :** `SecurityConfig.java:91, 149`

`isKeycloakAccessible()` effectue un appel HTTP de 3 secondes max, et est appelé **deux fois** au démarrage de Spring (une fois dans `securityFilterChain`, une fois dans `jwtDecoder`).

Si Keycloak met 3 secondes à répondre : **+6 secondes** sur le temps de démarrage.

**Correction :** Mémoriser le résultat (lazy init avec `@Lazy` ou un champ `volatile`).

---

### [PERF-04] Pool de Threads `@Async` Non Borné

**Fichier :** `BillingServiceImpl.java:110, 148`

Spring utilise le pool par défaut (`SimpleAsyncTaskExecutor`) pour `@Async`, qui crée **un thread par tâche**. Sous charge forte (1000 req/s), cela créerait 2000 threads par seconde pour les saves et historiques → saturation mémoire / OOM.

**Correction :**
```java
// AsyncConfig.java
@Bean("billingTaskExecutor")
public TaskExecutor billingTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(500);
    executor.setThreadNamePrefix("billing-async-");
    executor.setRejectedExecutionHandler(new CallerRunsPolicy()); // graceful fallback
    return executor;
}

// BillingServiceImpl.java
@Async("billingTaskExecutor")
protected void saveTransactionFee(...) { ... }
```

---

### [PERF-05] Dashboard Sans Cache

Les endpoints KPI (`/dashboard/**`) effectuent des agrégations complexes sur `TransactionFee` à chaque requête. Ces calculs (sommes, comptages, distributions) peuvent être coûteux sur un volume important.

**Correction :**
```java
@Cacheable(value = "dashboard-kpis", key = "#filter.period")
public DashboardKpisDTO getKpis(DashboardFilterDTO filter) { ... }

// Avec Spring Cache + Caffeine
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager("dashboard-kpis");
    manager.setCaffeine(Caffeine.newBuilder().expireAfterWrite(5, TimeUnit.MINUTES));
    return manager;
}
```

---

### [PERF-06] Bundle Angular Trop Volumineux

**Fichier :** `frontend/angular.json`
Budget configuré : 3MB initial, 6MB max — excessif pour une app d'administration bancaire.

**Analyse recommandée :**
```bash
npm run build:prod -- --stats-json
npx webpack-bundle-analyzer dist/stats.json
```

**Pistes de réduction :**
- `FullCalendar` (~400KB) : vérifier si vraiment utilisé
- `pdf-lib` (~300KB) : tree-shakeable ? Lazy-load uniquement sur les pages export
- `ApexCharts` + `Chart.js` : choisir l'un ou l'autre
- Lazy-load les modules heavy (transactions, simulateur)

---

## 🟠 IMPORTANT — Architecture & Design

---

### [ARCH-01] Critères Codés en Dur dans le Moteur — Couplage Fort

**Fichier :** `BillingServiceImpl.java` → `getValueForCriteria()`

La correspondance `code_critère → clé_payload` est une longue chaîne conditionnelle hardcodée :
```java
private Object getValueForCriteria(String criteriaCode, Map<String, Object> payload) {
    if ("DIR_EMM".equals(criteriaCode) || "DIR_REC".equals(criteriaCode)) {
        return payload.get("direction");
    } else if ("SENDER_CLIENT_TYPE_P".equals(criteriaCode) || ...) {
        return payload.get("sender_client_type");
    }
    // ... 20+ conditions
}
```

**Problème :** Ajouter un nouveau type de critère oblige à modifier le coeur du moteur de facturation — violation du principe Open/Closed.

**Correction — Pattern Registry :**
```java
// CriteriaPayloadResolver.java
@Component
public class CriteriaPayloadResolver {
    private final Map<String, Function<Map<String, Object>, Object>> resolvers = Map.of(
        "DIR_EMM", p -> p.get("direction"),
        "DIR_REC", p -> p.get("direction"),
        "SENDER_CLIENT_TYPE_P", p -> p.get("sender_client_type"),
        // ...
    );

    public Object resolve(String criteriaCode, Map<String, Object> payload) {
        return resolvers.getOrDefault(criteriaCode, p -> null).apply(payload);
    }
}
```
À terme, externaliser cette configuration en base de données (lier `CriteriaTypeRegistry` à sa clé payload).

---

### [ARCH-02] Double Authentification Non Documentée et Non Testée

Le système supporte deux flux d'authentification distincts dont la coexistence n'est pas clairement documentée ni testée :
- **Keycloak JWT** : pour les utilisateurs de l'interface admin
- **ApiToken propriétaire** : pour les intégrations service-à-service

**Risques :**
- Un ApiToken pourrait accéder à des endpoints protégés Keycloak si la SecurityConfig est mal ordonnée
- Pas de tests d'intégration validant chaque flux

**Recommandation :** Ajouter des tests de sécurité dédiés :
```java
@Test
void applyRules_withoutToken_shouldReturn401() { ... }
@Test
void applyRules_withInvalidToken_shouldReturn401() { ... }
@Test
void applyRules_withValidApiToken_shouldReturn200() { ... }
@Test
void ruleDefinitions_withApiToken_shouldReturn403() { ... } // ApiToken n'a pas accès au CRUD
```

---

### [ARCH-03] Frontend — Paradigmes Angular Mixtes et OAuth Déprécié

**Problèmes identifiés :**
1. **Mélange NgModules / Standalone Components** : Dashboard est standalone, les features sont NgModules. Incohérence qui complexifie la maintenance.
2. **Double state management** : TanStack Query ET RxJS Subjects — deux approches différentes selon les modules.
3. **ROPC OAuth2** : `grant_type=password` déprécié en OAuth 2.1, expose les credentials utilisateur à l'app cliente.

**Roadmap recommandée :**
- **Court terme :** Standardiser sur TanStack Query pour les appels API
- **Moyen terme :** Migrer vers Authorization Code + PKCE pour l'auth
- **Long terme :** Migration complète vers Standalone Components + Angular Signals

---

### [ARCH-04] TypeScript Strict Mode Désactivé

**Fichier :** `frontend/tsconfig.json`
```json
{ "strict": false }
```

Désactiver le mode strict permet : `any` implicites, nullability non vérifiée, paramètres non typés. Pour un système financier, c'est un risque de régression silencieuse.

**Migration progressive :**
```json
{
  "strictNullChecks": true,          // Activer d'abord
  "noImplicitAny": true,             // Puis
  "strict": true                     // Enfin
}
```

---

### [ARCH-05] Elasticsearch Optionnel mais Comportement Non Documenté

ES est intégré pour l'indexation des règles, mais optionnel en dev. Il n'est pas clair :
- Quels endpoints utilisent ES vs JPA en production
- Quel est le fallback si ES est indisponible en prod
- Si les règles sont indexées en temps réel ou en différé

**Recommandation :** Documenter le comportement dans `CLAUDE.md`, ajouter des health checks explicites, et tester les performances ES vs JPA sur des volumes réalistes.

---

## 🟡 AMÉLIORATION — Qualité du Code

---

### [CODE-01] Logging Trop Verbeux avec Emojis en Production

**Fichier :** `BillingServiceImpl.java` (multiples lignes)

Logs au niveau `INFO` pour chaque critère évalué, chaque règle testée, avec emojis ✅❌. Sur 1000 transactions/minute, ces logs génèrent des MBs de données et polluent les systèmes d'agrégation (ELK, Splunk).

**Standard recommandé :**
- `INFO` : Événements métier clés (règle sélectionnée, transaction facturée)
- `DEBUG` : Détail de l'évaluation (critères, payload)
- `WARN` : Situations anormales (aucune règle applicable)
- `ERROR` : Erreurs techniques

---

### [CODE-02] `console.log` / `console.error` dans le Frontend

Logs de debug dans les services Angular, visibles dans la console navigateur en production. Exposent des détails techniques aux utilisateurs finaux.

**Correction :** Créer un `LoggerService` qui désactive les logs en production :
```typescript
@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(...args: any[]) {
    if (!environment.production) console.log(...args);
  }
}
```

---

### [CODE-03] Validation Insuffisante sur `BillingRequestDTO`

Les champs critiques n'ont pas d'annotations de validation Bean Validation :
```java
public class BillingRequestDTO {
    private String direction;   // Pas de @NotBlank, pas de @Pattern pour les valeurs valides
    private BigDecimal amount;  // Pas de @NotNull, @DecimalMin("0.01")
    private String currency;    // Pas de @Size(min=3, max=3) pour les codes ISO
    // ...
}
```

Des données invalides traversent silencieusement le moteur et peuvent générer des NPE ou des calculs incorrects.

---

### [CODE-04] `@Getter` / `@Setter` sur un `@Service`

**Fichier :** `BillingServiceImpl.java:25-26`

Lombok `@Getter` et `@Setter` sur un service Spring n'ont pas de sens — les services doivent être stateless et ne pas exposer leurs dépendances injectées. Supprimer ces annotations.

---

### [CODE-05] Absence de Tests sur le Moteur de Règles

Le coeur du système (`BillingServiceImpl`) ne dispose pas de tests unitaires couvrant les cas limites :
- Règle sans critère → doit-elle toujours matcher ?
- Règle sans action → que retourner ?
- Payload incomplet → comportement attendu ?
- Deux règles à même priorité → déterminisme ?
- Transaction simulée → ne doit pas incrémenter le compteur

**Priorité haute :** Ces cas sont la définition contractuelle du comportement financier.

---

## 🟡 AMÉLIORATION — Fonctionnel & UX

---

### [FUNC-01] Absence de Trace de Décision dans la Réponse

La réponse billing indique quelle règle a été appliquée, mais pas **pourquoi** (quels critères ont matché, lesquels ont été rejetés pour les autres règles).

**Amélioration :**
```json
{
  "feeAmount": 500,
  "ruleApplied": "RULE-001",
  "decisionTrace": {           // Optionnel via ?debug=true
    "rulesEvaluated": 15,
    "matchedCriteria": ["DIR_EMM", "SENDER_CLIENT_TYPE_P"],
    "rejectedRules": [
      { "code": "RULE-002", "reason": "MONTHLY_TRANSFER_COUNT_UNDER_30 no match" }
    ]
  }
}
```

---

### [FUNC-02] Conflits de Priorité Non Déterministes

Si deux règles ont la même priorité, la sélection dépend de l'ordre retourné par la DB (non garanti). En cas de modification du schéma ou des index, le comportement peut changer silencieusement.

**Correction :** Contrainte `UNIQUE` sur la colonne `priority` de `pi_fact_rule_definition`, ou tri secondaire déterministe (`ORDER BY priority DESC, created_date ASC`).

---

### [FUNC-03] Pas d'Internationalisation (i18n)

Messages d'erreur et labels tous hardcodés en français dans le frontend. Acceptable pour une app CBAO mono-langue, mais bloquant si la banque opère dans plusieurs pays.

---

### [FUNC-04] Pas de Gestion Offline / Dégradée

Si le backend est indisponible, l'interface affiche des erreurs génériques sans guidance utilisateur. Pas de mode dégradé, pas de retry avec feedback.

---

### [FUNC-05] Expiration des Tokens API Non Notifiée

Les `ApiToken` expirent (colonne `expirationDate`). Les clients intégrés découvrent l'expiration quand leurs appels commencent à retourner 401, sans préavis.

**Amélioration :** Job planifié `@Scheduled` qui envoie une notification (email, webhook) 7 jours avant l'expiration.

---

## Plan d'Action Priorisé

### Sprint 1 — Corrections Critiques (≈ 2 semaines)

| # | Issue | Fichier | Effort |
|---|---|---|---|
| 1 | [SEC-01] Corriger `ApiTokenFilter` | `ApiTokenFilter.java` | 2h |
| 2 | [SEC-02] Fail-fast si Keycloak inaccessible en prod | `SecurityConfig.java` | 3h |
| 3 | [BUG-01] Corriger race condition `transactionId` | `BillingServiceImpl.java` | 4h |
| 4 | [SEC-03] Externaliser la clé JWT fallback | `SecurityConfig.java` | 1h |
| 5 | [SEC-04] Externaliser tokens Consul ACL | `application-*.yml` | 1h |
| 6 | [BUG-03] Implémenter idempotency key | `BillingController + Service` | 1j |

### Sprint 2 — Corrections Importantes (≈ 2 semaines)

| # | Issue | Fichier | Effort |
|---|---|---|---|
| 7 | [BUG-02] Implémenter `PaymentGatewayService` | `PaymentGatewayServiceImpl.java` | 2j |
| 8 | [PERF-01+02] Cache règles + JOIN FETCH | `RuleDefinitionRepository.java` | 1j |
| 9 | [SEC-09] Logs : niveau DEBUG + masquage données | `BillingServiceImpl.java` | 3h |
| 10 | [CODE-03] Validation `@Valid` sur DTOs billing | `BillingRequestDTO.java` | 2h |
| 11 | [SEC-07] Désactiver Swagger/Actuator en prod | `application-prod.yml` | 1h |

### Sprint 3 — Qualité & Architecture (≈ 3 semaines)

| # | Issue | Effort |
|---|---|---|
| 12 | [CODE-05] Tests unitaires `BillingServiceImpl` | 3j |
| 13 | [ARCH-01] Refactorer `getValueForCriteria()` en Registry | 2j |
| 14 | [PERF-04] Configurer `ThreadPoolTaskExecutor` | 4h |
| 15 | [FUNC-01] `decisionTrace` dans la réponse | 1j |
| 16 | [ARCH-04] Activer `strict: true` TypeScript progressivement | 2j |

### Sprint 4 — Évolution Long Terme

| # | Issue | Effort |
|---|---|---|
| 17 | Migration OAuth2 ROPC → Authorization Code + PKCE | 5j |
| 18 | [ARCH-03] Migration Angular vers Standalone + Signals | 3 semaines |
| 19 | [PERF-05] Spring Cache sur Dashboard | 1j |
| 20 | [FUNC-02] Unicité contrainte priorité règles | 2h |

---

## Récapitulatif — Fichiers Prioritaires

| Priorité | Fichier | Issues |
|---|---|---|
| 🔴 | `config/ApiTokenFilter.java` | SEC-01 |
| 🔴 | `config/SecurityConfig.java` | SEC-02, SEC-03, SEC-06 |
| 🔴 | `service/impl/BillingServiceImpl.java` | BUG-01, SEC-09, PERF-01 |
| 🔴 | `resources/config/application-prod.yml` | SEC-04 |
| 🔴 | `resources/config/application-dev.yml` | SEC-04 |
| 🔴 | `frontend/src/environments/environment.ts` | SEC-05 |
| 🟠 | `repository/jpa/RuleDefinitionRepository.java` | PERF-02 |
| 🟠 | `dto/BillingRequestDTO.java` | CODE-03 |
| 🟠 | `service/impl/PaymentGatewayServiceImpl.java` | BUG-02 |
| 🟠 | `frontend/tsconfig.json` | ARCH-04 |

---

*Diagnostic réalisé sur la base d'une analyse statique complète du code source. Les corrections proposées respectent l'architecture existante.*
