**Situación actual:**
- Solo existen modos `debug` (implícito) y `release` en los logs de bootstrap
- environment.dart tiene `dev`/`prod` pero sin conciencia de `kProfileMode`
- launch.json no tiene configuraciones `profile`
- No se aprovechan los beneficios clave del modo profile de Flutter

**Beneficios del modo Profile:**
1. **AOT compilation** igual que release → métricas de rendimiento reales (no infladas por JIT de debug)
2. **DevTools habilitado** → CPU profiler, timeline, memory profiler activos
3. **Service extensions** conservados → se pueden conectar herramientas externas
4. **Crashlytics / logging** pueden mantenerse activos selectivamente
5. **Sin hot-reload overhead** → comportamiento de app campo real

### Beneficios del modo Profile habilitados

| Beneficio | Cómo se aprovecha |
|---|---|
| **AOT compilation** (misma que release) | Al lanzar con `--profile`, el compilador usa AOT → métricas de rendimiento reales |
| **Flutter DevTools activo** | CPU Profiler, Timeline, Memory Profiler disponibles al conectar DevTools |
| **Sin overhead de hot-reload** | Comportamiento idéntico al campo para medir jank, frame drops, etc. |
| **Backend de desarrollo** | `kEnvironment` en profile redirige al entorno `dev` para no contaminar datos de producción |
| **Crashlytics deshabilitado en no-release** | Evita contaminación de dashboards con crashes de sesiones de perfilado |

### Archivos modificados

**lib/environment/environment.dart**
- Agrega `import 'package:flutter/foundation.dart'`
- Expone `kBuildModeName` getter → `'debug'` / `'profile'` / `'release'`
- Convierte `kEnvironment` de `const` a getter: en profile/debug siempre usa `Environment.dev`; solo en release+prod flavor usa `Environment.prod`

**lib/bootstrap.dart**
- Log del modo ahora distingue los tres estados: debug / **profile** / release
- Añade `await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(kReleaseMode)` → Crashlytics solo activo en release

**.vscode/launch.json**
- Agrega 3 nuevas configuraciones de lanzamiento con `--profile`:
  - `Android/iOS Profile (Test)` → flavor `dev`
  - `Android/iOS Profile (Production)` → flavor `prod`
  - `Huawei Profile (Test)` → flavor `dev` + Huawei HMS defines

Para usar: seleccionar cualquiera de las nuevas configs en el panel **Run & Debug** de VS Code y conectar Flutter DevTools para analizar rendimiento.



-------
-------
-------



# Guía de Modo Profile — app_mobile

## 1. ¿Qué es el modo Profile?

Flutter tiene tres modos de compilación. Cada uno existe para un propósito distinto:

| Modo | Compilación | DevTools | Hot Reload | Crashlytics | Uso |
|---|---|---|---|---|---|
| **debug** | JIT (intérprete) | ✅ | ✅ | ❌ | Desarrollo diario |
| **profile** | AOT (nativo) | ✅ | ❌ | ❌ | **Medir rendimiento real** |
| **release** | AOT (nativo) | ❌ | ❌ | ✅ | Distribución en tienda |

El modo **profile** es el único que permite **medir rendimiento real** con las herramientas de análisis activas. Debug infla los tiempos de ejecución porque usa JIT; release deshabilita DevTools. Profile da lo mejor de ambos mundos.

---

## 2. ¿Cuándo usarlo?

Usar **siempre** modo profile cuando:

- La app se siente lenta o con jank (frames perdidos) y quieres ubicar la causa exacta.
- Antes de marcar una tarea como "lista" cuando involucra listas largas, animaciones o carga de mapas.
- Al implementar una nueva pantalla pesada (mapa, lista de vehículos, video VLC).
- Cuando Crashlytics o el equipo QA reporta ANRs o freezes en producción y necesitas reproducirlos.
- Al comparar el rendimiento de dos implementaciones (antes / después de un refactor).

**No** reemplaza las pruebas en release. Profile elimina el overhead de debug pero mantiene símbolos de depuración.

