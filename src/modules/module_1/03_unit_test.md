**Anatomía de las pruebas unitarias**

- **Propósito:**: Verificar el comportamiento de una unidad pequeña de código (una función, clase o método) de forma aislada.
- **Aislamiento:**: Evitar dependencias externas (red, BD, sistema de archivos) usando dobles de prueba como mocks o fakes.
- **Determinismo:**: Las pruebas deben producir el mismo resultado en cada ejecución.

**¿Qué es un mock?**

- **Definición:**: Un mock es un objeto simulado que reproduce el comportamiento de una dependencia real, permitiendo controlar su respuesta y verificar interacciones (p. ej. que se llamó un método con ciertos argumentos).
- **Cuándo usarlo:**: Cuando la dependencia es lenta, no determinista o difícil de configurar en el entorno de pruebas (APIs externas, bases de datos, etc.).
- **Diferencia con un fake/stub:**: Un stub devuelve valores predefinidos; un fake es una implementación simplificada; un mock además verifica interacciones (cómo se usó).

**Generación de `mock.mocks.dart` en Flutter con Mockito**

- **Por qué se genera:**: Mockito usa generadores de código para crear implementaciones de prueba (mocks) que implementan una interfaz o clase abstracta. El archivo `*.mocks.dart` contiene esas clases generadas.
- **Cómo funciona (esquema):**:

	1. Definir una interfaz o clase objetivo.
 2. Crear un archivo de prueba que anote las clases a mockear con `@GenerateMocks([MiClase])`.
 3. Ejecutar el generador (build_runner):

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

	4. Aparecerá `mi_test.mocks.dart` con la clase `MockMiClase` que puedes instanciar en las pruebas.

- **Ventajas del generador:**: Evita escribir manualmente mocks frágiles, mantiene tipos y facilita uso con null-safety.

**Por qué tener un archivo `data` con DTOs, entities y datos "quemados" (hardcoded)**

- **Objetivo:**: Centralizar ejemplos de datos de prueba (fixtures) para reutilizarlos en múltiples tests.
- **DTOs vs Entities:**: Los DTOs representan la estructura de transferencia (p. ej. JSON), las Entities son el modelo de dominio. Tener ambos permite probar conversiones y mapeos.
- **Ventajas de datos hardcoded:**:
- **Reproducibilidad:**: Datos predecibles facilitan pruebas deterministas.
- **Claridad:**: Los fixtures documentan la forma esperada de los datos.
- **Reutilización:**: Evita duplicar payloads en múltiples pruebas.

**¿Por qué las pruebas empiezan con `void main()`?**

- **Entrypoint de tests:**: En Dart/Flutter, `void main()` es el punto de entrada que registra y ejecuta los grupos y casos de prueba. El framework de pruebas detecta y ejecuta las pruebas declaradas dentro de `main`.

**¿Para qué sirve `setUp()` y `setUpAll()`?**

- **`setUp()`**: Se ejecuta antes de cada test dentro del `group` o `main`. Ideal para crear instancias limpias, inicializar mocks y dejar la unidad en estado conocido.
- **`setUpAll()`**: Se ejecuta una sola vez antes de todos los tests del archivo o grupo. Útil para operaciones costosas compartidas (p. ej. cargar fixtures globales).
- **`tearDown()` / `tearDownAll()`**: Limpian recursos después de cada test o al final del conjunto respectivamente.

**`group()` — Organización de pruebas**

- **Propósito:**: Agrupar tests relacionados para mejorar lectura y permitir `setUp`/`tearDown` específicos por grupo.
- **Buenas prácticas:**: Nombrar el `group` con la unidad bajo prueba (p. ej. `group('MiRepositorio', () { ... })`). Mantener grupos pequeños y con responsabilidad clara.

**Estructura AAA (Arrange-Act-Assert)**

