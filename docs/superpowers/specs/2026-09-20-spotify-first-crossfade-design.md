# Sonora — crossfade, orden de cola, vinilo, letras, nivelación de volumen y fade del sleep timer

**Fecha**: 2026-09-20
**Base**: fork de [sonorahq/sonora](https://github.com/sonorahq/sonora) en `rev e7eb953d5848fd97bacd00e6c0e33500765eaa63` (Cargo.lock actual del upstream)
**Punto de partida decidido**: extender/forkear Sonora (no una app nueva desde cero, no un skin tipo Spicetify) — Sonora ya resuelve el problema más caro (agregación multi-fuente: Spotify + YouTube Music + archivos locales + letras), y es Rust/GPUI activamente mantenido.

## Contexto y motivación

El usuario reportó que el "DJ" de Sonora no funciona. Investigación previa confirmó que es el AI DJ de Spotify (locución + selección automática), un endpoint interno propietario que ningún cliente de terceros (Sonora incluido, vía librespot) puede replicar sin atestación criptográfica del cliente oficial — no es un bug, es una limitación estructural fuera de alcance.

A partir de ahí, el usuario pidió analizar otros reproductores con funciones similares/distintas y proponer cómo incorporar lo mejor de cada uno en una app custom basada en Sonora. Tras una ronda de brainstorming y verificación directa del código, la lista se cerró en **6 funciones**:

1. Crossfade real, centrado en Spotify (el reproductor principal del usuario), con las demás plataformas como extra.
2. Orden inteligente de playlist.
3. Disco de vinilo visual en pantalla completa.
4. Arreglo real de las letras que fallan.
5. Nivelación de volumen (estilo ReplayGain) donde falta.
6. Fade-out real en el sleep timer.

**Spotify es el reproductor principal del usuario.** Toda decisión de diseño prioriza que estas funciones funcionen lo mejor posible ahí; las demás plataformas (YouTube Music, Subsonic, Deezer, Apple Music, archivos locales) las reciben como "extra", en la medida que la arquitectura ya compartida de Sonora lo permite sin esfuerzo desproporcionado.

**Corrección importante hecha a mitad de esta ronda**: una primera pasada de este documento asumía que Sonora no tenía pantalla completa de "now playing" ni visualizador ni color ambiental desde la portada. Es falso — las tres cosas ya existen (`shells/fullscreen.rs`, `ui::Visualizer`, `views/shared/ambient.rs`) y con un ecualizador de 10 bandas con UI completa en Ajustes. La función 3 (vinilo) se apoya en ese `FullscreenView` ya existente, no en algo nuevo desde cero.

## Fuera de alcance

- **AI DJ real de Spotify** — imposible sin atestación criptográfica propietaria del cliente oficial (ver arriba).
- **Plato/scratch manual, beatmatching por BPM, mezcla armónica tipo Camelot Wheel** (nivel Mixxx/Serato) — el usuario eligió explícitamente el nivel "auto-crossfade en cola normal" en vez de "dos decks manuales" o "DJ completo", por esfuerzo desproporcionado frente al beneficio.
- Investigar si la API "Pathfinder" interna de Spotify expone datos tipo audio-features, como alternativa a bliss-rs — no se investiga en esta spec (ver función 2).

---

## 1. Crossfade real, Spotify primero

### Validación técnica ya hecha (no es hipótesis)

Antes de comprometernos a este diseño, se corrió un spike descartable (`/tmp/.../dual-spike`, fuera del repo) que:

- Instaló Rust en la máquina del usuario.
- Compiló un binario mínimo contra el mismo fork/rev de librespot que usa Sonora.
- Autenticó con la sesión real de Spotify del usuario (reutilizando las credenciales cacheadas por la propia instalación Flatpak de Sonora, vía `librespot_core::cache::Cache` — nunca se leyeron en crudo).
- Levantó **dos instancias de `librespot_playback::player::Player` en paralelo**, ambas sobre el mismo `Session` clonado, cada una con un sink que solo contaba muestras (sin tocar hardware de audio real).

**Resultado**: ambos `Player` cargaron, reprodujeron y decodificaron la pista completa (mismo conteo exacto de muestras: 18 837 168 cada uno), sin error, sin `Unavailable`, sin que uno interrumpiera al otro. Esto confirma que **no hace falta parchear el fork de librespot** para el crossfade en Spotify: se puede construir enteramente sobre su API pública, ejecutando dos `Player` a la vez durante la ventana de transición.

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

---

## 2. Orden inteligente de playlist

**Hallazgo que descarta la ruta obvia**: Spotify cerró permanentemente su API pública de `audio-features`/`audio-analysis` (BPM, tonalidad, energía) el 27 de noviembre de 2024, sin reemplazo, para cualquier app nueva. No se puede pedir esa metadata a Spotify por la vía pública. (Su API interna "Pathfinder", que Sonora ya usa para todo lo demás, podría tenerlo, pero confirmarlo implica ingeniería inversa de un endpoint no documentado — frágil, puede romperse sin aviso, no se investiga en esta spec.)

**Sistema ya probado en el que basarse**: [bliss-rs](https://github.com/Polochon-street/bliss-rs) (`bliss-audio` en crates.io), usado en producción real por [blissify-rs](https://github.com/Polochon-street/blissify-rs) para "smart playlists" de MPD.

- Decodifica el audio real de cada pista (soporta **Symphonia**, que Sonora ya usa — no añade un decodificador nuevo) y calcula un vector de características: tempo, timbre, croma, sonoridad.
- Ordena una lista por distancia (euclidiana, o Mahalanobis si se quiere pesar unas características más que otras) desde una pista semilla — "las que suenan parecido quedan cerca".
- Licencia **GPL-3.0**, igual que Sonora — sin conflicto.
- Costo real a asumir: usa la librería C `aubio` por debajo (vía `aubio-rs`) para el análisis espectral — no es 100% Rust puro, aunque es una dependencia pequeña y común, muy por debajo del costo de algo como Essentia o el shim de C++ que ya usa el feature opcional de Widevine.

**Cómo encaja**: es un paso de *análisis*, no de reproducción — se corre una vez por pista (con caché en `storage`, igual que ya se cachean carátulas y letras) y decide el **orden** de la cola antes de reproducir; el mecanismo de crossfade no cambia. Para Spotify en particular, analizar una pista exige tener sus bytes de audio, así que reordenar una playlist completa implica descargar/decodificar cada pista por adelantado — un costo de tiempo/red real que **debe ser una acción explícita del usuario** ("reordenar esta playlist para que mezcle mejor"), nunca algo automático y silencioso en cada reproducción.

Fuera de alcance de esto: mezcla armónica por tonalidad tipo Camelot Wheel (Mixxx, Serato) — eso es para beatmatching real, descartado arriba.

---

## 3. Disco de vinilo visual

Se apoya en el `FullscreenView` ya existente (`crates/views/src/shells/fullscreen.rs`), que hoy muestra la portada grande con springs/motion y ya dibuja el visualizador de espectro (`ui::Visualizer`) y el fondo de color ambiental (`views/shared/ambient.rs`) detrás. El disco de vinilo es una alternativa a la portada plana actual, no un reemplazo del visualizador ni del color ambiental — ambos pueden seguir activos junto al disco.

- **Ajuste nuevo**: `appearance.cover_style` (`Flat | Vinyl`), mismo patrón que `VisualizerStyle` ya existente — id guardado como string, picker en Ajustes con `CoverStyle::ALL`. Opción alternativa, no reemplazo forzado: quien no la active sigue viendo la portada plana de siempre.
- **Componente nuevo en `ui`**, dibujado con `canvas`/`PathBuilder` — mismo enfoque que `visualizer.rs`/`ambient.rs`, nada de imágenes rasterizadas: círculos concéntricos de bajo contraste para los surcos, la portada recortada en círculo al centro como etiqueta, agujero del eje en el medio.
- **Giro**: velocidad angular constante mientras reproduce (33⅓ RPM real), calculada por tiempo transcurrido — no reactiva al audio, un tocadiscos real gira a velocidad fija. Se congela al instante al pausar (sin desaceleración simulada).
- **Brazo/aguja (tonearm) animado**: con `Springs`/`Motion`, que ya usa `fullscreen.rs` — dos posiciones de destino (reposo / sobre el disco), disparadas por play/pausa. A diferencia del giro (continuo), esto es exactamente para lo que sirven los springs: asentarse en un destino.
- **Costo real a advertir**: GPUI repinta toda la ventana cada frame, sin damage tracking (lo dice el propio `CLAUDE.md` del proyecto) — un giro continuo obliga a repintar todo el rato mientras suena algo, más costo de batería en laptop. Por eso el giro respeta `ui::motion::animates(cx)` (la preferencia de "reducir movimiento" del sistema, el mismo gate que ya usa `ambient.rs` para su campo de fondo) — con esa preferencia activada, el disco se ve pero no gira.

---

## 4. Arreglo real de las letras que fallan

Diagnóstico hecho en vivo, probando los endpoints reales (no es una suposición a partir del log):

- **Apple Music (`lyrics-api.binimum.org`) está muerto.** Probado en vivo: `404` tanto en la raíz como en la búsqueda, cacheado por Cloudflare desde hacía más de 8 horas al momento de probar — no es un bache pasajero, el servicio dejó de existir o cambió de dirección sin avisar. Es un proxy comunitario no oficial de terceros para el catálogo de letras de Apple Music; ningún cambio en Sonora lo revive.
- **LrcLib SÍ está vivo** — probado en vivo, responde 200 con datos reales. El `503` visto en el log del usuario fue casi seguro un bache pasajero de un servicio gratuito pequeño, no algo roto de forma permanente.
- **Musixmatch también respondió vivo** en la prueba. Sonora ya agrega 5 fuentes de letras (Spotify, YouTube Music, Apple Music, Musixmatch, LrcLib) rankeadas por confianza (`lyrics/catalog.rs`, con caída automática a la siguiente si una falla) — Apple Music muerto no es una letra que el usuario esté perdiendo de verdad, ya que las otras 4 fuentes cubren casi todo; es solo peso muerto que gasta una petición y ensucia el log en cada canción.

**Decisión**: quitar el proveedor Apple Music/binimum por completo (no dejarlo desactivado por si "vuelve"):
- Borrar `crates/music/src/binimum/`.
- Quitar `"Apple Music"` de la lista por defecto de `lyrics_providers` en `state/settings.rs`.
- Quitar el punto donde se instancia `Binimum` en el ensamblador de proveedores de letras (a ubicar exactamente en la fase de implementación).

**LrcLib**: agregar un reintento con backoff corto (una repetición tras ~500ms–1s si la respuesta es 5xx) en `crates/music/src/lrclib/mod.rs` — cambio chico y acotado sobre un servicio confirmado vivo.

---

## 5. Nivelación de volumen (estilo ReplayGain)

Hoy Spotify y YouTube Music ya nivelan volumen usando la loudness que cada plataforma provee nativamente (`youtube/playback.rs::normalisation()`, que usa `loudness_db` del propio formato). **Archivos locales, Subsonic, Deezer y Apple Music no tienen ninguna nivelación** — no hay de dónde sacar esa loudness por API, así que dos pistas de golpes de volumen muy distintos suenan dispares entre sí, y el crossfade puede notarse feo aunque la curva equal-power esté bien hecha.

**Base real en la que apoyarse**: [bs1770](https://github.com/ruuda/bs1770) — implementación en Rust puro de la medición de sonoridad ITU-R BS.1770/EBU R128, licencia Apache-2.0 (sin fricción con el GPL-3.0-or-later de Sonora). Es la base que usa **Musium**, otro reproductor en Rust del mismo autor, para el mismo propósito — no es una idea sin precedente real.

- **Primero, tags embebidos**: `lofty` (que Sonora ya usa para leer metadata local, ver `local/tags.rs`) sabe leer `ItemKey::ReplayGainTrackGain` si el archivo ya lo tiene grabado (de foobar2000, Mp3tag, etc.) — costo cero, se usa directo.
- **Si no hay tag**: se calcula una vez con `bs1770` sobre el audio ya decodificado, la primera vez que esa canción específica se reproduce — sin costo por adelantado ni escaneo masivo al importar la biblioteca. El resultado se cachea (mismo patrón de caché por ruta+fecha que ya usa el índice de la biblioteca local — ver "Only what changed is read" en el `CLAUDE.md`), así que las siguientes reproducciones de esa pista son gratis. La primera vez puede sonar sin nivelar mientras se calcula.
- **Se aplica en el mismo punto que ya existe**: como multiplicador sobre el `Chain.volume`/`SmoothGain` que ya maneja el volumen del usuario y el crossfade — no una segunda etapa de ganancia separada, para que componga bien con todo lo demás.
- Spotify/YouTube no cambian — ya nivelan con su propia loudness nativa; esto solo llena el hueco en local/Subsonic/Deezer/Apple.

---

## 6. Fade-out real en el sleep timer

Hoy (`state/playback.rs`), el temporizador de sueño por duración (`Sleep`, el `sleep_task`) simplemente llama `pause()` — un corte seco sin aviso. El de "al final de la canción actual" (`Sleep::EndOfTrack`) ya termina de forma natural y no necesita cambios.

- Cuando el temporizador por duración dispara, en vez de `pause()` directo: arranca la misma rampa de tiempo construida para el crossfade (unos segundos bajando a cero), y solo al llegar a cero llama `pause()` y restaura el volumen normal — para que el siguiente play no salga mudo.
- Si el fade del sleep timer coincide con un crossfade en curso: no se apilan dos rampas — se aplica como multiplicador sobre la que ya esté corriendo, mismo principio de composición que el volumen del usuario durante un crossfade (función 1).
- Si el usuario cancela el temporizador mientras el fade está bajando: se cancela al instante y el volumen vuelve a su nivel normal — mismo criterio de "acción explícita cancela la rampa" del resto del diseño.

---

## Testing

- **`engine.rs` / motor genérico**: tests unitarios sobre la lógica de solapamiento del crossfade (cuándo arranca el segundo `Playing<F>`, cómo se recorta `duration_ms`, qué pasa si la carga de la siguiente pista falla) — extendiendo los tests ya existentes en el módulo (`settle`, `Joined`, etc. ya tienen cobertura como referencia de estilo).
- **Camino de Spotify (crossfade)**: no es testeable con mocks razonables (depende de dos `Player` reales) — se valida con pruebas manuales dirigidas (como el spike, pero con audio real) antes de dar por buena cada iteración.
- **bliss-rs**: tests de que el orden resultante es determinista para un mismo conjunto de pistas analizadas, y que el caché de análisis no se recalcula si la pista no cambió.
- **bs1770 / nivelación de volumen**: tests de que el cálculo de ganancia es determinista para el mismo archivo, y que un tag ReplayGain embebido tiene prioridad sobre el cálculo propio.
- **LrcLib**: test de que el reintento respeta el backoff y no reintenta infinito.
- Verificación manual obligatoria en la app real (Sonora corriendo en Zorin) para crossfade, vinilo y sleep timer — son cambios de audio/UI en vivo que ninguna suite de tests reemplaza para "¿se ve/suena bien?".

## Notas de proceso para la implementación

- El repo vive en `~/Proyectos/sonora-custom`, `origin` = fork del usuario (`Arturitu33311/sonora`), `upstream` = `sonorahq/sonora`.
- Seguir el `CLAUDE.md` del propio repo al implementar: Conventional Commits, sin `Co-Authored-By`, PR de un encabezado y una lista nada más, nunca hacer `git push` sin pedir confirmación antes.
