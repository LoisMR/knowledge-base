# Patterns d’échange UI ↔ ViewModel en Kotlin (Compose/XML)

> Un tour d’horizon pragmatique des patterns courants (MVVM, MVP, MVI light, MVI complet, MVU/TEA, Redux, Machine à états, Flow-first) avec **schémas de flux**, **mini-exemples Kotlin** et un **tableau comparatif**.

---

## MVVM (observer simple)

```
UI ──onClick/inputs──▶ ViewModel ──updates──▶ State(Flow)
UI ◀──────────────observe State──────────────
```

```kotlin
data class LoginState(val loading: Boolean = false, val error: String? = null)

class LoginViewModel(private val repo: AuthRepo) : ViewModel() {
  private val _ui = MutableStateFlow(LoginState())
  val ui: StateFlow<LoginState> = _ui

  fun onLogin(u: String, p: String) = viewModelScope.launch {
    _ui.value = _ui.value.copy(loading = true)
    val ok = repo.login(u, p)
    _ui.value = LoginState(loading = false, error = if (ok) null else "Oups")
  }
}

@Composable fun LoginScreen(vm: LoginViewModel) {
  val s by vm.ui.collectAsState()
  if (s.loading) CircularProgressIndicator()
  Button(onClick = { vm.onLogin("u","p") }) { Text("Login") }
  s.error?.let { Text(it) }
}
```

**Quand** : petits/moyens écrans, équipe junior, besoin de simplicité.

---

## MVP (Passive View)

```
UI(View interface) ◀── render(...) ── Presenter
          ▲                         │
          └────── UI events ────────┘
```

```kotlin
interface LoginView { fun showLoading(); fun showError(msg: String); fun showHome() }

class LoginPresenter(private val v: LoginView, private val repo: AuthRepo) {
  fun onLogin(u: String, p: String) {
    v.showLoading()
    GlobalScope.launch { // remplace par scope injecté
      if (repo.login(u,p)) v.showHome() else v.showError("Oups")
    }
  }
}
```

**Quand** : projets XML legacy, besoin de testabilité sans dépendances Android sur le Presenter.

---

## MVI “light” (UDF simple)

```
UI ── Action ──▶ ViewModel/Reducer ──▶ State ──▶ UI
                      └──(Effect?)──▶ one-shot
```

```kotlin
sealed interface Action { data class Login(val u:String,val p:String): Action }

data class State(val loading:Boolean=false, val error:String?=null)

class VM(private val repo: AuthRepo) : ViewModel() {
  private val _state = MutableStateFlow(State())
  val state: StateFlow<State> = _state

  fun dispatch(a: Action) = when(a) {
    is Action.Login -> viewModelScope.launch {
      _state.value = State(loading = true)
      val ok = repo.login(a.u,a.p)
      _state.value = State(error = if (ok) null else "Oups")
    }
  }
}
```

**Quand** : Compose + unidirectionnel, sans toute la cérémonie Intent/Reducer formalisé.

---

## MVI “complet” (Intent/Action/Reducer/Effect)

```
UI ── Intent ─▶ Mapper ─▶ Action ─▶ Reducer(State,Action) -> NewState ─▶ UI
                                   └────────── Effect (one-shot) ─────▶ IO/Nav
```

```kotlin
sealed interface Intent { data class ClickLogin(val u:String,val p:String): Intent }
sealed interface Action { data class DoLogin(val u:String,val p:String): Action }
sealed interface Effect { data class Navigate(val dest:String): Effect }

data class State(val loading:Boolean=false, val error:String?=null)

class VM(private val repo: AuthRepo): ViewModel() {
  private val _state = MutableStateFlow(State())
  val state = _state.asStateFlow()
  private val _effects = MutableSharedFlow<Effect>()
  val effects = _effects // collect in UI for one-shots

  fun onIntent(i: Intent) = when(i) {
    is Intent.ClickLogin -> handle(Action.DoLogin(i.u,i.p))
  }

  private fun reduce(s: State, a: Action) = when(a) {
    is Action.DoLogin -> s.copy(loading = true)
  }

  private fun handle(a: Action) = viewModelScope.launch {
    _state.value = reduce(_state.value, a)
    val ok = repo.login((a as Action.DoLogin).u, a.p)
    _state.value = _state.value.copy(loading = false, error = if (ok) null else "Oups")
    if (ok) _effects.emit(Effect.Navigate("home"))
  }
}
```

