# Lineamiento (propuesta): Cubits sin `BuildContext` — State vs Effects

> **Estado:** propuesta interna. No está enlazada aún desde `AGENTS.md` ni otras entradas de agentes; conviene validar con el equipo antes de adoptarla como regla obligatoria.

## Contexto y objetivo

En el proyecto hay cubits que usan `BuildContext` para:

- Mostrar snackbars / modales
- Cambiar idioma (`setLocale`)
- Navegación
- (A veces) pasar `context` dentro de estados o servicios

Esto genera acoplamiento fuerte UI ↔ lógica, dificulta pruebas y hace los widgets ruidosos (muchos `if` / `try` / `catch` / listeners dispersos).

### Objetivos del lineamiento

- Cubits dedicados a **lógica y estado** (sin `BuildContext`).
- Widgets **legibles** (principalmente composición + `BlocBuilder`).
- Side-effects de UI (snackbar, navegación, locale, diálogo) **centralizados y consistentes**.
- Refactor **progresivo**, por feature, sin reescribir todo.

## Regla de oro

**Un cubit no debe depender de `BuildContext`.**

### Permitido en cubits

- Reglas de negocio y validaciones
- Mapeo DTO ↔ entidad de dominio
- Llamadas a servicios / repositorios (contratos de dominio)
- Manejo de concurrencia (debounce, cancel, retry)
- Emisión de **State** (datos replayables de la pantalla)
- Emisión de **intenciones de UI** vía **Effects** (ver más abajo)

### Prohibido en cubits

- `BuildContext`
- `Navigator` / `ScaffoldMessenger` / `showDialog`
- `context.setLocale(...)`
- `.tr()` / `.translate()` dentro del cubit (las traducciones se resuelven en la capa con contexto de localización)

### Nota sobre “context oculto”

Utilidades históricas como `NavigationService.navigatorKey.currentContext` **siguen siendo UI**: en la práctica acoplan la lógica al árbol de Flutter. Este lineamiento apunta a **eliminarlas de forma progresiva**.

---

## Modelo mental: State vs Effect

### State

Representa **cómo está la pantalla** y debe ser en lo posible **replayable**:

- `loading` / `loaded` / `error`
- Datos actuales (listas, formularios, flags, etc.)

### Effect (one-off)

Representa **“haz algo una vez”** y **no** debería quedar “pegado” como estado persistente de la pantalla:

- Mostrar snackbar
- Abrir modal
- Navegar
- Cambiar locale

Los efectos se **consumen en la UI**, donde sí existe `BuildContext` válido.

---

## Propuestas de arquitectura

### Opción A — Effects por cubit (stream interno)

Cada cubit expone su propio `effectsStream` (o equivalente).

**Cuándo elegirla**

- Migración 100% incremental por feature.
- Se aceptan **listeners por feature**, idealmente envueltos en un **widget wrapper** para no ensuciar cada pantalla.

**Ventajas**

- Muy fácil de introducir y migrar por módulos.
- No exige un bus global desde el día uno.

**Riesgos**

- Si cada pantalla monta su propia suscripción sin wrappers, se **multiplican** listeners y es difícil de auditar.

**Implementación recomendada (matiz)**

- Evitar `StreamBuilder` solo con `snapshot.data` para efectos one-off (puede **re-dispararse** o comportarse mal entre rebuilds).
- Preferir: `StatefulWidget` con `StreamSubscription` en `initState` + cancel en `dispose`, o un wrapper dedicado que encapsule la suscripción, o patrones probados del ecosistema (p. ej. librerías de “effects” para bloc si se adoptan).

**Ejemplo simplificado**

```dart
sealed class UiEffect {
  const UiEffect();
}

class ShowSnackbar extends UiEffect {
  final String messageKey;
  final SnackbarType type;
  final Map<String, String>? args;
  const ShowSnackbar(this.messageKey, {required this.type, this.args});
}

enum SnackbarType { info, success, error }

mixin UiEffectsEmitter on Cubit<Object> {
  final _effects = StreamController<UiEffect>.broadcast();
  Stream<UiEffect> get effectsStream => _effects.stream;

  void emitEffect(UiEffect effect) => _effects.add(effect);

  Future<void> closeEffects() async => _effects.close();

  @override
  Future<void> close() async {
    await closeEffects();
    return super.close();
  }
}

class PreferencesCubit extends Cubit<PreferencesState> with UiEffectsEmitter {
  PreferencesCubit() : super(PreferencesInitial());

  Future<void> save() async {
    emit(PreferencesLoading());
    // ... persistencia vía servicio ...
    emit(PreferencesSaved());
    emitEffect(const ShowSnackbar('preferences.messages.saved', type: SnackbarType.success));
  }
}
```

