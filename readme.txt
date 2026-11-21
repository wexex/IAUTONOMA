CEREBRO2 - Arquitectura y Guia Rapida
======================================

ESTRUCTURA DEL PROYECTO
-----------------------
- CONFIG                : Configuracion centralizada (rutas, puertos 8081/8082, modelos GGUF,
                          temperatura, reintentos, timeout, umbrales de similitud/refuerzo/
                          castigo, estimulos de vigilia, decaimientos y limites del visor).

- DESPERTAR/            : Servicios de fondo y ciclo biologico:
  - LENGUAJE            : Arranca llama-server en localhost:8081 con Qwen3.5-2B-Q4_K_M.gguf
                          (configurable a 4B / 1.7B / 0.8B / 0.6B en CONFIG).
  - MEMORIA             : Arranca llama-server en localhost:8082 con
                          Qwen3-Embedding-0.6B-Q8_0.gguf (--embedding).
  - OLVIDO              : Proceso independiente en tiempo real (funciona incluso con los
                          modelos dormidos). Degrada vectores segundo a segundo, poda
                          recuerdos bajo UMBRAL_SIM (0.45), elimina cajones vacios y
                          muestra un visor en vivo con limites configurables.
  - ABURRIMIENTO        : Vigila la inactividad segun TIEMPO_DESPIERTO (5 min por defecto).
                          A mitad de tiempo envia <<Todo esta en silencio>>, al cumplirse
                          envia <<Te duermes>> y apaga los puertos 8081 y 8082 dejando
                          OLVIDO intacto.

- CHAT                  : Interfaz interactiva de terminal (carga historial reciente para
                          flechas desde REGISTROS/historial.log, salida con q/Q).

- ENTRADA               : Punto de entrada principal:
                          - Despierta automaticamente LENGUAJE, MEMORIA y ABURRIMIENTO si
                            estan dormidos, enviando <<Te despiertas>> al despertar.
                          - Aplica inhibicion natural por similitud vectorial si detecta
                            correccion/rechazo (DECAIMIENTO_CASTIGO, ~1h de vida natural).
                          - Llama a AGENTE, vectoriza el turno con fecha/hora y guarda en:
                            - MEMORIA/hipocampo.json      (ultimos 6 turnos del presente).
                            - MEMORIA/episodica_*.json    (particiones tematicas, 7 dias).
                            - MEMORIA/persistente_*.json  (ascenso si norma >= 0.99, 1 anio).
                            - REGISTROS/historial.log     (registro completo sin limite).

- AGENTE                : Motor de razonamiento y herramientas:
                          - Vectoriza la consulta (anclada al hipocampo si es corta).
                          - Recupera hipocampo + Top-2 particiones de persistente y
                            episodica, reforzando (x1.10) y ascendiendo si alcanza >= 0.99.
                          - Inyecta en tools la especificacion de HERRAMIENTAS/bash.
                          - Ejecuta un bucle de hasta MAX_REINTENTOS (5) pasos para llamar
                            modulos, verificar salidas, reintentar o encadenar acciones.

- MEMORIA/              : Sistema de memoria en 3 capas + indice procedimental:
  - hipocampo.json      : Momento presente (MAX_HIPOCAMPO=6, decaimiento 24h).
  - episodica_*.json    : Medio plazo particionado + indice_episodica.json (7 dias).
  - persistente_*.json  : Largo plazo particionado + indice_persistente.json (1 anio).
  - castigo.json        : Vector cacheado de CONCEPTO_CASTIGO para inhibicion semantica.
  - MODULOS/            : Enrutador vectorial escalable de modulos (indice.json,
                          modulos_*.json y huella cksum; no decae con OLVIDO).

