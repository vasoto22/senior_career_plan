# Guía de Pruebas de Integración — app_mobile

## 1. ¿Qué son las pruebas de integración?

Una **prueba de integración** verifica que dos o más componentes del sistema funcionan correctamente cuando se conectan entre sí.

A diferencia de un **unit test**, que aísla una sola clase y reemplaza todas sus dependencias con mocks, la prueba de integración instancia las capas reales y solo simula los límites externos (red, base de datos física, dispositivo).

### Analogía
- **Unit test**: probar solo el motor de un carro en un banco de pruebas.
- **Prueba de integración**: probar que el motor, la transmisión y las ruedas funcionan juntos, sin necesidad de salir a la calle.
- **E2E**: manejar el carro completo en una ruta real.

---

## 2. La pirámide de tests en Flutter

```
                         ┌─────────────────────────────┐
                         │    E2E / integration_test    │  ← Requiere emulador/dispositivo
                         │  (app completa en pantalla)  │     Lento · Frágil · Costoso
                         └──────────────┬───────────────┘
                    ┌────────────────────────────────────────┐
                    │          Widget tests (UI + BLoC)      │  ← Sin emulador · Rápido
                    │  Renderizado, interacción, estados UI  │     Recomendado para features
                    └──────────────────┬─────────────────────┘
              ┌───────────────────────────────────────────────────┐
              │        Integración in-process (este documento)    │  ← Sin emulador · Rápido
              │  Capas reales conectadas, límites externos mocked │     Detecta bugs de contrato
              └───────────────────────────┬───────────────────────┘
        ┌──────────────────────────────────────────────────────────────┐
        │                       Unit tests                             │  ← Sin emulador · Muy rápido
        │  Una sola clase, todas las dependencias son mocks            │     Ya existentes en el proyecto
        └──────────────────────────────────────────────────────────────┘
```

---

## 3. Tipos de pruebas de integración

### 3.1 Integración lógica in-process
Conecta capas reales de dominio e infraestructura. Solo se mockean los límites externos (`Api`, `DbConnection`, etc.).

| Característica | Valor |
|---|---|
| Requiere emulador | No |
| Velocidad | Rápido (milisegundos) |
| Herramienta | `flutter_test` + `mockito` |
| Detecta | Bugs de mapeo, bugs de contrato entre capas |

**Ejemplo en este proyecto:**
```
VehicleServiceImpl (real) → VehicleSettingsRepositoryImpl (real) → MockApi
```

---

### 3.2 Integración de BLoC + capas reales
El cubit se instancia real junto con el servicio y el repositorio. Solo se mockea la capa de red.

| Característica | Valor |
|---|---|
| Requiere emulador | No |
| Velocidad | Rápido |
| Herramienta | `flutter_test` + `bloc_test` + `mockito` |
| Detecta | Bugs en la secuencia de estados cuando la lógica real procesa la respuesta |

**Ejemplo en este proyecto:**
```
VehicleUpdateCubit (real) → VehicleServiceImpl (real) → VehicleSettingsRepositoryImpl (real) → MockApi
```

---

### 3.3 Integración de guards con el locator
La función guard se ejecuta con un `VehicleService` real registrado en el `GetIt` locator. Solo se mockea el repositorio.

| Característica | Valor |
|---|---|
| Requiere emulador | No |
| Velocidad | Rápido |
| Herramienta | `flutter_test` + `mockito` |
| Detecta | Bugs en la resolución de dependencias vía locator, bugs de flujo en la guard |

---

### 3.4 Widget tests (integración UI + BLoC)
Renderiza widgets reales con el cubit real y verifica que la UI responde correctamente a los estados.

| Característica | Valor |
|---|---|
| Requiere emulador | No |
| Velocidad | Rápido |
| Herramienta | `flutter_test` (`testWidgets`) |
| Detecta | Botones mal conectados, campos no reactivos, overflow de layout, validaciones no visibles |

---

### 3.5 E2E con `integration_test` _(no implementado en este proyecto)_
La app completa corre en un emulador o dispositivo real con Firebase, backend y SignalR activos.

