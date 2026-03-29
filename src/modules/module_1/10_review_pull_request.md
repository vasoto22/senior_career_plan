# Dudas

## Podfile

Comprender mejor el contexto sobre para que se usa el podfile, es como el gradle y los archivos de configuración para Android?

## webrtc_client_service_impl.dart

**Comentario:** el HttpClient me genera duda, siento que esto va más en la capa de infrastructure, estamos exponiendo detalles de la implementación, si el día de mañana no se trabaja con HttpClient tendríamos que tocar el dominio

**Motivo:** Creo que es algo que no habría tenido en cuenta a la hora de revisar el PR, el tema de en qué capa estaban haciendo eso.


## streaming_controller_cubit.dart

**Comentario:** este cuando se emite?
**Motivo:** De dónde nace la duda para cuestionar el método, sé que es algo que se debería de usar para no hacer bulto o dejar basura en el código pero quisiera saber cómo llegó a esa conclusión, se revisó el código localmente? 


## streaming_audio_control_hls.dart

**Comentario:** 
- Considero que ese hls como una carpeta y adicional en el sufijo del nombre del widget es redundante, adicional si la carpeta HLS se crea por cada categoría de widgets (átomos, moléculas, organismos..) esto está creando una anidación más dificil de leer y no coincide con el estandar de carpetas que manejamos en el app, un ejemplo vehicle_detail.. si se encuentra que un feature tiene modulo o submodulo en presentación se crearía la carpeta referente al submodulo y por dentro irían las otras carpetas (widgets(atomos, moleculas...), cubit...)

- porque para este caso se usa ValueListenableBuilder? no digo que esté mal pero el manejador de estado principal del app es Cubit, quiero entender si para este caso en particular esta sería la mejor opción, y en caso de que den una justificación debería de quedar en la documentación la decisión y la aprobación de los arquitectos ya que se están mezclando manejadores de estados y esto puede traer inplicaciones o desorden en los demás equipos para decidir usar cosas porque sí.

**Motivo:** Más contexto al respecto y si le dieron respuesta sobre lo del manejador de estado o será que lo recomendó la IA y ni se dieron cuenta de ese detalle.

## video_player_hls.dart

**Motivo:** Cuándo me doy cuenta que un organismo ya se está desvirtuando? cómo ponerle un límite? como saber en qué momento está perdiendo la esencia de ser organismo.

## streaming_gateway.dart

**Comentario:** por qué no se está usando el widget Progress() del sistema de diseño?, se usa el CircularProgressIndicator
**Motivo:** Siento que esto es descuido del UX, ya que ellos mandan prototipos con cosas que no están en el app y a veces los desarrolladores se ven obligados a usar otras cosas, en eso siento que fallan, y es lo mismo, deberían sacar un rato para ver qué hay y qué no hay en el app para montar prototipos



[!IMPORTANT] **NOTA:** Veo demasiados comentarios en los métodos del código, si una lógica está bien hecha no tendría porqué explicar cada if que hace o que valida, o cada método, me parecen reuido visual los comentarios tan saturados, varios archivos, y son comentarios, no son documentación del código, emojis también en los logs, si es un error usar el log de error, y así, pero no usar emojis de X o de Warning