**Quand** : équipes/produits plus grands, besoin d’outils/debug, séparation nette des effets.

---

## MVU / TEA (Elm-like)

```
Msg ─▶ update(Model, Msg) -> (Model', Cmd?)
Cmd ──(async)──▶ produces Msg ─▶ update ...
View = f(Model)
```

```kotlin
data class Model(val loading:Boolean=false, val error:String?=null)
sealed interface Msg { data class Login(val u:String,val p:String): Msg; data class LoginResult(val ok:Boolean): Msg }
sealed interface Cmd { data class DoLogin(val u:String,val p:String): Cmd }

fun update(m: Model, msg: Msg): Pair<Model, Cmd?> = when(msg) {
  is Msg.Login -> m.copy(loading = true) to Cmd.DoLogin(msg.u,msg.p)
  is Msg.LoginResult -> m.copy(loading = false, error = if (msg.ok) null else "Oups") to null
}

class Loop(private val repo: AuthRepo) {
  val model = MutableStateFlow(Model())
  fun dispatch(msg: Msg) {
    val (m, cmd) = update(model.value, msg)
    model.value = m
    when(cmd) {
      is Cmd.DoLogin -> GlobalScope.launch {
        dispatch(Msg.LoginResult(repo.login(cmd.u, cmd.p)))
      }
      null -> {}
    }
  }
}
```

**Quand** : logique très pure, effets explicitement modélisés, bons invariants.

---

## Redux / Single Store

```
UI ── dispatch(Action) ─▶ Store ─▶ Reducer(State, Action) -> NewState ─▶ UI
                 └───────────── Middleware (effects) ────────────────┘
```

```kotlin
data class State(val loading:Boolean=false, val error:String?=null)
sealed interface Action { data class Login(val u:String,val p:String): Action; data class LoginDone(val ok:Boolean): Action }

fun reducer(s: State, a: Action) = when(a) {
  is Action.Login -> s.copy(loading = true)
  is Action.LoginDone -> s.copy(loading = false, error = if (a.ok) null else "Oups")
}

class Store(private val repo: AuthRepo) {
  private val _state = MutableStateFlow(State())
  val state = _state.asStateFlow()
  fun dispatch(a: Action) {
    _state.value = reducer(_state.value, a)
    if (a is Action.Login) GlobalScope.launch {
      dispatch(Action.LoginDone(repo.login(a.u,a.p)))
    }
  }
}
```

**Quand** : tooling fort, time-travel, plusieurs features synchronisées. Peut être verbeux.

---

## Machine à états (State Machine) / Workflow

```
[State] ──(Event)──▶ [Transition] ──▶ [New State]
              └──▶ [Effect one-shot]
```

```kotlin
sealed interface ScreenState {
  data object Idle: ScreenState
  data object Loading: ScreenState
  data class Error(val msg:String): ScreenState
  data object Success: ScreenState
}
sealed interface Event { data class Login(val u:String,val p:String): Event; data class Done(val ok:Boolean): Event }

class SM(private val repo: AuthRepo) {
  val state = MutableStateFlow<ScreenState>(ScreenState.Idle)

  fun on(e: Event) = when(val s = state.value) {
    ScreenState.Idle -> when(e) {
      is Event.Login -> {
        state.value = ScreenState.Loading
        GlobalScope.launch { on(Event.Done(repo.login(e.u,e.p))) }
      }
      else -> {}
    }
    ScreenState.Loading -> if (e is Event.Done)
      state.value = if (e.ok) ScreenState.Success else ScreenState.Error("Oups")
    else -> {}
    else -> {}
  }
}
```

**Quand** : processus complexes (checkout/KYC), transitions strictes, invariants forts.

---

## Event-driven / “Flow-first”

```
UI events (Flow) ─▶ operators (map/flatMapLatest/debounce) ─▶ State Flow ─▶ UI
```

