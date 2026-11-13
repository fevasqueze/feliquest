# Documentación Técnica y Funcional — Super Kid: Rescate Familiar

## 1. Resumen General del Proyecto
**Super Kid: Rescate Familiar** es un videojuego de plataformas 2D en formato vertical (**9:16 Portrait**, resolución base `540x960`) diseñado para dispositivos **Android** y navegadores web. Combina mecánicas clásicas de **Super Mario** (bloques `?`, salto de altura variable, pisotón sobre enemigos) y **Sonic the Hedgehog** (movimiento con inercia, recolección y dispersión de anillos protectores, resortes de impulso, *Spin-Jump* y *Spin-Dash*).

El proyecto está construido como una aplicación web autocontenida (sin dependencia de recursos gráficos ni audios externos, ya que todos los sprites y efectos de sonido se generan proceduralmente en tiempo de ejecución) y empaquetada de forma nativa para Android mediante **Capacitor**, generando el instalador `super-kid.apk`.

---

## 2. Stack Tecnológico y Herramientas
- **Motor de Juego:** Phaser `^3.85.2` (Motor de físicas Arcade 2D, escalado `Phaser.Scale.FIT` con centrado automático y soporte multitáctil de 4 punteros simultáneos).
- **Lenguaje:** TypeScript `^5.6.2` (Tipado estricto, ES2022).
- **Empaquetador Web:** Vite `^5.4.8`.
- **Contenedor Nativo Android:** Capacitor `^6.1.2` (`@capacitor/core`, `@capacitor/android`, `@capacitor/cli`).
- **Audio Procedural:** Web Audio API (`AudioContext` con síntesis de osciladores en tiempo real).
- **Gráficos Procedurales:** Phaser `Graphics` API compilados en texturas dinámicas en memoria.
- **Automatización de Compilación Android:** Script PowerShell (`build-apk.ps1`) con toolchain local de JDK, Android SDK y Gradle para producir `super-kid.apk`.

---

## 3. Historia, Personajes y Progresión de Mundos
El antagonista, el **Doctor Sombra**, ha secuestrado a toda la familia y mascotas del protagonista. El jugador controla a **Leo** (*El Hermano Menor*), quien debe ascender verticalmente a través de 5 mundos para rescatarlos a todos.

Toda la configuración narrativa y de personajes se encuentra centralizada y parametrizada en `src/config/familyConfig.ts`:

| Mundo | Título | Objetivo de Rescate | Miembros / IDs | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Mundo 1** | Colinas y Tejados Felinos | Salvar a los **2 Gatos** de la familia | `cat_white` (**Copito**, Gato Blanco)<br>`cat_black` (**Sombra**, Gato Negro) | **Jugable completo (3 niveles verticales + Jefe)** |
| **Mundo 2** | Cañón de los Ladridos | Salvar a los **3 Perros** | `dog_1` (**Toby**)<br>`dog_2` (**Luna**)<br>`dog_3` (**Rocky**) | Configurado en mapa y roster familiar |
| **Mundo 3** | Torres de Cristal | Salvar a las **2 Hermanas Mayores** | `sister_1` (**Sofi**)<br>`sister_2` (**Valeria**) | Configurado en mapa y roster familiar |
| **Mundo 4** | Fortaleza Cibernética | Salvar al **Hermano Más Mayor** | `oldest_brother` (**Mateo**) | Configurado en mapa y roster familiar |
| **Mundo 5** | Ciudadela del Doctor Sombra | Salvar a **Mamá y Papá** | `mom` (**Mamá**)<br>`dad` (**Papá**) | Configurado en mapa y roster familiar |

---

## 4. Arquitectura del Proyecto y Estructura de Archivos

