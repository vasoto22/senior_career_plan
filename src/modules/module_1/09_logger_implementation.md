# Lineamientos de Logging

## Introducción

Este documento define las reglas y mejores prácticas para el manejo de logs en la aplicación. El objetivo es asegurar un uso consistente, seguro y eficiente del sistema de logging.

## ⚠️ REGLA DE SEGURIDAD CRÍTICA

**NUNCA loguear información sensible**, esto incluye:
- Tokens de autenticación (access tokens, refresh tokens, JWT)
- Contraseñas o credenciales
- Datos personales sensibles (números de identificación, tarjetas de crédito)
- Claves de API o secretos
- Información de sesión completa

### Ejemplo de violación:
```dart
// ❌ INCORRECTO - Expone información sensible
log('Token: $token');
log('Authorization found ${authorization.tokenExpirationDate!}');
logDebug('Auth', 'User password: $password');

// ✅ CORRECTO - No expone información sensible
logDebug('Auth', 'Token refreshed successfully');
logDebug('Auth', 'Token expiration validated');
logInfo('Auth', 'User authenticated successfully');
```

## Uso Obligatorio de la Clase Logger

### ✅ PERMITIDO - Usar únicamente métodos de `lib/core/utils/logger/logger.dart`:

| Método | Uso | Cuándo usar |
|--------|-----|-------------|
| `logDebugFinest(header, message)` | Detalles muy finos de ejecución | Trazado detallado de flujo interno |
| `logDebugFiner(header, message)` | Detalles finos de ejecución | Trazado de operaciones detalladas |
| `logDebugFine(header, message)` | Información de depuración fina | Información contextual útil para debug |
| `logDebug(header, message)` | Información de depuración general | Información útil para desarrollo |
| `logInfo(header, message)` | Información relevante | Eventos importantes del sistema |
| `logSuccess(header, message)` | Operaciones exitosas | Confirmación de operaciones completadas |
| `logWarning(header, message)` | Advertencias | Situaciones inesperadas pero manejadas |
| `logError(header, message)` | Errores | Errores que no detienen la aplicación |
| `logErrorObject(header, error, message)` | Errores con objeto | Errores con objeto/excepción asociada |
| `logFatal(header, error, message)` | Errores fatales | Errores críticos que detienen funcionalidad |
| `logBuild(header)` | Métricas de build | Medir tiempo de construcción de widgets |
| `logTimerStart(header, message)` | Inicio de medición | Iniciar medición de tiempo |
| `logTimerStop(header, timer, message)` | Fin de medición | Finalizar medición de tiempo |

### ❌ PROHIBIDO - NO usar estos métodos directamente:

| Método Prohibido | Razón | Reemplazo |
|-----------------|-------|-----------|
| `print()` | No respeta release mode, no tiene formato | `logDebug(header, message)` |
| `debugPrint()` | No respeta release mode del Logger, inconsistente | `logDebug(header, message)` |
| `log()` de `dart:developer` | No respeta release mode del Logger | `logDebug(header, message)` |

### Excepción Permitida

El uso de `debugPrint()` está permitido **ÚNICAMENTE** dentro de la clase `Logger` (`lib/core/utils/logger/logger.dart`) ya que es el punto centralizado de logging.

## Niveles de Log y Cuándo Usarlos

### LogLevel.debugFinest / logDebugFinest
Información extremadamente detallada, útil solo para debugging profundo.
```dart
logDebugFinest('Repository', 'Entering method getVehicles()');
logDebugFinest('Mapper', 'Mapping field: vehicleId');
```

### LogLevel.debugFiner / logDebugFiner
Información detallada de ejecución.
```dart
logDebugFiner('Service', 'Processing batch of 50 vehicles');
```

### LogLevel.debugFine / logDebugFine
Información contextual útil para desarrollo.
```dart
logDebugFine('Pusher', 'Connecting to Platform Tracking Pusher...');
```

### LogLevel.debug / logDebug
Información general de depuración.
```dart
logDebug('VehicleCubit', 'Vehicles loaded successfully');
logDebug('Repository', 'API response status: 200');
```

### LogLevel.info / logInfo
Eventos importantes del sistema que no son errores.
```dart
logInfo('App', 'Application started');
logInfo('Auth', 'User session restored');
```

### LogLevel.success / logSuccess
Confirmación de operaciones importantes completadas.
```dart
logSuccess('Login', 'User authenticated successfully');
logSuccess('Sync', 'Data synchronized with server');
```

