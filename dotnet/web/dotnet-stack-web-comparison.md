# Comparatif des technos Web .NET (09/09/2025)
Guide de décision pour choisir entre **ASP\.NET MVC / Razor Pages**, **Blazor Server**, **Blazor WebAssembly**, et autres briques web .NET (Minimal APIs, Web API, gRPC, SignalR, Azure Functions, Blazor Web App .NET 8+).

> Objectif : fournir une synthèse structurée (avantages, limites, cas d’usage, spécificités techniques) pour justifier un choix d’architecture.

---

## Vue d’ensemble (résumé exécutif)

| Techno | Où s’exécute l’UI | Rendu initial | Interactivité | Sensible à la latence | Offline/PWA | SEO | Scalabilité & coûts serveur |
|---|---|---|---|---|---|---|---|
| **ASP\.NET MVC / Razor Pages** | Serveur (SSR) | **Très rapide** (HTML direct) | JS/îlots au besoin | Faible | Non (natif) | **Excellent** | **Très bonne** (stateless, CDN/cache) |
| **Blazor Server** | Serveur (UI + état) | **Rapide** via SSR + hydratation | **SignalR** (diff DOM côté serveur) | **Oui** (RTT) | Non | **Excellent** | **Coût par utilisateur** (1 circuit/client) |
| **Blazor WebAssembly** | Navigateur (WASM) | **Plus lourd** (runtime + DLL) | Locale (après bootstrap) | Non | **Oui (PWA)** | Moyen (mieux avec prérendu) | **Très bonne** (statique/CDN) |
| **Blazor Web App (.NET 8/9)** | Mixte par composant | **SSR** + streaming | **Server/WASM/Auto** par composant | Selon mode | WASM possible | **Excellent** | Mixte : SSR ok, Server = coût/circuit |
| **Minimal APIs / Web API** | Serveur | n/a (API) | n/a | n/a | n/a | n/a | **Très bonne** (stateless) |
| **gRPC / gRPC-Web** | Serveur & client | n/a | Streaming binaire | n/a | n/a | n/a | **Très performante** |
| **SignalR** | Serveur & client | n/a | Temps réel (WS) | Selon RTT | n/a | n/a | Dépend du nombre de connexions |
| **Azure Functions (HTTP)** | Serveurless | n/a | n/a | n/a | n/a | n/a | Élastique (cold starts possibles) |

---

## 1) ASP\.NET MVC / Razor Pages

**Description**  
Framework de rendu serveur (SSR) classique : contrôleurs + vues (MVC) ou **Razor Pages** (page-centric). Idéal pour sites publics, SEO, et montées en charge massives.

**Avantages**
- Rendu HTML **instantané** (TTFB/FCP faibles), **SEO natif**.
- **Stateless** par défaut → simple à **scaler horizontalement**, compatible **CDN/OutputCache/ETag**.
- Écosystème mature, testable, intégrable avec JS léger (**htmx/Alpine**) ou îlots **Razor Components**.
- Coûts serveurs **prédictibles** (pas de session interactive par client).

**Limites**
- UI très riche = plus de **JS** à écrire ou à encapsuler en composants/îlots.
- Moins d’état client “automatique” qu’un framework SPA.

**Cas d’usage recommandés**
- Sites/portails **publics** à fort trafic, contenu SEO, e‑commerce, marketing.
- Back-offices “classiques” sans interactions ultra fines.
- Fronts qui veulent **progressive enhancement** (SSR + îlots).

**Spécificités techniques**
- **SSR natif**, **OutputCache** (ASP\.NET Core 8/9), ETag/ResponseCaching.
- **Îlots** : Razor Components (render-mode Server/WASM) injectables dans des vues Razor.
- Auth : cookies/OIDC, BFF pattern possible pour sécuriser des appels JS.

---

## 2) Blazor Server

**Description**  
Composants Razor s’exécutant **sur le serveur**. Le navigateur garde un **circuit SignalR** (WebSocket) ; chaque événement DOM déclenche un traitement serveur et un **diff de rendu** (batch) est renvoyé.

**Avantages**
- **Productivité** : C# **full-stack** sans écrire beaucoup de JS.
- **Rendu initial rapide** via **SSR + hydratation**.
- **Accès sécurisé** aux ressources serveurs sans exposer de secrets côté client.
- Parfait sur **LAN/VPN** (latence faible).

**Limites**
- **Sensible à la latence** et aux pertes : chaque interaction = aller‑retour.
- **Coût/connexions** : 1 **circuit** par utilisateur (mémoire/CPU). Scale-out plus exigeant (souvent **Azure SignalR Service**).
- Mode offline **non** supporté.

**Cas d’usage recommandés**
- Applications **internes (LOB)** : formulaires riches, back‑office, dashboards sur réseau fiable.
- Outils métiers avec logique côté serveur et permissions fines.

