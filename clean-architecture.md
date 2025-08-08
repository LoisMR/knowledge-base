# 🏗 Clean Architecture – Structure du projet

Ce projet suit les principes de la **Clean Architecture**, qui vise à séparer clairement les responsabilités en couches indépendantes. Chaque couche a un rôle bien défini et ne dépend que des couches plus internes.

---

## 📐 Schéma global

```
+------------------+
|      UI          |  <- Android / Compose
| (Screens, VM)    |
+--------+---------+
         |
         v
+------------------+
|     Domain       |  <- Logique métier pure
| (Models, RepoInt)|
+--------+---------+
         |
         v
+------------------+
|      Data        |  <- Implémentations concrètes
| (RepoImpl, Mappers)
+---+----------+---+
    |          |
    v          v
+-------+  +--------+
| Local |  | Remote |
| (DB)  |  | (API)  |
+-------+  +--------+

[Common] : utilitaires partagés accessibles à toutes les couches
```

Flux de dépendances :

```
UI  ->  Domain  ->  Data  ->  Remote / Local
↑
└---- Common (utils)
```

---

## 🗂 Description des dossiers

### 1. **common/**

Fonctions utilitaires et extensions utilisées dans toutes les couches.

* `Anys.kt`, `Flows.kt`, `Utils.kt` : Helpers génériques.

---

### 2. **data/**

Implémentations concrètes de la couche **Domain**.
Contient la logique d’accès aux données (locale et distante).

* **local/** :

  * `objects/` : Entités de base pour la base locale (`BusObject`, `ReviewObject`).
  * `BusDao.kt`, `ReviewDao.kt` : Interfaces DAO pour Room.
  * `RMDatabase.kt` : Base de données Room.

* **remote/** :

  * `repositories/` : Implémentations concrètes des repositories définis dans le domaine (`BusRepositoryImpl`, `ReviewRepositoryImpl`).
  * `DataModule.kt` : Injection de dépendances / configuration data.

---

### 3. **domain/**

Cœur métier de l’application (indépendant de tout framework).

* Entitées :

  * `models/` : Contient l'entité et tous les modèles associés.
  * `XXXRepository` : Interfaces définissant les contrats d’accès aux données.

---

### 4. **ui/**

Couche de présentation (Android/Compose).

* **core/** :

  * `composables/` : Composants UI réutilisables (`Screen.kt`).
  * `navigation/` : Gestion des destinations et de la navigation (`Destination`, `Navigation.kt`).
  * `theme/` : Thème et style Compose.
  * `ViewModel` : Base commune pour les ViewModels.

* **screens/** :

  * `home/` : Écran Home (`HomeScreen.kt`) et son ViewModel (`HomeViewModel.kt`).

* `App.kt` : Entrée Compose de l’application.

* `MainActivity.kt` : Point d’entrée Android.

---

## 🔄 Communication entre couches

1. **UI → Domain**

   * Les ViewModels déclenchent des *use cases* ou appellent les interfaces `Repository` définies dans **domain**.

2. **Domain → Data**

   * Les `Repository` du domaine sont implémentés dans **data**, qui orchestre les sources locales et distantes.

3. **Data → Local/Remote**

   * `local/` : stockage persistant (Room).
   * `remote/` : API, services réseau.

4. **Common**

   * Fournit des utilitaires partagés accessibles à toutes les couches (ex: extensions Kotlin, helpers Flow).

---

## 📌 Règles clés

* La **couche Domain** ne dépend d’aucune autre (pas d’Android SDK).
* La **couche Data** dépend de Domain mais pas de UI.
* La **couche UI** dépend de Domain et peut observer directement les données exposées par les ViewModels.
* Les dépendances sont unidirectionnelles, du haut (UI) vers le bas (Local/Remote).
