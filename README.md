# Subway Jack: Juego completo (Hito 2 / Proyecto Final)

Videojuego retro 2D estilo *Subway Surfers* con **vista cenital** y **3 carriles verticales**, implementado en el lenguaje **Jack** del ecosistema **Nand2Tetris** (plataforma Hack).

El **Hito 2** convierte el renderizado estático del Hito 1 en un **juego completo y jugable**: máquina de estados, bucle principal no bloqueante, movimiento del jugador, obstáculos que caen y se reciclan, colisiones AABB, puntaje, vidas, niveles, reinicio y salida controlada. Todos los objetos se crean una sola vez y se liberan con `dispose()`; el heap se mantiene estable entre partidas (ver [Pruebas de control](#pruebas-de-control)).

| Enlace | URL |
|--------|-----|
| Repositorio | https://github.com/The-G-Man-Half-Life/DesarrolloEnJackFinal |
| Video de sustentación | POR CONFIRMAR |

---

## Estructura del repositorio

```
DesarrolloEnJackFinal/
|-- README.md
|-- projects/
|   `-- SubwayJack/      <- código fuente (.jack) y código compilado (.vm), 7 clases
`-- tools/               <- JackCompiler y VM Emulator de Nand2Tetris (requieren Java)
```

---

## Descripción del juego

- Pantalla monocromática de **512×256 px** (x: 0..511, y: 0..255).
- **Zona de juego:** x 128..383 (256 px de ancho), y 0..255.
- El jugador está abajo y los obstáculos **descienden** desde arriba. Esquívalos cambiando de carril.
- **Paneles de HUD** laterales: izquierdo con **PUNTOS / VIDAS / NIVEL** (dinámicos), derecho con la leyenda de controles (estática).

| Carril | Posición | Centro x (px) |
|--------|----------|---------------|
| 0 | Izquierdo | 170 |
| 1 | Central | 256 |
| 2 | Derecho | 341 |

| Obstáculo | Ancho × alto (px) |
|-----------|-------------------|
| Barrera | 56 × 18 |
| Tren | 48 × 60 |
| Cono | 24 × 26 |

---

## Reglas

- Empiezas con **3 vidas** y en **nivel 1**.
- **+1 punto** por cada obstáculo esquivado (sale por abajo sin golpearte).
- Cada **10 puntos** subes de nivel y el juego acelera: la caída es de `2 + (nivel - 1)` px por ciclo (tope 7) y la espera por ciclo es de `40 - 3 × (nivel - 1)` ms (mínimo 16).
- Al **chocar** pierdes 1 vida, el obstáculo golpeado se recicla y quedas **invulnerable** 25 ciclos (cerca de 1 s): el personaje **parpadea** para que no pierdas todas las vidas de golpe.
- Si las vidas llegan a **0**: **GAME OVER**.
- Al llegar al **nivel 10** (90 puntos): **GANASTE!** (condición de victoria).
- Siempre hay carriles libres: cada obstáculo reciclado aparece 70 px por encima del más alto y ningún obstáculo mide más de 60 px de alto, así que nunca hay dos a la misma altura.

---

## Controles

| Tecla | Acción |
|-------|--------|
| **Flecha izquierda** o **A** | Mover al carril de la izquierda |
| **Flecha derecha** o **D** | Mover al carril de la derecha |
| **ESPACIO** o **ENTER** | Iniciar el juego desde la pantalla de inicio |
| **R** | Reiniciar (en GAME OVER o victoria) |
| **ESC** | Salir del juego (en cualquier pantalla) |

El movimiento es **por flanco** (antirrebote): cada pulsación mueve exactamente un carril; mantener la tecla no salta varios carriles.

---

## Máquina de estados

1. **START**: pantalla de inicio con título, instrucciones y "Presiona ESPACIO para jugar". Mientras espera, un contador se incrementa y se usa como **semilla** del generador aleatorio (así cada partida es distinta).
2. **PLAYING**: juego en curso con bucle principal no bloqueante.
3. **GAMEOVER**: muestra "GAME OVER" (o "GANASTE!"), el puntaje y el nivel finales, y las opciones **R: reiniciar** / **ESC: salir**.

---

## Estructura de clases (un archivo por clase)

Todas en [`projects/SubwayJack/`](projects/SubwayJack/).

| Archivo | Responsabilidad |
|---------|-----------------|
| [`Main.jack`](projects/SubwayJack/Main.jack) | Punto de entrada. Crea `Game`, ejecuta `run()` y `dispose()`. |
| [`Game.jack`](projects/SubwayJack/Game.jack) | Orquestador: máquina de estados, game loop no bloqueante, entrada por flanco, colisiones AABB, puntaje/niveles, reinicio y salida. Dueño de `Board`, `Player`, `Hud` y el *pool* de 4 `Obstacle`. |
| [`Board.jack`](projects/SubwayJack/Board.jack) | Escenario estático: marco, bordes, separadores punteados, postes y los **marcos + textos estáticos** del HUD. `clearField()` limpia solo la zona de juego al mostrar GAME OVER. |
| [`Player.jack`](projects/SubwayJack/Player.jack) | Personaje compuesto (primitivas). `moveLeft/moveRight/setLane/reset`, `draw/erase` desacoplados y getters AABB. |
| [`Obstacle.jack`](projects/SubwayJack/Obstacle.jack) | Obstáculo con 3 tipos (barrera, tren, cono). `update(speed)` lo baja, `recycle(...)` lo reubica arriba sin `new`, dibujo **recortado** a la zona de juego. |
| [`Hud.jack`](projects/SubwayJack/Hud.jack) | Textos dinámicos **PUNTOS/VIDAS/NIVEL**: solo repinta un número cuando cambia y borra el anterior con espacios. `Hud.printText(s)` imprime un texto fijo y lo libera. |
| [`Rng.jack`](projects/SubwayJack/Rng.jack) | Generador pseudoaleatorio (LCG de 16 bits). `setSeed(s)` y `nextInt(max)` devuelve 0..max-1. |

### Diseño de memoria
- **Todos los objetos se crean una sola vez** (en `Game.new()`). Durante el juego y al reiniciar **no se llama a `new`**: los obstáculos se **reciclan** y el estado se reinicia en sitio con `reset()`.
- Cada clase implementa `dispose()`, que libera en cadena sus objetos secundarios y termina con `do Memory.deAlloc(this);`.
- En Jack cada literal de texto crea un `String` nuevo cada vez que se ejecuta. Por eso todo texto fijo se imprime con `Hud.printText`, que lo libera con `dispose()` justo después, y los números del HUD se borran con `Output.printChar(32)`, que no reserva memoria.

---

## Compilación y ejecución

**Requisito:** Java 8 o superior en el `PATH` (verificado con Java 11). Las herramientas de Nand2Tetris ya vienen en `tools/`.

Los `.vm` ya están compilados en `projects/SubwayJack/`. La línea del `JackCompiler` solo hace falta si cambias algún `.jack`; compila la **carpeta** completa y genera un `.vm` por clase.

### 1. Abrir el VM Emulator desde la terminal

Reemplaza la ruta del `cd` por la ubicación de tu copia (repositorio clonado o `.zip` descomprimido).

**Windows (PowerShell)**

```powershell
cd "C:\ruta\a\DesarrolloEnJackFinal"
.\tools\JackCompiler.bat projects\SubwayJack
.\tools\VMEmulator.bat
```

**Windows (Git Bash)**

```bash
cd "/c/ruta/a/DesarrolloEnJackFinal"
cmd //c 'tools\JackCompiler.bat projects\SubwayJack'
cmd //c 'tools\VMEmulator.bat'
```

**Linux / macOS (bash)**

```bash
cd ~/ruta/a/DesarrolloEnJackFinal
sh tools/JackCompiler.sh projects/SubwayJack
sh tools/VMEmulator.sh
```

Para obtener una copia nueva: `git clone https://github.com/The-G-Man-Half-Life/DesarrolloEnJackFinal.git`

### 2. Cargar y jugar (pasos manuales dentro del VM Emulator)

1. Menú **File > Load Program**.
2. El diálogo abre en `projects/SubwayJack`. Si abre en otra carpeta, ve a `DesarrolloEnJackFinal` > `projects` y selecciona la carpeta **`SubwayJack`** (con *Files of type: VM Files / Dirs*). También sirve entrar a la carpeta sin seleccionar ningún `.vm`: el campo *File name* debe mostrar la ruta de la carpeta, no la de un archivo.
3. Pulsa **Load Program**. Si el emulador pregunta si usar las implementaciones integradas (*built-in*) del sistema operativo, responde **Yes**.
4. En **Animate** elige **No animation** y lleva el control **Speed** a **Fast**.
5. Pulsa **Run** (botón `>>`).
6. Haz clic en el **ícono del teclado** que está debajo de la pantalla del emulador para que reciba tus teclas.
7. Pulsa **ESPACIO** para jugar. Usa las flechas o A/D; **R** reinicia y **ESC** sale.

### Problemas comunes

| Síntoma | Solución |
|---------|----------|
| No aparece la carpeta `SubwayJack` | Actualiza tu copia con `git pull` (en versiones anteriores la carpeta se llamaba `Prueba3`). |
| `java` no se reconoce como comando | Instala Java y agrégalo al `PATH`. |
| Error `<Clase>.vm not found or function ... not found` | Cargaste un `.vm` suelto: carga la carpeta `SubwayJack` completa. |
| El juego no responde a las teclas | Haz clic en el ícono del teclado debajo de la pantalla. |

---

## Pruebas de control

Ejecutadas el 19 de noviembre de 2026 con el `JackCompiler` y el VM Emulator 2.5 de `tools/`, con el teclado simulado.

| Prueba | Resultado |
|--------|-----------|
| Compilación de las 7 clases | Sin errores; los `.vm` del repositorio coinciden con una compilación limpia. |
| START > ESPACIO > PLAYING > choques hasta 0 vidas > GAME OVER > R | La partida reinicia con 0 puntos, 3 vidas y nivel 1. |
| Victoria (puntaje llevado a 89 y un obstáculo esquivado) | 90 puntos, nivel 10, pantalla GANASTE!; R reinicia el estado. |
| ESC en START y en PLAYING | El juego libera sus objetos y el programa termina. |
| Memoria en 40 partidas seguidas | El heap no crece entre partidas. |

---

## Créditos de autoría

- **Autores:** Mateo Montoya Ospina, Sebastian Ibarra Prada, Miguel Ángel Colorado Castaño
- **Curso:** Organización de Computadores (S2666-0322, clase 4329), Universidad EAFIT, 2026-2
- **Profesor:** Edison Valencia Díaz
- **Fecha:** 18 de noviembre de 2026
- **Proyecto:** Subway Jack, Hito 2 (Proyecto Final)
