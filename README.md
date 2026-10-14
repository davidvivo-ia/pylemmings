# pylemmings

Un clon de *Lemmings* (DMA Design, 1991) escrito desde cero en **Python moderno (3.12+)**, en **un único fichero: `pylemmings.py`**. Trae 50 niveles propios en cinco dificultades, las 8 habilidades, sonido y música sintetizados, demos y repeticiones.

No usa gráficos, sonidos ni niveles originales. Todo lo que se ve se dibuja por código y los niveles están escritos como texto dentro del propio fichero. Del original solo se toma la **mecánica**, que no tiene copyright. Su **expresión** sí lo tiene: sprites, niveles, música y nombres.

```bash
uv run pylemmings.py                 # jugar (uv instala pygame-ce y numpy solo)
python pylemmings.py --selftest      # física y los 50 niveles, sin ventana
python pylemmings.py --demo 10       # ver la solución automática del nivel 10
```

---

## 1. Requisitos y ejecución

- Python **3.12 o superior**.
- **pygame-ce** ≥ 2.5 y **numpy** ≥ 2.0.

Las dependencias van declaradas dentro del fichero con **PEP 723**, así que `uv run pylemmings.py` lo resuelve todo sin `requirements.txt`. La alternativa clásica:

```bash
python -m venv .venv && source .venv/bin/activate
pip install pygame-ce numpy
python pylemmings.py
```

| Opción | Efecto |
|---|---|
| `--level N` | Empieza en el nivel N, sin pasar por el menú. |
| `--scale S` | Escala de la ventana. Por defecto 3, y se reduce sola si no cabe en la pantalla. |
| `--demo N` | El autopiloto juega la solución del nivel N. |
| `--replay F` | Reproduce una partida guardada con `R` en la pantalla de resultado. |
| `--all` | Desbloquea todos los niveles. |
| `--mute` | Sin sonido. |
| `--selftest` | Tests de física y de niveles, sin ventana. |

El progreso se guarda en `~/.pylemmings.json`. Si ese fichero no se puede escribir, el juego funciona igual.

---

## 2. Controles

| Tecla o ratón | Acción |
|---|---|
| Clic izquierdo en el mapa | Asignar la habilidad seleccionada |
| `1`…`8` o clic en el panel | Elegir habilidad |
| `+` / `-` (o `F2` / `F1`) | Subir o bajar la tasa de salida. Manteniendo pulsado el botón del panel, se repite. |
| `Espacio` / `P` | Pausa (se puede asignar en pausa) |
| `F` | Avance rápido ×4 |
| `N` dos veces, o el botón rojo | Bomba nuclear |
| Flechas, `A`/`D`, ratón en el borde, rueda | Scroll (`Shift` va más rápido) |
| Clic en el minimapa | Saltar a esa zona |
| `M` | Música sí/no |
| `Esc` | Abandonar el nivel / volver al menú |

**Prioridad del cursor:** si hay varios lemmings bajo el cursor, la habilidad va al que puede recibirla. Por ejemplo, nunca se da un constructor a uno que ya está construyendo.

---

## 3. Los cincuenta niveles

Todos son de diseño propio. Están agrupados en cinco dificultades de diez niveles; en el menú, izquierda y derecha cambian de grupo. Cada nivel lleva su solución automática, que el `--selftest` ejecuta para demostrar que se puede ganar.

| Grupo | Niveles | Lemmings | Tasa | Margen de rescate | Habilidades de sobra |
|---|---|---|---|---|---|
| Paseo | 1–10 | 10–16 | 40–50 | amplio | muchas |
| Ingenio | 11–20 | 12 | 50 | ~20 % | +2 por habilidad |
| Sudor | 21–30 | 16 | 60 | ~12 % | +1 |
| Caos | 31–40 | 24 | 70 | ~6 % | +1 |
| Leyenda | 41–50 | 32–40 | 80 | un lemming | ninguna |

Los niveles 1–10 están diseñados a mano y enseñan una habilidad cada uno. Los 11–50 se compusieron con un taller de diseño, que no forma parte del repo, a partir de piezas con solución conocida:

- fosos de agua o fuego para el constructor, de 3 a 6 bloques;
- muros normales y de sentido único para el excavador;
- suelos que hay que cavar, a veces con una sola grieta en el acero;
- acantilados para el picador o para paraguas, y muros de escalada;
- pozos de los que se sale construyendo, escalones de acero y suelos finos para dinamita;
- trampas que hay que salvar por encima, rampas, techos de cueva, niveles espejados y niveles con dos entradas que convergen en una salida central.

Después, cada nivel se calibró simulándolo: el mínimo de rescate, el tiempo (entre 2 y 3,2 veces lo que tarda el autopiloto, nunca menos de 3 minutos) y las habilidades de sobra dependen de su grupo. Los mapas resultantes están escritos en ASCII dentro de `pylemmings.py`, como los diez primeros, y se pueden editar a mano.

Leyenda de los mapas (cada carácter es un bloque de 8×8 px; si un mapa tiene menos de 20 filas, se rellena con aire por arriba):

```
.  aire      #  tierra     S  acero      < >  un solo sentido
/  rampa ↗   \  rampa ↘    _  medio bloque     =  plataforma fina
~  agua      ^  fuego      T  trampa
E  entrada → e  entrada ←  X  salida (con suelo debajo)
```

Un escalón de un bloque (8 px) ya es un **muro**, porque supera los 6 px que puede subir un caminante. Las rampas y los medios bloques sí se pueden caminar.

---

## 4. Arquitectura del fichero único

```
pylemmings.py
├── 1. CONSTANTES      tick, física, máscaras de excavación
├── 2. SPRITES         pixel art como cadenas de texto
├── 3. TERRENO         Terrain: mapa numpy uint8 [y, x] + colores
├── 4. NIVELES         mapas ASCII, parser y soluciones (Rule)
├── 5. LEMMING         dataclass; la posición es el píxel de los PIES
├── 6. MUNDO           World.tick(): determinista, sin pygame
│      Autopilot / Replay / simulate()
├── 7. RENDER          Renderer (solo lee el mundo) y Sfx sintetizado
├── 8. INTERFAZ        App: menú, previa, partida, resultado
├── 10. SELFTEST       física y niveles, sin ventana
└── 9. main()          argumentos de línea de comandos
```

Las decisiones de diseño:

- **El terreno es un mapa de bytes.** Chocar es leer un píxel. Destruir es aplicar una máscara booleana de numpy (círculo, túnel o diagonal), y el acero y las paredes de un solo sentido se respetan. Construir es pintar píxeles. La superficie visible se regenera solo en el rectángulo que ha cambiado.
- **Paso fijo de 17 ticks/s y aritmética entera.** El render va a 60 fps por separado.
- **El mundo es determinista y no depende de pygame.** Las órdenes del jugador son tuplas (`("assign", id, skill)`, `("rr", v)`, `("nuke",)`) que se aplican antes de un tick concreto. Así una repetición es solo la lista de órdenes con su tick, y el autopiloto usa el mismo camino que el ratón.
- **Los bordes laterales del mapa son muros que no se pueden trepar.** Caer por debajo del mapa mata.

---

## 5. Física

Las constantes están al principio del fichero. Se han **contrastado con el código fuente del port de Palm OS** (Ardiri y Kerchner, el más cercano al original) **y con Lix**. Lix trabaja a doble resolución, así que sus valores se dividen entre dos.