**Consumo en UI (ideal: wrapper por feature)**

```dart
class PreferencesEffectsListener extends StatefulWidget {
  const PreferencesEffectsListener({super.key, required this.child});
  final Widget child;

  @override
  State<PreferencesEffectsListener> createState() => _PreferencesEffectsListenerState();
}

class _PreferencesEffectsListenerState extends State<PreferencesEffectsListener> {
  StreamSubscription<UiEffect>? _sub;

  @override
  void initState() {
    super.initState();
    final cubit = context.read<PreferencesCubit>();
    _sub = cubit.effectsStream.listen((effect) {
      if (!mounted) return;
      switch (effect) {
        case ShowSnackbar():
          final msg = effect.messageKey.tr(namedArgs: effect.args);
          ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(msg)));
      }
    });
  }

  @override
  void dispose() {
    _sub?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

---

### Opción B — Bus central de efectos (recomendada como eje global)

Un **`UiEffectsCubit`** (o servicio equivalente) recibe efectos y **un solo listener en la raíz** los ejecuta.

**Cuándo elegirla**

- Se quieren widgets **muy limpios** a nivel app.
- Ya existen patrones globales (p. ej. navegación centralizada, errores/snackbars desde capas base) y se quieren **ordenar** en un único punto.
- Hay o habrá migración a **GoRouter**: el listener raíz cambia el “driver” sin tocar los cubits.

**Ventajas**

- Un único punto de control para snackbar / diálogo / navegación / locale.
- Estándar transversal para todo el equipo.

**Riesgos**

- Puede convertirse en un “dios” si el modelo de `UiEffect` es **poco tipado** o el `switch` crece sin límite.

**Mitigaciones**

- `sealed class UiEffect` con jerarquía clara.
- Pocos efectos **genéricos**; payloads tipados; para casos muy específicos valorar **Opción A** en ese feature o un subtipo parametrizado (`ShowFeatureDialog(AlarmDialogId.x, payload)`).
- **Cola o batch de efectos:** si en un mismo método se encolan varios `push`, un `Cubit<UiEffect?>` con `emit(null)` entre medias puede **perder** el anterior. Diseñar desde el inicio: cola interna, `emit(UiEffectBatch([...]))`, o un pequeño buffer hasta que el listener haga `clear`.

**Ejemplo de bus**

```dart
sealed class UiEffect {
  const UiEffect();
}

class ShowSnackbar extends UiEffect {
  final String messageKey;
  final SnackbarType type;
  final Map<String, String>? args;
  const ShowSnackbar(this.messageKey, {required this.type, this.args});
}

class ChangeLocale extends UiEffect {
  final Locale locale;
  const ChangeLocale(this.locale);
}

class NavigateTo extends UiEffect {
  final String routeName;
  final Object? args;
  const NavigateTo(this.routeName, {this.args});
}

class UiEffectsCubit extends Cubit<UiEffect?> {
  UiEffectsCubit() : super(null);

  void push(UiEffect effect) => emit(effect);
  void clear() => emit(null);
}
```

**Listener único en root**

```dart
class AppEffectsListener extends StatelessWidget {
  const AppEffectsListener({super.key, required this.child});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return BlocListener<UiEffectsCubit, UiEffect?>(
      listenWhen: (prev, curr) => curr != null,
      listener: (context, effect) {
        if (effect == null) return;

        switch (effect) {
          case ShowSnackbar():
            final msg = effect.messageKey.tr(namedArgs: effect.args);
            ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(msg)));
          case ChangeLocale():
            context.setLocale(effect.locale);
          case NavigateTo():
            Navigator.of(context).pushNamed(effect.routeName, arguments: effect.args);
        }

        context.read<UiEffectsCubit>().clear();
      },
      child: child,
    );
  }
}
```

**Cubit expresa intención sin context**

```dart
class SomeCubit extends Cubit<SomeState> {
  SomeCubit(this._ui) : super(SomeInitial());
  final UiEffectsCubit _ui;

