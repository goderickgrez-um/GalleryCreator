# Galería Virtual 3D — Documentación completa

> **Propósito de este documento.** Describe de forma exhaustiva la aplicación **galeria.html** (museo/galería virtual 3D en un solo archivo). Está pensado para (a) mostrar todas las funciones a cualquier persona y (b) **cargarse en un chat nuevo** para continuar el desarrollo sin depender de la memoria. Incluye el modelo de datos, el catálogo de funciones, los controles, el pipeline de construcción/validación, el historial de versiones y la hoja de ruta de lo pendiente.
>
> **Versión documentada: v2.10.6** · Idioma de trabajo del proyecto: **español**.

---

## 1. Qué es

Aplicación web **autocontenida en un único archivo `galeria.html`** que permite:

- **Recorrer** una galería/museo en **primera persona** (escritorio con ratón + WASD; móvil con joystick táctil).
- **Editar** la galería: dibujar la planta (salas, pasillos, luces, recorrido guiado), colocar obras (imágenes y textos enriquecidos) y elementos de mobiliario (assets), ajustar colores, texturas e iluminación.
- **Exportar** una versión **solo-recorrido** a otro `.html` independiente (para compartir), o exportar/importar el proyecto completo en **JSON**.

Estética: museo de tema oscuro translúcido, acento **latón `#d6b873`**, motivo de cartela.

### Cómo ejecutarla
- Abrir `galeria.html` en un navegador moderno (Chrome, Edge, Firefox, Safari).
- **Requiere conexión a internet en la primera carga** porque importa **Three.js v0.160** desde CDN (unpkg) vía *importmap*:
  - `https://unpkg.com/three@0.160.0/build/three.module.js`
  - `https://unpkg.com/three@0.160.0/examples/jsm/...` (PointerLockControls, OrbitControls, BufferGeometryUtils).
- No necesita servidor; funciona como archivo local.

---

## 2. Arquitectura técnica

El archivo contiene **dos motores** escritos en Three.js:

1. **Motor editor (principal):** el `<script type="module">` principal. Usa `import`, sintaxis moderna (`const`/`let`, arrow functions). Es la app completa: recorrido + edición + editor de planta + exportadores.
2. **Motor exportado (recorrido):** vive como una **cadena de plantilla** `TOUR_TEMPLATE` dentro del motor principal. Al pulsar **Exportar HTML** se materializa en un `.html` nuevo, **solo-recorrido**, con su propio motor independiente escrito con `var` y concatenación de strings (sin backticks ni `${}`), por las reglas del *template literal* (ver §7).

### 2.1 Modelo de datos `plan`
Objeto serializable que describe la galería:

```
plan = {
  wallHeight,                            // (opcional) altura de pared global en metros (def. 4.0)
  rooms: [ {
      id, name,
      shape: 'rect' | 'circle' | 'ellipse' | 'poly',
      // poly: points:[[x,z],...], bulges:[n,...] (curvatura por arista, Bézier cuadrática)
      // circle: cx,cz,r   ·   ellipse: cx,cz,a,b
      colors: { wall, ceil, floor },     // hex
      wallColors: { [edgeIndex]: hex },  // (opcional) color de una pared concreta (por arista)
      tex:    { wall, ceil, floor },     // (opcional) dataURL de textura por superficie
      noCeiling: bool                    // (opcional) quita el techo de esa sala
  } ],
  corridors: [ { id, x, z, w, h } ],     // rectángulos que conectan salas
  corridorColors: { wall, floor, ceil },
  corridorTex:    { wall, floor, ceil }, // (opcional) texturas de todos los pasillos
  lights: [ {
      id, x, z, y, color, intensity,
      kind: 'spot' | 'point',
      dir,  // (spot) dirección horizontal del haz en radianes
      tilt  // (spot) 0..1 inclinación del haz (0 = recto hacia abajo)
  } ],
  tour: [ { x, z }, ... ],               // waypoints ordenados del recorrido guiado
  spawn: { x, z, angle },                // punto y orientación inicial
  lighting: 'museo'|'calida'|'fria'|'brillante'|'dramatica'|'noche'
}
```

### 2.2 Constructor de geometría: `buildInto(plan, group, opts)`
Núcleo compartido (existe en ambos motores). A partir de `plan`:
- Crea **suelos y techos** por sala/pasillo con `floorMesh`/`ceilMesh` (ShapeGeometry; UV en metros del mundo → las texturas embaldosan cada ~2 m).
- Respeta `room.noCeiling` (no genera techo en esa sala) cuando `opts.ceiling` está activo.
- Materiales mediante `surfMat(color, texURL, surf)` (principal) / `smk(...)` (exportado): `MeshStandardMaterial` con `map` de textura opcional (`RepeatWrapping`, anisotropía máxima).
- **Paredes con puertas automáticas:** recorre cada arista de sala/pasillo y la subdivide en segmentos de ~0.2 m, **omitiendo** los tramos que solapan con otra sala o pasillo (así aparecen vanos/puertas donde se tocan). Devuelve `{ wallMeshes, cells, wallSegs, roomMats, corridorMat }`.
- `opts.skirt` añade un zócalo oscuro.

`buildGalleryFromPlan(plan, resetCamera)` (principal) limpia el grupo anterior, llama a `buildInto`, asigna variables globales (celdas, segmentos de pared), construye luces (`buildPlanLights`) y refresca los destinos de color.

### 2.3 Colisión y navegación
- **2D sobre el plano** (la cámara va a altura de ojos `EYE = 1.62`).
- `resolveWalls(px,pz)`: empuje circular contra los segmentos de pared (`wallSegs`) con radio `PLAYER_R = 0.42`.
- `insideGallery(x,z)`: *point-in-polygon* contra las celdas (salas + pasillos).
- `tryMove(...)`: aplica el movimiento con deslizamiento por ejes (si choca en diagonal, intenta solo X o solo Z).
- Constantes clave: `H = 4.0` (altura de pared), `WALL_T = 0.18`, `SPEED = 4.4`, `BASE_H = 1.55` (escala base de obras de pared).

### 2.4 Objetos colocables (`placeables`)
Cada obra/elemento es un `art = { kind, group, data, ... }`. Tipos (`kind`):
- **`image`** — cuadro con marco, imagen y datos (título/fecha/descripción/marco on-off). Pickeable; abre el visor a pantalla completa.
- **`text`** — cartela de texto enriquecido (HTML → SVG `foreignObject` → CanvasTexture, con *fallback* a texto plano en canvas). Pickeable; no abre visor.
- **`asset`** — mobiliario decorativo (banca, puerta, puerta doble, postes con cinta). Se apoya en el piso; tiene una caja de selección invisible; **no** abre visor.

`snapshot()` serializa `plan` (clon profundo → incluye `tour`, `noCeiling`, `tex`, `corridorTex`, `dir`, `tilt` automáticamente) + cámara + `placeables` (con `kind`, transформación, `scale`, `aspect`, `asset`, datos). `restore(data)` reconstruye todo y vuelve a generar las flechas del tour.

