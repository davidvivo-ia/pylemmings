# pylemmings

Un clon de *Lemmings* (DMA Design, 1991) escrito desde cero en **Python moderno (3.12+)**, en **un único fichero: `pylemmings.py`**.

Sin gráficos, sonidos ni niveles originales. Todo lo que se ve en pantalla se dibuja por código y todos los niveles se definen dentro del propio fichero. Del original solo copiamos la **mecánica**, que no tiene copyright. Su **expresión** sí lo tiene: sprites, niveles, música y nombres de nivel.

---

## 1. Punto de partida: qué existe y qué no

**Dato:** el código fuente original de 1991 (Amiga/DOS) **nunca se ha publicado oficialmente**. Los derechos son de Sony, heredera de Psygnosis. Si alguien ofrece "el fuente original", desconfiad.

Lo que sí existe, de más a menos cercano al original, y para qué sirve en este proyecto:

| Fuente | Qué es | Uso en pylemmings |
|---|---|---|
| **Mike Dailly – Lemmings para ZX Spectrum Next** (`github.com/mikedailly/Lemmings`) | Reescritura en Z80 de uno de los programadores originales. Código © Dailly 2017-2020, con permiso de uso libre, pero **sin** permiso para distribuirlo con gráficos o niveles originales. Trae documentación recopilada de los formatos de nivel y un conversor en C# de los assets de Windows. | **Referencia principal de la mecánica**: bucle de las criaturas, terreno destructible, máscaras. Sirve para leer y entender, no para copiar assets. |
| **Port de Palm OS** (Ardiri & Kerchner, `palmdb.net/app/lemmings`) | Port no oficial con el código fuente completo liberado. | Segunda opinión sobre el bucle de juego en una máquina con pocos recursos. |
| **rustlings** (`github.com/DirtyHairy/rustlings`, GPLv3) | Decodificador y visor de los ficheros DOS, en Rust con SDL2. | Para entender los formatos. **No** se copia código: pylemmings es MIT y la GPLv3 contaminaría la licencia. |
| **modlem** (`davidgow.net/hacks/modlem.html`) | Herramienta para modificar los gráficos del port DOS. | Formato de sprites (solo como lectura). |
| **resource_dasm / lemmings_render** | Genera mapas de niveles a partir de la versión Mac clásica. | Visualizar niveles originales y estudiar el diseño. |
| **lemmings_3ds** | Homebrew en C para 3DS. | Estructura de un port pequeño en C. |
| **NeoLemmix** (Delphi), **Lix** (D), **Pingus** (C++) | Clones libres. NeoLemmix es la réplica más fiel de la física. *(Sin verificar a día de hoy.)* | **Valores numéricos de la física** (tabla de la sección 5) y casos límite. |

**Orden de estudio recomendado:** primero Dailly (quien diseñó el motor), luego NeoLemmix (para cotejar números y casos límite) y por último rustlings (formatos).

---

## 2. Reglas del proyecto (no negociables)

1. **Un solo fichero, `pylemmings.py`.** No hay paquetes, ni carpeta `assets/`, ni `levels/`. Si algo no cabe en el fichero, sobra.
2. **Cero assets originales.** Los sprites son matrices de píxeles definidas como texto en el código, y los niveles son ASCII o se generan proceduralmente.
3. **Física determinista.** Usamos *ticks* fijos y aritmética entera, y no aparece `dt` en la lógica. La misma partida con las mismas órdenes debe dar siempre el mismo resultado. Esto permite repeticiones (*replays*) y tests.
4. **La lógica no sabe que existe pygame.** El núcleo de simulación funciona sin ventana, y así se puede testear en modo *headless*.
5. **Licencia MIT** (ver `LICENSE`). Se puede leer código GPL para entender ideas, pero no se copia.

---

## 3. Requisitos y ejecución

- Python **3.12 o superior** (se usan `match`, `dataclass(slots=True)`, `enum.StrEnum` y tipado moderno).
- **pygame-ce** ≥ 2.5 (el fork comunitario mantenido de pygame).
- **numpy** ≥ 2.0 para el mapa de terreno.

El fichero lleva sus dependencias embebidas con **PEP 723** (*inline script metadata*), así que no hace falta `requirements.txt`:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["pygame-ce>=2.5", "numpy>=2.0"]
# ///
```

Para ejecutarlo:

```bash
# Opción recomendada (uv crea el entorno solo)
uv run pylemmings.py

