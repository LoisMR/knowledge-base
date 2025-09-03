# 🌐 Styles d’API – Guide de référence

Ce document résume les principaux styles d’API modernes au-delà du RESTful classique, avec leurs cas d’utilisation et un comparatif pratique.

---

## RESTful
Le plus répandu : chaque ressource est représentée par une URL, et les actions se font avec les verbes HTTP (`GET`, `POST`, `PUT`, `DELETE`).

**Exemple :**
```http
GET /users/123
POST /orders
PUT /products/42
DELETE /comments/99
```

✔️ Simple, largement adopté, facile à tester avec `curl` ou Postman.  
❌ Peut mener à des requêtes multiples ou à du sur/sous-fetching.

---

## GraphQL
Un seul endpoint, et le client définit exactement quelles données il veut.

**Exemple :**
```graphql
{
  user(id: "123") {
    name
    orders {
      id
      total
    }
  }
}
```

✔️ Parfait pour les applis front riches (mobile/web) → pas de sur/sous-récupération.  
❌ Plus complexe à mettre en place côté serveur, moins adapté au cache HTTP natif.  

---

## gRPC
Basé sur Protocol Buffers (Protobuf), hautes performances, communication binaire.

**Exemple `.proto` :**
```proto
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}
```

✔️ Idéal pour microservices, IoT, streaming temps réel.  
❌ Moins accessible pour les API publiques (besoin de générer du code client).  

---

## OData
Extension de REST qui standardise filtrage, tri et pagination.

**Exemple :**
```http
GET /Products?$filter=Price gt 100&$orderby=Name desc&$top=10
```

✔️ Super pour exposer des collections de données (ERP, BI, dashboards).  
❌ URLs verbeuses, moins courant en dehors de l’écosystème Microsoft/SAP.  

---

## HATEOAS
Les réponses contiennent des **liens** vers les actions possibles → API auto-découvrable.

**Exemple :**
```json
{
  "id": 123,
  "name": "Alice",
  "links": [
    { "rel": "self", "href": "/users/123" },
    { "rel": "orders", "href": "/users/123/orders" }
  ]
}
```

✔️ Pratique si l’API évolue souvent et qu’on veut guider les clients.  
❌ Peu utilisé en pratique, complexité supplémentaire.

---

## 🔎 Tableau comparatif

| Style     | Avantages | Inconvénients | Cas d’usage typique |
|-----------|-----------|---------------|----------------------|
| **RESTful** | Simple, universel, facile à tester | Sur/sous-récupération, endpoints multiples | API publiques, web et mobiles |
| **GraphQL** | Requêtes précises, évite sur-fetching | Complexité serveur, cache HTTP compliqué | Applis front riches (Facebook, GitHub) |
| **gRPC** | Performant, streaming bidirectionnel | Besoin de tooling spécial | Microservices, IoT, API internes |
| **OData** | Filtrage/tri standardisés | URLs compliquées, peu répandu | Dashboards, ERP, data APIs |
| **HATEOAS** | API auto-découvrable | Lourdeur, peu adopté | API évolutives, hypermedia-driven |

---

## 🏅 Mentions honorables

### RPC (Remote Procedure Call)
Endpoints = actions (style fonction).  
```http
POST /resetPassword
POST /calculateInvoice
```

### SOAP (Simple Object Access Protocol)
Ancien protocole XML encore utilisé dans certains secteurs (banques, assurances).  
```xml
<soap:Envelope>
  <soap:Body>
    <GetUser>
      <UserId>123</UserId>
    </GetUser>
  </soap:Body>
</soap:Envelope>
```

---

✍️ **Résumé :**
- REST est le standard par défaut.  
- GraphQL brille pour le front et la flexibilité.  
- gRPC domine pour la performance et le microservice.  
- OData pour la data “à la sauce SQL”.  
- HATEOAS pour les puristes REST qui veulent de l’auto-découverte.  
- RPC et SOAP : toujours là, mais plus niche.