- HERRAMIENTAS/         : Herramienta ejecutable (--spec y ejecucion en jq puro):
  - bash                : Selecciona los Top-MAX_MODULOS (20) mas relevantes para el
                          prompt, filtra comandos peligrosos y ejecuta comandos Bash con
                          TIMEOUT_HERRAMIENTA (20s) cargando todos los MODULOS/*.

- MODULOS/              : Funciones en Bash cargadas automaticamente en HERRAMIENTAS/bash:
  - Sistema y archivos  : ruta, listar [ruta], identificar <ruta|cmd>, mirar <ruta>,
                          crear_directorio <ruta> (crear_dir), crear_fichero <ruta>,
                          escribir <ruta> [ini] [fin] '<texto>', anadir <ruta> '<texto>',
                          editar <ruta> <buscar> <reemplazar>, copiar_fichero <origen>,
                          copiar_directorio <origen>, pegar_fichero [destino],
                          pegar_directorio [destino], mover <origen> <destino>,
                          renombrar <origen> <destino>, buscar <nombre> [ruta],
                          buscar_texto <texto> [ruta].
  - Entorno y tiempo    : fecha, hora, maquina.
  - Red y proyectos     : buscar_internet <consulta>, subir_github [mensaje].
  - Meteorologia        : meteo_ahora, meteo_hoy, meteo_manana, meteo_ayer,
                          meteo_semana, meteo_estacion.
  - Guias               : skill [nombre] (sin nombre lista SKILLS/).

- SKILLS/               : Guias bajo demanda leidas por el modulo skill:
  - tareas_bash.txt     : Reglas y 10 ejemplos para encadenar modulos con ";".

- REGISTROS/            : Logs y estado de ejecucion:
  - historial.log       : Historial completo de interacciones (incluye estimulos internos).
  - server.log          : Salida del servidor llama-server (rotacion automatica).
  - ultima_interaccion  : Marca Unix de la ultima interaccion para ABURRIMIENTO.
  - aburrimiento.pid    : PID activo del proceso ABURRIMIENTO (solo mientras esta despierto).


USO BASICO
----------
1. Modo automatico (recomendado):
   ./CHAT
   (Al enviar un mensaje, ENTRADA despierta solos LENGUAJE, MEMORIA y ABURRIMIENTO,
    y los duerme tras 5 minutos sin interaccion.)

2. Visor y olvido en tiempo real (terminal independiente, opcional):
   ./DESPERTAR/OLVIDO

3. Consulta directa de una sola linea:
   ./ENTRADA "Tu pregunta aqui"

4. Arranque manual de servidores (si prefieres verlos en terminales separadas):
   ./DESPERTAR/LENGUAJE
   ./DESPERTAR/MEMORIA


ADVERTENCIA
-----------
No ejecutes AGENTE ni los scripts de HERRAMIENTAS/ o MODULOS/ por separado de forma directa.
Deben ejecutarse siempre a traves de CHAT o ENTRADA para que se gestionen correctamente el
despertar automatico, el aburrimiento, el hipocampo, la consolidacion, el castigo y los logs.


PATOLOGIAS CONOCIDAS DEL MODELO COMPACTO (DIAGNOSTICO INTERNO)
--------------------------------------------------------------
- Sindrome del loro / elefante rosa:
   Sintoma: Repite literal lo que lee en memoria o en el turno previo sin adaptarlo,
            o reproduce un dato erroneo justo al leer una negacion sobre el.
   Por que: Un modelo compacto (2B Q4) con temperatura casi determinista (0.001)
            asigna mayor probabilidad a copiar el patron textual ya presente en el
            contexto que a razonar de nuevo o disparar una herramienta. En el efecto
            elefante rosa, la atencion se engancha al dato mencionado aunque vaya
            acompanado de un "no", y termina repitiendolo igual.

- Memoria de pez:
   Sintoma: Pierde el hilo si los recuerdos decaen por debajo de 0.45 o si la consulta
            es ambigua.
   Por que: El modelo no tiene estado propio; solo existe lo que cabe y se inyecta en
            el turno actual (hipocampo + Top-2 particiones de episodica/persistente).
            Si un recuerdo cae bajo UMBRAL_SIM (0.45) en OLVIDO, sale de MAX_HIPOCAMPO/
            MAX_MEMORIA, o la pregunta es tan ambigua que su embedding no alcanza
            similitud suficiente con el cajon correcto, el recuerdo no entra al prompt.

- Loropez (fusion):
   Sintoma: Repite como un loro el fallo o comando roto de su propio hipocampo reciente.
   Por que: Combina ambas patologias en un bucle de retroalimentacion: tras un error
            (o responder sin usar herramienta), ENTRADA guarda ese turno en el hipocampo
            y en episodica. En los turnos siguientes el modelo pierde la referencia real
            (pez), toma su propio error reciente como ejemplo valido y lo copia (loro);
            si se repite la consulta, el refuerzo (>= 0.80) puede incluso fortalecer el
            fallo hasta que lo rebaja la inhibicion por castigo (DECAIMIENTO_CASTIGO) o
            lo limpia OLVIDO.

- Fosilizacion de datos efimeros (Fosil persistente):
   Sintoma: Devuelve una hora, clima o ruta antigua como si fuera actual y deja de
            llamar a bash.
   Por que: Si un dato volatil se repite o rescata varias veces, su norma llega a >= 0.99
            y asciende a persistente (1 anio de vida). Aunque lleve fecha delante, el 2B
            confunde el recuerdo reforzado con el presente.

- Herramienta fantasma (Falsa ejecucion):
   Sintoma: Afirma haber ejecutado un modulo o inventa su salida (ej. /home/usuario)
            sin haber disparado ninguna llamada real de herramienta.
   Por que: El modelo compacto prioriza simular en texto que realizo la accion en vez de
            emitir el bloque estructurado tool_calls.

- Espejo Yo/Tu (Confusion de sujeto):
   Sintoma: Ante estimulos como <<Te duermes>> o <<Te despiertas>> responde como si el
            que durmiera o despertara fuera el usuario, o confunde quien hace que.
   Por que: Los estimulos internos entran con role "user" igual que el humano, y el 2B
            interpreta la segunda persona como conversacion dirigida al usuario.

- Complacencia ciega / Delirio de identidad (Sycophancy):
   Sintoma: Al corregirlo, da la razon a toda costa incluso contradiciendo su naturaleza
            (ej. afirmar que es una persona y no una IA) o repitiendo muletillas.
   Por que: El alineamiento conversacional de modelos compactos prioriza agradar al
            usuario tras una correccion por encima de mantener coherencia factica.

- Confusion Modulos vs Skills:
   Sintoma: Lista los modulos de bash rebautizandolos como skills en vez de ejecutar el
            modulo skill para leer SKILLS/.
   Por que: Ve los modulos enumerados dentro de la definicion de tools y asume que esa
            lista ya son sus skills.

- Bloqueo RLHF de fabrica (Negacion de capacidades):
   Sintoma: Afirma no tener acceso al sistema operativo ni memoria persistente pese a
            tenerlos activos.
   Por que: Cuando algo falla o el usuario cuestiona una accion, el reflejo de seguridad
            del modelo base se impone sobre las instrucciones del system prompt.

- Tartamudeo XML (Bucle de tokens):
   Sintoma: Genera etiquetas internas rotas (<tool_call>, </parameter> repetido) hasta
            agotar el limite de tokens (-n 2048).
   Por que: Al intentar invocar una herramienta en texto libre con temperatura casi cero
            (0.001), el token de cierre se retroalimenta en un bucle determinista.

- Punto ciego del router (Latente):
   Sintoma: Improvisa con un modulo equivocado (ej. mirar una URL) o alucina que no
            puede realizar una accion que si existe.
   Por que: Con mas modulos que MAX_MODULOS (20), una consulta ambigua puede dejar el
            modulo correcto fuera del Top-20 vectorial inyectado por HERRAMIENTAS/bash.

- Castigo ciego / Falso castigo (Latente):
   Sintoma: Debilita un recuerdo sano tras hablar de errores de codigo, o no alcanza un
            fallo ocurrido hace mas de un turno.
   Por que: La inhibicion (DECAIMIENTO_CASTIGO) se activa por similitud vectorial contra
            CONCEPTO_CASTIGO (0.58) y actua solo sobre el ultimo turno del hipocampo.

- Aritmetica fragil (Latente):
   Sintoma: Cruza bien un recuerdo con un dato de herramienta pero falla restas simples
            de horas o cantidades.
   Por que: Limitacion propia del razonamiento numerico en un modelo 2B cuantizado Q4.