```text
super-kid/
├── android/                        # Proyecto nativo Android generado con Capacitor
│   └── app/src/main/
│       └── AndroidManifest.xml     # Configurado en orientación vertical (portrait)
├── dist/                           # Build de producción web generado por Vite
├── src/
│   ├── main.ts                     # Punto de entrada y configuración de Phaser.Game
│   ├── audio/
│   │   └── SoundManager.ts         # Sintetizador de efectos de sonido (Web Audio API)
│   ├── config/
│   │   └── familyConfig.ts         # Datos del héroe, familia, 5 mundos y guardado LocalStorage
│   ├── entities/
│   │   └── Player.ts               # Física, estados, animaciones y combate del jugador
│   ├── gfx/
│   │   └── AssetGenerator.ts       # Generador procedural de todas las texturas y sprites
│   ├── levels/
│   │   └── World1Levels.ts         # Diseño estructural de los niveles 1-1, 1-2 y 1-3
│   ├── scenes/
│   │   ├── BootScene.ts            # Escena de arranque y generación de texturas
│   │   ├── TitleScene.ts           # Pantalla de título, historia y vista previa familiar
│   │   ├── WorldMapScene.ts        # Mapa de selección de niveles y vista de los 5 mundos
│   │   ├── GameScene.ts            # Escena principal de juego (escalada, colisiones y jefe)
│   │   └── VictoryScene.ts         # Pantalla de victoria, rescate de gatos y puntuación
│   └── ui/
│       └── TouchControls.ts        # Controles táctiles en pantalla, teclado y HUD superior
├── build-apk.ps1                   # Script automatizado para compilar web + APK Android
├── capacitor.config.ts             # Configuración de Capacitor (com.fvasquez.superkid)
├── index.html                      # Contenedor HTML5 optimizado para móviles (sin zoom/scroll)
├── package.json                    # Scripts y dependencias del proyecto
├── super-kid.apk                   # APK compilada lista para instalar en Android
├── tsconfig.json                   # Configuración del compilador TypeScript
└── vite.config.ts                  # Configuración de empaquetado con rutas relativas ('./')
```

---

## 5. Detalle de Módulos Implementados

### 5.1. Configuración Inicial (`src/main.ts` e `index.html`)
- **Viewport Móvil (`index.html`):** Bloquea gestos de navegador (`touch-action: none`, `user-scalable=no`) para evitar desplazamientos o zoom accidental durante el juego táctil.
- **Configuración de Phaser (`src/main.ts`):**
  - Resolución lógica: `540 x 960` píxeles (relación de aspecto 9:16 vertical).
  - Gravedad vertical Arcade: `1480 px/s²`.
  - Punteros activos: `4` punteros táctiles simultáneos para permitir correr, deslizar el pulgar y saltar/impulsarse al mismo tiempo.

### 5.2. Generación Procedural de Gráficos (`src/gfx/AssetGenerator.ts`)
No requiere imágenes PNG externas; construye dinámicamente todas las texturas al iniciar el juego en `BootScene`:
1. **Texturas del Héroe (`Leo`):**
   - `hero_idle`, `hero_run1`, `hero_run2`, `hero_jump`: Niño con gorra roja estilo Mario, camiseta azul con estrella dorada, capa roja y zapatillas veloces estilo Sonic.
   - `hero_spin1`, `hero_spin2`: Bola de energía giratoria para el *Spin-Jump* y *Spin-Dash*.
2. **Mascotas y Familia:**
   - `cat_white`: Sprite de **Copito** (gato blanco con orejas rosadas, ojos celestes y cascabel dorado).
   - `cat_black`: Sprite de **Sombra** (gato negro con orejas púrpuras, ojos dorados y cascabel).
   - Iconos de familia para el mapa y pantalla de inicio: `family_dog`, `family_sister1`, `family_sister2`, `family_brother`, `family_mom`, `family_dad`.
3. **Terreno y Bloques Interactivos:**
   - `tile_grass` (`40x40`): Suelo ajedrezado estilo *Green Hill*.
   - `tile_rooftop` (`40x40`): Tejas de terracota para el nivel de tejados.
   - `tile_tower` (`40x40`): Bloques metálicos con remaches luminosos para la torre final.
   - `platform_oneway` (`40x18`): Plataformas atravesables desde abajo.
   - `block_question` y `block_used` (`40x40`): Bloques de interrogación `?` estilo Mario y su estado agotado.
   - `spring_up` (`38x28`): Resortes rojos/amarillos estilo Sonic.
   - `spike_up` (`36x24`): Trampas de púas metálicas.