### 2.5 Persistencia
- **Autoguardado local**: `localStorage` con clave `galeria3d_v2` (botón **Guardar**). Si no cabe (texturas/imágenes en base64 son grandes), avisa de usar **Exportar**.
- **Exportar JSON**: descarga `galeria-AAAA-MM-DD.json` (proyecto completo).
- **Importar**: carga un JSON exportado.

---

## 3. Catálogo completo de funciones

### 3.1 Modos
- **Recorrido** y **Edición** (alternar con **Tab** o el conmutador superior).
- En **Recorrido** apuntar a una obra muestra su ficha; clic la abre a pantalla completa.
- En **Edición** aparece el panel lateral con todas las herramientas.

### 3.2 Barra superior
- **Nuevo proyecto** → abre el editor de planta **en blanco**.
- **Editar planta** → abre el editor de planta **con la galería actual cargada** (permite seguir modificándola después de construida). *(v2.0)*
- **Guardar** → guarda en el navegador.
- **Exportar HTML** → genera un `.html` independiente solo-recorrido (escritorio + móvil). *(v1.2)*
- **Exportar JSON** / **Importar** → proyecto completo.

### 3.3 Editor de planta (vista superior 2D + vista previa 3D)
**Herramientas** (barra del editor):
- **Seleccionar** — mover salas/pasillos/luces; arrastrar **vértices** (cuadrados) para deformar polígonos; **puntos medios** (círculos) para **curvar** una pared (bulge); **doble-clic** en una arista **añade** vértice y en un vértice lo **elimina**.
- **Sala** — arrastrar para crear la forma elegida; **clic sobre una sala existente la mueve**. *(movimiento por herramienta, v2.1)*
- **Pasillo** — arrastrar para crear; **clic sobre un pasillo existente lo mueve**. *(v2.1)*
- **Luz** — clic en vacío crea una luz; **clic sobre una luz existente la mueve**. *(v2.1)*
- **Ruta** — clic añade **waypoints** del recorrido guiado (se ven como línea verde numerada; en el recorrido aparecen como **flechas en el piso**). *(v2.0)*
- **Borrar** — elimina la sala/pasillo/luz o el **punto de ruta** más cercano.

**Formas de sala**: rectángulo, triángulo, pentágono, hexágono, octágono, círculo, elipse.

**Propiedades de SALA** (al seleccionarla):
- Nombre; colores de **paredes/techo/piso**.
- **Quitar techo** (casilla) — para escaleras/espacios abiertos. *(v2.0)*
- **Texturas** por superficie (**Paredes/Techo/Piso**): cargar imagen (se reescala a 1024 px, JPEG) + **Quitar texturas**. *(v2.0)*

**Propiedades de PASILLO**: **Texturas** (se aplican a todos los pasillos) + eliminar.

**Propiedades de LUZ**:
- Color + **colores rápidos** (paleta guardada en memoria de sesión). *(v2.1)*
- **Intensidad** y **Altura** con slider **+ campo numérico** (se pueden teclear valores). El rango de altura llega hasta **6 m**. *(v2.1)*
- Tipo: **Foco** (haz dirigido) o **Punto** (omnidireccional).
- Para focos: **Dirección** (0–360°) e **Inclinación** (0–1) → **rotación del haz** (la dirección se dibuja en el plano). *(v2.1)*

**Vista previa 3D**: panel con `OrbitControls` (arrastrar rota, rueda zoom). Refleja salas, pasillos, colores, texturas y luces en vivo.

Botones del editor: **Vista 3D** (mostrar/ocultar preview), **Plantilla** (carga una planta de ejemplo), **Vaciar**, **Cancelar**, **Generar galería**.

### 3.4 Panel — Añadir
- **Subir imagen** (clic o arrastrar y soltar). Al subir, **se autocompletan Título (nombre del archivo) y Fecha (año del archivo)**, editables después. *(v2.1)*
- **Añadir cuadro de texto**.
- **Colocar lo pendiente frente a mí**.
- **Elementos (assets)**: **Banca**, **Puerta**, **Puerta doble**, **Postes con cinta**. Aparecen frente a ti sobre el piso. *(v2.0)*

### 3.5 Panel — Acabados por sala
- **Aplicar a**: Todas las salas / una sala concreta / pasillos.
- Colores de **paredes/techo/piso** + **colores rápidos** (paleta de sesión; **＋ Guardar** añade el color activo; clic en un chip lo aplica a la última superficie usada). *(v2.1)*
- **Altura de paredes** (slider + número, 2.4–8 m, hasta 12 m tecleando) — **global** a toda la galería; reconstruye al instante. *(v2.2)*
- **Pintar una pared** (botón) — modo en el que apuntas a una pared en la vista y haces clic para aplicarle el color de **Paredes**; vuelve a pulsar para salir. Pintar con el color base de la sala revierte el override. *(v2.2)*

### 3.6 Panel — Iluminación (ambiente)
Presets de iluminación global: **Museo neutro, Cálida, Fría, Brillante, Dramática, Penumbra**. (Los puntos de luz concretos se colocan en el editor de planta.)

### 3.7 Panel — Selección (obra/elemento elegido)
- **Imagen**: Título, Fecha, Descripción, Mostrar marco.
- **Texto**: barra de formato (negrita, cursiva, subrayado, alineación, color) sobre un editor `contenteditable`, más **Ancho** y **Alto** del cuadro (slider + número) para que **quepa todo un párrafo** variando largo y alto a gusto, independientemente del escalado. *(v2.2)*
- **Tamaño**: slider **+ campo numérico** (teclear el factor de escala). *(v2.1)*
- **Mover sobre la pared (G)**, **Eliminar**.
- Con **varios elementos seleccionados** (Ctrl+clic), el tamaño, la escala con rueda, el movimiento, el copiar/pegar y el borrado se aplican **a todos a la vez**. *(v2.2)*

### 3.8 Interacciones en la vista (Edición)
- **Apuntar + clic** a una pared coloca lo pendiente; a una obra la selecciona.
- **Oclusión**: la selección **ya no detecta obras situadas en otras salas** a través de las paredes. *(v2.1)*
- **Multiselección**: **Ctrl+clic** (apuntando a la obra) añade o quita elementos de la selección; las operaciones se aplican al conjunto. *(v2.2)*
- **Rueda**: escala la selección.
- **Flechas**: mover (obras de pared en el plano de la pared; **assets** en X/Z sobre el piso).
- **Q / E**: girar un **asset**. *(v2.0)*
- **G**: agarrar y deslizar sobre la pared · **M**: reubicar frente a ti.
- **Copiar / pegar** (Ctrl+C / Ctrl+V) de obras y elementos; **duplicar en el sitio** (Ctrl+D); **seleccionar todo** (Ctrl+A). *(v2.1/v2.3)*
- **Deshacer / rehacer** (Ctrl+Z / Ctrl+Y, o Ctrl+Shift+Z): historial de hasta 30 pasos que cubre colocar, mover, escalar, girar, borrar, pegar/duplicar, editar texto, pintar pared, color de sala, altura de paredes y regenerar la planta. Las ráfagas (mantener una flecha, arrastrar la rueda o el color) se agrupan en un solo paso. El deshacer **no mueve la cámara**. **Ctrl+S** guarda. *(v2.3)*
- **Supr / Backspace**: eliminar la selección.