| Parámetro | pylemmings | Referencia |
|---|---|---|
| Ticks/s | 17 | Lix 15, Palm 12,5 (escalado); DOS ≈ 15–17 |
| Escalón que se sube | 6 px | Lix 12 hi-res = 6 |
| Bajada sin caer | 3 px | Lix < 7 hi-res ≈ 3 |
| Caída | 3 px/tick | DOS constante 3 (Lix acelera a propósito) |
| Caída mortal | > 60 px | Lix > 63, Palm ≥ 56 (escalado) |
| Paraguas | se abre tras ~30 px | Lix 60 hi-res = 30 |
| Escalador | ½ px/tick | Palm 1 px cada 2 frames; Lix 8 hi-res / 8 frames |
| Constructor | 12 ladrillos de 6×1, +2 / −1 por ladrillo, 16 ticks cada uno, aviso en los 3 últimos | Coinciden Palm y Lix |
| Cavador | 9 px de ancho, 1 fila cada 8 ticks | Palm 1 fila/8 frames; Lix 18 hi-res = 9 |
| Picador | +2 / +1 cada 12 ticks | Pendiente 1:2 |
| Bombardero | 5 s, «¡oh, no!» de 16 ticks (sin él si va por el aire), cráter de radio 16 centrado 6 px sobre los pies | Lix: centro 6 px sobre el pie, radio 11 |
| Bloqueador | campo de ±6 px | Lix \|dx\| < 14 hi-res ≈ 7 |
| Tasa de salida | `max(4, 53 − RR // 2)` ticks | Equivale a la fórmula *legacy* de Lix `4 + (99 − RR) / 2` |

El cráter es deliberadamente más grande que el de Lix: el nivel 7 está diseñado para ese radio.

---

## 6. Calidad y tests

```bash
python pylemmings.py --selftest
ruff check pylemmings.py
```

El selftest comprueba, sin ventana:

- **Física:**
  - Un escalón de 6 px se sube y uno de 7 px es un muro.
  - Una caída de 60 px se sobrevive y una de 61 px mata.
  - El paracaidista sobrevive a 110 px.
  - El constructor pone 12 ladrillos de 6 px.
  - El acero resiste la bomba y la tierra no.
  - El cavador se para en el acero.
  - El excavador atraviesa un muro, salvo uno de sentido contrario.
  - El picador baja en pendiente 1:2.
  - El escalador corona el muro.
  - El bloqueador da media vuelta a los demás.
- **Niveles (los 50):** cada uno se gana con su solución, una repetición grabada reproduce la partida exactamente y dos ejecuciones dan el mismo terreno final.

---

## 7. Fuentes consultadas

**Dato:** el código original de 1991 nunca se ha publicado; los derechos son de Sony (heredera de Psygnosis).

| Fuente | Estado | Uso |
|---|---|---|
| Port de Palm OS, `github.com/ardiri/palmos-lemmings` (y el `lemmings-4.2-foss.zip` de palmdb.net) | Descargado y leído | Física de referencia |
| **Lix**, `github.com/SimonN/LixD` | Descargado y leído | Física (dividida entre dos) |
| **Pingus**, `github.com/Pingus/pingus` | Descargado | Consulta |
| **rustlings**, `github.com/DirtyHairy/rustlings` (GPLv3) | Descargado | Formatos; no se copia código, por la licencia |
| **modlem**, `github.com/sulix/modlem`, y `digger-decoder` | Descargados | Formatos de gráficos |
| **resource_dasm** (`lemmings_render`) | Descargado | Formatos Mac |
| `lemmings_lvl_file_format.txt` (camanis.net) | Descargado | Formato de nivel DOS |
| **Mike Dailly**, `github.com/mikedailly/Lemmings` | **No accesible**: esa ruta no responde como repositorio público | Pendiente de verificar la URL |
| **NeoLemmix** | La web responde, pero no se ha localizado el repositorio de código | Pendiente |
| **lemmings_3ds** | Sin URL concreta | Pendiente |

De ninguna de estas fuentes se ha copiado código ni assets. Solo se han contrastado números.

---

## 8. Aviso legal

*Lemmings* es una obra protegida cuyos derechos pertenecen a Sony Interactive Entertainment (antes Psygnosis). Este proyecto no está afiliado a sus titulares ni lo respaldan. **No contiene ni distribuye** código, gráficos, sonidos ni niveles originales.

La melodía es una composición propia y los niveles son de diseño propio.

Licencia: **MIT** (ver [`LICENSE`](LICENSE)).