4. **Enemigos y Jefe:**
   - `enemy_walker` (`40x36`): Robot patrullero terrestre (*Badnik* tipo escarabajo/cangrejo).
   - `enemy_flyer` (`44x34`): Dron volador con oscilación sinusoidal.
   - `boss_captor` (`80x70`): Jefe **Robot Captor** en nave flotante con cúpula y propulsores.
5. **Objetos y Efectos:**
   - `ring_star` (`28x28`): Anillo dorado coleccionable.
   - `power_shield` (`30x30`): Burbuja de escudo protector cian.
   - `cage_bars` (`64x64`): Jaula con botón de apertura superior donde están atrapados los gatos.
   - `goal_flag` (`64x80`): Bandera de meta del Nivel 1-1.
   - `particle_spark` (`10x10`): Partículas de destello para impactos y recolección.

### 5.3. Motor de Sonido Sintetizado (`src/audio/SoundManager.ts`)
Implementa la clase `SoundManager` (`soundFX`) utilizando `AudioContext` nativo del navegador con soporte de silenciamiento (`muted`):
- `playJump(superJump)`: Onda cuadrada ascendente (frecuencia más alta para el doble salto).
- `playSpinDash()`: Onda diente de sierra con rampa de aceleración y desaceleración.
- `playRing()`: Doble tono sinusoidal clásico (`B5 -> E6`, `987.77 Hz -> 1318.51 Hz`).
- `playSpring()`: Onda triangular de rebote elástico (`180 Hz -> 880 Hz`).
- `playStomp()`: Onda triangular descendente al aplastar enemigos.
- `playHurt()`: Secuencia descendente en diente de sierra al recibir daño.
- `playCatMeow(isBlackCat)`: Maullido sintetizado con modulación de frecuencia (tono más agudo para Copito y más grave para Sombra).
- `playPowerup()`: Arpegio ascendente de 6 notas en onda cuadrada al obtener escudo o derrotar al jefe.
- `playVictory()`: Fanfarria melódica al completar un nivel.

### 5.4. Jugador y Mecánicas de Movimiento (`src/entities/Player.ts`)
La clase `Player` extiende `Phaser.Physics.Arcade.Sprite` e implementa:
- **Movimiento Horizontal con Inercia:**
  - Velocidad máxima estándar: `300 px/s`.
  - Aceleración en suelo (`1500 px/s²`) y en aire (`1100 px/s²`), con bonificación de giro rápido (`turnBoost = 2.2x`) al cambiar de dirección.
  - Fricción progresiva suave al soltar los controles.
- **Salto Avanzado y Doble Salto en Giro:**
  - *Coyote Time* (`140 ms`): Permite saltar brevemente tras dejar el borde de una plataforma.
  - *Jump Buffering* (`150 ms`): Registra la pulsación de salto momentos antes de tocar el suelo.
  - *Altura de Salto Variable:* Corta la velocidad ascendente (`* 0.62`) si el jugador suelta el botón de salto temprano.
  - *Doble Salto (Aerial Spin-Jump):* Segundo salto en el aire (`-590 px/s`) que activa el estado `isSpinning`, permitiendo derrotar enemigos al contacto.
- **Spin-Dash (Impulso Sonic):**
  - Activa un impulso horizontal a `430 px/s` durante `360 ms` en estado de bola giratoria (`isSpinning = true`), capaz de destruir enemigos al chocar y dar un pequeño impulso vertical si se usa cayendo en el aire.
- **Sistema de Anillos, Escudo y Daño (`receiveHit`):**
  - El jugador inicia con **3 anillos protectores** para una experiencia accesible.
  - Si posee el **Escudo Protector** (`hasShield`), el golpe consume únicamente el escudo sin perder anillos.
  - Si no tiene escudo pero tiene anillos (`rings > 0`), pierde hasta 8 anillos que salen disparados rebotando por el escenario y pueden volver a recogerse.
  - Si recibe daño con `0` anillos, reaparece inmediatamente en el **último checkpoint vertical alcanzado** con 3 anillos restaurados, manteniendo el flujo del juego sin pantallas de *Game Over* frustrantes.