# Opción clásica
python -m venv .venv && source .venv/bin/activate
pip install pygame-ce numpy
python pylemmings.py
```

Opciones de línea de comandos previstas:

```bash
python pylemmings.py --level 3        # empezar en un nivel concreto
python pylemmings.py --scale 3        # escala de ventana (pixel art nítido)
python pylemmings.py --selftest       # tests de física sin ventana (headless)
python pylemmings.py --replay f.json  # reproducir una partida grabada
```

---

## 4. Arquitectura dentro del fichero único

El fichero se ordena en secciones fijas, de arriba abajo. Cada sección solo depende de las que tiene encima.

```
pylemmings.py
├── 0. Cabecera PEP 723, docstring, imports
├── 1. CONSTANTES      tick, tamaños, tabla de física (sección 5)
├── 2. SPRITES         pixel art como cadenas de texto → Surfaces al arrancar
├── 3. TERRENO         clase Terrain: mapa numpy uint8 + operaciones de máscara
├── 4. NIVELES         definiciones ASCII/procedurales + parser
├── 5. LEMMING         dataclass + máquina de estados (un método por estado)
├── 6. MUNDO           clase World: tick(), entradas, trampas, salida, recuento
├── 7. RENDER          dibuja World sobre una Surface; no muta nada
├── 8. INTERFAZ        panel de habilidades, cursor, minimapa, scroll
├── 9. BUCLE PRINCIPAL lógica a paso fijo + render libre
└── 10. SELFTEST       aserciones de física sin ventana
```

### 4.1 Bucle a paso fijo

El original avanzaba la lógica a unos **17 ticks por segundo** (≈ 60 ms por tick). Ese ritmo es parte de la sensación del juego, así que se respeta. El render puede ir a 60 fps.

```python
TICK_MS = 60
acc = 0
while running:
    acc += clock.tick(60)
    while acc >= TICK_MS:
        world.tick()          # lógica pura, determinista
        acc -= TICK_MS
    render(world, screen)     # solo lee
```

El avance rápido (*fast-forward*) llama a `world.tick()` varias veces por iteración y nunca cambia `TICK_MS`.

### 4.2 Terreno: un mapa de bytes, no sprites

Esta es la clave del motor original y también de este proyecto. El nivel es **un mapa de bits** en el que cada píxel guarda un tipo:

```python
AIR, DIRT, STEEL, ONEWAY_L, ONEWAY_R, WATER, FIRE = range(7)
terrain = np.zeros((H, W), dtype=np.uint8)   # [y, x]
```

- **Colisión** = consultar un píxel: `terrain[y, x] != AIR`. No hay *bounding boxes*.
- **Destruir** (bombardero, excavador, picador, minero) = aplicar una **máscara booleana** con `np.where`. El acero nunca se borra.
- **Construir** (constructor) = pintar píxeles `DIRT`.
- La *Surface* visible se regenera solo en la zona sucia (*dirty rect*), nunca entera en cada tick.
- Los bordes del mapa se tratan como aire en los laterales y como muerte por debajo del suelo.

### 4.3 El lemming

```python
@dataclass(slots=True)
class Lemming:
    x: int            # píxel de los PIES, no esquina superior
    y: int
    dir: int          # +1 derecha, -1 izquierda
    state: State      # WALK, FALL, FLOAT, CLIMB, BUILD, BASH, MINE, DIG, BLOCK, ...
    frame: int = 0
    fall_dist: int = 0
    is_climber: bool = False   # permanente
    is_floater: bool = False   # permanente
    bomb_timer: int | None = None
    bricks_left: int = 0
```

La posición se toma en **los pies**, igual que en el original y en NeoLemmix. Todas las consultas de terreno son relativas a ese píxel.

La máquina de estados es un `match self.state:` que despacha a un método por estado. Cada método solo puede devolver un estado nuevo o `None`, y no toca el estado de otros lemmings. La única excepción son los bloqueadores: `World` resuelve su efecto antes de mover a nadie.

---

## 5. Física de referencia

Valores tomados de la documentación comunitaria y de NeoLemmix, que reproduce el original. **Hay que cotejarlos con el repo de Dailly antes de dar el motor por cerrado.** Están todos en el bloque `CONSTANTES` para poder ajustarlos sin buscar números mágicos.

| Parámetro | Valor de referencia | Confianza |
|---|---|---|
| Ticks de lógica | ~17/s (60 ms) | Alta |
| Tamaño de nivel (DOS) | 1600 × 160 px | Alta |
| Ventana de juego | 320 × 160 px + panel inferior | Alta |
| Avance al caminar | 1 px/tick | Alta |
| Escalón máximo que sube caminando | 6 px (si es más alto: da la vuelta, o trepa si es escalador) | Alta |
| Bajada máxima sin pasar a caer | 3 px | Media |
| Velocidad de caída | 3 px/tick | Media |
| Caída mortal | > 60 px (sin paraguas) | Alta |
| Constructor | 12 ladrillos de 6 px de ancho y 1 px de alto; aviso en los 3 últimos | Alta |
| Bombardero | Cuenta atrás de 5 s, cráter circular, no destruye acero | Alta |
| Tasa de salida (*release rate*) | 1–99; intervalo ≈ `53 - RR // 2` ticks | Media, a verificar |
| Lemming | ≈ 10 px de alto | Media |