### 3.9 Tour guiado con flechas
Los waypoints de `plan.tour` se materializan en **flechas de latón sobre el piso** que apuntan hacia el siguiente punto, con **animación de onda** (opacidad/altura) que recorre el camino, y un **anillo pulsante** en el destino. Visibles solo en modo Recorrido y **también en el HTML exportado**. *(v2.0)*

### 3.10 Visor a pantalla completa
Al hacer clic/tocar una imagen se abre con título, fecha y descripción (con scroll). En móvil del HTML exportado se corrigió el parpadeo por *click sintético* y se permite el scroll nativo de la descripción. *(v1.3)*

---

## 4. Controles

### Editor (escritorio)
| Acción | Tecla/gesto |
|---|---|
| Mirar | Clic en la vista (bloquea cursor) + ratón |
| Caminar / correr | W A S D · Shift |
| Liberar cursor | Esc |
| Recorrido ↔ Edición | Tab |
| Escalar selección | Rueda |
| Mover selección | Flechas |
| Girar asset | Q / E |
| Agarrar / reubicar | G / M |
| Copiar / pegar | Ctrl+C / Ctrl+V |
| Duplicar selección | Ctrl+D |
| Seleccionar todo | Ctrl+A |
| Deshacer / rehacer | Ctrl+Z / Ctrl+Y (o Ctrl+Shift+Z) |
| Guardar en el navegador | Ctrl+S |
| Eliminar | Supr / Backspace |

### HTML exportado — escritorio
WASD o flechas para moverse, ratón para mirar (clic para bloquear, Esc para liberar), clic en un cuadro para ampliarlo.

### HTML exportado — móvil
Joystick táctil para moverse, arrastrar con el dedo para mirar, tocar un cuadro para abrirlo a pantalla completa. (Mientras el visor está abierto los controles del recorrido se desactivan para permitir cerrar tocando fuera y desplazar la descripción.)

---

## 5. Formato del proyecto (snapshot JSON)

```
{
  v: 2,
  plan: { ...ver §2.1... },
  lighting: 'museo',
  camera: { pos:[x,y,z], rot:[x,y,z] },
  placeables: [
    { kind:'image', pos, quat, scale, aspect, dataURL, title, date, desc, frameOn },
    { kind:'text',  pos, quat, scale, aspect, w, h, html },   // w,h = ancho/alto del cuadro en metros
    { kind:'asset', pos, quat, scale, asset:'bench'|'door'|'door2'|'stanchion' }
  ]
}
```

El HTML exportado embebe este estado en un `<script id="state" type="application/json">` (con `<` escapado como `\u003c`) y lo lee al arrancar.

**Modelo `plan` multi-piso (v2.5):**
```
plan: {
  floors: [ { name, rooms:[...], corridors:[...], corridorColors, corridorTex, lights:[...], tour:[...] }, ... ],
  stairs: [ { id, type:'straight'|'u'|'spiralL'|'spiralR', floor:int /*piso inferior*/, x, z, w, run, dir } ],
  activeFloor: int,          // piso activo (editor / colisión inicial)
  floorHeight: number,       // separación vertical entre pisos (m)
  wallHeight, spawn, lighting, sun, envColor, envIntensity   // a nivel de plan
}
```
`plan.rooms / corridors / lights / tour / corridorColors / corridorTex` son **alias** (misma referencia) al piso activo: todo el código del editor opera sobre el piso activo sin cambios. `migratePlan()` envuelve automáticamente los proyectos antiguos de un solo piso en `floors[0]`, por lo que **los guardados previos siguen abriéndose**.

Cada **sala** admite además `wallHeight` (altura de pared propia) y `noCeiling`; cada **pasillo** admite `wallHeight`. Si no se define, se usa `plan.wallHeight`. Cada **escalera** se describe con `{ id, type, floor, x, z, w, run, dir }` (ver §7, v2.5).

---

## 6. Estructura del archivo y anclajes (para desarrollo)

`galeria.html` (un solo archivo) contiene, en orden:
1. **`<style>`** — todo el CSS (tema, panel, editor, controles `.sld`, paletas `.swatches`, assets `.assetRow`, texturas `.texRow`).
2. **HTML del UI** — barra superior (`#topbar`), panel lateral (`#panel`), HUD de recorrido, visor (`#viewer`), modal de ayuda (`#help`), editor de planta (`#planner`).
3. **`<script type="importmap">`** — mapeo de `three` y `three/addons/`.
4. **`<script type="module">`** (motor editor):
   - Constantes y escena (`H, WALL_T, EYE, PLAYER_R, SPEED, BASE_H`, renderer, cámara, `PointerLockControls`).
   - `defaultPlan()`, helpers de geometría (`ellipsePoly, roomPoly, corridorPoly, roomCentroid, pointInPoly, inRect, floorMesh, ceilMesh`).
   - `buildInto`, `buildGalleryFromPlan`, `buildPlanLights`, `applyLighting`.
   - `surfMat`, `texFileToURL`; **assets** (`ASSET_MATS`, `buildAssetNode`, `createAssetArt`, `placeAssetInFront`); **tour** (`arrowGeometry`, `buildTourArrows`, `animateTour`, `tourGroup`).
   - Placeables (`newPlaceableBase, createImageArt, createTextArt, renderText, applyScale, selectArt, deleteArt`), colisión, **portapapeles** (`serializeOne, createFromData, pasteClipboard`), teclado.
   - `setMode`, acabados por sala (`setColor`) + **paletas** (`renderRoomSwatches`), subida de imágenes (`ingest`, `niceTitle`), panel de selección.
   - `snapshot`/`restore`, botones Guardar/Exportar/Importar.
   - `TOUR_TEMPLATE` (motor exportado, ver abajo) + `exportTourHTML()`.
   - **Editor de planta**: `draft`, `emptyDraft, normalizeDraft, openPlanner`, dibujo (`drawPlan, drawSelection`), punteros (`mousedown/mousemove/mouseup/wheel/dblclick`), `hitTest, hitWaypoint`, propiedades (`showProps`) con helpers `sliderRow/bindSlider/swatchHTML/bindSwatches`, vista previa (`initPreview...buildPreview`), `generateFromDraft`.
   - Ejemplos (`seedExamples`) y arranque.

### Pipeline de validación (obligatorio antes de entregar)
1. **Módulo principal:** extraer el primer `<script type="module">` con la regex no-greedy
   `/<script type="module">([\s\S]*?)<\/script>/` y correr `node --check`.