### 5.5. Controles Táctiles y HUD (`src/ui/TouchControls.ts`)
Diseñado específicamente para jugarse cómodamente con los pulgares en pantallas verticales:
- **Controles Táctiles Inferiores:**
  - Botones direccionales `◀` y `▶` en la esquina inferior izquierda, con soporte de **deslizamiento continuo del pulgar** entre izquierda y derecha.
  - Botón **B (`SPIN DASH`)** y Botón **A (`SALTO`)** en la esquina inferior derecha con retroalimentación visual de escala y color al presionar.
- **Soporte Completo de Teclado (PC):**
  - Movimiento: `Flechas Izquierda/Derecha` o `A / D`.
  - Salto / Doble Salto: `Espacio`, `W` o `Flecha Arriba`.
  - Spin-Dash: `Shift`, `Z` o `X`.
- **HUD Superior Fijo:**
  - Contador de anillos (cambia a color rojo de alerta cuando llega a `0`).
  - Indicador de nivel actual y puntaje acumulado (`PTS`).
  - Insignia de `ESCUDO` activa.
  - **Barra de Progreso de Altitud (`CIMA`):** Muestra en tiempo real qué tan cerca está el jugador de alcanzar la meta en la cima del nivel.
  - Botón para silenciar/activar sonido (`🔊` / `🔇`) y botón rápido para volver al `MAPA`.
  - **Barra de Vida del Jefe (`ROBOT CAPTOR`):** Se despliega automáticamente durante el combate final del Nivel 1-3.

### 5.6. Diseño de Niveles Verticales (`src/levels/World1Levels.ts`)
El Mundo 1 cuenta con **3 niveles verticales completos** de dificultad progresiva:

1. **Nivel 1-1: Jardín de los Saltos (`1080 x 2600 px`)**
   - **Ambientación:** Colinas verdes bajo cielo diurno.
   - **Estructura:** 7 pisos de ascenso con plataformas de césped y plataformas *one-way*, 6 bloques `?`, 7 resortes Sonic, 8 enemigos (*walkers* y *flyers*), 10 líneas de anillos y 2 trampas de púas.
   - **Meta:** Bandera de meta en la cima (`y = 240`).
2. **Nivel 1-2: Tejados Felinos (`1080 x 2800 px`)**
   - **Ambientación:** Tejados de terracota al atardecer púrpura/naranja.
   - **Estructura:** 7 secciones de escalada urbana, 6 bloques `?`, 8 resortes de gran potencia, 9 enemigos patrullando y volando, 7 líneas de anillos y 3 zonas de púas.
   - **Meta:** Romper la jaula en el campanario superior para rescatar a **Copito (Gato Blanco)**.
3. **Nivel 1-3: Torre del Robot Captor (`1080 x 2900 px`)**
   - **Ambientación:** Torre metálica nocturna del Doctor Sombra.
   - **Estructura:** 6 secciones de ascenso + Arena de Jefe en la cima con plataformas laterales y resortes de apoyo para ataques aéreos. Incluye 8 bloques `?`, 9 resortes, 8 enemigos y 4 zonas de púas.
   - **Jefe Final del Mundo 1:** El **Robot Captor** protege la jaula de **Sombra (Gato Negro)** con un campo de fuerza láser. El jugador debe golpear al jefe **4 veces** (saltando encima o con *Spin-Jump* / *Spin-Dash*). Al derrotarlo, el campo de fuerza desaparece y se libera a Sombra.

