# 🚀 Optimización de Rendimiento: Dart Isolates en Feature Vehicle

## 1. Introducción y Justificación

En Flutter, el código se ejecuta en un único hilo llamado **Main Isolate** (Event Loop). Cuando procesamos grandes volúmenes de datos, como una lista de más de 2000 vehículos, el hilo principal se satura realizando tareas de computación (mapeo, normalización y cruce de datos), lo que provoca el bloqueo de la interfaz de usuario (Jank) y pérdida de frames.

**Objetivo:** Trasladar la carga computacional del "Enrichment" de vehículos a un hilo secundario para mantener la app fluida a 60 FPS.

---

## 2. Investigación Técnica

### ¿Por qué Isolates y no solo Future/Async?

* **Async/Await:** Siguen corriendo en el hilo principal. Son buenos para tareas de I/O (esperar una API), pero no para tareas de CPU intensas.
* **Isolates:** Crean un espacio de memoria y un hilo de ejecución independiente. Son ideales para procesamiento de datos pesados sin interrumpir las animaciones de la UI.

### El método `compute`

Para esta implementación, se seleccionó la función `compute` de Flutter, que encapsula la creación de un Isolate, ejecuta una función de alto nivel y retorna el resultado de forma simplificada, gestionando automáticamente el ciclo de vida del hilo secundario.

---

## 3. Plan de Implementación

### A. Definición de la Lógica Aislada (Worker)

Se creó la función `enrichVehiclesInIsolate`. Esta función debe ser **global** o **estática** y trabajar únicamente con datos que puedan ser enviados a través de un mensaje (memoria aislada).

```dart
// Esta función corre en el hilo secundario
Future<List<Map<String, dynamic>>> enrichVehiclesInIsolate(
  Map<String, dynamic> paramsMap,
) async {
  final vehiclesList = paramsMap['vehiclesList'] as List<dynamic>;
  final providerMapData = paramsMap['providerMap'] as Map<String, dynamic>;
  final specialServicesMapData = paramsMap['specialServicesMap'] as Map<String, dynamic>;
  final userInfoVehiclesMap = paramsMap['userInfoVehiclesMap'] as Map<String, dynamic>;

  final vehicles = <Map<String, dynamic>>[];
  for (final vehicleMap in vehiclesList) {
    final vehicleData = Map<String, dynamic>.from(vehicleMap as Map);
    final serviceCode = vehicleData['serviceCode'] as String?;
    if (serviceCode == null) continue;

    // Cruce de información (Enrichment)
    final userInfoVehicleData = userInfoVehiclesMap[serviceCode] as Map<String, dynamic>?;
    final providerData = providerMapData[serviceCode] as Map<String, dynamic>?;
    final specialServicesData = specialServicesMapData[serviceCode] as List<dynamic>?;

    vehicles.add(
      VehicleNormalizeMapper.mapToMap(
        vehicleData: vehicleData,
        providerData: providerData,
        specialServicesData: specialServicesData,
        nickName: userInfoVehicleData?['nickName'] as String?,
        tripData: userInfoVehicleData?['trip'] as Map<String, dynamic>?,
      ),
    );
  }
  return vehicles;
}

```

### B. Orquestación en el Repositorio

Se implementó un umbral de decisión (**threshold**) para decidir cuándo usar un Isolate. Si la lista es pequeña (< 2000 registros), el costo de crear un Isolate es mayor que procesarlo en el Main Isolate.

```dart
final shouldUseIsolate = vehiclesList.length >= 2000;

if (shouldUseIsolate) {
  // 1. Preparamos los parámetros en un Map plano (dtoToMap)
  final isolateParams = VehicleEnrichmentNormalizeMapper.dtoToMap(vehiclesList, enrichmentMaps);
  
  // 2. Disparamos la computación en el hilo secundario
  final vehiclesMaps = await compute(enrichVehiclesInIsolate, isolateParams);
  
  // 3. Retornamos a modelos tipados en el hilo principal
  return vehiclesMaps.map((map) => VehicleNormalizeMapper.mapToModel(map)).toList();
}

```

---

## 4. Beneficios Obtenidos

1. **UI Fluida:** Las animaciones de carga y el scroll de la app no se congelan al recibir una respuesta masiva del backend.
2. **Desacoplamiento:** La lógica de normalización de datos queda aislada y es más fácil de mantener.
3. **Eficiencia Energética:** Al distribuir la carga en los núcleos del procesador, se evita que el hilo principal se sature y se caliente el dispositivo innecesariamente.

