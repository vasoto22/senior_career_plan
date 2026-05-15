# Análisis de la capa `presentation/` — feature `vehicle`

Documento único de revisión para `lib/features/vehicle/presentation/`, evaluado contra:

- Principios de **Clean Code** y **Clean Architecture**.
- Atomic Design.
- Lista de sugerencias del equipo (jerarquía, responsabilidades de cubits/widgets, design system, naming, sufijo `Page`, dependencias).

> Alcance: solo carpeta `presentation/`. No se propone reescribir negocio; se identifican fugas hacia presentación, acoplamientos y deuda estructural.

> Severidades: **🔴 alta** (rompe arquitectura o tiene bug), **🟠 media** (acoplamiento o legibilidad), **🟡 baja** (mejora menor).

---

## 0. Inventario actual

### Estructura de carpetas

```text
lib/features/vehicle/presentation/
├── presentation.dart          # barrel
├── extensions/
│   └── build_context.dart
├── utils/
│   └── vehicle_event_dispatcher.dart   # singleton con RxDart
├── view_models/
│   └── vehicle_filter_view_model.dart
├── vehicle/                   # submódulo: lista
│   ├── cubit/  (5 cubits: list, item, order, pinned, action_service)
│   ├── pages/  vehicle_list_page.dart   → clase VehiclesListPage
│   └── widgets/ atoms / molecules / organisms / vehicle_item_builder.dart (suelto)
├── vehicle_detail/            # submódulo: detalle
│   ├── cubit/  vehicle_detail_cubit
│   ├── pages/  vehicle_detail_page.dart
│   └── widgets/ molecules / organisms      ← sin atoms
├── vehicle_filter/            # submódulo: filtros
│   ├── cubit/  list/{gr,vh}, result, state    ← subcarpetas por flujo
│   └── widgets/ atoms / molecules / organisms / templates
├── vehicle_map/               # submódulo: mapa
│   ├── pages/  vehicle_map_page.dart    → clase MapPage
│   └── widgets/ atoms / organisms       ← sin molecules
└── vehicle_update/            # submódulo: edición alias
    ├── cubit/, pages/, utils/, widgets/{atoms,molecules,organisms}
```

### Cubits identificados

| Cubit | Líneas aprox. | Importa Material/UI | BuildContext en API | Acceso a snackbars/modals |
|-------|---------------|---------------------|---------------------|----------------------------|
| `VehicleListCubit` | 240 | **Sí** (`flutter/material.dart`) | No | No |
| `VehicleItemCubit` | 180 | No | No | No |
| `VehicleOrderCubit` | 50 | **Sí** (`flutter/material.dart`) | No | No |
| `VehiclePinnedCubit` | 50 | No | No | No |
| `VehicleServiceCubit` (acción) | 35 | No | No | No |
| `VehicleDetailCubit` | 55 | No | No | No |
| `VehicleUpdateCubit` | 95 | No (usa `core/design_system`) | No | **Sí** (`SnackBarControl`) |
| `VehicleFilterStateCubit` | 220 | No | No | No |
| `VehicleFilterListGroupCubit` | 140 | No | No | No |
| `VehicleFilterListVehicleCubit` | 110 | No | No | No |
| `VehicleFilterResultCubit` | 130 | No | No | No |

### Páginas (clases con sufijo `Page`)

| Archivo | Clase | Registrada en `PageConstants` | Ruta dinámica |
|---------|-------|-------------------------------|----------------|
| `vehicle_list_page.dart` | `VehiclesListPage` (plural) | No (vive embebida en `home`) | No |
| `vehicle_detail/pages/vehicle_detail_page.dart` | `VehicleDetailPage` | `vehicleDetail = '/vehicle_detail'` | Sí |
| `vehicle_update/pages/vehicle_update_page.dart` | `VehicleUpdatePage` | `vehicleUpdate = '/vehicle_update'` | Sí |
| `vehicle_map/pages/vehicle_map_page.dart` | **`MapPage`** | No (tab embebido) | No |

---

## 1. Cubits con lógica que no aplica

### 1.1 `VehicleListCubit` 🔴

[lib/features/vehicle/presentation/vehicle/cubit/vehicle_list_cubit.dart](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_list_cubit.dart)

Problemas:

1. **Mezcla 4 responsabilidades** en un solo cubit:
   - Cargar lista de vehículos.
   - Suscribirse a 4 streams distintos (`eventStream`, `eventTripStream`, `eventEmitterStream`, `eventPlatformStream`).
   - Hacer fan-out a `VehicleEventDispatcher` (pertenece a infra/dominio).
   - Enriquecer con `safetyRouteService.enrichVehicles(...)` y `liveModeService.enrichVehiclesWithLiveMode(...)` directamente.
2. **Auto-interrogación**: mantiene la lista `_vehiclesAutoInterrogated` (estado de UI/anti-spam) acoplada al ciclo de vida del listado.
3. Emite `VehicleUpdated()` (estado sin payload) tras cada evento para forzar rebuild → señal de que se está usando estado como event bus.
4. Importa `package:flutter/material.dart` sin necesidad real (solo `@immutable`).

Acción:

- Sustituir el import por `package:flutter/foundation.dart`.
- Mover `VehicleEventDispatcher().dispatch*` fuera del cubit (a infra o servicio de dominio).
- Mover suscripciones a un `VehicleEventsCubit` o servicio de dominio.
- Lista `_vehiclesAutoInterrogated` → servicio de dominio `AutoInterrogateTracker`.

División sugerida:

| Cubit | Responsabilidad |
|-------|-----------------|
| `VehicleListCubit` | `getVehicles`, `onResume`, `updateVehiclesAfterReorder`. |
| `VehicleEventsCubit` (nuevo) | Suscripciones a streams y disparo de updates. |
| Servicio dominio | Fan-out de eventos y tracker de auto-interrogación. |

---

### 1.2 `VehicleItemCubit` 🟠

[lib/features/vehicle/presentation/vehicle/cubit/vehicle_item_cubit.dart](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_item_cubit.dart)

Problemas:

1. **Muta la entidad de dominio** directamente: `_vehicle.name = newName` antes de confirmar respuesta del servidor (rollback manual luego). Debería trabajar con `copyWith` y emitir nuevos estados inmutables.
2. **3 suscripciones a streams** en el constructor sin un método `start()` explícito. Las cancela en `close()`, pero el contrato no es claro.
3. `getLastLocation()` retorna `VehicleEntity?` (rompe el patrón de cubit: debería emitir estado, no devolver datos).
4. Métodos sin tipo de retorno: `connectToEvents`, `connectPlatformToEvents`, `connectToThirdPartyEvents`, `_emitUpdateVehicle`. Hay además `await _emitUpdateVehicle()` sobre función `void`.
5. Magic numbers: `Duration(milliseconds: 300)` repetido en throttles.

Acción: tipos de retorno explícitos, `_streamThrottle` como constante, `copyWith` en lugar de mutación.

---

### 1.3 `VehicleOrderCubit` 🟠

[lib/features/vehicle/presentation/vehicle/cubit/vehicle_order_cubit.dart](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_order_cubit.dart)

Problemas:

1. Importa `package:flutter/material.dart` solo por `@immutable`.

Acción: cambiar a `package:flutter/foundation.dart`.

---

### 1.4 `VehicleUpdateCubit` 🔴

[lib/features/vehicle/presentation/vehicle_update/cubit/vehicle_update_cubit.dart](../lib/features/vehicle/presentation/vehicle_update/cubit/vehicle_update_cubit.dart)

Problemas:

1. **Llama a `SnackBarControl` desde el cubit** (líneas 60–85). Mover hacia `BlocListener` en la page.
2. Mezcla 2 responsabilidades: validación de nickname (`validateNickname`, `isNicknameValid`) + flujo de update. La validación delega a `VehicleUpdateValidationUtils`, pero el cubit aún expone ambos métodos a la UI.
3. Acepta `VehicleListCubit?` por constructor (dependencia entre cubits hermanos). Mejor: emitir un estado y que la page le pida a `VehicleListCubit` refrescar.
4. `currentSaveEnabled` se recalcula tras error pero el snackbar ya se mostró; no hay garantía de orden.

Refactor sugerido:

```dart
// Estado
class VehicleUpdateInitial extends VehicleUpdateState {
  final bool isSaveEnabled;
  final VehicleUpdateMessageKey? message;   // success | duplicate | error | null
  // ...
}

// Page (BlocListener)
BlocListener<VehicleUpdateCubit, VehicleUpdateState>(
  listenWhen: (p, c) => c is VehicleUpdateInitial && c.message != null,
  listener: (context, state) {
    final s = state as VehicleUpdateInitial;
    switch (s.message!) {
      case VehicleUpdateMessageKey.success:
        SnackBarControl.showSnackBar(message: 'vehicles.update.success'.translate(), ...);
      case VehicleUpdateMessageKey.duplicate:
        SnackBarControl.showSnackBarError(...);
      case VehicleUpdateMessageKey.error:
        SnackBarControl.showSnackBarError(...);
    }
  },
)
```

---

### 1.5 `VehicleFilterStateCubit` 🔴

[vehicle_filter_state_cubit.dart](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_state/vehicle_filter_state_cubit.dart)

Problemas:

1. **Decide `IconData`, `Color` y `backgroundSelectedColor` para cada filtro** (`_getFiltersList`, líneas 110–165). Los view models contienen tipos de Flutter → cubit acoplado a `material`/design system.
2. Método `_getFiltersList()` de ~55 líneas con 6 entradas hardcodeadas; cualquier nuevo estado obliga a modificar el cubit.
3. `initSubscriptions()` se invoca desde el widget (`VehicleFiltersMenu.initState`). El cubit debería ser dueño de su ciclo de vida.
4. Mantiene `vehiclesDashcam` cacheado en propiedad pública.
5. `subscription!.cancel()` puede fallar si `subscriptionListFiltered` quedó null.

Acción: la construcción del view model con íconos/colores → mapper en `presentation/view_models/vehicle_filter_view_factory.dart`. El cubit emite enums (`VehicleStateEnum`, `VehicleServiceEnum`) + conteos; el widget aplica el ícono/color.

---

### 1.6 `VehicleFilterResultCubit` 🟠

[vehicle_filter_result_cubit.dart](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_result/vehicle_filter_result_cubit.dart)

Problemas:

1. **Construye copy con i18n keys (`'vehicles.filter.state'`, `'vehicles.filter.group'`, `'vehicles.filter.vehicle'`) directamente en el cubit** dentro de `_resetToFilterByState` / `_resetToFilterByVehicles`.
2. **Muta in-place** los `VehicleFilterViewModel` del estado (`filter.selected = true; filter.label = ...`) en lugar de emitir copias. Rompe el contrato “estados inmutables”.
3. `filterByState`, `filterByVehicles` mezclan: actualización de selecciones + cálculo de resultados + emisión. Cada uno tiene 20+ líneas con ramificaciones.
4. `log(size.toString())` log perdido en `filterByState`.

Acción:

- Mover textos a un mapper de presentación.
- Sustituir mutaciones por `copyWith` en `VehicleFilterViewModel`.
- Dividir `filterByVehicles` en pasos privados claros: `_selectCurrent`, `_clearOtherType`, `_applyState`, `_compute`, `_emit`.

---

### 1.7 `VehicleFilterListGroupCubit` y `VehicleFilterListVehicleCubit` 🟠

[vehicle_filter_list_gr_cubit.dart](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_list/vehicle_filter_list_gr_cubit.dart), [vehicle_filter_list_vh_cubit.dart](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_list/vehicle_filter_list_vh_cubit.dart)

Problemas:

1. `buildItemText(...)` construye **texto de UI** (`"$name(${vehicles.length})"`) en el cubit. Mover a widget o view model.
2. `String all = 'vehicles.filter.all'.translate();` como propiedad **mutable** del cubit, evaluada en construcción → riesgo si cambia el `Locale` después. Mover a constante o helper.
3. `VehicleFilterListVehicleCubit` mantiene flags `isFiltering` y dos listas (`list`, `filteredList`); `emitResultsBasedOnFilter` decide cuál emitir. Modelar como **un único stream de “lista mostrada”** evitaría esta duplicidad.
4. Métodos `getList`, `filterByText`, `onCheckChanged`, `handleCheckChange`, `_uncheckAll`, `_checkAllIfAllChecked` exponen lógica de “seleccionar todo”. Puede vivir en un servicio compartido `MultiSelectController<T>` o en `VehicleFilterListCubit` base.
5. Estado con typo: `VechielFilterListFailure` (5 ocurrencias).

---

### 1.8 `VehicleDetailCubit` 🟠

[vehicle_detail_cubit.dart](../lib/features/vehicle/presentation/vehicle_detail/cubit/vehicle_detail_cubit.dart)

Problemas:

1. **Estados como eventos**: `VehicleDetailCommandVisible`, `TakeScreenshot` son señales para la UI, no estados. Usar `StreamController` de efectos o un `BlocListener` que reaccione a `VehicleDetailState` con un `Action` enum.
2. `getFueLevel()` consulta `UserInfoService` y `FuelLevelService` (2 servicios), valida `hasVehicleAvailableService`, y emite. Esa decisión de “si tiene servicio” puede vivir en dominio (`CanLoadFuelLevelUseCase`).
3. Hardcodea `measureUnits: 'Kilometros'` (string literal); ya hay `GlobalSettings.measureUnits`.

---

### 1.9 `VehicleServiceCubit` (acciones) 🟡

[vehicle_action_service_cubit.dart](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_action_service_cubit.dart)

Problemas:

1. Dos métodos hacen casi lo mismo (`getActionService`, `isServiceActive`) pero emiten estados distintos (`VehicleServiceEnabled`, `VehicleHasService`). Aclarar nombres o unificar.
2. `emitError` y `emitWhenNoInternet` vacíos: errores silenciados.

---

## 2. Widgets con lógica que no aplica

### 2.1 `VehicleItem` 🔴

[vehicle_item.dart](../lib/features/vehicle/presentation/vehicle/widgets/organisms/vehicle_item.dart)

Lógica fuera de lugar:

| Método | Problema | Mover a |
|--------|----------|---------|
| `_shouldAutoInterrogate(vehicle, context)` (líneas 70–84) | Reglas de negocio: `isSatrackProvider`, `noSignal && noReport`, `inMinutes > 2`, ya interrogado. | Servicio de dominio `AutoInterrogateService.canInterrogate(vehicle)`. |
| `_navigateToDetailPage` (líneas 90–105) | Decide reglas (`actorTrip == thirdWithAssignedTrip`, `vehicleState != noReport`) → es regla de UX/negocio. | Extraer `VehicleNavigationGuard.canNavigateToDetail(vehicle)`. |
| `_showMessageVehicleInRouteWithOtherCompany` (líneas 200–230) | Construye un diálogo grande inline en el widget. | Extraer a un widget propio `VehicleUnavailableDialog`. |
| Creación de `InterrogateCubit`, `VehiclePinnedCubit` en `initState` con `locator.get(...)` | Composición de cubits dentro de un atom/molecule. | Subir a la Page (`VehicleListPage`) o a `VehicleItemBuilder`. |
| `_isNavigating` como debounce manual | Reinicia inmediatamente; el “debounce” no funciona. | Usar `InkWell` deshabilitado o un `Throttler` real. |
| Magic numbers (`Duration(seconds: 2)`, `inMinutes > 2`) | Sin nombre. | Constantes nombradas. |

**Bug 🟠:** el “debounce” no debouncea: `_isNavigating = true; navigate; _isNavigating = false;` se ejecuta sincrónico.

---

### 2.2 `VehicleItemBuilder` 🟠

[vehicle_item_builder.dart](../lib/features/vehicle/presentation/vehicle/widgets/vehicle_item_builder.dart)

- Es un **proveedor de cubits** disfrazado de widget. Nombre engañoso: no construye nada visualmente; solo monta `VehicleLiveModeCubit` y `VehicleItemCubit`. Renombrar a `VehicleListItemProvider` y dejarlo en `widgets/templates/` o `widgets/organisms/`.
- Vive suelto en `widgets/`, fuera del nivel atómico.

---

### 2.3 `OptimizedVehicleItem` 🟠

[optimized_vehicle_item.dart](../lib/features/vehicle/presentation/vehicle/widgets/organisms/optimized_vehicle_item.dart)

- Implementa **cache manual de un widget** (`_cachedWidget`) y reglas de invalidación basadas en propiedades de `VehicleEntity` (`_hasRelevantChanges`). Esto es lógica de optimización que duplica la responsabilidad de `BlocBuilder.buildWhen` ya presente en `VehicleListCubit`. Evaluar si es realmente necesaria — si el rebuild se controla con `buildWhen` correcto y `ValueKey`, esta clase sobra.
- Si se mantiene, mover `_hasRelevantChanges` a un extension method o equality del `VehicleEntity` (en dominio).
- Naming: el sufijo `Optimized` describe implementación, no rol → `VehicleListItem`.

---

### 2.4 `VehicleActionServices` 🟠

[vehicle_enabled_action_services.dart](../lib/features/vehicle/presentation/vehicle/widgets/molecules/vehicle_enabled_action_services.dart)

- `_showActionServices(context)` se invocaba dentro de `build`, disparando emisión del cubit en cada rebuild. **Anti-patrón**: mover a `initState` con `addPostFrameCallback`.
- `_showDashcamMenu` construye 3–4 `DashcamMenuViewModel` con condicionales (`cameraNosignal`, `ignition == 0`, `services.any(...)`) → reglas de presentación + producto. Mover a una factoría `DashcamMenuFactory.buildOptions(vehicle, services)`.
- `_getOptionDashcam`, `_getOptionOndemand`, `_getSettingOption`, `_getOptionDataUsage` son builders de view models dentro del widget. Llevarlos a `presentation/view_models/` o a la propia feature `dashcam/`.
- Naming: el widget pinta el menú de dashcam, no “services” → `VehicleDashcamActionsMenu`.

---

### 2.5 `VehicleItemBody` 🟠

[vehicle_item_body.dart](../lib/features/vehicle/presentation/vehicle/widgets/molecules/vehicle_item_body.dart)

- La sub-clase `_LastReportWidget`:
  - Crea un `Stream.periodic(1s)` **dentro del `build`** y usa `.shareValue()`. Se reconstruye en cada rebuild del padre.
  - Calcula diferencias de tiempo y elige label `now` vs template con i18n → lógica de formato.
  - Mover a un cubit `LastReportCubit(vehicle)` o a un helper puro `LastReportFormatter` + `TickerProvider` cuando aplique.

---

### 2.6 `VehicleEnabledServices` 🟡

[vehicle_enabled_services.dart](../lib/features/vehicle/presentation/vehicle/widgets/molecules/vehicle_enabled_services.dart)