**Spécificités techniques**
- **SignalR** (WebSocket) obligatoire en interactif.
- **SSR + hydratation** en .NET 8/9 ; **render modes** au niveau composant (Server).
- Optimisations : réduire événements (debounce/throttle), limiter zones interactives, héberger proche des utilisateurs, Azure SignalR.

---

## 3) Blazor WebAssembly (WASM)

**Description**  
L’app .NET tourne **dans le navigateur** via WebAssembly. Le runtime .NET et tes assemblies sont téléchargés au démarrage (ou à l’hydratation si prérendu).

**Avantages**
- **Interactions locales** après bootstrap : insensible à la latence réseau.
- **Offline/PWA** possibles (cache, Service Worker).
- Déploiement **statique** (CDN), coûts serveurs faibles (principalement des APIs).
- Même langage C# côté front/back.

**Limites**
- **Chargement initial** plus lourd (runtime + DLL). **AOT** accélère l’exécution mais **alourdit** le bundle.
- Accès direct aux ressources protégées plus délicat (tokens côté client) → privilégier **BFF** ou API sécurisée.
- SEO plus faible en SPA pure (mitigé par **prérendu SSR**).

**Cas d’usage recommandés**
- **PWA** et apps publiques nécessitant **offline**/latence locale.
- Apps orientées **widget/outil** avec peu de pages lourdes en SEO.
- Scénarios à coûts serveur minimaux (CDN).

**Spécificités techniques**
- **Prérendu** possible (SSR) pour accélérer le premier paint.
- **AOT** sélectif pour les écrans chauds, **lazy‑load** d’assemblies, trimming + Brotli.
- Auth : **BFF** (Backend for Frontend) conseillé pour éviter de manipuler des tokens longs‑vécus côté client.

---

## 4) Blazor Web App (.NET 8/9) et Razor Components

**Description**  
Template unifié avec **Razor Components** + **render modes** : **SSR** pour le premier rendu, puis interactivité **Server**, **WebAssembly** ou **Auto** **par composant**.

**Avantages**
- **Meilleur des deux mondes** : SSR rapide + choix fin de l’interactivité.
- **Granularité** : on active l’interactif uniquement où c’est utile.
- SEO et initial paint **excellents** (SSR + streaming).

**Limites**
- Complexité de **mix** (Server vs WASM) et gestion d’état entre îlots.
- Coût potentiel si on active **Server** partout (circuits).

**Cas d’usage recommandés**
- Sites avec **SEO** + quelques zones **très interactives**.
- Apps internes qui veulent combiner **fluidité LAN** (Server) et **latence locale** (WASM) sur des widgets précis.

**Spécificités techniques**
- `@rendermode InteractiveServer | InteractiveWebAssembly | Auto` par composant.
- **Hydratation** et **streaming rendering** intégrés.
- Peut cohabiter avec **MVC/Razor Pages**.

---

## 5) Minimal APIs & ASP\.NET Core Web API

**Description**  
Construction d’APIs HTTP. **Minimal APIs** proposent une surface très légère (handlers/fonctions). **Web API** (contrôleurs) offre conventions, filtres et structure plus riche.

**Avantages**
- **Perf** et **simplicité** (surtout Minimal APIs).
- Parfait pour **BFF**, microservices, et backends Blazor/SPA.
- **Stateless** → scale/CDN côté API Gateway.

**Limites**
- Pas d’UI (à combiner avec MVC/Blazor/SPA).

**Cas d’usage recommandés**
- Backend d’apps web/mobile, microservices, endpoints à fort RPS.

**Spécificités techniques**
- Endpoints `MapGet/MapPost` (Minimal APIs), binding/validation, OpenAPI/Swagger, **rate limiting**, auth (JWT/cookies).

---

## 6) gRPC & gRPC‑Web

**Description**  
RPC binaire hautes performances sur HTTP/2. Pour navigateurs, **gRPC‑Web** via proxy (Kestrel sait parler gRPC‑Web).

**Avantages**
- **Très performant** (binaire, streaming), contrats `.proto` forts.
- Idéal **service‑to‑service** et clients natifs.

**Limites**
- Navigateur = **gRPC‑Web** (quelques limitations par rapport à gRPC pur).
- Pas de SEO/HTML : à combiner avec une UI.

**Cas d’usage recommandés**
- Microservices, temps réel léger (server streaming), clients desktop/mobile performants.

---

## 7) SignalR (temps réel)

**Description**  
Abstraction temps réel (WebSocket, SSE, long‑polling). Souvent utilisé **avec** MVC/Blazor pour push/notifications.

**Avantages**
- Temps réel **simple** (broadcast, groupes, users).
- Intégration **Azure SignalR Service** pour échelle mondiale.