---

## 3. Modos disponibles en este proyecto

Las configuraciones están en [`.vscode/launch.json`](../.vscode/launch.json) y son accesibles desde el panel **Run & Debug** (`Ctrl+Shift+D`):

| Configuración VS Code | Flavor | Backend | Servicio de mapas |
|---|---|---|---|
| `Android/iOS Profile (Test)` | dev | **Test** (Azure App Config label: `Test`) | Google FCM |
| `Android/iOS Profile (Production)` | prod | **Producción** (Azure App Config label: `Production`) | Google FCM |
| `Huawei Profile (Test)` | dev | **Test** | Huawei HMS |

> **Regla de oro**: usar siempre `Profile (Test)` para medir. Solo usar `Profile (Production)` cuando necesitas confirmar un problema específico que solo ocurre con datos reales de producción y bajo supervisión.

---

## 4. Paso a paso: lanzar en modo Profile

### 4.1 Desde VS Code

1. Conectar un dispositivo físico **o** iniciar un emulador.  
   > Para medir rendimiento real, se recomienda **dispositivo físico** — los emuladores no reflejan el GPU/CPU del usuario final.

2. Abrir el panel **Run & Debug** (`Ctrl+Shift+D`).

3. En el dropdown superior seleccionar la configuración deseada, por ejemplo `Android/iOS Profile (Test)`.

4. Presionar **F5** o el botón ▶.

5. La app compilará en modo AOT (tarda más que debug en la primera compilación) y se ejecutará en el dispositivo.

### 4.2 Desde terminal

```bash
# Test environment + Google FCM
flutter run --flavor dev --profile \
  --dart-define-from-file=config/dart_defines.json \
  --dart-define-from-file=config/dart_defines_google_fcm.json

# Production environment + Google FCM
flutter run --flavor prod --profile \
  --dart-define-from-file=config/dart_defines.json \
  --dart-define-from-file=config/dart_defines_google_fcm.json

# Test environment + Huawei HMS
flutter run --flavor dev --profile \
  --dart-define-from-file=config/dart_defines.json \
  --dart-define-from-file=config/dart_defines_huawei_hms.json
```

---

## 5. Abrir Flutter DevTools

Una vez que la app esté corriendo en profile:

1. Abrir la paleta de comandos: `Ctrl+Shift+P`.
2. Escribir `Flutter: Open DevTools` y presionar Enter.
3. DevTools se abre en el navegador con las siguientes pestañas:

| Pestaña | Qué mide | Cuándo usarla |
|---|---|---|
| **Performance** | Frame rendering, UI thread, Raster thread | Jank, animaciones lentas |
| **CPU Profiler** | Qué funciones consumen CPU | Lógica lenta, parseo, BLoC |
| **Memory** | Allocations, leaks, GC | Crecimiento de memoria, leaks |
| **Network** | Requests HTTP, tiempos | Llamadas lentas a API |
| **Logging** | Logs de la app | Verificar flujo de ejecución |

---

## 6. Analizar rendimiento con la pestaña Performance

Esta es la herramienta más usada en profile mode.

### 6.1 El frame chart

```
UI Thread  ───█████░░░░░░────█████░░░░│░░────
Raster     ───░░███░░░░░────░░░███░░░░│░░────
                                      ↑
                                  16ms budget (60fps)
```

- Cada barra es un frame. El objetivo es **< 16ms por frame** (60fps) o **< 8ms** (120fps).
- **Barras rojas o amarillas** indican frames que superaron el presupuesto (jank visible).
- **UI Thread** lento → problema en Dart (BLoC, parseo, lógica de negocio).
- **Raster Thread** lento → problema en widgets costosos (sombras, opacidades, imágenes grandes).

### 6.2 Flujo recomendado

1. En DevTools > **Performance**, presionar **Record**.
2. Reproducir la acción problemática en el dispositivo (ej: hacer scroll en la lista de vehículos, abrir el mapa).
3. Presionar **Stop**.
4. Identificar los frames rojos/amarillos y expandir el flame chart para ubicar la función costosa.