2. **Motor exportado:** **materializarlo** evaluando la plantilla:
   - localizar `` TOUR_TEMPLATE = ` `` , tomar `inner` hasta el siguiente backtick, sustituir `__STATE__` por un estado mínimo y `eval('`'+inner+'`')`;
   - del HTML resultante extraer su `<script type="module">` y correr `node --check`;
   - verificar que los `</script>` reales estén **balanceados** y que existan `importmap` y `BufferGeometryUtils`.
3. Copiar a `/mnt/user-data/outputs/galeria.html` y presentar.

### Reglas del *template literal* (motor exportado) — IMPRESCINDIBLE
Dentro de `TOUR_TEMPLATE` (una cadena con backticks):
- **No** usar backticks ni `${...}`.
- Cualquier `</script>` literal **rompe** el `<script>` padre → escribir **`<\/script>`** (cocina a `</script>`).
- Las barras invertidas que deban aparecer en el export se **duplican**: p. ej. la regex `\s` se escribe `\\s` en la plantilla (si se escribe `\s` se pierde la barra). `\/` es intencional y queda `/`.
- El motor exportado usa `var` y concatenación de strings.

---

## 7. Historial de versiones

- **v1.0 — Galería base.** Recorrido en 1ª persona; editor de planta con formas (rect/polígonos/círculo/elipse), vértices y bulges; colores por sala; presets de iluminación + luces colocables; colocación de imágenes y textos en paredes; persistencia local; exportar/importar JSON.
- **v1.1 — Cuadros de texto SVG.** Corrección de cartelas invisibles: `renderText` serializa a XHTML con `XMLSerializer` (autocierra `<br/>`, normaliza entidades, asegura `xmlns`) con *fallback* a texto plano en canvas.
- **v1.2 — Exportador a HTML solo-recorrido.** Botón **Exportar HTML**; plantilla independiente con controles de escritorio (pointer-lock + ratón + WASD) y móvil (joystick táctil + arrastrar para mirar + tap para ampliar).
- **v1.3 — Visor en móvil.** Corrección del parpadeo por *click sintético* tras `touchend` (ventana de gracia + supresión del click) y scroll nativo de la descripción; los controles del recorrido se desactivan con el visor abierto.
- **v2.0 — Mobiliario, texturas, techo, tour y edición posterior.**
  - **Assets**: banca, puerta decorativa simple, puerta doble, postes con cinta.
  - **Texturas** de paredes/techo/piso por sala y de pasillos.
  - **Quitar techo** por sala (espacios abiertos / paso de escaleras).
  - **Tour guiado** con **flechas animadas en el piso** (editor + recorrido + exportado).
  - **Editar planta** después de construida la galería.
- **v2.1 — Productividad de edición.**
  - **Autocompletado** de título (nombre de archivo) y fecha (año) al subir imágenes, editables.
  - **Oclusión por paredes**: la selección/colocación ignora obras de otras salas.
  - **Copiar/Pegar** (Ctrl+C / Ctrl+V) de obras y elementos.
  - **Campos numéricos** en los sliders (escala, intensidad, altura) — se pueden teclear valores; **altura de luces** ampliada a 6 m.
  - **Rotación e inclinación de focos** (dirección 0–360° + inclinación) con indicador en el plano.
  - **Colores rápidos**: paletas guardadas en sesión en luces y en acabados por sala.
  - **Movimiento por herramienta** en el editor: con Sala/Pasillo/Luz, clic sobre un elemento existente lo mueve.
- **v2.2 — Acabados, alturas, multiselección y texto variable.**
  - **Color de pared individual**: pintar una pared concreta apuntando y haciendo clic; serializado por arista (`room.wallColors`).
  - **Altura de paredes** ajustable y global (`plan.wallHeight`), reflejada en editor, vista previa y export.
  - **Multiselección** con **Ctrl+clic**: escalar/mover/copiar/pegar/borrar en lote.
  - **Cuadros de texto de tamaño variable**: ancho y alto del cuadro independientes del escalado, para alojar párrafos completos (`text.w`, `text.h`).
- **v2.3 — Historial y atajos de teclado.**
  - **Deshacer / rehacer** (Ctrl+Z / Ctrl+Y / Ctrl+Shift+Z) con historial de 30 pasos basado en `snapshot()`, con agrupación de ráfagas y sin mover la cámara.
  - **Ctrl+D** duplicar en el sitio, **Ctrl+A** seleccionar todo, **Ctrl+S** guardar (intercepta el guardado del navegador).
  - Sección de **atajos** añadida a la ayuda integrada.
- **v2.4 — Editor de planta avanzado, mobiliario interactivo, transparencia y tipografías.**
  - **Multiselección en el editor de planta** (Ctrl+clic): mover varios elementos a la vez y editar **todas las luces seleccionadas** simultáneamente (color, intensidad, altura, dirección, inclinación, tipo).
  - **Inclinación de luces 0–180°** en el editor: a 0° apuntan al suelo, a 90° horizontal, a 180° **al techo**.
  - **Offset de pasillos**: los pasillos se construyen ligeramente más grandes en planta y, además, con el **suelo 2 cm más abajo y el techo 2 cm más arriba** que los de las salas, de modo que en las uniones **no se solapan las texturas** (se elimina el *z-fighting*) sin que el desnivel se perciba al caminar.
  - **«Pinta con clic»**: apunta a una **pared, piso o techo** y haz clic para aplicarle el color elegido en un **recuadro** junto al botón (sustituye a «pintar una pared»).
  - **Presets Dramática y Penumbra diferenciados**: Dramática = luz clave cálida intensa con sombras; Penumbra = luz fría tenue y uniforme.
  - **Sol y ambiente en el editor de planta**: panel con **color de ambiente** + intensidad y una **luz de sol** direccional con **color, intensidad, azimut y elevación**; se refleja en la vista previa, en la galería y en el HTML exportado (`plan.sun`, `plan.envColor`).
  - **PNG transparentes**: las imágenes conservan su canal alfa; **con marco** se les pone fondo **blanco**, **sin marco** se muestran con su **fondo transparente**.
  - **Cuadros de texto reflowables**: el ancho y el alto **reflujan** el texto sin cambiar el tamaño de letra; el control **Tamaño** sigue escalando todo el cuadro. El texto se dibuja con un **renderizador en canvas 2D** (parser de negrita/cursiva/subrayado/color/alineación) que **respeta las fuentes cargadas** y actualiza en vivo de forma fiable, tanto en el editor como en el HTML exportado.
  - **Importar fuentes** (TTF/OTF/WOFF) por cuadro de texto: se registran con la API `FontFace` y se **guardan y exportan** (registro `fonts` en el snapshot); aparecen en el editor, en el recorrido y en el HTML exportado.
  - **Mobiliario interactivo**: puertas y postes con cinta **no son interactuables**; la **banca**, en recorrido, **sienta la cámara** (altura de asiento, mirada libre, sin desplazamiento) y al volver a hacer clic **se levanta**. En el recorrido (editor y HTML exportado) aparece un **cartel al apuntar**: «Clic para sentarse» / «Clic para ponerse de pie».
- **v2.5 — Multi-piso y escaleras (versión actual).**
  - **Galerías de varios pisos**: el modelo pasa de un único plano a `plan.floors[]` (cada piso con sus **salas, pasillos, luces, colores, texturas y ruta** propios). Toda la galería se construye **apilada en Y** a una separación configurable (`plan.floorHeight`). Compatible hacia atrás: los proyectos de un solo piso se migran solos (`migratePlan`).
  - **Pestañas de piso** en el editor de planta: **cambiar** de piso, **＋ Piso** para añadir uno encima, **Eliminar piso**, **renombrar** (doble clic en la pestaña) y campo **Altura entre pisos** (m). Cada piso se diseña por separado en la misma vista superior.
  - **Herramienta «Escalera»** con **4 tipos**: **Recta**, **En U**, **Caracol ↺** y **Caracol ↻**. Se coloca con un clic y **conecta el piso actual con el de encima** (si no existe, se crea automáticamente). Propiedades por escalera: **tipo, largo/radio, ancho y orientación**.
  - **Huecos automáticos**: la losa del piso superior y el techo del inferior se **perforan** con la huella de cada escalera (sin *z-fighting*), de modo que el cubo de escalera queda abierto.
  - **Movimiento vertical real**: al pisar una escalera la cámara **sigue la pendiente** de los peldaños y, al llegar arriba o abajo, **cambia el piso activo** (con su propia colisión). Portado a los **dos motores** (editor y HTML exportado). El indicador de sala muestra «Escalera» y **· Planta N**.
  - **Geometría de escalera**: peldaños (huella + contrahuella) orientados a lo largo de la línea central, con **postes de barandal** a ambos lados; las espirales describen un arco de 1,5 vueltas y la U gira sobre un rellano.
  - **Vista previa 3D y export multi-piso**: la previsualización del editor muestra **todos los pisos apilados con sus escaleras**; el HTML exportado reproduce la galería completa, con recorrido vertical funcional en escritorio y móvil.
- **v2.6 — Acabado de escaleras, alturas por sala y luz por piso (versión actual).**
  - **Plataforma superior y cubo de escalera**: cada escalera lleva ahora una **plataforma plana arriba** (elimina la sensación de vacío al salir) y un **cubo de muros** alrededor de su huella, construido como un pasillo pero en vertical: **existe donde el hueco queda fuera de una sala y desaparece donde queda dentro** (se recorta contra las salas de cada planta). *(Nota: estos muros son visuales; no añaden colisión.)*
  - **Altura de pared por sala y por pasillo**: cada sala y cada pasillo puede tener su **propia altura** (`room.wallHeight` / `corridor.wallHeight`); en sus propiedades hay un control **Altura de pared** y un botón **Usar altura global**. Si no se define, se usa la altura global del plan.
  - **Luces de pisos superiores en la vista previa**: la previsualización del editor ahora coloca **las luces de todos los pisos** a su elevación (antes solo se veían las del piso activo).
  - **Los pisos bloquean la luz hacia abajo** *(intentado en v2.6, revertido en v2.6.1)*: se probó aislar las luces por planta con **capas (layers)**, pero en Three.js una luz que no comparte capa con la cámara **se desactiva por completo** (el renderizador no hace iluminación selectiva por objeto). El resultado fue que **todas las luces colocadas dejaron de iluminar**. Se revirtió: las luces vuelven a la capa 0 (siempre activas). La contrapartida es que la luz **vuelve a atravesar los pisos**; bloquearla de verdad exigiría mapas de sombra (con coste de rendimiento).
  - **Ocultar pisos en la vista previa**: botón **«Solo este piso / Ver todos los pisos»** en la cabecera de la previsualización para aislar la planta que se está editando. **Solo afecta a la vista previa**; al generar la galería siguen estando todos los pisos.
  - **Correcciones multi-piso**: el **anclaje de obras a paredes**, el **empuje con flechas** y la detección **suelo/techo del pintado** ahora son **relativos a la elevación de cada planta** (antes usaban la altura del piso bajo y hundían/colocaban mal los elementos de plantas superiores).

---

## 8. Hoja de ruta (pendiente) — con especificaciones

Lo siguiente está **identificado y especificado** pero **no implementado** todavía. Se documenta para retomarlo en una sesión nueva.

### 8.1 Multi-piso + escaleras — ✅ IMPLEMENTADO (v2.5)
Implementado en ambos motores. Modelo `plan.floors[]` con alias al piso activo (todo el editor existente sigue operando sobre el piso activo sin cambios), `plan.stairs[]`, `plan.floorHeight`. Cuatro tipos de escalera (recta, U, caracol ↺/↻), pestañas de piso, huecos automáticos en losa/techo y movimiento vertical siguiendo la pendiente con cambio de piso activo. Ver §7 (v2.5). Posibles mejoras futuras: **pasamanos continuo** (hoy solo postes), **ascensores**, **rellanos con baranda** y afinado de las curvas de espiral/U.

### 8.2 Mejoras de texturas
- Embaldosado de **paredes** por tamaño real (hoy es aproximado porque la geometría de pared está fusionada; requeriría UV por segmento o materiales por arista).
- Control de **repetición/escala** de textura por superficie.

> **Nota:** color de pared individual, altura de paredes ajustable, multiselección (Ctrl+clic), cuadros de texto de tamaño variable, multiselección en el editor de planta, inclinación de luces 0–180°, pintar con clic, sol/ambiente, PNG transparentes, texto reflowable, importar fuentes, mobiliario interactivo **y el sistema multi-piso con escaleras** —antes en esta hoja de ruta— ya están **implementados (v2.2–v2.5)**.

---

## 9. Limitaciones y notas
- **Tamaño del proyecto:** las imágenes y texturas se guardan **en base64**, lo que infla el JSON y el HTML exportado, y puede **superar la cuota de `localStorage`** (en ese caso, usar **Exportar**).
- **Primera carga del HTML exportado:** necesita **internet** (Three.js por CDN). Una vez cargado, el recorrido funciona sin más interacción de red.
- **Texturas en paredes:** el embaldosado es aproximado (ver §8.2). En suelos y techos es nítido (UV en metros del mundo).
- **Mobiliario:** los assets son **decorativos** y no bloquean el paso. Puertas y postes con cinta **no** son interactuables; la **banca** sí permite **sentarse** en recorrido (clic para sentarse/levantarse).
- **Fuentes importadas:** se incrustan en base64, lo que **aumenta** el tamaño del guardado y del HTML exportado; pueden tardar un instante en aparecer en el primer render mientras la fuente se decodifica.
- **Escaleras (v2.5–v2.7):**
  - Cada escalera lleva **plataforma plana arriba** (con holgura para no chocar con el suelo superior), un **cubo de muros** alrededor (presente fuera de las salas, ausente dentro) y, desde v2.7, una **losa de piso propia** bajo su huella (situada por debajo de los suelos de sala, así sirve de piso cuando la escalera está fuera de un edificio y queda oculta cuando hay sala).
  - **Prioridad de la muralla de sala (v2.7):** donde la huella de la escalera **cruza** una muralla, se recorta la muralla para abrir el paso; donde la huella es **colineal** con una muralla, **gana la muralla de la sala** (se conserva) y el muro del cubo se suprime, evitando paredes dobles/raras.
  - **Huecos de techo/suelo robustos (v2.7):** el hueco se **recorta al contorno de la sala**, de modo que el techo se abre aunque la huella roce o sobresalga la muralla, sin geometría inválida.
  - La **escalera en U** tiene un **rellano cuadrado plano** en el giro (ya no una diagonal) (v2.6.1).
  - El barandal tiene **postes y un pasamanos continuo** (cilindros que unen los topes de los postes, siguiendo la pendiente y el rellano) (v2.8).
  - El **pasamanos corta en los giros bruscos**: en la escalera en U el riel se interrumpe limpio en el rellano (detección por cambio de tangente >0.9 rad) en vez de cruzar el hueco; en recta y caracol sigue continuo (v2.9).
  - **Barrera de colisión sobre el hueco** (`overOpening`): en la planta superior solo se puede transitar por el carril de entrada de la escalera (a la altura del piso); el resto de la apertura está bloqueado, evitando quedar atrapado entre el borde del suelo y la escalera (v2.9).
  - **Barandilla de seguridad alrededor del hueco** superior (postes + pasamanos dorado en los 4 lados, con un vano en la entrada de la escalera) (v2.9).
  - Huecos de suelo/techo **agrandados y más robustos** (se identifica la sala por el centro real de la huella) para dar holgura a las escaleras de caracol (v2.9).
  - El **cubo cierra el entrepiso**: entre el techo de la planta inferior y el suelo de la superior se dibuja siempre una banda de muro en los 4 lados, de modo que **no se ve el vacío ni el resto del mapa** a través del hueco de la escalera (v2.8).
  - Cada escalera debe colocarse **dentro de una sala o pasillo** (su base y su llegada han de quedar sobre suelo transitable para poder pisarla).
  - El hueco del cubo de escalera en el piso superior es **transitable**: el jugador no cae —al pisarlo **vuelve a entrar en la escalera** y desciende—.
  - Una escalera conecta el piso en que se coloca con el **inmediatamente superior**; para subir varios niveles se encadenan varias escaleras.
- **Luz entre plantas y salas (v2.7):** **ahora SÍ se bloquea**, mediante **mapas de sombra** (no por *layers*, que apagaba las luces). Suelos, techos, paredes y escaleras proyectan/reciben sombras, así la losa superior frena la luz hacia abajo y cada sala puede tener su propio ambiente; el sol y los focos/puntos proyectan sombra. Hay un interruptor **«Sombras»** en *Sol y ambiente* (activo por defecto): si la galería va lenta con muchas luces —sobre todo luces **de tipo punto**, que usan mapas de cubo y son más costosas—, conviene desactivarlo. Coste de rendimiento real; medirlo en el equipo de destino.
- **Compatibilidad:** requiere WebGL y navegador moderno.

---

## 10. Cambios v2.10

### 10.1 Iluminación corregida en el editor — ✅
**Síntoma:** todos los presets (Museo neutro, Fría, Brillante, Dramática, Penumbra) dejaban la escena **negra** en modo Edición; solo «Cálida» se veía. En la vista previa se veían bien.
**Causa:** la vista previa (`previewLighting`) **ignora el preset** (usa ambiente+hemisférico fijos), por eso nunca fallaba. En cambio `applyLighting` (edición/recorrido) sí aplica presets, y todos salvo «Cálida» añadían una **luz direccional con sombras** (`cfgDir`). Esa configuración de sombra direccional estaba mal: faltaba `updateProjectionMatrix()` tras fijar los límites de la cámara ortográfica, faltaba **añadir el `target`** de la luz a la escena, y usaba un `bias` agresivo sin `normalBias`. Con las superficies recibiendo sombra de un mapa mal orientado, todo quedaba en sombra → negro.
**Solución (ambos motores + vista previa):** las direccionales de los presets pasan a ser **luz de relleno sin sombra**; **solo el sol** proyecta sombra direccional. El `cfgDir` del sol se corrigió: `mapSize 2048`, límites ortográficos ±45, `near 0.5`/`far 160`, **`updateProjectionMatrix()`**, `bias -0.0004`, **`normalBias 0.05`** y se **añade `target` al grupo de luces**. Como el sol está **apagado por defecto**, los presets se ven correctamente sin sombras pesadas; al activarlo, la sombra del sol funciona bien.

### 10.2 Entrada/salida de escaleras como pasillos — ✅
**Síntoma:** desde el 2.º piso la entrada a la escalera quedaba bloqueada y el visitante quedaba **pegado en el tope** sin poder moverse.
**Causa:** al llegar al tope (su.t≈1), el detector de escalera **re-agarraba** al jugador inmediatamente, atrapándolo en un bucle agarrar/soltar.
**Solución (ambos motores):** `stairAt` solo «sube» al jugador a la escalera cuando pisa la **rampa** (su.t entre 0.05 y 0.95). Los **extremos** (su.t < 0.05 abajo, > 0.95 arriba) se comportan como **descansos caminables**, igual que la boca de un pasillo: se entra y se sale andando, sin re-agarre ni atrapamiento. Los umbrales de salida (0.03 / 0.97) quedan fuera de la zona de agarre, eliminando la oscilación. El vano de la barandilla de seguridad se amplió (×1.2) para dejar la entrada despejada.

### 10.3 Rotar salas en el editor de planta — ✅ (solo editor)
En las propiedades de una sala aparecen botones **⟲ 15° / ⟳ 15° / ⟳ 45°**. `rotateRoom(r,deg)` convierte la sala a **polígono** (rect → 4 vértices; elipse → 40; el círculo se omite porque rotarlo es idéntico) y gira sus puntos alrededor del **centroide**. Tras rotar, la sala es un polígono editable normal (vértices arrastrables). Se redibuja el plano y la vista previa al instante.

### 10.4 Recorrido guiado por nodos (editor normal) — ✅ (ambos motores)
Sustituye a la antigua «Ruta» del editor de planta (eliminada). Modelo nuevo **global**: `plan.tourNodes[]`, cada nodo `{t:'path'|'dest', x, y, z, fl, yaw, pitch}`.
- **Programación (modo Edición, 1.ª persona):** botón **«Programar recorrido»** (sección 05 del panel). Con el ratón capturado, caminando y mirando: **Z** coloca un **nodo de camino** (guía la ruta sin parar, para el *path-finding*); **Espacio** coloca un **nodo de destino** (guarda posición + **punto de mira** —yaw/pitch— para encuadrar una obra). Botones **Deshacer nodo** y **Vaciar recorrido**. Los nodos se ven como marcadores en el suelo (discos azules = camino, anillos dorados + flecha de mira = destino) y un **token verde** en el inicio.
- **Reproducción (modo Recorrido y export):** se pulsa el **círculo/token de inicio** (clic o toque) o la **flecha derecha** para empezar. **→** avanza al **siguiente destino** atravesando los nodos de camino con una animación suave (posición interpolada por longitud de arco + giro/inclinación con *easing*); **←** vuelve al **anterior**. Durante el recorrido WASD lo abandona para caminar libre. En el export, las flechas izquierda/derecha se reservan para navegar el recorrido cuando hay destinos (el avance/retroceso siguen en W/S o ↑/↓ y el joystick en móvil). Verificado por sintaxis (`node --check` en ambos motores) y por **lógica** (secuencia de destinos adelante/atrás y la interpolación `posAlong`); **no** verificado el render real (sin WebGL en el entorno).

> **Nota de rendimiento:** si la galería va lenta, el interruptor **«Sombras»** sigue disponible. El recorrido por nodos es muy ligero (marcadores `MeshBasic`).

### 10.5 Escaleras en U: acceso y baranda corregidos (v2.10.1) — ✅
**Acceso bloqueado (de nuevo):** la barrera del hueco (`overOpening`) era demasiado estricta: solo dejaba pasar por un **carril angosto** y con una condición de altura que, en la escalera en U, dejaba la entrada en una esquina imposible. Ahora un punto del hueco es **transitable** si está **en el carril** de la escalera (al pisar la rampa, `stairAt` te agarra y bajas/subes) **o** en la **zona alta** cerca del nivel del piso superior (el descanso de entrada, a lo ancho). Verificado con **simulación de movimiento real** (tryMove + stairAt + transición de piso): se baja de planta 2 a 1 y se sube de 1 a 2 de forma fluida, sin atascarse, siguiendo la línea de la U.
**Baranda rota de la U:** el riel del **rellano plano** quedaba como una barra flotante (la T) y el giro generaba cruces. Ahora el pasamanos **no se dibuja en los tramos planos** (el rellano) y **se corta cuando el salto lateral es grande** (el giro de 180°), dejando un riel limpio y continuo en cada tramo inclinado. Las escaleras **recta** y de **caracol** no se ven afectadas (no tienen rellano plano).

### 10.6 Techos transparentes en escalera + colisión de barandillas (v2.10.2) — ✅
**Techos al usar la escalera:** ahora, mientras el visitante está **sobre una escalera** (`onStair`), **todos los techos se ocultan** automáticamente, de modo que se ve con claridad el hueco y ambos niveles durante la subida/bajada; al pisar de nuevo el suelo, los techos reaparecen. Implementado etiquetando cada techo (`userData.ceil`) al construirlo y alternando su visibilidad según el estado de escalera (ambos motores).
**Colisión de la barandilla del piso superior:** la baranda de seguridad alrededor del hueco ya **no se atraviesa**. Una función (`stairRailColliders`) genera segmentos de colisión que **igualan el perímetro renderizado de la baranda** (la huella de la escalera) **con el mismo vano de entrada** (junto al tope), y se añaden a las paredes del piso superior (`FLOORS[pisoSuperior].wallSegs`). El visitante queda bloqueado en todo el perímetro de la baranda y solo puede entrar por el vano; verificado con simulación de movimiento: con los colliders activos se sigue **bajando de planta 2 a 1 y subiendo de 1 a 2 de forma fluida**, sin atascarse. Las escaleras rectas y de caracol también reciben su collider perimetral.

### 10.7 Techo selectivo en escalera + salto de cámara (v2.10.3) — ✅
**Techos:** ocultar **todos** los techos al pisar la escalera era excesivo (parecía que el edificio perdía el tejado). Ahora se oculta **solo el techo del piso inferior de la escalera** (la losa entre pisos), que es la que estorba la vista hacia arriba; el **techo exterior del piso superior se conserva**. Cada techo se etiqueta con su número de piso (`userData.ceilFloor`) y, al estar en una escalera, solo se oculta el del `floorLo` de esa escalera. Ambos motores.
**Salto brusco de la vista (movementX):** en navegadores basados en Chromium, `movementX/movementY` del puntero bloqueado a veces devuelve un valor enorme y la vista "salta". Se corrigió **limitando ese delta** (clamp a ±110 px por evento) en ambos motores. En el editor, además, se **neutralizó el giro interno de PointerLockControls** (`pointerSpeed=0`) y se implementó un giro propio con el límite, de modo que ningún pico de ratón teletransporta la cámara.

### 10.8 Hueco de techo horneado + tapa de escalera + dedup + slider de rotación (v2.10.4) — ✅
Refactor que **elimina** la maquinaria de ocultado de techos en tiempo de ejecución del v2.10.2/v2.10.3 y la sustituye por geometría correcta horneada. Cuatro frentes:

**1) Hueco de techo robusto (causa raíz del bug).** El bug de fondo: al pisar la escalera se ocultaba dinámicamente el techo del piso inferior porque el hueco real (`holesIn`) **solo se calaba en la sala cuyo centro contenía la huella**. Si la escalera quedaba **a caballo entre dos salas** (o entre sala y pasillo), la sala vecina conservaba su techo y tapaba la vista, lo que obligó al parche de "ocultar todo el piso". Ahora `holesIn` **recorta el hueco a la bbox de cada celda y lo abre en TODAS las que solapan la huella** (unión), sin la prueba de "centro dentro del polígono". Resultado: el agujero del techo es continuo aunque la escalera cruce el muro entre dos salas. Verificado con prueba de lógica (la sala que antes quedaba con "labio" ahora también se cala). Ambos motores.

**2) Eliminada toda la maquinaria de ocultado en runtime.** Fuera `ceilMeshes`, `ceilHideFloor`, el etiquetado `userData.ceil`/`userData.ceilFloor`, la recolección de techos tras construir y los **dos toggles por-frame** del bucle de animación. El techo ya no tiene estado dinámico: se construye una vez, con su agujero, y no se vuelve a tocar. Menos estado global y menos trabajo por frame.

**3) Tapa de techo de la escalera.** Antes, una escalera **al aire libre** (fuera de cualquier sala/pasillo) no tenía techo: se veía el vacío arriba. Ahora `buildStairwell` añade una **losa de tapa** sobre la huella a la altura del techo del piso superior (`eHi + hiH + 0.06`), por **encima** de los techos de sala. Es el espejo de la losa de piso de la escalera (que va por **debajo** de los suelos de sala): cuando hay sala, el techo de la sala oculta la tapa desde abajo; cuando la escalera está en el vacío, la tapa es el techo visible. Así la escalera **siempre tiene techo**. Ambos motores.

**4) Plataforma superior a ras.** El descanso superior de la escalera estaba a `eHi-0.09` (hundido un poco para no chocar con el suelo superior); como el suelo superior tiene su hueco sobre la huella, ya no hay riesgo de choque y se sube a `eHi-0.04`, **a ras del piso superior**, para una transición limpia al caminar. Ambos motores.

**Optimización (dedup dentro de cada motor).** Se factorizó la duplicación **interna** de cada motor (la duplicación *entre* motores es obligatoria por las reglas del `TOUR_TEMPLATE`):
- `footCorners(f)` — las cuatro esquinas de la huella, antes repetidas literalmente en `stairRailColliders` y dos veces en `buildStairwell`.
- `orientedWallGeom(x1,z1,x2,z2,yBase,hgt)` — caja de muro orientada por el ángulo del segmento, compartida por `box()` de `buildInto` y `segBox()` de `buildStairwell`.
- `flatSlab(cx,cy,cz,wx,wz,mat,thick)` — losa plana, usada por la losa de piso **y** la nueva tapa de techo de la escalera.
Los **barandales** (`buildStairNode` vs `buildStairwell`) se dejan **sin unificar a propósito**: su lógica difiere (riel inclinado con postes y corte por salto vs. riel perimetral de seguridad con vano de entrada), y unificarlos arriesgaría el comportamiento.

**Slider de rotación de salas (editor de planta).** Las propiedades de una sala ahora incluyen, además de los botones ⟲15°/⟳15°/⟳45°, un **slider continuo «Girar» (−180°…180°)**. La matemática de rotación se factorizó en `rotatePointsAround(points, c, deg)` y `roomToPoly(r)`, compartidas por los botones y el slider. El slider rota **desde una base capturada al empezar a arrastrar** (la sala se convierte a polígono y se guarda su forma y centroide) y **re-fija esa base al soltar**, reseteando el slider a 0; así el giro libre **no acumula error de coma flotante** ni se pelea con el arrastre de vértices. Solo existe en el motor principal (el recorrido exportado no tiene editor).

**Validación.** `node --check` del módulo principal y del `TOUR_TEMPLATE` materializado; prueba de lógica del nuevo `holesIn` y de la rotación (sin deriva en 4×90°); y **smoke test de ejecución real** con un mock de THREE que recorre `stairData → buildStairwell → buildInto` en dos plantas con escalera a caballo entre salas, en **ambos motores** (paridad idéntica: mismas mallas y celdas). La verificación **visual** con WebGL no es posible en el entorno de construcción; se recomienda una pasada visual al abrir el archivo.

### 10.9 Huecos de escalera robustos e independientes (v2.10.5) — ✅
**Bug:** al conectar **varias escaleras al mismo piso superior** (p. ej. dos salas de la planta baja que suben a una única sala de arriba), la **segunda** escalera mostraba el **techo bloqueado** (su hueco no se calaba), y en el piso superior quedaba el suelo **entrecortado**. El defecto **no dependía** de tener pisos conectados de cierta forma, sino de **cómo se calaban los huecos**.

**Causa raíz.** En v2.10.4, para arreglar el cruce de dos salas, se quitó la prueba de "centro de la huella dentro del polígono" y se pasó a calar la huella en **toda celda cuya bbox solapara**. Eso introdujo dos fragilidades en la triangulación de `ShapeGeometry` (earcut), que se comprobaron empíricamente con Three.js real:
1. **Huecos que se solapan o están duplicados rompen earcut**: dos huecos idénticos dejan la sala **sin calar** (área completa), y dos solapados producen un recorte incorrecto. Esto ocurría cuando dos rellanos caían cerca en la sala superior (huellas + padding solapados), o cuando la huella de una escalera vecina se "colaba" en una sala contigua y solapaba el hueco real.
2. **Contaminación cruzada**: sin la prueba de centro, el *padding* de una escalera situada junto a un muro medianero calaba un trozo del techo de la **sala vecina**.

**Arreglo (cada escalera, independiente).** Se rediseñó `holesIn` en ambos motores con dos reglas:
- **Membresía por solape real:** una huella solo cala una celda si la solapa de forma **sustancial** (≥22% del área de la huella, medido sin el margen de expansión). Así se mantiene el **cruce legítimo de dos salas** (cada sala recibe su parte) pero se descarta el **roce del padding** de una escalera vecina (contaminación eliminada).
- **Fusión de huecos solapados:** los huecos que aún se solapen dentro de una misma celda se **fusionan en su bbox conjunta** hasta que el conjunto sea **disjunto**. earcut nunca recibe huecos solapados, de modo que **el resultado no depende de cuántas escaleras haya** ni de su cercanía. Dos rellanos juntos producen un único hueco que cubre ambos; dos separados, dos huecos limpios.

**Verificación.** Además de la sintaxis y los tests previos, se añadió una **prueba integral con `ShapeGeometry` real de Three.js** que ejecuta el `holesIn` **extraído del propio archivo** sobre los escenarios reportados: (1) dos escaleras en dos salas distintas hacia una sala superior — **ambos** techos inferiores se calan y el piso superior conserva **dos huecos**; (2) rellanos **cercanos** con huellas solapadas — se **fusionan** y el piso superior **sí** se cala (antes quedaba bloqueado); (3) escalera **a caballo** de dos salas — sin labio; (4) sala vecina **sin** escalera — intacta; (5) **tres** escaleras al mismo piso — tres huecos independientes. 13/13 asserts en verde.

---

### 10.10 Caracol retirado + techos de salas no rectangulares + optimización (v2.10.6) — ✅
Entrega integral orientada a clientes. Tres frentes:

**1) Escaleras de caracol eliminadas.** Presentaban varios defectos visuales (barandal y geometría irregulares) y se han **retirado por completo** del código en ambos motores: fuera la rama `spiralL`/`spiralR` de `stairData`, las opciones del selector de creación y del panel de propiedades, la etiqueta y el ajuste de label del slider (ya no hay "Radio", solo "Largo"). Quedan únicamente **Recta** y **En U**. Para no romper proyectos antiguos, `migratePlan` **convierte** cualquier escalera de caracol guardada en **recta** al cargar (ambos motores).

**2) Techos/pisos de salas NO rectangulares (bug de fondo).** En salas con forma de **círculo, elipse o polígono cóncavo**, el hueco de la escalera quedaba mal: `holesIn` recortaba el hueco a la **bbox** del polígono, no al polígono real, de modo que el rectángulo del hueco **cruzaba el contorno curvo** de la sala. Se comprobó con Three.js real que, cuando un hueco cruza el borde, **earcut produce basura** (área triangulada *mayor* que la propia sala). Arreglo: cada hueco (ya disjunto tras la fusión de v2.10.5) se **recorta al polígono real de la sala** mediante **Sutherland-Hodgman** (`clipRectToPoly`), y los huecos se pasan a `floorMesh`/`ceilMesh` como **polígonos** (no rectángulos). El hueco sigue ahora el borde curvo y nunca lo cruza. En salas rectangulares el resultado es idéntico al anterior (un hueco de 4 vértices), sin regresión.

**3) Optimización.** `roomPoly(r)` se calculaba **dos veces por sala** en cada reconstrucción de `buildInto` (una para suelo/techo y otra para los muros), costoso en salas circulares/elípticas (decenas de vértices). Ahora el polígono se **cachea** (`roomPolys[i]`) y se reutiliza para los muros, en ambos motores. Se mantiene la dedup interna previa (`footCorners`, `orientedWallGeom`, `flatSlab`) y se añade `clipRectToPoly`/`polyAreaAbs` como helpers compartidos por motor.

**Verificación.** Sintaxis (`node --check`) de ambos motores; smoke test de ejecución real ampliado con **sala circular** y **escalera en U**; prueba de rotación; y una **prueba integral con `ShapeGeometry` real de Three.js** que ejecuta el `holesIn` extraído del propio archivo sobre: círculo (hueco que sobresale → calado limpio siguiendo el borde curvo), elipse, polígono cóncavo (L), huella en la muesca de la L (fuera → no cala), más todos los escenarios multi-escalera previos y la no-regresión en salas rectangulares. **15/15** asserts en verde. La verificación **visual** con WebGL sigue sin ser posible en el entorno de construcción; se recomienda una pasada visual con una sala redonda y la escalera dentro.

---

*Fin del documento — v2.10.6.*