- `_buildSaveMode().buildWhen` repite la cláusula `current is SafetyRouteEnabled && current.serviceCode == vehicle.serviceCode || current is SafetyRouteDisabled`. Funciona, pero el cubit debería emitir un estado scopeado por `vehicle` (filtrado en el cubit, no en cada item).
- Naming: “Services” muy genérico → `VehicleEnabledServicesRow` o `VehicleServicesBadgeRow`.

---

### 2.7 `VehicleListAppBarBase` / `_VehicleAppBarHeader` 🟠

[vehicle_list_appbar.dart](../lib/features/vehicle/presentation/vehicle_filter/widgets/organisms/vehicle_list_appbar.dart)

- Crea **5 BlocProviders** dentro del organism (líneas 50–85). Esos providers pertenecen a la **Page**.
- `_getPlaces` filtra y mapea entidades (`vehicleState != noReport`, lat/long != 0, `actorTrip != thirdWithAssignedTrip`) → lógica de dominio dentro del widget.
- `_VehicleAppBarHeader._updateSuspensionMessageContent` hace `setState` dentro de `addPostFrameCallback` para reaccionar a un mensaje externo: indica acoplamiento. Considerar un cubit de `InvoiceSuspensionCubit`.

---

### 2.8 `VehicleMap` 🟠

[vehicle_map.dart](../lib/features/vehicle/presentation/vehicle_map/widgets/organisms/vehicle_map.dart)

- `_handleUpdatePlace`/`_handleUpdatePlaceByThirdParty` resuelven el vehículo desde `VehicleListCubit.vehicles.where(...).first` (acoplamiento) y construyen un `Place` con callback de navegación.
- TODO en código sobre uso de `'owner'` en lugar del enum `ActorTripEnum`. Resolverlo.
- Toda esa orquestación debería estar en un `VehicleMapCubit`, no en el widget.

**Bug 🔴:** `_subscribeToEvents` y `_subscribeToEmitter` sobreescriben la misma variable `subscription`; la primera suscripción se pierde. Separar en `_eventSubscription` y `_emitterSubscription`, cancelar ambas en `dispose`.

---

### 2.9 `VehicleDetailPage` 🔴

[vehicle_detail_page.dart](../lib/features/vehicle/presentation/vehicle_detail/pages/vehicle_detail_page.dart)

- El `build` del `StatelessWidget` raíz parsea `arguments as Map<String, dynamic>`, hace `map['interrogateCubit'] != null` para decidir si crea o reutiliza cubits, y luego construye **6 BlocProviders**. Línea 30–95: difícil de leer y propenso a typos.
- Tipar argumentos (`VehicleDetailRouteArgs`) + extraer una factoría `VehicleDetailDependencies.providers(args)`.
- `_VehicleDetailPageState`:
  - `didChangeAppLifecycleState`: cuenta máquina de estados manual con `_lastLifecycleState`. Funciona, pero amerita encapsular en `AppLifecycleObserver`.
  - `_handleBackNavigation()` lee `GlobalSettings.vehicleBehavior` y decide navegación condicional. Pertenece a una guarda de dominio.
  - Mezcla `Screenshot`, `MapDetailBase`, `BlocConsumer` de `SafetyRouteCubit`, `BlocListener` de `VehicleLiveModeCubit`, `MapBaseLayerCubit.enableDisableRoadEvents`. Es la página más “sucia” del feature.

---

### 2.10 `VehicleDetailMap` 🔴

[vehicle_detail_map.dart](../lib/features/vehicle/presentation/vehicle_detail/widgets/organisms/vehicle_detail_map.dart)

`takeScreenshot()` (líneas 100–140) ejecuta:

1. `analyticsCubit.registerEvent(...)`.
2. `cubit.commands(false)`.
3. `ProgressControl.showProgressIndicator(canPop: false)`.
4. `await Future.delayed(500ms)` (mágico).
5. `mapCubit.takeSnapshot()`.
6. `screeShotCubit.buildScreenshot(mapImage!)` con `!` no validado.
7. `await Future.delayed(500ms)`.
8. `widget.screenshotController!.capture(delay: 100ms)`.
9. `img.decodeImage(image!)`.
10. `share(img.encodeJpg(detail!))`.
11. `cubit.commands(true)`; `screeShotCubit.buildScreenshot(Uint8List(0))`.
12. `catch { log(e.toString()) }`; `finally { ProgressControl.hideProgressIndicator() }`.

**Síntomas:** lógica de aplicación (encoding/share/file system) en un widget, dependencia de `path_provider`, `image`, `share_plus`, `screenshot`. Métodos `await Future.delayed` mágicos. 5 `!` sin validar.

**Acción:** crear `ShareVehicleLocationUseCase` en dominio o un cubit `VehicleScreenshotCubit` que orqueste paso a paso. El widget solo dispara el caso y escucha estado (`Capturing`, `Sharing`, `Done`).

---

### 2.11 `VehicleIndicationsModal` 🟠

[vehicle_indications_modal.dart](../lib/features/vehicle/presentation/vehicle_detail/widgets/organisms/vehicle_indications_modal.dart)

- `_openMapsSheet`:
  - Llama a `MapLauncher.installedMaps` (servicio externo) desde el widget.
  - Si hay 1 mapa, lo abre; si hay más, abre un `ModalControl.showModal` y arma un `Wrap` de `ListTile` dinámicos.
- Mover la lógica “seleccionar maps disponibles” a un `OpenExternalMapUseCase` y dejar el modal solo para pintar.

---

### 2.12 `VehicleUpdatePage` 🟡

[vehicle_update_page.dart](../lib/features/vehicle/presentation/vehicle_update/pages/vehicle_update_page.dart)

- Pasa al cubit `VehicleListCubit` para refrescar tras update. Es una dependencia entre cubits hermanos (acoplamiento). Mejor: emitir un evento global o que la Page de origen escuche el resultado y refresque.
- `_handleSuccess` decide el valor de pop (`newNickname.isNotEmpty ? newNickname : widget.serviceCode`); aceptable, pero documentar.

---

## 3. Métodos complejos / difíciles de leer

| Archivo | Método | Problema | Sugerencia |
|---------|--------|----------|------------|
| `vehicle_detail_page.dart` | `_VehicleDetailPage.build` (raíz) | Parsea `arguments`, construye 6 providers, inicializa cubits. 60+ líneas con branching. | Extraer `VehicleDetailArgs.from(arguments)` + `VehicleDetailProviders.from(args).build(child: ...)`. |
| `vehicle_detail_map.dart` | `takeScreenshot()` | 11 pasos secuenciales con `Future.delayed`, 5 `!`, captura/encoding/share. | Cubit dedicado + use case. |
| `vehicle_filter_state_cubit.dart` | `_getFiltersList()` | 55 líneas con 6 entries hardcodeadas. | Iterar sobre lista declarativa `[(state, icon, color, type), ...]` o sustituir por mapper. |
| `vehicle_filter_result_cubit.dart` | `filterByVehicles(...)` | 22 líneas mezclando: seleccionar, resetear otro, ajustar estado, filtrar, emitir. | Dividir en `_selectCurrent`, `_clearOther`, `_syncState`, `_emitResult`. |
| `vehicle_list_page.dart` | `_hasRelevantVehicleStateChanged(...)` | Iteraciones anidadas + `firstWhere` con `orElse`; revisa 5 propiedades. | Definir `Equatable` extendido o `VehicleListSignature` que se compare globalmente. |
| `vehicle_list_cubit.dart` | `getVehicles({refresh})` | 25 líneas con 3 await secuenciales + try/catch + emisiones. | Extraer `_enrichVehicles(user, refresh)` en método aparte. |
| `vehicle_item.dart` | `_VehicleItemState.build` indirectamente vía `_providers` + 2 `BlocBuilder` anidados | El widget hace todo: providers, render, navegación, slidable. | Separar `VehicleListItemProvider` (cubits) y `VehicleListItemView` (render). |
| `vehicle_filter_list_gr_cubit.dart` | `onCheckChanged` + `_handleCheckChange` + `_updateTotal` + `_emitResultsBasedOnFilter` | Cadena de 4 métodos privados con efectos colaterales mutables. | Refactor a estado puro: `state.copyWith(...)`. |
| `vehicle_indications_modal.dart` | `_openMapsSheet(...)` | Mezcla I/O (`MapLauncher.installedMaps`), branching y construcción de UI. | Servicio + render. |

---

## 4. Lógica que debería moverse de capa

| Hoy en `presentation/` | Pertenece a | Razón |
|------------------------|-------------|-------|
| `VehicleEventDispatcher` (`utils/`) | `infrastructure/` o `domain/services/` | Transporte de eventos; estado mutable global. |
| `_shouldAutoInterrogate` en `VehicleItem` | `domain/services/auto_interrogate_service.dart` | Reglas de negocio. |
| `VehicleNavigationGuard.canNavigateToDetail` (a crear) basado en `vehicle.trip`, `vehicleState` en `VehicleItem` | `domain/` | Política de UX/negocio. |
| `MapLauncher.installedMaps` en `VehicleIndicationsModal` | `domain/services/external_map_service.dart` | I/O. |
| Captura+share en `VehicleDetailMap.takeScreenshot` | `domain/use_cases/share_vehicle_location_use_case.dart` | Compone múltiples servicios. |
| Filtros (icono/color/i18n key) en `VehicleFilterStateCubit` y `VehicleFilterResultCubit` | `presentation/view_models/` (mapper) | Decoración de UI. |
| `'vehicles.filter.all'.translate()` y `buildItemText` en `VehicleFilterListXxxCubit` | Widget | Formato de texto. |
| `_getPlaces` en `VehicleListAppBarBase` | `domain/` o extension de `VehicleEntity` | Reglas de filtrado para mapa. |
| `_handleBackNavigation` en `VehicleDetailPage` (depende de `GlobalSettings.vehicleBehavior`) | `domain/` (guarda) | Decisión sobre estado global. |

