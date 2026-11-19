# Subway Jack — Juego completo (Hito 2 / Proyecto Final)

Videojuego retro 2D estilo *Subway Surfers* con **vista cenital** y **3 carriles verticales**, implementado en el lenguaje **Jack** del ecosistema **Nand2Tetris** (plataforma Hack).

El **Hito 2** convierte el renderizado estático del Hito 1 en un **juego completo y jugable**: máquina de estados, bucle principal no bloqueante, movimiento del jugador, obstáculos que caen y se reciclan, colisiones AABB, puntaje, vidas, niveles, reinicio y salida controlada, todo **sin fugas de memoria** y **sin crear objetos dentro del game loop**.

---

## Descripción del juego

- Pantalla monocromática de **512×256 px** (x: 0..511, y: 0..255).
- **Zona de juego:** x 128..383 (256 px de ancho), y 0..255.
- **3 carriles verticales**:
  | Carril | Posición | Centro x |
  |-------|----------|----------|
  | 0 | Izquierdo | 170 |
  | 1 | Central   | 256 |
  | 2 | Derecho   | 341 |
- El jugador está abajo y los obstáculos **descienden** desde arriba. Esquívalos cambiando de carril.
- **Paneles de HUD** laterales: izquierdo con **PUNTOS / VIDAS / NIVEL** (dinámicos), derecho con la leyenda de controles (estática).

---

## Reglas

- Empiezas con **3 vidas** y en **nivel 1**.
- **+1 punto** por cada obstáculo esquivado (sale por abajo sin golpearte).
- Cada **10 puntos** subes de nivel: el juego **acelera** (más velocidad de caída y menor espera por ciclo), con un tope para que siga siendo jugable.
- Al **chocar** pierdes 1 vida, el obstáculo golpeado se recicla y quedas **invulnerable** unos instantes (el personaje **parpadea**) para no perder todas las vidas de golpe.
- Si las vidas llegan a **0** → **GAME OVER**.
- Al llegar al **nivel 10** → **¡GANASTE!** (condición de victoria).
- Siempre hay al menos un carril libre: los obstáculos se reciclan con separación vertical suficiente para que el juego sea ganable.

---

## Controles

| Tecla | Acción |
|-------|--------|
| **Flecha izquierda** o **A** | Mover al carril de la izquierda |
| **Flecha derecha** o **D** | Mover al carril de la derecha |
| **ESPACIO** (o **ENTER**) | Iniciar el juego desde la pantalla de inicio |
| **R** | Reiniciar (en GAME OVER / victoria) |
| **ESC** | Salir del juego |

El movimiento es **por flanco** (antirrebote): cada pulsación mueve exactamente un carril; mantener la tecla no salta varios carriles.

---

## Máquina de estados

1. **START** — Pantalla de inicio con título, instrucciones y "Presiona ESPACIO para jugar". Mientras espera, un contador se incrementa y se usa como **semilla** del generador aleatorio (así cada partida es distinta).
2. **PLAYING** — Juego en curso: bucle principal no bloqueante.
3. **GAMEOVER** — Muestra "GAME OVER" (o "¡GANASTE!"), el puntaje y el nivel finales, y las opciones **R: reiniciar** / **ESC: salir**.

---

## Estructura de clases (un archivo por clase)

| Archivo | Responsabilidad |
|---------|-----------------|
| `Main.jack` | Punto de entrada. Crea `Game`, ejecuta `run()` y `dispose()`. |
| `Game.jack` | Orquestador: máquina de estados, game loop no bloqueante, entrada por flanco, colisiones AABB, puntaje/niveles, reinicio y salida. Dueño de `Board`, `Player`, `Hud` y el *pool* de `Obstacle`. |
| `Board.jack` | Escenario estático: marco, bordes, separadores punteados, postes y los **marcos + textos estáticos** del HUD. Método `clearField()` para limpiar solo la zona de juego en el reinicio. |
| `Player.jack` | Personaje compuesto (primitivas). `moveLeft/moveRight/setLane/reset`, `draw/erase` desacoplados y getters AABB. |
| `Obstacle.jack` | Obstáculo con 3 tipos (barrera, tren, cono). `update(speed)` lo baja, `recycle(...)` lo reubica arriba sin `new`, dibujo **recortado** a la zona de juego. |
| `Hud.jack` | Textos dinámicos **PUNTOS/VIDAS/NIVEL**: solo repinta un número cuando cambia y borra el anterior con espacios. |
| `Rng.jack` | Generador pseudoaleatorio (LCG de 16 bits). `setSeed(s)` y `nextInt(max)` → 0..max-1. |

### Diseño de memoria
- **Todos los objetos se crean una sola vez** (en `Game.new()`). Durante el juego y al reiniciar **no se llama a `new`**: los obstáculos se **reciclan** y el estado se reinicia en sitio con `reset()`.
- Cada clase implementa `dispose()` que libera en cadena sus objetos secundarios y termina con `do Memory.deAlloc(this);`.
- Los textos fijos se imprimen solo al pintar una pantalla o el HUD; para números se usa `Output.printInt` (no se crean `String` en el loop).

---

## Compilación y ejecución

### 1. Compilar con JackCompiler
Coloca los archivos `.jack` en una carpeta (p. ej. `SubwayJack/`) y compílala (sobre la **carpeta**, no archivos sueltos):

```bash
# Desde la raíz de las herramientas de Nand2Tetris:
./tools/JackCompiler.sh ruta/a/SubwayJack
# Windows:
tools\JackCompiler.bat ruta\a\SubwayJack
```

Genera un `.vm` por cada clase.

### 2. Ejecutar en el VM Emulator
1. Abre el **VM Emulator** (`tools/VMEmulator.sh` / `VMEmulator.bat`).
2. **File → Load Program** y selecciona la **carpeta** `SubwayJack` (carga todos los `.vm` + el OS integrado).
3. Pon la animación en **"No Animation"** y la velocidad alta.
4. Pulsa **Run** (▶).
5. En la pantalla de inicio pulsa **ESPACIO** para jugar. Usa las flechas o A/D. Pulsa **R** para reiniciar o **ESC** para salir.


---

## Créditos de autoría

- **Autor(es):** Sebastian Ibarra, Mateo Montoya, Miguel Colorado
- **Materia / Curso:** Organización de computadores
- **Fecha:** 18 de noviembre del 2026
- **Proyecto:** Subway Jack — Hito 2 (Proyecto Final)