- **Arrange (Preparar):**: Crear las dependencias, configurar mocks y preparar los datos (fixtures). Aquí se usa `setUp()` o variables locales en el test.
- **Act (Actuar):**: Ejecutar la función/metodo bajo prueba.
- **Assert (Afirmar):**: Verificar el resultado y/o interacciones con los mocks (`verify()` en Mockito).

Ejemplo simple en Dart/Flutter con Mockito:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'my_service.dart';
import 'my_service_test.mocks.dart';

@GenerateMocks([RemoteApi])
void main() {
	late MockRemoteApi mockApi;
	late MyService service;

	setUp(() {
		mockApi = MockRemoteApi();
		service = MyService(api: mockApi);
	});

	group('fetchData', () {
		test('devuelve lista cuando API responde correctamente', () async {
			// Arrange
			when(mockApi.getData()).thenAnswer((_) async => ['a', 'b']);

			// Act
			final result = await service.fetchData();

			// Assert
			expect(result, ['a', 'b']);
			verify(mockApi.getData()).called(1);
		});
	});
}
```

**Consejos prácticos y buenas prácticas**

- **Mantén las pruebas rápidas**: mocks en lugar de llamadas reales.
- **Un solo assert por test** (ideal): facilita identificar la causa de fallo; si hay varios, que sean estrechamente relacionados.
- **Nombres descriptivos**: `test('fetchData devuelve lista vacía cuando no hay datos', ... )`.
- **Fixtures centrales**: coloca ejemplos en `test/fixtures` o un archivo `data` para reutilización.
- **Verifica interacciones relevantes**: usar `verify()` para comprobar efectos secundarios.

**Resumen rápido**

- Los mocks aíslan dependencias; Mockito genera `*.mocks.dart` para mantener tipos y compatibilidad.
- `void main()` agrupa y ejecuta tests; `setUp()` prepara cada caso; `group()` organiza.
- La estructura AAA hace las pruebas claras y mantenibles.

---

Si quieres, puedo: 1) añadir ejemplos más avanzados (streams, excepciones), 2) crear fixtures de ejemplo en `src/modules/module_1/data/`, o 3) generar un template de prueba para tu proyecto.
 
**Explicaciones del ejemplo provisto y patrones comunes**

- **Imports clave**:
	- `dart:async`: para `StreamController` y utilidades de streams.
	- `bloc_test`: provee `blocTest` para probar cubits/blocs con `build`/`act`/`expect`/`verify`.
	- `flutter_test`: helpers y bindings (p. ej. `TestWidgetsFlutterBinding`).
	- `mockito`: creación y verificación de mocks; el archivo generado (`mocks.mocks.dart`) expone las clases `MockXxx`.

- **`TestWidgetsFlutterBinding.ensureInitialized()`**: garantiza que el binding de Flutter esté listo (útil si el código bajo prueba usa `ServicesBinding`, `PlatformChannels` o cualquier API que requiera inicialización de Flutter). Es seguro llamarlo en pruebas unitarias que interactúan con partes de Flutter.

- **`late` variables**: se declaran como `late` para inicializarlas en `setUp()` y garantizar que cada test reciba instancias limpias.

- **StreamController y `.broadcast()`**:
	- Se usan `StreamController<...>.broadcast()` cuando habrá múltiples listeners (p. ej. varios suscriptores en el cubit o en tests).
	- Siempre cerrar los controllers en `tearDown()` para evitar fugas y efectos colaterales entre tests.

- **Stubbing de streams en `setUp()`**:
	- `when(mockService.someStream).thenAnswer((_) => controller.stream);` — se devuelve la `Stream` del controller para simular eventos.
	- Se usa `thenAnswer` porque estamos devolviendo un objeto (la stream) o una respuesta calculada; para `Future` asincrónico se usa `thenAnswer((_) async => value)`.

- **`thenAnswer` vs `thenReturn` y `doNothing()`**:
	- `thenReturn(value)`: para valores sincrónicos simples.
	- `thenAnswer((_) async => value)`: para devolver un `Future` asíncrono.
	- `thenAnswer((_) => stream)`: para devolver una `Stream` o cualquier objeto calculado dinámicamente.
	- Para métodos `void` o que normalmente se usan por su efecto, Mockito recomienda `doNothing().when(mock).method(args);`. En muchos tests verás `when(mock.dispose()).thenReturn(null);` — eso también evita ejecutar la implementación real.

- **Uso de `any` y `anyNamed`**:
	- `any` coincide con cualquier argumento posicional.
	- `anyNamed('isRealTime')` coincide con el parámetro nombrado `isRealTime`.
	- Útil cuando el valor exacto no es relevante para el comportamiento que estamos stubbeando.

- **`when(...).thenAnswer((_) async {})` vs `thenReturn`**: cuando stubbeas un método que devuelve `Future`, usa `thenAnswer((_) async => value)`. Si usas `thenReturn` con un `Future.value(...)` también funciona, pero `thenAnswer` es más claro y flexible.

- **Por qué stubtear `dispose()`**: en algunos mocks el `dispose()` podría ejecutar lógica real o lanzar excepciones si no se ha inicializado algo; stubearlo garantiza que el close/dispose no tenga efectos secundarios indeseados durante el test.

- **`Future.delayed(Duration.zero)`**:
	- Se usa en tests para permitir que microtasks y callbacks asíncronos pendientes se ejecuten antes de hacer `verify` o `expect` (equivalente a dejar que el event loop procese la cola inmediata).

- **`blocTest` — partes y motivación**:
	- `build`: crea la instancia del cubit/bloc bajo prueba (o devuelve la que ya configuraste en `setUp`).
	- `act`: invoca la acción o evento a probar (p. ej. `cubit.getVehicles()`).
	- `expect`: lista de matchers que describen los estados que el bloc/cubit debe emitir en orden.
	- `verify`: comprobaciones adicionales, típicamente `verify(mock.method()).called(n);` para confirmar interacciones.
	- Ejemplo: en tu test hay dos `VehicleListLoaded` porque el cubit emite un estado inicial de carga y luego otro tras recibir datos del streaming interno; por eso `expect` incluye varias entradas.

- **Matchers y `verify`**:
	- `isA<Type>()` verifica que el estado sea del tipo esperado.
	- `verify(mock.method()).called(1)` asegura que el método se llamó exactamente una vez.
	- `verify(mock.prop).called(greaterThan(0))` comprueba que la propiedad/método fue accedida más de cero veces.

- **Verificación de streams**:
	- Para comprobar que tu cubit delega un stream al servicio, se stubea la stream en `setUp()` y luego en un test se verifica que la propiedad/stream del servicio fue accedida.

- **Estado interno y aserciones**:
	- Es válido usar `expect(cubit.vehicles.length, n)` en `verify` para comprobar que el estado interno cambió según lo esperado.

- **Prácticas para evitar tests frágiles**:
	- Crear controllers y mocks nuevos por test (usar `setUp`), no compartir estado mutable entre tests.
	- Cerrar/cancelar subscriptions y controllers en `tearDown`.
	- Usar fixtures centralizados para datos (`createMixedVehicleList()`, `createTestEvent()`) para consistencia.

**Resumen de consejos aplicados al ejemplo**

- Usa `StreamController.broadcast()` cuando múltiples listeners requieran la misma stream.
- Usa `when(...).thenAnswer((_) => controller.stream)` para inyectar streams simuladas.
- Para métodos `Future` usa `thenAnswer((_) async => ...)`; para `void` considera `doNothing().when(mock)...` o `when(...).thenReturn(null)` si ya es común en tu código.
- Añade `Future.delayed(Duration.zero)` cuando necesites esperar a microtasks/completados asíncronos antes de verificar.
- Mantén `setUp` y `tearDown` robustos: inicializa y cierra recursos allí para evitar flakeos.

---

**Ejemplo comentado: `blocTest` real (línea a línea)**

```dart
blocTest<VehicleListCubit, VehicleListState>(
	'emits [VehicleListLoading, VehicleListLoaded] when getVehicles is successful',
	build: () {
		// Se preparan fixtures y stubs necesarios para el test
		final vehicles = createMixedVehicleList(); // fixture: lista de vehículos de ejemplo
		// Registrar stubs para llamadas asíncronas del cubit
		when(mockEventSyncService.registerEmitterPusher()).thenAnswer((_) async {});
		when(mockEventSyncService.registerPlatformPusher()).thenAnswer((_) async {});
		when(mockVehicleService.getVehicles(refresh: false)).thenAnswer((_) async => vehicles);
		when(mockVehicleService.getStreamingPlaying(any)).thenAnswer((_) async => vehicles);
		when(mockUserServiceDb.getUser()).thenAnswer((_) async => testUserEntity);
		when(mockSafetyRouteService.enrichVehicles(any, any)).thenAnswer((_) async {});
		when(mockLiveModeService.enrichVehiclesWithLiveMode(any, refresh: anyNamed('refresh')))
				.thenAnswer((_) async {});
		when(mockEventSyncService.registerLocationPusher(isRealTime: anyNamed('isRealTime'))).thenAnswer((_) async {});
		// Retornar la instancia del cubit configurada
		return cubit;
	},
	act: (cubit) => cubit.getVehicles(),
	// Expect: secuencia de estados esperada en el orden emitido
	expect: () => [
		isA<VehicleListLoading>(),
		isA<VehicleListLoaded>(),
		// Segundo VehicleListLoaded del streaming interno
		isA<VehicleListLoaded>(),
	],
	verify: (_) {
		// Verificar interacciones esperadas con servicios/mocks
		verify(mockEventSyncService.registerEmitterPusher()).called(1);
		verify(mockEventSyncService.registerPlatformPusher()).called(1);
		verify(mockVehicleService.getVehicles(refresh: false)).called(1);
	},
);
```

Notas rápidas sobre este bloque:
- `build`: prepara el cubit con todos los stubs necesarios — es el `Arrange` de AAA.
- `act`: ejecuta la acción que dispara cambios de estado (`getVehicles`) — es el `Act`.
- `expect`: lista el flujo de estados esperados — es el `Assert`.
- `verify`: comprueba efectos secundarios como llamadas a servicios externos.

**Fixtures de ejemplo (snippets Dart)**

Ejemplo simple de fixtures/constructores usados en los tests. Ajusta tipos reales (`VehicleEntity`, `EventDto`, etc.) según tu proyecto.

```dart
// src/modules/module_1/data/fixtures.dart (ejemplo)