```kotlin
class VM(private val repo: AuthRepo): ViewModel() {
  private val loginClicks = MutableSharedFlow<Pair<String,String>>() // (u,p)
  val state: StateFlow<State> =
    loginClicks.flatMapLatest { (u,p) ->
      flow {
        emit(State(loading=true))
        val ok = repo.login(u,p)
        emit(State(loading=false, error = if (ok) null else "Oups"))
      }
    }.stateIn(viewModelScope, SharingStarted.Eagerly, State())

  fun onLogin(u:String,p:String) { viewModelScope.launch { loginClicks.emit(u to p) } }
}

data class State(val loading:Boolean=false, val error:String?=null)
```

**Quand** : orchestration asynchrone riche (search, debounces), pipelines déclaratifs.

---

# Tableau comparatif (benchmark qualitatif)

| Pattern           | Courbe d’apprentissage                 | Boilerplate           | Testabilité                         | Prédictibilité du state                 | Gestion des effets        | Scalabilité équipe/app | Idéal pour                        | Pièges courants                              |
| ----------------- | -------------------------------------- | --------------------- | ----------------------------------- | --------------------------------------- | ------------------------- | ---------------------- | --------------------------------- | -------------------------------------------- |
| **MVVM**          | 🎯 Facile                              | 🟢 Faible             | 🟡 Bonne (si logique hors UI)       | 🟡 Correcte (si one-shots bien séparés) | 🟠 Parfois mêlée au state | 🟡 OK pour moyen       | Écrans simples/moyens             | Mélange state/effects, fuite de scope        |
| **MVP**           | 🟡 Moyenne                             | 🟠 Élevé (interfaces) | 🟢 Excellente (Presenter pur)       | 🟡 Correcte                             | 🟡 Explicite via View     | 🟡 OK (legacy/XML)     | Projets hérités/tests unitaires   | Verbosité, couplage View-Presenter           |
| **MVI light**     | 🟡 Moyenne                             | 🟡 Modéré             | 🟢 Bonne                            | 🟢 Bonne (unidirectionnel)              | 🟡 À séparer manuellement | 🟡 Bonne               | Compose, features isolées         | Effets one-shot oubliés, reducers impurs     |
| **MVI complet**   | 🔵 Plus raide                          | 🟠 Élevé              | 🟢 Excellente                       | 🟢 Excellente                           | 🟢 Canal d’effets dédié   | 🟢 Très bonne          | Apps/équipes large                | Sur-ingénierie sur petits écrans             |
| **MVU/TEA**       | 🔵 Raide (changement de mental modèle) | 🟡 Modéré             | 🟢 Excellente (update pur)          | 🟢 Excellente                           | 🟢 Cmd explicites         | 🟡 Bonne               | Invariants forts                  | Boilerplate Cmd/Msg                          |
| **Redux**         | 🔵 Raide                               | 🔴 Haut               | 🟢 Excellente                       | 🟢 Excellente                           | 🟢 Middleware             | 🟢 Très bonne          | Debug/time-travel, multi-features | Verbosité, action spam                       |
| **State machine** | 🟡–🔵 Variable                         | 🟡 Modéré             | 🟢 Excellente (transitions finies)  | 🟢 Excellente                           | 🟢 Effets par transition  | 🟢 Très bonne          | Processus complexes               | Explosion d’états si mal découpé             |
| **Flow-first**    | 🟡 Moyenne                             | 🟢 Faible             | 🟡 Bonne (attention aux opérateurs) | 🟡 Variable (concurrence)               | 🟡 Souvent inline         | 🟡 OK                  | Recherche, streaming              | Fuites/cancellations, opérateurs mal choisis |

**Légende** : 🟢 bon / 🟡 moyen / 🟠 élevé / 🔴 très élevé / 🔵 courbe raide

---

## Conseils pratiques transverses

* Sépare **State immuable** et **Effects one-shot** (navigation, toasts). Utilise `SharedFlow`/`Channel` ou un `Effect` dédié.
* Un **seul owner** de l’état par écran/feature. Propager des **valeurs dérivées** plutôt que recalculer partout.
* Garde les **reducers/update purs** (sans IO). Les effets asynchrones vivent ailleurs (use cases, middleware, effect handler).
* Pour Compose : préfère `StateFlow`/`SnapshotState` bien scellés (`asStateFlow()`), et une **UI stateless** qui reçoit `State` + callback d’Intent/Action.
* Documente le **contrat d’Intent/Msg** par écran (liste exhaustive des événements et states).

---
