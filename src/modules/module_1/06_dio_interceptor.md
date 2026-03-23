## Estado actual de los interceptores

| Interceptor | `onRequest` | `onResponse` | `onError` |
|---|---|---|---|
| `ConnectivityInterceptor` | ✅ Verifica conectividad | — | — |
| `AuthInterceptor` | ✅ Inyecta token | — | — |
| `ErrorInterceptor` | — | — | ✅ Convierte DioException → AppException |

Ninguno implementa `onResponse`.

## ¿Sería útil?

### Casos donde **sí aportaría valor real**:

**1. Logging de respuestas exitosas**
El `LogManagerService` ya se usa en `onError` para reportar errores HTTP, pero no existe un registro de respuestas exitosas. Un `onResponse` permitiría tracing completo request→response (útil para debugging y auditoría).

**2. Centralizar `_getResponseDto`**
En api.dart, la función `_getResponseDto` se llama en los 7 métodos HTTP (`post`, `get`, `put`, `patch`, `delete`, `getUrl`, `download`). Mover esa lógica a `onResponse` eliminaría esa repetición.

**3. Extracción de headers de respuesta**
Si en algún momento el backend emite headers como `X-Token-Expiry` o nuevos tokens en respuesta, `onResponse` sería el lugar correcto para procesarlos centralizadamente.

### Casos donde **NO es necesario**:

- **Manejo de errores**: Los errores 4xx/5xx ya los captura Dio en `onError` antes de llegar a `onResponse` (por el `validateStatus` por defecto).
- **Autenticación**: El token se agrega en `onRequest`; no hay patrón de token-refresh reactivo pendiente.
- **Transformación de datos**: El parsing ya ocurre en los repositorios.

## Conclusión

En el estado actual **no es urgente**, pero hay un caso concreto que justificaría añadirlo: **logging de respuestas exitosas** para tener trazabilidad completa. Hoy solo se registran los errores. Un interceptor así quedaría así:

```dart
class LoggingInterceptor extends QueuedInterceptor {
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    locator<LogManagerService>().reportHttpResponse(
      url: response.requestOptions.uri.toString(),
      method: response.requestOptions.method,
      statusCode: response.statusCode ?? 200,
    );
    handler.next(response);
  }
}
```

----


Después de revisar la arquitectura completa, la respuesta es clara: **no vale la pena el esfuerzo**.

## Por qué el caso de `_getResponseDto` no funciona bien en `onResponse`

Parece el candidato ideal, pero tiene un problema de diseño:

```dart
// En ApiImpl: tiene tipo genérico <T>
ResponseDto<T> _getResponseDto<T>(Response response) { ... }

// En un interceptor: NO hay genérico, es siempre dynamic
void onResponse(Response response, ResponseInterceptorHandler handler) { ... }
```

Moverlo al interceptor **pierde el tipo genérico**, que es lo que permite que los repositorios reciban `ResponseDto<Map<String, dynamic>>` tipado. Habría que rediseñar toda la capa de `ResponseDto` y cada repositorio. Trabajo considerable sin ganancia real, porque la "repetición" actual es solo `return _getResponseDto(response)` — ya está encapsulada.

## Casos clásicos de `onResponse` y si aplican aquí

| Patrón | ¿Aplica? | Motivo |
|---|---|---|
| Token refresh en 401 | ❌ | Eso va en `onError`, no en `onResponse`. Ya no hay implementación de retry. |
| Normalizar estructura de respuesta del backend | ❌ | Cada repositorio hace su propio mapping. No hay formato inconsistente documentado. |
| Cache de respuestas HTTP | ❌ | No hay necesidad identificada de caching a nivel HTTP. |
| Extraer headers especiales (ej: `X-Rate-Limit`) | ❌ | El backend actual no emite headers que requieran procesamiento central. |
| Logging de respuestas exitosas | ⚠️ | El único caso real, pero es opcional y de bajo impacto. |

## Veredicto

El estado actual es coherente y consistente: los interceptores atienden preocupaciones transversales bien delimitadas (conectividad → auth → errores). Forzar un `onResponse` ahora sería añadir complejidad accidental sin resolver un problema real. **No gastar energía en ello.**

Si en el futuro el backend emite un header como `X-New-Token` para renovar sesiones silenciosamente, ahí sí sería el momento natural de introducirlo.