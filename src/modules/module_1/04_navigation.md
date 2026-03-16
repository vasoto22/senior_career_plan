# Análisis Comparativo de Librerías de Navegación para Satrack

Este documento evalúa las opciones de navegación en Flutter para la arquitectura empresarial de **Satrack**, comparando la solución oficial **go_router** contra las alternativas más sólidas de la comunidad: **auto_route** y **beamer**.

---

## 1. go_router (La Opción Oficial)
Es el paquete mantenido directamente por el equipo de Flutter. Está diseñado para simplificar Navigator 2.0 y es el estándar de la industria actualmente.

### ✅ Ventajas
* **Respaldo de Google:** Garantiza compatibilidad inmediata con nuevas versiones de Flutter.
* **Deep Linking Nativo:** Manejo excepcional de URLs, ideal si Satrack requiere versiones web o integración con notificaciones push que redirigen a puntos específicos.
* **Sintaxis Declarativa:** Código limpio y fácil de seguir, lo que facilita el mantenimiento por parte de diferentes equipos.
* **ShellRoute:** Permite crear fácilmente navegación anidada (menús laterales o barras inferiores que no se recargan al navegar).

### ❌ Desventajas
* **Seguridad de Tipos (Type Safety):** Basado originalmente en strings (`context.go('/rastreo')`), lo que puede causar errores de dedo.
* **Menos Automatización:** Requiere definir manualmente las rutas y parámetros, a menos que se use un generador adicional.

## 🛡️ Manejo de Guards en go_router (Seguridad de Rutas)

Una de las preocupaciones en aplicaciones empresariales como **Satrack** es restringir el acceso a ciertas pantallas (ej. que un usuario no autenticado no vea el mapa de flotas). 

A diferencia de `auto_route`, que usa clases separadas para los guards, `go_router` utiliza la propiedad `redirect`. 

### ¿Cómo funciona un Guard en go_router?

El `redirect` es una función que se ejecuta antes de procesar cualquier ruta. Puedes usarlo de dos formas:

1.  **Global Redirect:** Protege toda la aplicación (ej. chequear si el usuario está logueado).
2.  **Route-level Redirect:** Protege una ruta específica (ej. solo usuarios con rol 'Administrador' pueden entrar a 'Reportes de Combustible').

#### Ejemplo de Lógica de Guard (Pseudo-código):

```dart
GoRouter(
  initialLocation: '/login',
  refreshListenable: authService, // Se dispara automáticamente si el usuario cierra sesión
  redirect: (context, state) {
    final bool loggedIn = authService.isLoggedIn;
    final bool loggingIn = state.matchedLocation == '/login';

    // GUARDIA: Si no está logueado y no está en login, mándalo a login
    if (!loggedIn) return '/login';

    // GUARDIA: Si ya está logueado y trata de ir al login, mándalo al Home
    if (loggedIn && loggingIn) return '/home';

    // Si todo está bien, no redirigir a ningún lado (continúa su camino)
    return null;
  },
  routes: [ ... ]
);

GoRouter(
  // 1. ESCUCHA CAMBIOS: Si el usuario cierra sesión, el router se entera solo.
  refreshListenable: authProvider, 

  redirect: (BuildContext context, GoRouterState state) {
    // 2. OBTENER ESTADO: ¿Está el usuario logueado?
    final bool isAuthenticated = authProvider.isLoggedIn;
    
    // 3. SABER A DÓNDE VA: ¿La ruta actual es una zona privada?
    final bool isGoingToLogin = state.matchedLocation == '/login';

    // CASO A: No está logueado y quiere entrar a una zona privada
    if (!isAuthenticated && !isGoingToLogin) {
      return '/login'; // <--- El Guard lo rebota al Login
    }

    // CASO B: Ya está logueado pero intenta volver al Login manualmente
    if (isAuthenticated && isGoingToLogin) {
      return '/home'; // <--- El Guard lo manda al inicio, ya no necesita login
    }

    // CASO C: Todo está en regla
    return null; // <--- "Siga su camino", no hay redirección.
  },
  
  routes: [
    GoRoute(path: '/login', builder: (context, state) => LoginPage()),
    GoRoute(path: '/home', builder: (context, state) => HomePage()),
    GoRoute(path: '/monitoreo', builder: (context, state) => MonitoringPage()),
  ],
);
```
---

## 2. auto_route (Basado en Generación de Código)
Es el competidor más fuerte, enfocado en robustez y seguridad en tiempo de compilación.

### ✅ Ventajas
* **Strongly Typed:** Al generar código, las rutas son clases. No usas `/detalle`, sino `DetalleRoute()`. Si cambias un nombre, el código no compila, evitando errores en producción.
* **Route Guards:** API muy potente para manejar middleware (ej. verificar si el vehículo tiene GPS activo antes de entrar a la pantalla).
* **Parámetros Simplificados:** El paso de argumentos entre pantallas es automático y tipado.

### ❌ Desventajas
* **Dependencia de `build_runner`:** Obliga al equipo a ejecutar comandos de generación de código constantemente, lo que puede ralentizar el flujo de trabajo diario.
* **Código "Mágico":** Al haber mucho código generado automáticamente, depurar errores internos de la librería puede ser más complejo.

---

## 3. Beamer (Arquitectura por "Locations")
Una opción flexible que separa la lógica de las rutas de la interfaz de usuario.

### ✅ Ventajas
* **Modularidad Total:** Permite dividir la navegación en "módulos de negocio" independientes.
* **Control de Historial:** Ofrece un control granular sobre cómo se reconstruye la pila de navegación.

### ❌ Desventajas
* **Curva de Aprendizaje:** Es el más difícil de aprender. Para una empresa con rotación de personal o equipos grandes, el tiempo de capacitación es mayor.
* **Comunidad Reducida:** Menos tutoriales y ejemplos disponibles comparado con las otras dos opciones.

---

## Cuadro Comparativo

| Característica | go_router | auto_route | Beamer |
| :--- | :---: | :---: | :---: |
| **Soporte/Mantenimiento** | Google (Alto) | Comunidad (Alto) | Comunidad (Medio) |
| **Seguridad de Tipos** | Media (Opcional) | **Alta (Nativa)** | Media |
| **Generación de Código** | No requerida | **Requerida** | No requerida |
| **Navegación Web** | Excelente | Muy Buena | Buena |
| **Facilidad de Uso** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |

---

## Conclusión y Recomendación

Para el ecosistema de **Satrack**, la recomendación técnica es implementar **go_router**.

### ¿Por qué go_router es la mejor opción?
1.  **Mantenibilidad:** En una aplicación empresarial, la longevidad del código es vital. Al ser oficial de Flutter, el riesgo de que la librería quede obsoleta es mínimo.
2.  **Facilidad de Onboarding:** Un desarrollador nuevo puede integrarse al proyecto y entender la navegación en minutos, sin pelear con generadores de código.
3.  **Híbrido Perfecto:** Si la seguridad de tipos es una prioridad crítica, se puede añadir `go_router_builder` más adelante para obtener rutas tipadas sin abandonar el estándar oficial.

> **Decisión Sugerida:** Adoptar `go_router` por su equilibrio entre simplicidad, soporte oficial y potencia para aplicaciones de gran escala.

---