**Limites**
- Connexions persistantes à gérer (ressources, quota).

**Cas d’usage recommandés**
- Dashboards, chat, notifications, co‑édition légère.

---

## 8) Azure Functions (HTTP & event‑driven)

**Description**  
Serverless. Handlers déclenchés par HTTP, files, timers, messages, etc.

**Avantages**
- **Élastique** et **facturation à l’usage**.
- Idéal pour **webhooks**, jobs, intégrations, BFF simples.

**Limites**
- **Cold starts** possibles, timeouts plan tarifaire.
- Pas d’UI, pas d’état long‑vécu (sauf Durable Functions).

**Cas d’usage recommandés**
- API ponctuelles, traitements asynchrones, colle entre services.

---

## Matrice de décision rapide

- **Site public massif / SEO / coûts maîtrisés** → **ASP\.NET MVC/Razor Pages** (+ îlots Razor Components au besoin).
- **App interne LOB sur LAN/VPN (Entra ID), formulaires riches** → **Blazor Server** (SSR + hydratation, Azure SignalR pour scale).
- **PWA/offline, interactions locales après bootstrap, coûts serveur bas** → **Blazor WebAssembly** (prérendu + BFF).
- **Mix SEO + interactivité au cas par cas** → **Blazor Web App (.NET 8/9)** (SSR + `InteractiveServer/WASM` par composant).
- **Backend API** → **Minimal APIs/Web API** ; **inter‑service** → **gRPC** ; **temps réel** → **SignalR** ; **spikes/automations** → **Azure Functions**.

---

## Patterns hybrides recommandés

- **MVC/Razor Pages + îlots Razor Components** : SSR global, composants interactifs ciblés (Server ou WASM).
- **Blazor Server pour l’admin**, **MVC** pour le public.
- **BFF** en façade d’un client WASM/SPA (auth OIDC, cookies httpOnly).
- **SignalR** pour push dans un site SSR (live notifications).

---

## Performance & coûts (rappels clés)

- **SSR** améliore le **rendu initial** (TTFB/FCP/LCP).  
- **Blazor Server** : latence perçue ≈ **RTT + temps serveur**. Fluide en **LAN** ; sur Internet, privilégier `onchange` vs `oninput`, debounce, et proximités régionales.  
- **WASM** : **bootstrap** plus gros ; ensuite interactions locales. Utiliser **prérendu + lazy‑load + AOT ciblé**.  
- **Stateless** = **scale facile**. Éviter sessions serveurs si possible.  
- **Cache** : CDN + `OutputCache` + ETag sur SSR ; pour API, **caching** et pagination stricte.  
- **Coûts** : Blazor Server → **coût par utilisateur connecté** (RAM/CPU + slots WS). MVC/WASM → coût surtout **par requête**.

---

## Sécurité & identité (Microsoft Entra ID / OIDC)

- **MVC / Razor Pages / Blazor Server** : Auth cookie OIDC côté serveur ; contrôles d’accès côté back, secrets protégés.  
- **WASM/SPA** : éviter de stocker des tokens longue durée sur le client ; préférer **BFF** (cookie httpOnly + anti‑CSRF) ou appels courts via API sécurisée.  
- **Réseaux internes** : soigner **timeouts/keep‑alive** pour SignalR, et la **résilience** (reconnexion, gestion d’état).

---

## Checklists rapides

**Si tu choisis MVC/Razor Pages :**
- Activer **OutputCache** / ETag, compresser (Brotli), éviter la session serveur.  
- Ajouter des **îlots** Razor Components uniquement où nécessaire.  
- Observabilité : OpenTelemetry, health checks.

**Si tu choisis Blazor Server :**
- **SSR + hydratation** par défaut ; limiter les composants interactifs.  
- **Azure SignalR Service** pour le scale-out.  
- Réduire les événements (debounce), héberger proche des utilisateurs.

**Si tu choisis Blazor WASM :**
- **Prérendu** pour le first paint ; **lazy‑load** assemblies ; **AOT** ciblé.  
- Pattern **BFF** pour l’auth ; PWA si offline requis.

---

## En bref (TL;DR)

- **MVC/Razor Pages** : meilleur pour **public massif + SEO + coûts**.  
- **Blazor Server** : meilleur pour **interne LAN + formulaires riches + C# full‑stack** (attention au scale par connexion).  
- **Blazor WebAssembly** : meilleur pour **PWA/offline + latence locale**, charge initiale plus lourde.  
- **Blazor Web App** : **mix** intelligent de SSR + modes interactifs **au cas par cas**.  
- Compléments : **Minimal APIs/Web API**, **gRPC**, **SignalR**, **Azure Functions** selon le besoin.