---

## 5. Bugs identificados

| Archivo | Bug | Severidad | Estado |
|---------|-----|-----------|--------|
| `vehicle_map.dart` | Doble asignación de `subscription` (la primera se pierde). | 🔴 | Pendiente. |
| `vehicle_item.dart` | `_isNavigating` debounce sin asincronía no funciona. | 🟠 | Pendiente. |
| `vehicle_detail_map.dart` | Cinco `!` en cadena (`mapImage!`, `image!`, `detail!`, `widget.screenshotController!`) sin validación; cualquiera puede fallar silenciosamente. | 🟠 | Pendiente. |
| `vehicle_filter_state_cubit.dart` | `subscription!.cancel()` puede fallar si `subscriptionListFiltered` quedó null. | 🟡 | Pendiente. |
| `vehicle_update_cubit.dart` | `currentSaveEnabled` se recalcula tras error pero el snackbar ya se mostró; no hay garantía de orden. | 🟡 | Pendiente. |
| `vehicle_item_cubit.dart` | Muta `_vehicle.name` antes de confirmar; si el rollback falla, queda desincronizado. | 🟠 | Pendiente. |

---

## 6. Hallazgos transversales

### 6.1 Naming

| Actual | Problema | Sugerido |
|--------|----------|----------|
| `VehiclesListPage` | Plural inconsistente con el resto. | `VehicleListPage` |
| `MapPage` | Nombre genérico (pisa contextos de otras features). | `VehicleMapPage` o `VehicleMapView` (según se registre como ruta). |
| `VehicleActionServices` | Pinta el menú de dashcam, no “services”. | `VehicleDashcamActionsMenu` |
| `VehicleEnabledServices` | “Services” muy genérico. | `VehicleEnabledServicesRow` |
| `VehicleDetailItem` | Demasiado genérico. | `VehicleDetailInfoRow` |
| `VehicleDetailAction` | Singular pero contiene varios botones. | `VehicleDetailHeaderActions` |
| `OptimizedVehicleItem` | El sufijo describe implementación, no rol. | `VehicleListItem` |
| `VehicleItemBuilder` | “Builder” no es rol UI; provee cubits. | `VehicleListItemProvider` |
| `VehicleUpdateContent` | Wrapper redundante de un único hijo. | Eliminar capa o renombrar a `VehicleUpdateForm`. |
| `VechielFilterListFailure` | **Typo**. | `VehicleFilterListFailure` |
| `current_location.dart` | Atom con nombre vago. | `CurrentLocationButton` |

**Regla:** nombre = rol + contexto. Evitar “Builder/Helper/Optimized/Manager” salvo justificación.

---

### 6.2 Atomic Design

- `vehicle_detail/widgets/` no tiene `atoms/` aunque define molecules y organisms.
- `vehicle_map/widgets/` no tiene `molecules/`.
- `vehicle/widgets/vehicle_item_builder.dart` está suelto, fuera de cualquier nivel atómico.
- `vehicle/widgets/atoms/vehicle_slidable.dart`: contiene `Slidable` con acciones de pin → atom dudoso, es **molecule** (compone gestos + estado).
- `vehicle_detail/widgets/molecules/vehicle_detail_section_title.dart`: es atom y debería vivir en `atoms/`.
- `vehicle_filter/widgets/templates/vehicle_filters_base.dart`: nivel `templates/`

Reubicación sugerida:

| Archivo | Nivel correcto |
|---------|----------------|
| `vehicle_slidable.dart` | molecules |
| `vehicle_detail_section_title.dart` | atoms |
| `vehicle_filter_list_appbar.dart` | organisms |
| `vehicle_item_builder.dart` | organisms o templates |
| `current_location.dart` (renombrado) | atoms |

---

### 6.3 Sufijo `Page`, `PageConstants` y rutas dinámicas

- `VehiclesListPage` y `MapPage` usan sufijo `Page` pero **no son rutas registradas**. Se montan como sub-vistas dentro del `home`. Según el lineamiento, “únicamente las páginas principales deben llevar `Page`”, lo cual aquí se viola al revés.
- `VehicleDetailPage` y `VehicleUpdatePage` sí son rutas y están en `PageConstants` ✅.
- Validar que ambas estén dentro de `routes_dynamic.dart` / `routes_config.dart`.
- Reemplazar `arguments: <String, dynamic>{...}` por una clase tipada (`VehicleDetailRouteArgs`).

| Archivo | Acción |
|---------|--------|
| `vehicle_list_page.dart` → `VehiclesListPage` | Renombrar a `VehicleListView` (no es ruta) **o** registrar `PageConstants.vehicleList` y dejar `VehicleListPage`. |
| `vehicle_map/pages/vehicle_map_page.dart` → `MapPage` | Renombrar a `VehicleMapView` (no ruta) **o** registrar `PageConstants.vehicleMap` y renombrar a `VehicleMapPage`. |

---

### 6.4 Dependencias: presentación → solo dominio

- Imports directos de `infrastructure/` desde `presentation/`: **0** ✅.
- Fugas indirectas:
  - `VehicleListCubit` ejecuta `VehicleEventDispatcher().dispatch*()` (singleton bajo `presentation/utils/` que opera sobre DTOs de dominio).
  - `VehicleItem` y `VehicleItemBuilder` usan `locator.get<VehicleService>()`, `locator.get<UserServiceDb>()`, `locator.get<LiveModeService>()` desde el widget.
  - `VehicleDetailPage` instancia 6 BlocProviders con `locator.get<...>()` inline.

---

### 6.5 Reutilización del Design System

Bien: se usan `Button`, `Appbar`, `BorderedItem`, `MyColors`, `MyTextStyle`, `MyDecoration`, `MySizeBox`, `NoDataContainer`, `ContainerMap`, `ButtonImage`, `ButtonFloatingCircle`, `ItemCheckBox`, `SearchAppbar`.

Brechas:

- [vehicle_update_header.dart#L40-L48](../lib/features/vehicle/presentation/vehicle_update/widgets/organisms/vehicle_update_header.dart#L40-L48): compone manualmente “campo deshabilitado”. Debe ser variante en el DS: `MyDecoration.inputDisabled()`.
- `EdgeInsets.symmetric(horizontal: 20, vertical: 10)`, `padding: EdgeInsets.only(right: 5, bottom: 5)`, `SizedBox(height: 25)` aparecen literales. Aplicar tokens de `MySizeBox`.
- `Image(image: AssetImage('assets/images/${vehicle.vehicleState.getVehicleImage()}'))` repetido — extraer atom `VehicleStateAvatar(vehicle: ...)`.

---

### 6.6 Otros hallazgos de Clean Code

| Tema | Hallazgo |
|------|----------|
| Visibilidad | `vehiclesAutoInterrogated`, `vehicles` expuestos como campos públicos en cubits → exponer copia inmutable o tras getter. |
| Mezcla de idiomas | Comentarios en español/inglés mezclados; unificar a español según [commit-guidelines.md](commit-guidelines.md). |
| TODO sin owner | `// TODO: Synecta...` en `vehicle_map.dart` sobre uso de enum `owner`. |
| Logging | `log(e.toString())` directo en `VehicleListCubit`, `VehicleItemCubit`, `VehicleMap`, `VehicleDetailMap`. Aplicar `logging-guidelines.md`. |
| Magic numbers | `Duration(milliseconds: 300/500/700)`, `inMinutes > 2`, `value.length > 50` → constantes nombradas. |

---

## 7. Plan de acción priorizado

> Estructura: **Q** = quick wins (cambios cosméticos, sin tocar dominio/infra). **M** = medio (cambia shape de estado o naming amplio). **H** = mayor (toca dominio/infra). **L** = pulido visual.

### 7.1 Quick wins (Q)

| # | Acción | Estado | Notas |
|---|--------|--------|-------|
| Q1 | Quitar `import 'package:flutter/material.dart'` de `VehicleOrderCubit` | Pendiente | Sustituir por `package:flutter/foundation.dart` para mantener `@immutable`. |
| Q2 | Quitar `import 'package:flutter/material.dart'` de `VehicleListCubit` | Pendiente | Mismo cambio: importar solo `foundation`. |
| Q3 | Bug doble asignación de `subscription` en `VehicleMap` | Pendiente | Separar en `_eventSubscription` y `_emitterSubscription`; cancelar ambas en `dispose`. |
| Q4 | Sacar `_showActionServices(context)` del `build` de `VehicleActionServices` | Pendiente | Renombrar a `_loadActionServices()` y disparar desde `initState` con `addPostFrameCallback`. |
| Q5 | Rename typo `VechielFilterListFailure` → `VehicleFilterListFailure` | Pendiente | 5 ocurrencias (estado + 2 cubits + organism). |
| Q6 | Tipos de retorno explícitos en `VehicleItemCubit` | Pendiente | `connectToEvents`, `connectPlatformToEvents`, `connectToThirdPartyEvents`, `_emitUpdateVehicle` → `void`. Quitar `await` indebido sobre `_emitUpdateVehicle`. |
| Q7 | Constantes para magic numbers locales | Pendiente | `_autoInterrogateDelay`, `_autoInterrogateThresholdMinutes` en `VehicleItem`; `_streamThrottle` en `VehicleItemCubit`; `_nicknameMaxLength` en `VehicleUpdateValidationUtils`. |
| Q8 | Rename `VehiclesListPage` → `VehicleListPage` | Pendiente | Clase, State y referencia en `lib/core/navigation/home_pages.dart`. |

### 7.2 Refactors medios (M)

| # | Acción | Estado | Notas |
|---|--------|--------|-------|
| M1 | Mover snackbars fuera de `VehicleUpdateCubit` → `BlocListener` con `messageKey` en estado | Pendiente | Cambia shape del estado. |
| M2 | Tipar argumentos de ruta (`VehicleDetailRouteArgs`, `VehicleUpdateRouteArgs`) | Pendiente | Toca call sites de `NavigationService.pushNamed`. |
| M3 | Renombrar `MapPage` → `VehicleMapView`/`VehicleMapPage` y `OptimizedVehicleItem` → `VehicleListItem` | Pendiente | Rename refactor + ajustar `presentation.dart`. |
| M4 | Reubicar `vehicle_slidable.dart` (atoms → molecules) y `vehicle_detail_section_title.dart` (a atoms) | Pendiente | Solo mover archivos + exports. |

### 7.3 Refactors mayores (H)

| # | Acción | Estado | Notas |
|---|--------|--------|-------|
| H1 | Mover `VehicleEventDispatcher` fuera de `presentation/` | Pendiente | Toca muchos imports. |
| H2 | Extraer `_shouldAutoInterrogate` (en `VehicleItem`) a dominio | Pendiente | Crear use case en `domain/`. |
| H3 | Extraer captura/compartir mapa (`VehicleDetailMap`) a use case + cubit | Pendiente | Saca `share_plus`, `image`, `path_provider` del widget. |
| H4 | Mapper de presentación para `VehicleFilterStateCubit` (color/ícono fuera del cubit) | Pendiente | Cubit emite enum/clave; widget mapea a `IconData/Color`. |
| H5 | Registrar `VehicleListPage`/`VehicleMapPage` en `PageConstants` + `routes_dynamic` si se tratan como rutas | Pendiente | Decisión de arquitectura previa. |
| H6 | Completar jerarquía atómica donde falta (`vehicle_detail/atoms/`, `vehicle_map/molecules/`) | Pendiente | Reubicación de archivos. |
| H7 | Logging conforme a `logging-guidelines.md` (sustituir `log(...)` sueltos) | Pendiente | Varios cubits y widgets. |
| H8 | Dividir `VehicleListCubit` en `VehicleListCubit` + `VehicleEventsCubit` + `AutoInterrogateTracker` (dominio) | Pendiente | Separa carga de orquestación de eventos. |
| H9 | Reemplazar mutaciones de `VehicleFilterViewModel` por `copyWith` en `VehicleFilterResultCubit` | Pendiente | Estados inmutables. |
| H10 | Convertir efectos `VehicleDetailCommandVisible`/`TakeScreenshot` en `BlocListener` con `Action` enum | Pendiente | Estados ≠ eventos. |

### 7.4 Pulido (L)

| # | Acción | Estado | Notas |
|---|--------|--------|-------|
| L1 | Reemplazar `EdgeInsets`/`SizedBox` literales por tokens de `MySizeBox` en toda la capa | Pendiente | Pulido visual. |
| L2 | Crear variante `MyDecoration.inputDisabled()` en design system y reutilizar en `VehicleUpdateHeader` | Pendiente | Toca `core/design_system`. |
| L3 | Reorganizar `extensions/build_context.dart` (renombrar a `vehicle_filters_reset_extension.dart` y documentar) | Pendiente | Mover/renombrar archivo. |
| L4 | Mover `VehicleFilterListGroupCubit.buildItemText` a widget o view model | Pendiente | Texto de UI fuera del cubit. |
| L5 | Extraer atom `VehicleStateAvatar(vehicle)` para evitar `Image(AssetImage(...))` repetido | Pendiente | Reuso visual. |
| L6 | Resolver TODO de `'owner'` → `ActorTripEnum` en `vehicle_map.dart` | Pendiente | Limpia código pendiente. |

---

## 8. Checklist de “Definition of Done” para esta capa

- [ ] Ningún cubit importa `package:flutter/material.dart`, `core/design_system`, `core/controls/snack_bar_control.dart`, `core/controls/modal_control.dart`, `core/controls/progress_control.dart`.
- [ ] Ningún cubit invoca `Navigator`, `BuildContext`, ni controles globales de UI.
- [ ] Ningún widget contiene reglas de negocio (cadenas de `if` con políticas, validaciones de producto, transformaciones de DTO).
- [ ] Ningún widget llama directamente a `locator.get<...>` salvo en la Page raíz del flujo.
- [ ] Toda página con sufijo `Page` está en `PageConstants` y registrada en `routes_dynamic.dart`. Vistas no-ruta usan sufijo `View`.
- [ ] Toda subcarpeta `widgets/` tiene `atoms/`, `molecules/`, `organisms/` cuando aplica; archivos no quedan sueltos.
- [ ] No se construyen botones, decoraciones ni textos “a mano” cuando existe componente del design system.
- [ ] Estados con `equatable`, sin tipos `IconData`/`Color` en el payload.
- [ ] Todos los métodos públicos del cubit declaran tipo de retorno.
- [ ] Logging conforme a `logging-guidelines.md` (sin `log(...)` sueltos).
- [ ] Argumentos de ruta tipados; no `Map<String, dynamic>`.

---

## 9. Resumen ejecutivo

- **Cubits que más lógica indebida tienen:** `VehicleListCubit`, `VehicleFilterStateCubit`, `VehicleFilterResultCubit`, `VehicleUpdateCubit`.
- **Widgets que más lógica indebida tienen:** `VehicleDetailMap`, `VehicleItem`, `VehicleDetailPage`, `VehicleIndicationsModal`, `VehicleListAppBarBase`.
- **Métodos más complejos:** `takeScreenshot`, `_VehicleDetailPage.build`, `_getFiltersList`, `filterByVehicles`, `_hasRelevantVehicleStateChanged`.
- **Bugs reales:** doble subscription en `VehicleMap` (🔴), debounce roto en `VehicleItem` (🟠), cadena de `!` en `VehicleDetailMap` (🟠), mutación previa a confirmación en `VehicleItemCubit` (🟠).
- **Capas a fortalecer:** crear `domain/use_cases/` y `domain/services/` para descargar a `presentation/` de I/O, reglas y orquestación.
- **Estructura general** y **dependencia hacia dominio** se cumplen razonablemente; las brechas principales están en fugas de UI hacia cubits, reglas de negocio dentro de widgets, naming inconsistente y deuda de Atomic Design.

El refactor sugerido es incremental: cada ítem del plan §7 puede tomarse como un PR independiente.
# Análisis de la capa `presentation/` — feature `vehicle`

Documento de revisión y propuestas de mejora para `lib/features/vehicle/presentation/`, evaluado contra:

- Principios de **Clean Code** y **Clean Architecture**.
- Atomic Design.
- Lista de sugerencias del equipo (jerarquía, responsabilidades de cubits/widgets, design system, naming, sufijo `Page`, dependencias).

> Alcance: solo carpeta `presentation/`. No se propone reescribir negocio; se identifican fugas hacia presentación, acoplamientos y deuda estructural.

---

## 1. Inventario actual

### Estructura de carpetas

```text
lib/features/vehicle/presentation/
├── presentation.dart          # barrel
├── extensions/
│   └── build_context.dart
├── utils/
│   └── vehicle_event_dispatcher.dart   # singleton con RxDart
├── view_models/
│   └── vehicle_filter_view_model.dart
├── vehicle/                   # submódulo: lista
│   ├── cubit/  (5 cubits: list, item, order, pinned, action_service)
│   ├── pages/  vehicle_list_page.dart   → clase VehiclesListPage
│   └── widgets/ atoms / molecules / organisms / vehicle_item_builder.dart (suelto)
├── vehicle_detail/            # submódulo: detalle
│   ├── cubit/  vehicle_detail_cubit
│   ├── pages/  vehicle_detail_page.dart
│   └── widgets/ molecules / organisms      ← sin atoms
├── vehicle_filter/            # submódulo: filtros
│   ├── cubit/  list/{gr,vh}, result, state    ← subcarpetas por flujo
│   └── widgets/ atoms / molecules / organisms / templates
├── vehicle_map/               # submódulo: mapa
│   ├── pages/  vehicle_map_page.dart    → clase MapPage
│   └── widgets/ atoms / organisms       ← sin molecules
└── vehicle_update/            # submódulo: edición alias
    ├── cubit/, pages/, utils/, widgets/{atoms,molecules,organisms}
```

### Cubits identificados

| Cubit | Líneas aprox. | Importa Material/UI | BuildContext en API | Acceso a snackbars/modals |
|-------|---------------|---------------------|---------------------|----------------------------|
| `VehicleListCubit` | 240 | **Sí** (`flutter/material.dart`) | No | No |
| `VehicleItemCubit` | 180 | No | No | No |
| `VehicleOrderCubit` | 50 | **Sí** (`flutter/material.dart`) | No | No |
| `VehiclePinnedCubit` | 50 | No | No | No |
| `VehicleServiceCubit` (acción) | 35 | No | No | No |
| `VehicleDetailCubit` | 55 | No | No | No |
| `VehicleUpdateCubit` | 95 | No (usa `core/design_system`) | No | **Sí** (`SnackBarControl`) |
| `VehicleFilterStateCubit` | 220 | No | No | No |
| `VehicleFilterListGroupCubit` | 140 | No | No | No |
| `VehicleFilterListVehicleCubit` | 110 | No | No | No |
| `VehicleFilterResultCubit` | 130 | No | No | No |

### Páginas (clases con sufijo `Page`)

| Archivo | Clase | Registrada en `PageConstants` | Ruta dinámica |
|---------|-------|-------------------------------|----------------|
| `vehicle_list_page.dart` | `VehiclesListPage` (plural, sin sufijo `Page` correctamente aplicado) | No (vive embebida en `home`) | No |
| `vehicle_detail/pages/vehicle_detail_page.dart` | `VehicleDetailPage` | `vehicleDetail = '/vehicle_detail'` | Sí |
| `vehicle_update/pages/vehicle_update_page.dart` | `VehicleUpdatePage` | `vehicleUpdate = '/vehicle_update'` | Sí |
| `vehicle_map/pages/vehicle_map_page.dart` | **`MapPage`** | No (tab embebido) | No |

---

## 2. Hallazgos por lineamiento

### 2.1 Estructura y jerarquía de carpetas / submódulos

**Estado:** parcialmente correcto. Hay subcarpetas por submódulo (`vehicle`, `vehicle_detail`, `vehicle_filter`, `vehicle_map`, `vehicle_update`)

**Problemas detectados**

- Atomic design **inconsistente**:
  - `vehicle_detail/widgets/` no tiene `atoms/` aunque define molecules y organisms.
  - `vehicle_map/widgets/` no tiene `molecules/`.
  - En `vehicle/widgets/` el archivo [vehicle_item_builder.dart](../lib/features/vehicle/presentation/vehicle/widgets/vehicle_item_builder.dart) está suelto, fuera de cualquier nivel atómico.
- En `vehicle_filter/cubit/` existen subcarpetas (`vehicle_filter_list/`, `vehicle_filter_result/`, `vehicle_filter_state/`) sin un README ni `barrel` que explique el criterio. La división por flujo es válida, pero falta convención.
- `extensions/build_context.dart` queda colgado a nivel `presentation/` y mezcla acciones de varios cubits (reset de filtros). Es difícil de descubrir.
- `utils/vehicle_event_dispatcher.dart` es un singleton global con estado mutable bajo `presentation/`. Por su responsabilidad (fan-out de eventos de dominio) **no pertenece a presentación**.
- `view_models/vehicle_filter_view_model.dart` está aislado; otros view models del feature están dispersos (p. ej. `DashcamMenuViewModel` se construye dentro de un widget).

**Propuesta**

```text
presentation/
├── presentation.dart
├── shared/                       # nuevo: utilidades comunes del feature
│   ├── extensions/
│   └── view_models/
├── vehicle_list/                 # renombrar 'vehicle/' → 'vehicle_list/'
│   ├── cubit/
│   ├── pages/
│   └── widgets/{atoms,molecules,organisms}
├── vehicle_detail/
│   └── widgets/{atoms,molecules,organisms}
├── vehicle_filter/
│   ├── cubit/{list,result,state}/   # ya existe; documentar
│   └── widgets/{atoms,molecules,organisms,templates}
├── vehicle_map/
│   └── widgets/{atoms,molecules,organisms}   # crear molecules vacío o evitarlo
└── vehicle_update/
```

Mover `VehicleEventDispatcher` a `lib/features/vehicle/infrastructure/` (es transporte, no UI) o a `domain/services/` si el equipo lo considera servicio.

**Submódulos:** mantener la regla actual (Opción A). Aplicar Opción B solo si un submódulo crece a su propio `cubit + domain` independiente.

---

### 2.2 Cubit: solo estado, sin lógica de presentación ni `BuildContext`

**Hallazgos**

| Cubit | Problema | Evidencia |
|-------|----------|-----------|
| `VehicleUpdateCubit` | Llama a `SnackBarControl.showSnackBar*` desde el cubit. Mezcla copy/i18n + UX en lógica de estado. | [vehicle_update_cubit.dart#L60-L85](../lib/features/vehicle/presentation/vehicle_update/cubit/vehicle_update_cubit.dart#L60-L85) |
| `VehicleListCubit` | Importa `package:flutter/material.dart` y llama a `VehicleEventDispatcher().dispatch*()` (efecto secundario fuera del estado); además ejecuta `vehicleService.saveVehicleByEvent` en cadena con stream subscriptions. | [vehicle_list_cubit.dart#L1-L3](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_list_cubit.dart#L1-L3), [#L95-L140](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_list_cubit.dart#L95-L140) |
| `VehicleOrderCubit` | Importa `flutter/material.dart` sin usarlo (basta `equatable`). | [vehicle_order_cubit.dart#L1](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_order_cubit.dart#L1) |
| `VehicleItemCubit` | Suscribe a streams en `connectToEvents` retornando dinámico, sin tipo de retorno; muta directamente la entidad de dominio (`_vehicle.name = newName`). | [vehicle_item_cubit.dart#L96-L106](../lib/features/vehicle/presentation/vehicle/cubit/vehicle_item_cubit.dart#L96-L106) |
| `VehicleDetailCubit` | Estados con responsabilidades mezcladas: `VehicleDetailCommandVisible` y `TakeScreenshot` son “señales de UI”, no estados de dominio. Mejor `event/effect` o BlocListener específico. | [vehicle_detail_cubit.dart](../lib/features/vehicle/presentation/vehicle_detail/cubit/vehicle_detail_cubit.dart) |
| `VehicleFilterStateCubit` | Inicializa subscriptions desde el widget (`initSubscriptions()`); el cubit decide colores, íconos y backgrounds de filtros (ver `_getFilterEntity`). Esto es lógica visual. | [vehicle_filter_state_cubit.dart](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_state/vehicle_filter_state_cubit.dart) |
| `VehicleFilterListGroupCubit` | Construye texto de UI (`buildItemText`, `'(N)'`) y usa `'vehicles.filter.all'.translate()` en propiedad de instancia → presentación dentro del cubit. | [vehicle_filter_list_gr_cubit.dart#L18-L25](../lib/features/vehicle/presentation/vehicle_filter/cubit/vehicle_filter_list/vehicle_filter_list_gr_cubit.dart#L18-L25) |

**Pregunta del equipo: ¿cómo garantizar que un cubit solo tenga lógica de estado?**

Reglas operativas a adoptar:

1. **Prohibir imports**: ningún cubit debe importar `package:flutter/material.dart`, `widgets.dart`, `core/design_system/*`, `core/controls/snack_bar_control.dart`, `core/controls/modal_control.dart`, `core/controls/progress_control.dart`. Agregar regla a `analysis_options.yaml` (custom_lint o `dependency_validator`).
2. **Estados con `equatable`** que contengan datos puros (`errorKey`, `successKey`, `isLoading`, `result`), no instancias de `IconData` ni `Color`.
3. **Side-effects de UI** (snackbar/modal/navegación) → mover a la Page con `BlocListener` y mapear desde el estado.
4. **Color/icono de filtros** → cubit emite enum/clave; un mapper de presentación (`view_models/`) traduce a `IconData/Color`.
5. **Subscripciones**: se arman en el constructor o en un método explícito `start()`, pero **el cubit es el dueño** de cancelarlas en `close()`. No pedir a un widget llamar `initSubscriptions`.
6. **Tests**: cada cubit debe poder testearse con `bloc_test` sin pumpWidget.

**Propuesta concreta — refactor `VehicleUpdateCubit`**

```dart
// Estado
class VehicleUpdateInitial extends VehicleUpdateState {
  final bool isSaveEnabled;
  final VehicleUpdateMessageKey? message;   // success | duplicate | error | null
  // ...
}

// Page (BlocListener)
BlocListener<VehicleUpdateCubit, VehicleUpdateState>(
  listenWhen: (p, c) => c is VehicleUpdateInitial && c.message != null,
  listener: (context, state) {
    final s = state as VehicleUpdateInitial;
    switch (s.message!) {
      case VehicleUpdateMessageKey.success:
        SnackBarControl.showSnackBar(message: 'vehicles.update.success'.translate(), ...);
      case VehicleUpdateMessageKey.duplicate:
        SnackBarControl.showSnackBarError(...);
      case VehicleUpdateMessageKey.error:
        SnackBarControl.showSnackBarError(...);
    }
  },
)
```

---

### 2.3 Widgets: solo pintan, lógica afuera

**Hallazgos**

- [vehicle_item.dart](../lib/features/vehicle/presentation/vehicle/widgets/organisms/vehicle_item.dart):
  - Define `_shouldAutoInterrogate(...)` (reglas: `noSignal`, `noReport`, `timeDifference.inMinutes > 2`). **Es regla de negocio en un widget**.
  - Crea `InterrogateCubit` y `VehiclePinnedCubit` con `locator.get(...)` directamente en `initState`. Acopla widget a la composición de dependencias.
  - Maneja navegación con argumentos `Map<String, dynamic>` (no tipada).
- [vehicle_action_service_cubit.dart consumido por `VehicleActionServices`](../lib/features/vehicle/presentation/vehicle/widgets/molecules/vehicle_enabled_action_services.dart): el widget llama `_showActionServices(context)` **dentro de `build`**, disparando una emisión de cubit en cada rebuild. Anti-patrón.
- [vehicle_update_page.dart](../lib/features/vehicle/presentation/vehicle_update/pages/vehicle_update_page.dart) maneja `TextEditingController`, `FocusNode`, `GlobalKey<FormState>`, hace `NavigationService.navigatorKey.currentState?.pop(...)` desde callbacks → correcto que esté en el widget; pero `_handleSuccess` decide el valor de retorno con `_nicknameController.text` (regla de presentación aceptable).
- [vehicle_detail_map.dart](../lib/features/vehicle/presentation/vehicle_detail/widgets/organisms/vehicle_detail_map.dart) ejecuta `await Future.delayed(...)`, captura screenshots, decodifica imágenes y comparte archivo → **lógica de aplicación dentro de un widget**.

**Pregunta del equipo: ¿cómo garantizar que un widget solo dibuje?**

1. Lint convention: archivos en `widgets/` no deben importar `package:image/*`, `path_provider`, `share_plus`, `screenshot`, `app_mobile/features/.../infrastructure/*`, `locator`.
2. `Stateful` permitido solo cuando se manejan controllers/animations/lifecycle. Cualquier lógica con `await` sobre servicios → mover a cubit/use case.
3. Eliminar invocaciones a cubits dentro de `build`. Disparadores deben ir en `initState` o callbacks (`onTap`).
4. Tipar argumentos de ruta: en lugar de `Map<String, dynamic>`, crear `VehicleDetailRouteArgs`.

**Propuestas concretas**

- Mover `_shouldAutoInterrogate` a `domain/services/vehicle_auto_interrogate_service.dart`.
- Extraer la lógica de captura/compartir mapa de `VehicleDetailMap` a un `MapShareUseCase` invocado por un cubit (`MapShareCubit` o reusar `ScreenshotCubit`).
- Convertir `_showActionServices(context)` en un disparador en `initState`.

---

### 2.4 Reutilizar Design System (no construir “de cero”)

**Estado:** se usan componentes del DS (`Button`, `Appbar`, `BorderedItem`, `MyColors`, `MyTextStyle`, `MyDecoration`, `MySizeBox`, `NoDataContainer`, `ContainerMap`, `ButtonImage`, `ButtonFloatingCircle`, `ItemCheckBox`, `SearchAppbar`). Buen punto de partida.

**Brechas**

- [vehicle_update_header.dart#L40-L48](../lib/features/vehicle/presentation/vehicle_update/widgets/organisms/vehicle_update_header.dart#L40-L48) compone `MyDecoration.inputWithBorders('').copyWith(filled: true, fillColor: MyColors.switchTrackInactive, ...)` para “campo deshabilitado”. Esto debería ser un token/variante en el DS: `MyDecoration.inputDisabled()`.
- `EdgeInsets.symmetric(horizontal: 20, vertical: 10)`, `padding: EdgeInsets.only(right: 5, bottom: 5)`, `SizedBox(height: 25)` aparecen literalmente. Tokens en `MySizeBox` cubren la mayoría; aplicar en TODOS los puntos.
- [vehicle_item_body.dart](../lib/features/vehicle/presentation/vehicle/widgets/molecules/vehicle_item_body.dart) y `vehicle_enabled_services.dart`: revisar si los íconos/badges replican algo existente del DS.
- `Image(image: AssetImage('assets/images/${vehicle.vehicleState.getVehicleImage()}'))` repetido — extraer atom `VehicleStateAvatar(vehicle: ...)`.

**Cómo garantizarlo**

- Checklist de PR: cualquier `Color(0x...)`, `TextStyle(...)`, `EdgeInsets(...)` con valores literales en `features/` requiere justificación explícita.
- Lint regla: prohibir `ColorXxx` directos y `TextStyle(` literal en features.

---

### 2.5 Naming claro y orientado al rol

**Hallazgos**

| Actual | Problema | Sugerido |
|--------|----------|----------|
| `VehiclesListPage` (clase) en `vehicle_list_page.dart` | Plural inconsistente con el resto. | `VehicleListPage` |
| `MapPage` (clase) en `vehicle_map/pages/vehicle_map_page.dart` | Nombre genérico (“Map” pisa contextos de otras features). | `VehicleMapPage` (ver §2.7 antes de mantener el sufijo) |
| `VehicleSlidable` (atom) | OK, claro. | — |
| `VehicleActionServices` (molecule) | Pinta el menú de dashcam, no “services”. | `VehicleDashcamActionsMenu` |
| `VehicleEnabledServices` | “Services” muy genérico. | `VehicleEnabledServicesRow` o `VehicleServicesBadgeRow` |
| `VehicleDetailItem` | Demasiado genérico (Item ¿de qué?). | `VehicleDetailInfoRow` |
| `VehicleDetailAction` | Singular pero contiene varios botones. | `VehicleDetailHeaderActions` |
| `VehicleFilterListAppbar` (molecule) | Es un appbar de búsqueda → naming OK, pero ubicarlo en organisms. | mover a `organisms/` |
| `VehicleFiltersMenu` | OK. | — |
| `OptimizedVehicleItem` | Sufijo `Optimized` describe implementación, no rol. | `VehicleListItem` y dejar `VehicleItem` como organism interno o renombrar a `VehicleListItemView`. |
| `VehicleItemBuilder` | “Builder” no es rol UI. Realmente provee cubits. | `VehicleListItemProvider` |
| `VehicleUpdateContent` | Es un wrapper de un único hijo (header). Redundante. | Eliminar capa o renombrar a `VehicleUpdateForm` |
| `VehicleNameField` / `VehicleNameContainer` | OK. | — |
| `_VehicleAppBarHeader` (privado) | OK. | — |
| `VechielFilterListFailure` (estado, ojo typo) | **Typo**: “Vechiel” en lugar de “Vehicle”. | `VehicleFilterListFailure` |

**Regla:** nombre = rol + contexto. Evitar “Builder/Helper/Optimized/Manager” salvo justificación.

---

### 2.6 Atomic Design

**Hallazgos**

- `vehicle/widgets/atoms/vehicle_slidable.dart`: contiene `Slidable` con acciones de pin → atom dudoso, es **molecule** (compone gestos + estado).
- `vehicle/widgets/atoms/vehicle_service_icon.dart` → atom correcto.
- `vehicle_detail/widgets/`: solo molecules y organisms; algunos elementos (p. ej. `VehicleDetailSectionTitle`) son atoms y deberían vivir en `atoms/`.
- `vehicle_map/widgets/atoms/current_location.dart`: nombre vago para un atom; renombrar `CurrentLocationButton`.
- `vehicle_filter/widgets/templates/vehicle_filters_base.dart`: nivel `templates/` 
- `vehicle/widgets/vehicle_item_builder.dart` fuera de niveles → mover a `organisms/` (o `templates/`).

**Propuesta:** completar la jerarquía atómica en todos los submódulos y reubicar los archivos según el cuadro:

| Archivo | Nivel correcto |
|---------|----------------|
| `vehicle_slidable.dart` | molecules |
| `vehicle_detail_section_title.dart` | atoms |
| `vehicle_filter_list_appbar.dart` | organisms |
| `vehicle_item_builder.dart` | organisms o templates |
| `current_location.dart` (renombrado) | atoms |

---

### 2.7 Sufijo `Page`, `PageConstants` y rutas dinámicas

**Hallazgos**

- `VehiclesListPage` y `MapPage` usan sufijo `Page` pero **no son rutas registradas**. Se montan como sub-vistas dentro del `home`. Según el lineamiento, “únicamente las páginas principales deben llevar `Page`”, lo cual aquí se viola al revés: tienen sufijo sin ser rutas.
- `VehicleDetailPage` y `VehicleUpdatePage` sí son rutas y están en `PageConstants` ✅.
- No se encontró referencia explícita de estas clases en `lib/core/navigation/` desde grep (probablemente vía generación dinámica de rutas). Hay que validar que ambas estén dentro de `routes_dynamic.dart` / `routes_config.dart`.

**Propuesta**

| Archivo | Acción |
|---------|--------|
| `vehicle_list_page.dart` → `VehiclesListPage` | Renombrar a `VehicleListView` (no es ruta) **o** registrarla como ruta con constante `PageConstants.vehicleList` y dejar `VehicleListPage`. |
| `vehicle_map/pages/vehicle_map_page.dart` → `MapPage` | Renombrar a `VehicleMapView` (no ruta) **o** registrar `PageConstants.vehicleMap` y renombrar clase a `VehicleMapPage`. |
| `VehicleDetailPage` | Verificar registro en `routes_dynamic.dart`. |
| `VehicleUpdatePage` | Verificar registro y tipado de argumentos. |

Adicionalmente:

- Reemplazar `arguments: <String, dynamic>{...}` por una clase tipada (`VehicleDetailRouteArgs`) co-ubicada con la Page.
- Centralizar `'/vehicle_detail'`, `'/vehicle_update'` (ya están en `PageConstants`); cualquier nuevo string de ruta NO debe vivir hardcodeado en widgets.

---

### 2.8 Dependencias: presentación → solo dominio

**Hallazgos**

- Búsqueda de imports de `infrastructure/` en `presentation/`: **0 imports directos** ✅.
- Sin embargo, hay **fugas indirectas**:
  - `VehicleListCubit` ejecuta `VehicleEventDispatcher().dispatch*()` (singleton bajo `presentation/utils/` que opera sobre DTOs de dominio). Mover a infra/dominio (ver §2.1).
  - `VehicleItem` y `VehicleItemBuilder` usan `locator.get<VehicleService>()`, `locator.get<UserServiceDb>()`, `locator.get<LiveModeService>()` **desde el widget**. Aunque los tipos son contratos de dominio, el `locator` desde un widget acopla composición. Preferir BlocProvider arriba en el árbol.
  - `VehicleDetailPage` instancia 6 BlocProviders con `locator.get<...>()` inline: aceptable como composition root del flujo, pero conviene extraer a un `VehicleDetailDependencies.providers(vehicle)` para test/legibilidad.

---

### 2.9 Otros hallazgos de Clean Code

| Tema | Hallazgo |
|------|----------|
| Tipos de retorno | `connectToEvents()`, `connectPlatformToEvents()`, `connectToThirdPartyEvents()`, `_emitUpdateVehicle()`, varios `_showActionServices(...)` no declaran retorno → agregar `void`/`Future<void>`. |
| Visibilidad | `vehiclesAutoInterrogated`, `vehicles` expuestos como campos públicos en cubits → exponer copia inmutable o tras getter. |
| Mutación de entidades de dominio en UI | `VehicleItemCubit.updateVehicleName` muta `_vehicle.name` antes de confirmar respuesta. Mejor `copyWith`. |
| `log(...)` vs guidelines de logging | Se usa `log(e.toString())` directo en cubits (`VehicleListCubit`, `VehicleItemCubit`, `VehicleMap`, `VehicleDetailMap`). Aplicar `logging-guidelines.md` (`logError`, `logInfo` con tag). |
| Comentarios en español/inglés mezclados | Unificar a español según [commit-guidelines.md](commit-guidelines.md). |
| `// TODO: Synecta...` | TODO en `vehicle_map.dart#L66` sobre uso de enum `owner` → crear issue o resolver. |
| `_showActionServices(context)` invocado en `build` | Provoca emisiones del cubit en cada rebuild. Mover a `initState`. |
| `setState` dentro de `addPostFrameCallback` en `_VehicleAppBarHeader._updateSuspensionMessageContent` | Aceptable pero indica acoplamiento de mensajes externos; documentar. |
| Streams duplicados en `VehicleMap._subscribeToEvents`/`_subscribeToEmitter` | Ambos sobreescriben `subscription`; el primero se pierde. **Bug**. Usar lista o variables separadas. |
| Nulabilidad | `widget.screenshotController!.capture(...)` con `!` después de hacerlo opcional. Mejor exigirlo requerido. |
| Estados con typo | `VechielFilterListFailure`. |
| Magic numbers | `Duration(milliseconds: 300/500/700)`, `inMinutes > 2`, `value.length > 50` → constantes nombradas. |

---

## 3. Plan de acción priorizado

### Progreso de implementación

> Nota: la ronda 1 de quick wins fue aplicada y luego **revertida** por el equipo. Los ítems quedan **pendientes** y se conservan aquí como referencia priorizada.

**Ronda 1 — quick wins (revertidos, pendientes de re-aplicar):**

| # | Acción | Estado | Detalle |
|---|--------|--------|---------|
| Q1 | Quitar `import 'package:flutter/material.dart'` de `VehicleOrderCubit` | Pendiente | Sustituir por `package:flutter/foundation.dart` para mantener `@immutable`. |
| Q2 | Quitar `import 'package:flutter/material.dart'` de `VehicleListCubit` | Pendiente | Mismo cambio: importar solo `foundation`. |
| Q3 | Bug doble asignación de `subscription` en `VehicleMap` | Pendiente | Separar en `_eventSubscription` y `_emitterSubscription`; cancelar ambas en `dispose`. |
| Q4 | Sacar `_showActionServices(context)` del `build` de `VehicleActionServices` | Pendiente | Renombrar a `_loadActionServices()` y disparar desde `initState` con `addPostFrameCallback`. |
| Q5 | Rename typo `VechielFilterListFailure` → `VehicleFilterListFailure` | Pendiente | 5 ocurrencias (estado + 2 cubits + organism). |
| Q6 | Tipos de retorno explícitos en `VehicleItemCubit` | Pendiente | `connectToEvents`, `connectPlatformToEvents`, `connectToThirdPartyEvents`, `_emitUpdateVehicle` → `void`. Quitar `await` indebido sobre `_emitUpdateVehicle`. |
| Q7 | Constantes para magic numbers locales | Pendiente | `_autoInterrogateDelay`, `_autoInterrogateThresholdMinutes` en `VehicleItem`; `_streamThrottle` en `VehicleItemCubit`; `_nicknameMaxLength` en `VehicleUpdateValidationUtils`. |
| Q8 | Rename `VehiclesListPage` → `VehicleListPage` | Pendiente | Clase, State y referencia en `lib/core/navigation/home_pages.dart`. |

**Pendiente — siguientes rondas:**

| # | Acción | Estado | Notas |
|---|--------|--------|-------|
| M1 | Mover snackbars fuera de `VehicleUpdateCubit` → `BlocListener` con `messageKey` en estado | Pendiente | Cambia shape del estado. |
| M2 | Tipar argumentos de ruta (`VehicleDetailRouteArgs`, `VehicleUpdateRouteArgs`) | Pendiente | Toca call sites de `NavigationService.pushNamed`. |
| M3 | Renombrar `MapPage` → `VehicleMapView`/`VehicleMapPage` y `OptimizedVehicleItem` → `VehicleListItem` | Pendiente | Rename refactor + ajustar `presentation.dart`. |
| M4 | Reubicar `vehicle_slidable.dart` (atoms → molecules) y `vehicle_detail_section_title.dart` (a atoms) | Pendiente | Solo mover archivos + exports. |
| H1 | Mover `VehicleEventDispatcher` fuera de `presentation/` | Pendiente | Toca muchos imports. |
| H2 | Extraer `_shouldAutoInterrogate` (en `VehicleItem`) a dominio | Pendiente | Crear use case en `domain/`. |
| H3 | Extraer captura/compartir mapa (`VehicleDetailMap`) a use case + cubit | Pendiente | Saca `share_plus`, `image`, `path_provider` del widget. |
| H4 | Mapper de presentación para `VehicleFilterStateCubit` (color/ícono fuera del cubit) | Pendiente | Cubit emite enum/clave; widget mapea a `IconData/Color`. |
| H5 | Registrar `VehicleListPage`/`VehicleMapPage` en `PageConstants` + `routes_dynamic` si se tratan como rutas | Pendiente | Decisión de arquitectura previa. |
| H6 | Completar jerarquía atómica donde falta (`vehicle_detail/atoms/`, `vehicle_map/molecules/`) | Pendiente | Reubicación de archivos. |
| H7 | Logging conforme a `logging-guidelines.md` (sustituir `log(...)` sueltos) | Pendiente | Varios cubits y widgets. |
| L1 | Reemplazar `EdgeInsets`/`SizedBox` literales por tokens de `MySizeBox` en toda la capa | Pendiente | Pulido visual. |
| L2 | Crear variante `MyDecoration.inputDisabled()` en design system y reutilizar en `VehicleUpdateHeader` | Pendiente | Toca `core/design_system`. |
| L3 | Reorganizar `extensions/build_context.dart` (renombrar a `vehicle_filters_reset_extension.dart` y documentar) | Pendiente | Mover/renombrar archivo. |
| L4 | Mover `VehicleFilterListGroupCubit.buildItemText` a widget o view model | Pendiente | Texto de UI fuera del cubit. |

> Para el detalle de **lógica fuera de lugar** (cubits con responsabilidades indebidas, widgets con reglas de negocio, métodos complejos, bugs), ver el documento complementario [review-vehicle-misplaced-logic.md](review-vehicle-misplaced-logic.md). Allí se enumeran los hallazgos archivo por archivo que alimentan los ítems M1, H1, H2, H3 y H4 de esta tabla.

---

## 4. Checklist de “Definition of Done” para esta capa

- [ ] Ningún cubit importa `package:flutter/material.dart`, `core/design_system`, `core/controls/snack_bar_control.dart`, `core/controls/modal_control.dart`, `core/controls/progress_control.dart`.
- [ ] Ningún cubit invoca `Navigator`, `BuildContext`, ni controles globales de UI.
- [ ] Ningún widget contiene reglas de negocio (cadenas de `if` con políticas, validaciones de producto, transformaciones de DTO).
- [ ] Ningún widget llama directamente a `locator.get<...>` salvo en la Page raíz del flujo.
- [ ] Toda página con sufijo `Page` está en `PageConstants` y registrada en `routes_dynamic.dart`. Vistas no-ruta usan sufijo `View`.
- [ ] Toda subcarpeta `widgets/` tiene `atoms/`, `molecules/`, `organisms/` cuando aplica; archivos no quedan sueltos.
- [ ] No se construyen botones, decoraciones ni textos “a mano” cuando existe componente del design system.
- [ ] Estados con `equatable`, sin tipos `IconData`/`Color` en el payload.
- [ ] Todos los métodos públicos del cubit declaran tipo de retorno.
- [ ] Logging conforme a `logging-guidelines.md` (sin `log(...)` sueltos).
- [ ] Argumentos de ruta tipados; no `Map<String, dynamic>`.

---

## 5. Notas para futuras revisiones

- Considerar añadir tests `bloc_test` para los cubits de filtro (`VehicleFilterStateCubit`, `VehicleFilterResultCubit`) que hoy concentran mucha lógica.
- Evaluar dividir `VehicleListCubit` en dos: `VehicleListCubit` (lista) y `VehicleEventsCubit` (orquestación de streams). Hoy mezcla ambos roles.

---

**Resumen ejecutivo:** la capa cumple bien con la estructura general y con la dependencia hacia dominio, pero presenta fugas claras de UI hacia cubits (snackbars, imports de Material), reglas de negocio dentro de widgets (auto-interrogación, captura de mapa, construcción de filtros), naming inconsistente (`MapPage`, `OptimizedVehicleItem`, `VechielFilter…`) y deuda de Atomic Design (carpetas atómicas incompletas). El refactor sugerido es incremental, sin tocar dominio ni infraestructura.
