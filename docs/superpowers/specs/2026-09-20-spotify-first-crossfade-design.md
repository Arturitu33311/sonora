# Sonora — crossfade centrado en Spotify, con orden de cola inteligente

**Fecha**: 2026-09-20
**Base**: fork de [sonorahq/sonora](https://github.com/sonorahq/sonora) en `rev e7eb953d5848fd97bacd00e6c0e33500765eaa63` (Cargo.lock actual del upstream)
**Punto de partida decidido**: extender/forkear Sonora (no una app nueva desde cero, no un skin tipo Spicetify) — Sonora ya resuelve el problema más caro (agregación multi-fuente: Spotify + YouTube Music + archivos locales + letras), y es Rust/GPUI activamente mantenido.

## Contexto y motivación

El usuario reportó que el "DJ" de Sonora no funciona. Investigación previa confirmó que es el AI DJ de Spotify (locución + selección automática), un endpoint interno propietario que ningún cliente de terceros (Sonora incluido, vía librespot) puede replicar sin atestación criptográfica del cliente oficial — no es un bug, es una limitación estructural fuera de alcance.

A partir de ahí, el usuario pidió analizar otros reproductores con funciones similares/distintas (en particular la visualización de "disco de vinilo giratorio", presente en apps como Retro Music Player o el tema Spicetify "Turntable" — puramente decorativa — frente a software de DJ real como Mixxx, donde el plato controla de verdad la mezcla) y proponer cómo incorporar lo mejor de cada uno en una app custom basada en Sonora.

**Spotify es el reproductor principal del usuario.** Toda decisión de diseño prioriza que el crossfade y las demás funciones nuevas funcionen lo mejor posible ahí; las demás plataformas (YouTube Music, Subsonic, Deezer, Apple Music, archivos locales) reciben las mismas funciones como "extra", en la medida que la arquitectura ya compartida de Sonora lo permite sin esfuerzo desproporcionado.

## Alcance de esta spec

1. **Crossfade real** entre pistas consecutivas de la cola normal de reproducción (nivel "Plexamp Sweet Fades": un solo hilo de reproducción, transición automática, sin control manual de plato/scratch — el usuario descartó explícitamente el nivel "dos decks manuales" y "DJ completo con BPM sync" por ser demasiado esfuerzo para el beneficio).
2. **Orden inteligente de playlist** opcional, para que las transiciones caigan entre pistas que suenan parecido, en vez de un orden arbitrario.
3. Fuera de alcance: AI DJ real de Spotify (imposible sin atestación propietaria — ver arriba), plato/scratch manual, beatmatching por BPM, disco de vinilo visual (quedó como interés secundario, no priorizado por el usuario en esta ronda — ver "Trabajo futuro no incluido").

## Validación técnica ya hecha (no es hipótesis)

Antes de comprometernos a este diseño, se corrió un spike descartable (`/tmp/.../dual-spike`, fuera del repo) que:

- Instaló Rust en la máquina del usuario.
- Compiló un binario mínimo contra el mismo fork/rev de librespot que usa Sonora.
- Autenticó con la sesión real de Spotify del usuario (reutilizando las credenciales cacheadas por la propia instalación Flatpak de Sonora, vía `librespot_core::cache::Cache` — nunca se leyeron en crudo).
- Levantó **dos instancias de `librespot_playback::player::Player` en paralelo**, ambas sobre el mismo `Session` clonado, cada una con un sink que solo contaba muestras (sin tocar hardware de audio real).

**Resultado**: ambos `Player` cargaron, reprodujeron y decodificaron la pista completa (mismo conteo exacto de muestras: 18 837 168 cada uno), sin error, sin `Unavailable`, sin que uno interrumpiera al otro. Esto confirma que **no hace falta parchear el fork de librespot** para el crossfade en Spotify: se puede construir enteramente sobre su API pública, ejecutando dos `Player` a la vez durante la ventana de transición.

## Arquitectura

### Hallazgo de partida (código actual)

- Cada proveedor abre **su propio** dispositivo de salida: Spotify vía `spotify/sink.rs` → `OutputSink` envuelve un `Paced` propio; Subsonic/Deezer/Apple/local comparten el bucle genérico de `engine.rs`, cada instancia con su propio `Paced` también. `Paced` (en `sink.rs`) encola paquetes en **un** `rodio::Player` (cola secuencial) — no mezcla dos fuentes.
- `audio.rs::Output::open()` es quien de verdad abre el stream de `cpal` y el `stream.mixer()` de rodio 0.21. Rodio **ya tiene** el primitivo de mezcla real (sumar N fuentes simultáneas) — hoy cada `Output` solo le agrega una fuente (`SmoothGain<Equalized<...>>`).
- El "gapless join" que ya existe (`Job::Queue` en `engine.rs`, `gapless: true` en `PlayerConfig` de librespot) es un **corte duro** por conteo de muestras, no una mezcla — decide, en el instante exacto en que la pista A se acaba, pasar a la pista B.
- librespot (el fork, revisado en `playback/src/player.rs`) tiene la misma forma: un solo campo `decoder` en su máquina de estados (`PlayerState::Playing { decoder, ... }`). Su "gapless" solo evita parar el dispositivo de audio entre pistas, no mezcla.

### El cambio central: un mixer compartido a nivel de app

Hoy: **N proveedores → N `Output` → N mixers de rodio**, cada uno con una sola fuente.
Propuesto: **1 `Output`/mixer por app**, al que cualquier motor de reproducción activo le agrega su(s) fuente(s) con un `SmoothGain` propio.

Esto requiere:
1. Sacar la construcción del `Output`/`stream.mixer()` de `Output::open()` (que hoy vive dentro de cada `Paced::open()`) y subirla a un nivel más alto — construido una vez, compartido (`Arc`) entre todos los proveedores activos.
2. Adaptar `Paced` para aceptar un handle al mixer compartido en vez de abrir uno propio.
3. Durante una transición, en vez de un solo `Job::Queue` + corte duro, el orquestador de crossfade (componente nuevo, ver abajo) agrega **dos** fuentes al mismo mixer a la vez: la que termina (con una rampa de volumen bajando) y la que empieza (con una rampa subiendo) — cada una es la misma envolvente `SmoothGain` que ya existe, solo que manejada por una cuenta de tiempo en vez de por el volumen que fija el usuario.

### Camino para Spotify (prioridad, validado)

- El orquestador, al detectar que faltan `duration_ms` para el final de la pista actual, construye un **segundo** `librespot_playback::player::Player` sobre el mismo `Session` clonado (exactamente como en el spike), le manda `load()` de la siguiente pista, y conecta su `OutputSink` al mixer compartido con una rampa de entrada.
- Al mismo tiempo, sustituye (o envuelve) el sink del `Player` que se está terminando por uno con rampa de salida.
- Cuando la rampa de salida llega a cero, se descarta el primer `Player` (drop) y el segundo pasa a ser "el actual" — mismo patrón de `settle()`/`epoch` que ya usa `engine.rs` para invalidar lo viejo.
- **No se toca el código del fork de librespot.** Todo esto se construye sobre su API pública (`Player::new`, `Sink` trait), igual que hace Sonora hoy.

### Camino para las demás plataformas (extra, mismo patrón, menor riesgo)

- Subsonic, Deezer, Apple Music y archivos locales ya comparten (o son estructuralmente iguales a) `engine.rs`, que Sonora controla por completo — no hay una sesión externa de por medio como con Spotify Connect, así que no hace falta validar nada: se abre el segundo `Playing<F>` en paralelo dentro del mismo `audio_loop`, con la misma pareja de rampas, sumado al mismo mixer compartido.
- YouTube Music tiene su propio módulo de playback (`youtube/playback.rs`, no confirmado si reusa `engine.rs`) — a revisar en la fase de implementación si necesita el mismo tratamiento o uno análogo.
- Transición que cruza de un proveedor a otro (p. ej. termina en Spotify, sigue en un archivo local) se degrada a un fade simple sin superposición (bajar a silencio, sin blend real) — no hay un tipo de decodificador compartido entre el `Player` de librespot y el `Playing<F>` de `engine.rs`, así que no hay dos fuentes que mezclar de verdad ahí. Se acepta como límite razonable dado que la mayoría de la cola real del usuario es Spotify→Spotify.

### Ajustes de usuario

Nuevo bloque en `settings.json`, junto a `gapless`/`normalisation` ya existentes:

```json
"crossfade": {
  "enabled": false,
  "duration_ms": 6000
}
```

- `enabled`: apagado por defecto (aditivo, no cambia el comportamiento actual de quien no lo activa).
- `duration_ms`: ventana de superposición, tópicamente 2000–12000.
- La curva de fade queda fija en **equal-power** (mantiene el volumen percibido constante durante la transición) — no se expone como opción; es de esas configuraciones que casi nadie toca y solo añade superficie de UI. Se puede añadir después si alguien lo pide.

### Casos límite

Reusando la idea de `epoch` que ya invalida cargas en curso en `engine.rs` — cualquier acción explícita del usuario **cancela** el fade en curso en vez de intentar conciliarlo:

| Caso | Comportamiento |
|---|---|
| Pausa durante el fade | Ambas rampas se congelan en su punto exacto; reanudan igual al dar play. |
| Seek manual durante el fade | Se cancela: se descarta el decodificador saliente, el que queda salta a volumen 100%. |
| Saltar de pista a mano durante el fade | Se cancela, igual que el seek — no se encadena un segundo fade sobre el primero. |
| Pista siguiente más corta que `duration_ms` | Se recorta a `min(duration_ms, duración_siguiente / 2)`. |
| La pista siguiente falla al cargar | No se inicia fade hacia nada — cae al comportamiento actual (`PlaybackEvent::Unavailable`). |
| Cola algorítmica (radio/autoplay) decide tarde qué sigue | Si no hay suficiente antelación para decodificar, se degrada a fade corto o al gapless normal — nunca se bloquea la reproducción esperando. |
| Cambio de volumen del usuario durante el fade | Se aplica como multiplicador sobre la rampa en curso, no la reemplaza (mismo patrón que ya usa `SmoothGain` con el volumen normal). |

## Orden inteligente de playlist (función independiente, compone con el crossfade)

**Hallazgo que descarta la ruta obvia**: Spotify cerró permanentemente su API pública de `audio-features`/`audio-analysis` (BPM, tonalidad, energía) el 27 de noviembre de 2024, sin reemplazo, para cualquier app nueva. No se puede pedir esa metadata a Spotify por la vía pública. (Su API interna "Pathfinder", que Sonora ya usa para todo lo demás, podría tenerlo, pero confirmarlo implica ingeniería inversa de un endpoint no documentado — frágil, puede romperse sin aviso, no se investiga en esta spec.)

**Sistema ya probado en el que basarse**: [bliss-rs](https://github.com/Polochon-street/bliss-rs) (`bliss-audio` en crates.io), usado en producción real por [blissify-rs](https://github.com/Polochon-street/blissify-rs) para "smart playlists" de MPD.

- Decodifica el audio real de cada pista (soporta **Symphonia**, que Sonora ya usa — no añade un decodificador nuevo) y calcula un vector de características: tempo, timbre, croma, sonoridad.
- Ordena una lista por distancia (euclidiana, o Mahalanobis si se quiere pesar unas características más que otras) desde una pista semilla — "las que suenan parecido quedan cerca".
- Licencia **GPL-3.0**, igual que Sonora — sin conflicto.
- Costo real a asumir: usa la librería C `aubio` por debajo (vía `aubio-rs`) para el análisis espectral — no es 100% Rust puro, aunque es una dependencia pequeña y común, muy por debajo del costo de algo como Essentia o el shim de C++ que ya usa el feature opcional de Widevine.

**Cómo encaja**: es un paso de *análisis*, no de reproducción — se corre una vez por pista (con caché en `storage`, igual que ya se cachean carátulas y letras) y decide el **orden** de la cola antes de reproducir; el mecanismo de crossfade no cambia. Para Spotify en particular, analizar una pista exige tener sus bytes de audio, así que reordenar una playlist completa implica descargar/decodificar cada pista por adelantado — un costo de tiempo/red real que **debe ser una acción explícita del usuario** ("reordenar esta playlist para que mezcle mejor"), nunca algo automático y silencioso en cada reproducción.

Fuera de alcance de esto: mezcla armónica por tonalidad tipo Camelot Wheel (Mixxx, Serato) — eso es para beatmatching real, que el usuario ya descartó al elegir el nivel "auto-crossfade en cola normal" en vez de "dos decks manuales" o "DJ completo".

## Testing

- **`engine.rs` / motor genérico**: tests unitarios sobre la lógica de solapamiento (cuándo arranca el segundo `Playing<F>`, cómo se recorta `duration_ms`, qué pasa si la carga de la siguiente pista falla) — extendiendo los tests ya existentes en el módulo (`settle`, `Joined`, etc. ya tienen cobertura como referencia de estilo).
- **Camino de Spotify**: no es testeable con mocks razonables (depende de dos `Player` reales) — se valida con pruebas manuales dirigidas (como el spike, pero con audio real) antes de dar por buena cada iteración.
- **bliss-rs**: tests de que el orden resultante es determinista para un mismo conjunto de pistas analizadas, y que el caché de análisis no se recalcula si la pista no cambió.
- Verificación manual obligatoria en la app real (Sonora corriendo en Zorin) para el crossfade en sí — es un cambio de la ruta de audio, que ninguna suite de tests reemplaza para "¿suena bien?".

## Trabajo futuro no incluido en esta spec

- Disco de vinilo visual en el "now playing" (Sonora hoy solo tiene `chrome/player_bar.rs`, sin vista de pantalla completa) — quedó como interés secundario del usuario, no priorizado en esta ronda.
- Mezcla armónica real / BPM sync / plato manual (nivel Mixxx) — descartado explícitamente por el usuario por esfuerzo desproporcionado frente al auto-crossfade elegido.
- Investigar si la API "Pathfinder" interna de Spotify expone datos tipo audio-features, como alternativa a bliss-rs.