| Característica | Valor |
|---|---|
| Requiere emulador | **Sí** |
| Velocidad | Lento (segundos a minutos por test) |
| Herramienta | `integration_test` (paquete de Flutter SDK) |
| Detecta | Problemas de integración con servicios externos reales |
| Recomendado para | Flujos críticos transversales (login, checkout) — no para features individuales |

---

## 4. Ventajas y desventajas

### Ventajas de las pruebas de integración (niveles 1–4)

| Ventaja | Descripción |
|---|---|
| **Detectan bugs de contrato** | Si una capa cambia su interfaz, los tests de integración fallan inmediatamente aunque los unit tests sigan pasando |
| **Cubren código real** | El código que se ejecuta es el mismo que va a producción, no doubles ni stubs intermedios |
| **No requieren emulador** | Los niveles 1–4 corren con `flutter test` en CI sin infra adicional |
| **Alta confianza** | Cuando pasan, garantizan que las capas cooperan correctamente |
| **Documentan el flujo** | Cada test describe un escenario completo de usuario, no solo una unidad aislada |

### Desventajas / consideraciones

| Desventaja | Descripción |
|---|---|
| **Más lógica de setup** | Se necesita construir y conectar múltiples objetos reales en cada `setUp` |
| **Más difíciles de aislar** | Un fallo puede venir de cualquier capa de la cadena, no de una sola clase |
| **Más lentos que unit tests** | Aunque rápidos vs E2E, involucran más código ejecutándose |
| **Más frágiles al refactoring** | Si se reestructuran capas internas, hay más tests que actualizar |

---

## 5. Tests de integración implementados en este proyecto

Los archivos están en `test/features/vehicle/`.

---

### Test 1 — Integración lógica: Service + Repository + Api
**Archivo:** `test/features/vehicle/integration/vehicle_service_settings_integration_test.dart`

**Cadena probada:**
```
VehicleServiceImpl ──► VehicleSettingsRepositoryImpl ──► VehicleRepositoryImpl ──► MockApi / MockDbConnection
```

**¿Por qué es integración y no unit test?**
En los unit tests existentes (`vehicle_service_impl_test.dart`) el repositorio es un `MockVehicleSettingsRepository`. Eso significa que si hay un bug en cómo `VehicleSettingsRepositoryImpl` interpreta la respuesta de la API (por ejemplo, si confunde el string `"Vehicle ID Duplicated"` con un éxito), el unit test del servicio nunca lo detecta porque el mock devuelve directamente el enum.

Este test instancia los repositorios **reales** y solo mockea `MockApi` y `MockDbConnection`.

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Success | La cadena completa devuelve `VehicleUpdateResultEnum.success` y el repositorio actualiza la Db local |
| duplicateNickname | El repositorio real interpreta `"Vehicle ID Duplicated"` correctamente y **nunca** llama a `DbConnection.update` |
| serverError | Una respuesta HTTP de error se traduce a `VehicleUpdateResultEnum.serverError` |
| Vehículo no en Db | Si no existe en Db local, devuelve `success` sin intentar actualizar |
| Múltiples casos | Los tres casos del `mockVehicleUpdateTestCases` recorren la cadena real |

---

### Test 2 — Integración de cubit: Cubit + Service + Repository
**Archivo:** `test/features/vehicle/integration/vehicle_update_cubit_integration_test.dart`

**Cadena probada:**
```
VehicleUpdateCubit ──► VehicleServiceImpl ──► VehicleSettingsRepositoryImpl ──► MockApi / MockDbConnection
```

**¿Por qué es integración y no unit test?**
En `vehicle_update_cubit_test.dart` el servicio es un `MockVehicleService`. Eso significa que si `VehicleServiceImpl.updateVehicleNickname` devuelve un resultado inesperado porque el repositorio real mapea mal la respuesta, el cubit test nunca lo verifica. Este test hace que el cubit llame al servicio **real**, que a su vez llama al repositorio **real**.

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Success | El cubit emite `[VehicleUpdateLoading, VehicleUpdateSuccess]` con las tres capas reales |
| duplicateNickname | El cubit vuelve a `VehicleUpdateInitial` y el repositorio real no actualiza la Db |
| serverError | El cubit emite `[VehicleUpdateLoading, VehicleUpdateInitial]` sin llamar a `DbConnection.update` |
| Flujo completo de UI | `onNicknameChanged` → `vehicleUpdate` emite los estados en el orden correcto |