  void onSaveSuccess() {
    _ui.push(const ShowSnackbar('common.saved', type: SnackbarType.success));
    _ui.push(const NavigateTo('/home'));
  }
}
```

---

## Híbrido recomendado (propuesta operativa)

| Situación | Enfoque |
|-----------|---------|
| Snackbar genérico, navegación de app, locale, sesión | **Opción B** (bus + listener raíz) |
| Efectos muy específicos de un feature (muchos modales de un wizard) | **Opción A** (stream + wrapper del feature) **o** efectos parametrizados en B con mapa acotado |

No se contradicen: B da **columna vertebral**; A evita inflar el bus con ruido local.

---

## Navegación y GoRouter

Los cubits emiten **intención** (`NavigateTo(routeName, args)`), no llaman al router.

- El **listener raíz** decide si usa `Navigator` o `GoRouter`.
- Tras migrar: `context.go(effect.routeName, extra: effect.args)` (o la API que definan), **sin** cambiar firmas de negocio en cubits.

---

## Idioma: fuente de verdad vs aplicación en UI

El idioma puede cambiar por: preferencias en backend/DB, login, refresh de sesión, remote config o UI.

### Recomendación

1. **`LocaleCubit` (o `AppSettingsCubit`)** como fuente de verdad del `Locale` actual.
2. Cualquier origen (servicio, otro cubit) **actualiza** ese cubit; **ninguno** llama directamente `context.setLocale`.
3. Un **`BlocListener`** en la raíz aplica: `context.setLocale(state.locale)`.
4. Features con datos traducidos desde backend (p. ej. alarmas): escuchan cambios de locale (`distinct` por `languageCode`) y **refrescan** con debounce, sin `BuildContext` en el cubit de dominio/presentación de datos.

```dart
class LocaleState {
  final Locale locale;
  const LocaleState(this.locale);
}

class LocaleCubit extends Cubit<LocaleState> {
  LocaleCubit() : super(const LocaleState(Locale('es')));

  void setLocale(Locale locale) => emit(LocaleState(locale));
}
```

```dart
BlocListener<LocaleCubit, LocaleState>(
  listener: (context, state) => context.setLocale(state.locale),
  child: /* ... */,
);
```

```dart
// En un cubit que debe recargar datos al cambiar idioma
_localeSub = localeCubit.stream
    .map((s) => s.locale.languageCode)
    .distinct()
    .listen((_) => getAlarms());
```

---

## Traducciones

- Los cubits emiten **keys** (`'errors.no_internet'`), la UI hace `.tr()` / `.translate()`.
- Evita textos hardcodeados en efectos que luego quedan desalineados si el locale cambia después del evento.

---

## Refactor progresivo (sugerido)

### Paso 0 — Congelar deuda

A partir de la adopción del lineamiento: **no** añadir nuevo `BuildContext` en cubits nuevos.

### Paso 1 — Contrato e infraestructura

- Definir `UiEffect` tipado (snackbar, navegación, locale como mínimo).
- Crear `UiEffectsCubit` e instalar `AppEffectsListener` en el root.
- (Opcional) `LocaleCubit` + listener raíz que aplique `setLocale`.
- Criterio: app compila; UX igual si aún no se migran llamadas.

### Paso 2 — Un flujo piloto (p. ej. preferencias / idioma)

- Fuente de verdad → `LocaleCubit` o `ChangeLocale` vía bus.
- Cubits que dependen de backend traducido reaccionan al locale **sin** context.

### Paso 3 — Reducir ruido en widgets

- Sustituir `BlocConsumer` muy cargado en listener por **listener central** + widgets con `BlocBuilder` y design system.

### Paso 4 — `BaseCubit` y capas base

- Dejar de ejecutar UI directa; emitir efectos o estado de error con keys.
- Modo híbrido temporal si hace falta compatibilidad.

---

## ¿Cuándo A vs B? (decisión rápida)

**Opción A** si: se prioriza avanzar por feature sin cambio global coordinado; el equipo prefiere evitar un bus compartido; se aceptan wrappers por feature.

**Opción B** si: se quieren vistas muy limpias; ya hay dependencias globales que conviene **unificar**; hay o habrá **GoRouter**; la navegación debe ser intención en cubit y ejecución en un solo “driver”.

**Regla práctica:** efecto **global** o transversal (locale, nav, snackbars genéricos) → tiende a **B**. Efecto **muy local** a un feature → **A** o efecto parametrizado en B.

---

## Checklist para PRs (cuando se adopte)

- [ ] El cubit no usa `BuildContext`.
- [ ] El cubit no hace `.tr()` / `.translate()` con strings de producto.
- [ ] Las acciones de UI salen como `UiEffect` (o stream de efectos del feature).
- [ ] La UI es la única que ejecuta `setLocale`, snackbars, navegación, `showDialog`.
- [ ] Features con datos traducidos desde backend refrescan al cambiar locale (stream + debounce).

---

## Referencia cruzada (documentación existente)

- [presentation-layer.md](presentation-layer.md) — responsabilidades de presentación y cubits.
- [architecture-refactoring-and-feature-model.md](architecture-refactoring-and-feature-model.md) — análisis de capas, sesión y refactors alineados.
