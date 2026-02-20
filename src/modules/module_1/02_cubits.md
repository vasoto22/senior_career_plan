# Cubits

1. Cual debería ser la responsabilidad de un Cubit?
2. Que buenas practicas deberían considerarse en los Cubit?
3. Menciona algunas buenas practicas en los Cubit que apuntan a garantizar el rendimiento del App, pueden ser de las practicas mencionadas en el item anterior
4. Revisa en la App cual es un Cubit candidato a dividirse o cual Cubit está teniendo responsabilidades que no le corresponden y cual sería la propuesta de mejora

## Solución:

1. La responsabilidad principal de un Cubit en Flutter es gestionar el estado de la interfaz de usuario (UI) de forma reactiva y sencilla, actuando como intermediario entre la lógica de negocio y la presentación. Debe encapsular estados (states), exponer funciones para modificarlos, manejar errores y separar la lógica de la UI. 

2. Para implementar Cubit en Flutter de forma eficiente, se debe separar la lógica de negocio de la interfaz, utilizando estados inmutables y métodos claros para emitir cambios. Es vital centralizar la lógica, mantener cubits pequeños y escalables, y utilizar herramientas como BlocBuilder y BlocListener para una reconstrucción eficiente. 

### Mejores Prácticas en Cubit Flutter:
- **Inmutabilidad de Estados:** Definir clases de estado que sean inmutables (usando equatable o freezed) para asegurar que el BlocBuilder solo reconstruya la UI cuando el estado realmente cambie.
- **Métodos enfocados:** Los métodos dentro del Cubit deben ser simples y realizar acciones específicas (ej. emailChanged, loginSubmitted), en lugar de una función genérica updateState.
- **Separación de Conceptos:** Utilizar una estructura de carpetas clara para mantener los Cubits en la capa de presentación o dominio, separándolos de los repositorios de datos.
- **Uso de BlocBuilder vs BlocListener:**
    - Usar BlocBuilder solo para reconstruir la UI basada en el estado (ej. mostrar un spinner o texto).
    - Usar BlocListener para acciones laterales que ocurren una sola vez, como navegar a otra pantalla, mostrar un diálogo o un SnackBar.
- **Manejo de Estados Complejos:** Para formularios, segmentar el estado global en estados más pequeños para evitar reconstrucciones innecesarias de toda la pantalla.
- **Testeo Unitario:** Aprovechar que los Cubits facilitan las pruebas. Se deben escribir tests unitarios para verificar que una función emita los estados esperados en orden.
- **Evitar Lógica en la UI:** La UI solo debe llamar a funciones del Cubit y escuchar estados, sin realizar lógica de negocio interna. 


4. Cubits existentes en el app:

  350 lib\features\map\presentation\cubit\map\gms_map\gms_map_adapter_cubit.dart                                   
  307 lib\features\map\presentation\cubit\map\hms_map\hms_map_adapter_cubit.dart                                   
  350 lib\features\map\presentation\cubit\map\gms_map\gms_map_adapter_cubit.dart                                   
  307 lib\features\map\presentation\cubit\map\hms_map\hms_map_adapter_cubit.dart                                   
  306 lib\features\vehicle\presentation\vehicle\cubit\vehicle_cubit.dart                                           
  302 lib\features\dashcam\presentation\streaming\cubit\streaming_cubit.dart                                       
  259 lib\features\command\presentation\interrogate\cubit\interrogate_cubit.dart                                   
  256 lib\features\dashcam\presentation\ondemand\cubit\ondemand_cubit.dart                                         
  240 lib\features\vehicle_route\presentation\cubit\vehicle_route_cubit.dart                                       
  223 lib\features\alarm\presentation\alarm_composed\cubit\config\alarm_config_cubit.dart                          
  206 lib\features\login\presentation\cubit\login\login_cubit.dart                                                 
  196 lib\features\alarm\presentation\alarm\cubit\alarm_cubit.dart                                                 
  193 lib\features\home\presentation\cubit\home_cubit.dart                                                         
  189 lib\features\vehicle\presentation\vehicle_filter\cubit\vehicle_filter_list\vehicle_filter_list_gr_cubit.dart 
  184 lib\features\vehicle\presentation\vehicle\cubit\vehicle_item_cubit.dart                                      
  184 lib\features\alarm\presentation\alarm_composed\cubit\channel\alarm_channel_cubit.dart                        
  171 lib\features\vehicle\presentation\vehicle_filter\cubit\vehicle_filter_state\vehicle_filter_state_cubit.dart  
  159 lib\core\base\cubit\base_cubit.dart                                                                          
  152 lib\features\alarm\presentation\alarm_composed\cubit\alarm_composed_cubit.dart                               
  149 lib\features\alarm\presentation\alarm_detail\cubit\alarm_detail_cubit.dart                                   
  148 lib\features\alarm\presentation\alarm_composed\cubit\children\alarm_children_cubit.dart                      
  147 lib\features\vehicle\presentation\vehicle_filter\cubit\vehicle_filter_list\vehicle_filter_list_vh_cubit.dart 
  145 lib\features\vehicle\presentation\vehicle_filter\cubit\vehicle_filter_result\vehicle_filter_result_cubit.dart
  143 lib\features\safety_route\presentation\cubit\safety_route_notifications\safety_route_notifications_cubit.dart
  133 lib\features\safety_route\presentation\cubit\safety_route_main\safety_route_main_cubit.dart                  
  132 lib\features\payment\presentation\cubit\payment_cubit.dart                                                   
  128 lib\features\onboarding\presentation\cubit\onboarding_cubit.dart   


  