Algunos **casos límite** que el motor debe reproducir porque forman parte del diseño de niveles clásico:

- El bloqueador da la vuelta a los que llegan, pero **no** a los que caen sobre él.
- Si un constructor choca contra la pared, se detiene y da media vuelta.
- Si un excavador o picador toca acero, se detiene.
- Una pared de un solo sentido (*one-way*) solo deja excavar en su dirección.
- Escalador más paracaidista es un *athlete*: ambos rasgos son permanentes y se acumulan.
- La bomba nuclear (*nuke*) pone a todos en cuenta atrás y corta la salida de nuevos lemmings.

---

## 6. Las ocho habilidades

| Tecla | Habilidad | Tipo | Efecto sobre el terreno |
|---|---|---|---|
| `1` | Escalador (*Climber*) | permanente | — |
| `2` | Paracaidista (*Floater*) | permanente | — |
| `3` | Bombardero (*Bomber*) | temporizada | borra un círculo |
| `4` | Bloqueador (*Blocker*) | estado | — |
| `5` | Constructor (*Builder*) | estado | pinta una escalera |
| `6` | Excavador (*Basher*) | estado | túnel horizontal |
| `7` | Picador (*Miner*) | estado | túnel diagonal hacia abajo |
| `8` | Cavador (*Digger*) | estado | pozo vertical |

Otros controles:

- `+` / `-`: subir o bajar la tasa de salida.
- `P`: pausa.
- `F`: avance rápido.
- `N` dos veces: bomba nuclear.
- Flechas o ratón en el borde: scroll.
- Clic izquierdo: asignar la habilidad seleccionada.

**Prioridad del cursor:** cuando hay varios lemmings bajo el cursor, la habilidad se asigna al que puede recibirla y no a uno que ya la tiene. Por ejemplo, no se da un constructor al que ya está construyendo. Sin esta regla el juego resulta injugable.

---

## 7. Niveles

Los niveles se definen en el propio fichero como **ASCII**, con una leyenda simple, y se escalan a píxeles al cargarlos:

```
# Cada carácter equivale a un bloque de 8×8 px
LEVEL_1 = Level(
    name="Primeros pasos",
    lemmings=10, to_save=5, release_rate=50, time_s=300,
    skills={"dig": 10},
    map="""
.....E..........................
................................
######################.....#####
######################.....##X##
################################
""")
# E = entrada (trampilla)   X = salida   # = tierra
# S = acero   < > = un solo sentido   ~ = agua   ^ = fuego
```

El objetivo es tener unos 10 niveles de dificultad creciente, **todos de diseño propio**. Está prohibido transcribir niveles originales, aunque sea "de memoria".

A medio plazo se añadirá un generador procedural con semilla (`--seed`) para partidas infinitas.

---

## 8. Hoja de ruta

Cada hito es jugable y se puede comprobar por sí mismo:

1. **Terreno + caminante.** Un mapa numpy, un lemming que camina, sube escalones, cae y muere. `--selftest` verifica escalón y caída mortal.
2. **Entrada, salida, tasa de salida y recuento.** Ya es un juego: se gana o se pierde.
3. **Habilidades destructivas:** cavador, excavador, picador y bombardero (máscaras).
4. **Constructor y bloqueador.**
5. **Escalador y paracaidista.**
6. **Acero, paredes de un solo sentido, agua, fuego y trampas.**
7. **Interfaz completa:** panel, minimapa, scroll, pausa, avance rápido y bomba nuclear.
8. **Replays deterministas** (lista de `(tick, lemming_id, skill)` en JSON).
9. **Sonido sintetizado** con `pygame.sndarray`, sin muestras externas.
10. **Pulido:** paletas de color, animaciones y transiciones.

---

## 9. Calidad

- `ruff check pylemmings.py && ruff format pylemmings.py` antes de cada commit.
- Tipado completo y validado con `mypy --strict` o `pyright`.
- `python pylemmings.py --selftest` debe pasar siempre. Comprueba la física en modo *headless* (`SDL_VIDEODRIVER=dummy`): escalón de 6 px frente a 7 px, caída de 60 px frente a 61 px, que el constructor coloca 12 ladrillos y que el acero resiste al bombardero.
- Si los tests empiezan a crecer, se pueden mover a `test_pylemmings.py` con pytest. Es la única excepción tolerada a la regla del fichero único, y solo para tests.

---

## 10. Aviso legal

*Lemmings* es una marca y una obra protegida cuyos derechos pertenecen a Sony Interactive Entertainment (antes Psygnosis). Este proyecto no está afiliado a sus titulares ni lo respaldan. **No contiene ni distribuye** código, gráficos, sonidos ni niveles originales. Las mecánicas de juego no son objeto de copyright, pero su expresión sí, y por eso aquí todo es de creación propia.

Licencia: **MIT** (ver [`LICENSE`](LICENSE)).