---

### Test 3 — Integración guard + locator + Service
**Archivo:** `test/features/vehicle/integration/vehicle_guards_integration_test.dart`

**Cadena probada:**
```
builderVehicleDetailGuard ──► locator.get<VehicleService>() ──► VehicleServiceImpl ──► MockVehicleRepository
```

**¿Por qué es integración y no unit test?**
La guard obtiene el servicio vía `locator.get<VehicleService>()` en lugar de inyección directa. Ningún unit test puede verificar que esa resolución de dependencia funciona correctamente ni que la guard muta `args['vehicle']` e `args['isPush']` como corresponde. Este test registra un `VehicleServiceImpl` real en el locator.

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Vehicle ya poblado | Retorna `true` sin llamar al servicio |
| ServiceCode null | Retorna `false` sin llamar al servicio |
| ServiceCode vacío | Retorna `false` sin llamar al servicio |
| Nombre no encontrado en Db | El repositorio devuelve `""` → la guard retorna `false` sin llamar a `getVehicle` |
| Flujo completo | La guard llama a `getServiceCodeByName` luego a `getVehicle`, popula `args['vehicle']` e `args['isPush']` |
| Múltiples vehículos | Los cuatro `mockValidServiceCodes` recorren la cadena correctamente |

---

### Test 4 — Widget test: VehicleNameField
**Archivo:** `test/features/vehicle/presentation/vehicle_update/widgets/vehicle_name_field_test.dart`

**Qué se prueba:** el átomo `VehicleNameField` en aislamiento total (sin BLoC ni dependencias externas).

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Renderizado | El campo existe en el árbol de widgets y muestra el texto inicial |
| Controller vacío | El campo existe aunque el controller esté vacío |
| readOnly=true | El usuario no puede modificar el texto |
| readOnly=false | El usuario puede escribir normalmente |
| onChanged | El callback se invoca con el texto correcto en cada cambio |
| Validador con error | El mensaje de error aparece en pantalla |
| Validador sin error | No aparece ningún mensaje de error |
| Dos instancias | Dos campos con distintos `identifierKey` conviven sin conflicto |

---

### Test 5 — Widget test: VehicleUpdateContent (formulario + cubit real)
**Archivo:** `test/features/vehicle/presentation/vehicle_update/widgets/vehicle_update_content_test.dart`

**Qué se prueba:** el formulario completo (`VehicleUpdateContent` → `VehicleUpdateHeader` → 2× `VehicleNameField`) con el cubit instanciado de forma real.

> **Nota técnica:** `easy_localization` sin inicializar devuelve las claves raw como texto. El helper `pumpWide` amplía la pantalla de test a 2000×2000px para evitar el overflow del `Row` del header que contiene esas claves largas.

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Dos campos presentes | El formulario renderiza exactamente dos `TextFormField` |
| serviceCode visible | El código del vehículo aparece en pantalla |
| Nickname visible | El nickname actual aparece en pantalla |
| serviceCode readOnly | `EditableText.readOnly == true` en el primer campo |
| nickname editable | `EditableText.readOnly == false` en el segundo campo |
| Validador: nickname válido | `cubit.validateNickname` devuelve null |
| Validador: solo espacios | `cubit.validateNickname` devuelve un error no nulo |
| Validador: > 50 caracteres | `cubit.validateNickname` devuelve un error no nulo |
| `Form.validate()` false | El formulario no válida cuando el nickname es inválido |
| `Form.validate()` true | El formulario valida cuando el nickname es correcto |
| isSaveEnabled reactivo | `onNicknameChanged` actualiza el estado del cubit |
| Listener del controller | Escribir en el campo activa el listener del `TextEditingController` |
| maxLength=51 | El campo no acepta más de 51 caracteres |

---