List<Map<String, dynamic>> createMixedVehicleList() {
	return [
		{'id': 1, 'serviceCode': 'AAA-001', 'status': 'active'},
		{'id': 2, 'serviceCode': 'AAA-002', 'status': 'idle'},
		{'id': 3, 'serviceCode': 'AAA-003', 'status': 'active'},
		{'id': 4, 'serviceCode': 'AAA-004', 'status': 'stopped'},
	];
}

Map<String, dynamic> createTestEvent() {
	return {
		'vehicleId': 1,
		'type': 'location',
		'payload': {'lat': -34.0, 'lng': -58.0},
	};
}

Map<String, dynamic> createTestEventEmitter() {
	return {
		'vehicleId': 1,
		'emitter': 'third_party',
		'payload': {'trip': {'actorTrip': 'ownWithThirdPartyTrip'}},
	};
}

// Si tu código usa clases, convierte estos Map en instancias reales:
// VehicleEntity.fromJson(createMixedVehicleList()[0])
```

**Cómo usar las fixtures en los tests**

- En el `build` del `blocTest` o en `setUp()` llama a las funciones de fixtures para obtener datos consistentes.
- Si tu proyecto usa `VehicleEntity`, crea helpers `createTestVehicle(...)` que devuelvan instancias tipadas.

---

¿Quieres que cree físicamente `src/modules/module_1/data/fixtures.dart` con estos snippets y que adapte los tipos a tus entidades reales?