---

## 7. Identificar y corregir problemas comunes

### Jank en listas (`ListView` / `VehicleList`)

**Síntoma**: scroll con tagueo, frames > 16ms en UI thread.

**Herramienta**: Performance > flame chart.

**Soluciones típicas**:
- Usar `ListView.builder` en lugar de `ListView` con items preconstruidos.
- Extraer `const` constructors en los items.
- Evitar reconstrucciones innecesarias con `const` o `BlocSelector`.

### Leak de memoria

**Síntoma**: memory crece con el tiempo y no baja al navegar.

**Herramienta**: Memory > **Track Allocations** + **Snapshot**.

**Soluciones típicas**:
- Verificar que los `StreamSubscription` se cancelen en `close()` del BLoC.
- Verificar que los `AnimationController` se dispongan en `dispose()`.

### Llamadas de red lentas

**Síntoma**: la pantalla tarda en cargar aunque la UI no tenga jank.

**Herramienta**: Network tab.

**En este proyecto**: buscar llamadas a Azure App Config o a los endpoints de la API que tarden > 500ms y considerar caché.

---

## 8. Comportamiento específico del proyecto en Profile

### Crashlytics deshabilitado

En [`bootstrap.dart`](../lib/bootstrap.dart) se configura:

```dart
await FirebaseCrashlytics.instance.setCrashlyticsCollectionEnabled(kReleaseMode);
```

Esto significa que en profile **no se envían crashes a Firebase**. El log de la app seguirá funcionando en consola.

### Backend de desarrollo activo

En [`environment.dart`](../lib/environment/environment.dart):

```dart
Environment get kEnvironment {
  if (appFlavor == 'prod' && kReleaseMode) return Environment.prod;
  return Environment.dev;
}
```

En profile mode **siempre apunta al entorno Test**, incluso si el flavor es `prod`. Esto protege los datos de producción durante sesiones de análisis.

### Identificar el modo en runtime

Desde cualquier parte del código puedes verificar el modo actual:

```dart
import 'package:app_mobile/environment/environment.dart';

// Retorna 'debug', 'profile' o 'release'
print(kBuildModeName);

// Para lógica condicional
if (kProfileMode) {
  // Solo activo en profile: habilitar overlays de diagnóstico, etc.
}
```

---

## 9. Buenas prácticas

| ✅ Hacer | ❌ Evitar |
|---|---|
| Medir en dispositivo físico de gama media | Medir solo en emulador o en dispositivo flagship |
| Comparar antes/después del cambio en el mismo build | Comparar un build debug con uno profile |
| Usar `Profile (Test)` para análisis rutinario | Usar `Profile (Production)` para análisis diario |
| Cerrar otras apps durante la medición | Dejar apps pesadas corriendo en paralelo |
| Reproducir el escenario 3 veces y promediar | Fiarse de una sola medición |
| Reportar el P90 de tiempos de frame | Reportar solo el mejor caso |

---

## 10. Preguntas frecuentes

**¿Por qué la compilación en profile tarda más que en debug?**  
Porque genera código AOT (máquinas nativas). Solo ocurre en la primera build; builds incrementales son más rápidas.

**¿Puedo hacer hot reload en profile?**  
No. Profile usa AOT. Para iterar rápido en código usa debug, y cuando estés listo para medir cambia a profile.

**¿Por qué no veo DevTools cuando corro en release?**  
Release deshabilita el protocolo de servicio de Flutter. Profile lo mantiene activo.

**¿Profile afecta el comportamiento de la app (APIs, autenticación)?**  
No. Solo cambia la compilación y el acceso a herramientas. El flujo de negocio es idéntico al de debug/release con el mismo flavor.

**¿Cómo sé que estoy en profile y no en debug?**  
El log de bootstrap imprime: `Running in profile mode`. También puedes verificar con `kBuildModeName` o inspeccionar `kProfileMode` de `flutter/foundation.dart`.