### Test 6 — Widget test: flujo completo de UI (cubit + BLoC → UI)
**Archivo:** `test/features/vehicle/presentation/vehicle_update/vehicle_update_page_widget_test.dart`

**Qué se prueba:** el ciclo completo de interacción de usuario con la página de actualización. Se usa un widget envolvente `_VehicleUpdateTestPage` que replica la estructura de `_VehicleUpdateView` sin depender de `locator` ni de `VehicleCubit` padre.

> **Nota técnica:** el `Button` del design system tiene un debounce de 1.5 segundos. Los tests que hacen `tap` en el botón usan `await tester.pump(const Duration(milliseconds: 1600))` para que el timer del debounce complete antes de que el test finalice.

**Escenarios cubiertos:**

| Test | Qué verifica |
|---|---|
| Botón deshabilitado inicial | `ConfirmationActions.disabledConfirm == true` al cargar |
| Botón se activa | Al escribir un nickname distinto y válido, el `BlocBuilder` reconstruye el botón habilitado |
| Botón se deshabilita de nuevo | Al escribir el nickname original, el botón vuelve a deshabilitarse |
| Nickname con espacios | El botón no se activa si el nickname solo tiene espacios |
| Tap en guardar | El callback `onConfirm` se invoca al tocar el botón (con debounce resuelto) |
| Success → callback | Al recibir `VehicleUpdateSuccess`, el `BlocConsumer` invoca `onSuccessCallback` |
| duplicateNickname → no callback | `VehicleUpdateInitial` no dispara `onSuccessCallback` |
| Dos campos visibles | El formulario renderiza los dos `TextFormField` |
| serviceCode visible | El código aparece en pantalla al cargar |
| Nickname visible | El nickname actual aparece en pantalla al cargar |

---

## 6. Cómo ejecutar los tests

```bash
# Todos los tests de integración y widget de la feature vehicle
flutter test test/features/vehicle/integration/
flutter test test/features/vehicle/presentation/vehicle_update/

# Un archivo específico
flutter test test/features/vehicle/integration/vehicle_service_settings_integration_test.dart

# Todos los tests del proyecto
flutter test
```

Ninguno de estos comandos requiere emulador, dispositivo físico ni servicios externos activos.





------

## Para alguien técnico (dev del equipo)

Usa la **cadena de flechas** que está en el documento:

> *"Un unit test prueba `VehicleServiceImpl` pero le pasa un `MockVehicleSettingsRepository` — entonces nunca sabe si el repositorio real mapea bien la respuesta. Nuestro test de integración le pasa el repositorio **real**, y solo mockea el `Api`. Así probamos que el bug de contrato entre las dos capas no existe."*

La clave que hace clic: **"¿qué capa dejaste de mockear?"**

---

## Para alguien semi-técnico (tech lead, QA)

Usa la analogía de las piezas de Lego:

> *"Un unit test prueba cada pieza por separado. Un test de integración ensambla varias piezas reales y verifica que encajan bien. Nosotros ensamblamos el cubit + el servicio + el repositorio y solo simulamos la red."*

---

## Para alguien no técnico (PM, stakeholder)

Usa la analogía del carro del documento, pero más simple:

> *"Imagina que prueban cada parte del carro por separado: el motor funciona, el volante funciona. Los test de integración verifican que el motor y las ruedas funcionan **juntos**, sin salir a la calle. Eso es lo que hicimos: probar que las partes del código cooperan sin necesitar el servidor real ni el celular."*

---

## El argumento de valor (para justificar el esfuerzo)

Este es el más importante si alguien pregunta *"¿para qué, si ya tenemos unit tests?"*:

> *"Teníamos un bug potencial que los unit tests no podían detectar: si `VehicleSettingsRepositoryImpl` cambia cómo interpreta la respuesta del API — por ejemplo, si `'Vehicle ID Duplicated'` pasa a ser `'DUPLICATE'` — el unit test del servicio seguiría pasando porque usa un mock. El test de integración fallaría de inmediato porque ejecuta el código real del repositorio."*

Ese es el argumento que convence: **los unit tests prueban el contrato que tú mismo declaraste. Los tests de integración prueban que el contrato real se cumple.**