### 5.7. Flujo de Escenas (`src/scenes/`)
- **`BootScene.ts`:** Invoca `AssetGenerator.generateAll(this)` y transiciona a `TitleScene`.
- **`TitleScene.ts`:** Muestra el logo de *Super Kid*, la sinopsis de la historia, el estado actual de rescate de **Copito** y **Sombra** en el Mundo 1, el listado de los Mundos 2 al 5 (Perros, Hermanas, Hermano Mayor y Padres), animación inferior de personajes y botones para ir al Mapa o Jugar directamente el nivel desbloqueado.
- **`WorldMapScene.ts`:** Presenta las estadísticas globales (anillos totales y gatos rescatados), las tarjetas interactivas de los niveles `1-1`, `1-2` y `1-3` con sus récords de puntuación y estado de desbloqueo, además de la vista previa de los Mundos 2 al 5.
- **`GameScene.ts`:** Construye la geometría del nivel, administra la cámara vertical con seguimiento suavizado (`lerp 0.12`, offset vertical `-75px`), gestiona colisiones, checkpoints automáticos cada `280px` de ascenso, dispersión física de anillos, IA de enemigos y del jefe, animaciones de ruptura de jaula y transición a la victoria.
- **`VictoryScene.ts`:** Muestra la celebración del nivel superado, animaciones del héroe junto al gato rescatado (en el nivel 1-3 aparecen ambos gatos juntos celebrando), cita de agradecimiento del personaje rescatado, desglose de puntos (`+100 pts` por anillo conservado + `1000 pts` de bonificación de nivel) y acceso al siguiente nivel o al mapa.

### 5.8. Persistencia de Datos (`localStorage`)
En `src/config/familyConfig.ts`, las funciones `loadProgress()` y `saveLevelCompletion()` guardan automáticamente el progreso bajo la clave `super_kid_save_v1`:
- `unlockedLevel`: Nivel más alto desbloqueado en el Mundo 1 (`1` a `3`).
- `rescuedIds`: Lista de identificadores de familiares/mascotas rescatados (`['cat_white', 'cat_black']`).
- `totalStars`: Total acumulado de anillos conservados al finalizar niveles.
- `bestScores`: Récord histórico de puntuación por cada nivel (`w1_1`, `w1_2`, `w1_3`).

---

## 6. Empaquetado para Android y Comandos del Proyecto

### 6.1. Configuración Nativa Android
- **ID de Aplicación (`capacitor.config.ts`):** `com.fvasquez.superkid`
- **Nombre de la App:** `Super Kid: Rescate Familiar`
- **Orientación Fija (`AndroidManifest.xml`):** Configurado con `android:screenOrientation="portrait"` en `MainActivity` para garantizar la experiencia vertical 9:16 en todo momento.

### 6.2. Scripts Disponibles (`package.json`)
- `npm run dev`: Inicia el servidor de desarrollo local con Vite en `http://localhost:5173`.
- `npm run build`: Verifica tipos con TypeScript (`tsc --noEmit`) y genera el empaquetado web optimizado en la carpeta `dist/`.
- `npm run preview`: Previsualiza localmente la compilación de producción.
- `npm run android:init`: Inicializa la plataforma Android con Capacitor (`npx cap add android`).
- `npm run android:sync`: Compila el proyecto web y sincroniza los archivos con el proyecto nativo Android (`npm run build && npx cap sync android`).
- `npm run android:open`: Abre el proyecto Android en Android Studio.
- `npm run apk:build`: Ejecuta el script `build-apk.ps1` para generar directamente el archivo `super-kid.apk`.

### 6.3. Proceso Automatizado de Generación de APK (`build-apk.ps1`)
El script `build-apk.ps1` automatiza de principio a fin la creación del instalador Android sin requerir configuración manual del entorno:
1. Configura las variables de entorno locales (`JAVA_HOME`, `ANDROID_HOME`, `ANDROID_SDK_ROOT`, `GRADLE_USER_HOME`).
2. Ejecuta `npm run build` (TypeScript + Vite).
3. Ejecuta `npx cap sync android` para actualizar los assets web dentro de la app nativa.
4. Ejecuta `gradlew.bat assembleDebug --no-daemon` para compilar el paquete Android.
5. Copia el archivo resultante desde `android/app/build/outputs/apk/debug/app-debug.apk` hacia la raíz del proyecto como **`super-kid.apk`**.