### LogLevel.warning / logWarning
Situaciones inesperadas pero que no impiden el funcionamiento.
```dart
logWarning('Network', 'Slow connection detected, retrying...');
logWarning('Cache', 'Cache expired, fetching fresh data');
```

### LogLevel.error / logError
Errores que no detienen la aplicación.
```dart
logError('VehicleCubit', 'Error getting vehicles: ${e.toString()}');
```

### LogLevel.fatal / logFatal
Errores críticos que impiden el funcionamiento de una funcionalidad.
```dart
logFatal('Database', e, 'Critical database connection failure');
```

## Uso en Bloques Catch

### En Cubits

```dart
// ✅ CORRECTO
try {
  await _vehicleService.getVehicles();
} catch (e) {
  logError('VehicleCubit', 'Error getting vehicles: ${e.runtimeType}');
  super.manageError(e);
}

// ❌ INCORRECTO - No usar print
try {
  await _vehicleService.getVehicles();
} catch (e) {
  print('Error: $e'); // PROHIBIDO
}
```

### En Repositorios

```dart
// ✅ CORRECTO - Usar LaunchExceptionService para errores de API
try {
  final response = await apiService.getData();
  return response;
} catch (e) {
  // El logging se maneja en capas superiores
  throw LaunchExceptionService.getAppException(e, ErrorCodeEnum.getData);
}

// ✅ CORRECTO - Si necesitas loguear adicionalmente
try {
  final response = await apiService.getData();
  return response;
} catch (e) {
  logError('Repository', 'API call failed: ${e.runtimeType}');
  throw LaunchExceptionService.getAppException(e, ErrorCodeEnum.getData);
}
```

### En Servicios

```dart
// ✅ CORRECTO
try {
  return await _repository.getData();
} catch (e) {
  logError('ServiceName', 'Operation failed: ${e.runtimeType}');
  rethrow;
}
```

## Headers del Log

El header (`h` parameter) debe ser corto y descriptivo:

```dart
// ✅ CORRECTO
logInfo('VehicleCubit', 'Vehicles loaded');
logError('AuthService', 'Token refresh failed');
logWarning('NetworkRepo', 'Connection timeout');

// ❌ INCORRECTO - Headers muy largos
logInfo('VehicleRepositoryImplementation', 'Vehicles loaded'); 
logError('AuthenticationServiceImplementation', 'Failed');
```

**Recomendación**: Usar máximo 15 caracteres para el header.

## Configuración del Logger

El Logger es un singleton configurable:

```dart
// Configurar nivel mínimo de log
Logger().setLogLevel(LogLevel.debug);

// Configurar ancho del header
Logger().setHeaderWidth(15);

// Habilitar solo logs a DevTools (sin terminal)
Logger().setLogDevToolsOnly(true);
```

## Medición de Rendimiento

Para medir tiempos de ejecución:

```dart
// Método simple para widgets
logBuild('MyWidget');

// Medición manual
final timer = logTimerStart('Operation', 'Starting heavy operation...');
await heavyOperation();
logTimerStop('Operation', timer, 'Heavy operation completed');
```

## Verificación en Code Review

Durante la revisión de código, verificar:

1. **No hay `print()` o `debugPrint()` fuera de la clase Logger**
2. **No hay `log()` de dart:developer fuera de la clase Logger**
3. **No se loguean datos sensibles** (tokens, passwords, credentials)
4. **Se usa el nivel de log apropiado** para cada mensaje
5. **Los headers son concisos** (≤15 caracteres)
6. **Los catch blocks usan `logError`** cuando es apropiado

## Resumen de Reglas

| Regla | Severidad |
|-------|-----------|
| No loguear información sensible | 🔴 CRÍTICA |
| No usar `print()` | 🔴 ERROR |
| No usar `debugPrint()` (excepto en Logger) | 🔴 ERROR |
| No usar `log()` de dart:developer (excepto en Logger) | 🔴 ERROR |
| Usar nivel de log apropiado | 🟡 WARNING |
| Headers concisos (≤15 chars) | 🟢 SUGERENCIA |

## Migración de Código Existente

Si encuentras código que viola estas reglas:

```dart
// ANTES (incorrecto)
print('Error al obtener: $e');
debugPrint(contacts.length.toString());
log('Authorization found ${authorization.tokenExpirationDate!}');

// DESPUÉS (correcto)
logError('Repository', 'Error fetching data: ${e.runtimeType}');
logDebug('Contacts', 'Contacts count: ${contacts.length}');
logDebug('Auth', 'Authorization validated successfully');
```
