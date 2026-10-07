# Ruta Catastral — planificador de visitas de campo (IGAC)

Aplicación web de **un solo archivo** (`RutaCatastral.html`) para:

1. **Espacializar** los predios del Excel de comisión cruzándolos (por NPN) con las capas `u_terreno` y `r_terreno`.
2. Calcular la **ruta óptima** de visita sobre vías reales (OpenRouteService u OSRM) o, sin internet, un orden aproximado.
3. Entregar: **KMZ** (Google Earth / My Maps), el **mismo Excel con la columna `Orden` diligenciada** + horas estimadas, itinerario CSV, `ruta.geojson`, `ruta.gpx`, **incidencias en Excel** y enlaces de Google Maps.

- Sin instalación, sin servidor, sin build y sin CDN: todo el código está dentro del `.html` (≈290 KB, versión 1.4.0).
- Interfaz por pasos (1 a 6) plegables, con un indicador de estado en cada encabezado. Las explicaciones y la ayuda están en desplegables "ℹ" para no saturar la pantalla.
- **Privacidad:** el Excel y los shapefiles se procesan solo en el navegador. Solo salen las **coordenadas** hacia el motor de rutas (y el texto que escriba en el buscador de direcciones).
- **No contiene ninguna API key.** Cada persona usa la suya o trabaja sin clave (OSRM).

> **Nota técnica:** el archivo no incluye librerías de terceros (Leaflet, SheetJS, shpjs, JSZip, proj4, Turf). Sus funciones se implementaron dentro del mismo archivo: lector de shapefile, lector y escritor XLSX/ZIP, proyecciones MAGNA-SIRGAS, visor de mapa y optimizador. Por eso el archivo es liviano y no depende de licencias externas.
>
> **El Excel conserva su formato:** la aplicación edita directamente el XML de la hoja, así que se mantienen colores, anchos de columna, fórmulas y celdas combinadas. Esto es una diferencia frente a SheetJS. Aun así, el itinerario también se entrega en CSV.

---

## 1. Modos de ejecución (la aplicación detecta el modo y lo muestra en la cabecera)

| Modo | Cómo se abre | Comentario |
|---|---|---|
| **A** | Doble clic sobre `RutaCatastral.html` (`file://`) | Funciona casi siempre. Si el navegador bloquea la conexión (CORS), la aplicación lo detecta al iniciar y ofrece alternativas. La clave guardada puede no conservarse entre sesiones. |
| **B (RECOMENDADO)** | Publicado por HTTPS (GitHub Pages u otro alojamiento estático) | Conexión al motor de rutas garantizada (CORS). Basta con la URL; no hay que llevar el archivo. Los datos siguen procesándose solo en su navegador. |
| **C** | `python -m http.server 8000` en la carpeta del archivo → `http://localhost:8000/RutaCatastral.html` | Para quien tenga Python. Origen `http://localhost` estable. |

**Recomendación: use el modo B.** Es el más eficiente para trabajar desde cualquier equipo del IGAC sin copiar archivos.

## 2. Publicar en GitHub Pages (paso a paso)

1. Cree una cuenta en https://github.com (si no tiene una) e inicie sesión.
2. Pulse **New repository**. Póngale un nombre (por ejemplo `ruta-catastral`), márquelo como **Public** y pulse **Create repository**.
3. En el repositorio, elija **Add file → Upload files**. Suba `RutaCatastral.html` **renombrado como `index.html`** (y este `README.md`, si quiere). Pulse **Commit changes**.
4. Vaya a **Settings → Pages**. En *Build and deployment* elija **Source: Deploy from a branch**, **Branch: `main`**, carpeta **`/ (root)`** y pulse **Save**.
5. Espere 1 o 2 minutos. La dirección aparecerá en la misma página: `https://<su-usuario>.github.io/ruta-catastral/`
6. Comparta esa URL con sus colegas. **Nunca suba al repositorio Excel, shapefiles ni claves.** El código es público y la aplicación no los necesita en el servidor.
7. Para actualizar la aplicación, suba de nuevo `index.html` (el nuevo reemplaza al anterior).

### Abrirlo localmente
- **Modo A:** doble clic sobre `RutaCatastral.html` (Chrome, Edge o Firefox actualizados).
- **Modo C:** abra una terminal en la carpeta y ejecute `python -m http.server 8000`. Luego abra `http://localhost:8000/RutaCatastral.html`.

## 3. Obtener la API key de OpenRouteService (opcional)

1. Entre a **https://account.heigit.org**. Es esa página y no otra:
   - **No es** https://openrouteservice.org/dev/#/api-docs: esa es la documentación, y ahí la clave solo aparece como marcador (`INSERT_YOUR_KEY` / `YOUR-KEY`).
   - **Tampoco es** https://routing.heigit.org: ese es el cliente de mapas.
2. Inicie sesión con su **correo electrónico**, o con *Sign in with GitHub* si se registró por GitHub.
   - Desde la migración del 25/02/2025 ya no se puede entrar con nombre de usuario.
   - La contraseña debe tener mínimo 8 caracteres.
3. En el **Dashboard**, sección de tokens / API keys, copie la clave del **Standard Plan**.
4. Si no aparece ninguna, créela: elija el tipo de token, póngale un nombre y pulse crear.
5. Pegue la cadena **completa** en el paso 1 de la aplicación, **incluido el `=` final**. Pulse **Probar conexión**.
6. Si el portal lo muestra desconectado al pasar entre dominios, permita las cookies de terceros de openrouteservice.org y desactive la protección antirrastreo para ese sitio. Pasa sobre todo en Firefox.

**Cómo usa la clave la aplicación:**
- Valida el formato en el propio navegador, sin gastar cuota: la clave debe decodificar a un JSON con los campos `org`, `id` y `h`.
- La envía **solo** en el encabezado `Authorization` a `https://api.heigit.org`. Nunca va en la URL ni aparece en la consola.
- Solo se guarda si marca **"Recordar esta clave en este navegador"**. El botón **"Olvidar clave guardada"** la borra.

Cuotas gratuitas (Plan Estándar):

| Servicio | Peticiones por día |
|---|---|
| Directions | 10.000 |
| Matrix | 2.500 |
| Optimization | 2.500 |
| Geocoding | 15.000 |

Se restablecen 24 h después de la primera petición. Una jornada típica gasta 4 peticiones: 1 snap + 1 matrix + 1 optimization + 1 directions. Si repite una jornada idéntica, las respuestas salen de la caché local y no consumen cuota.

Endpoints usados:

| Uso | Endpoint |
|---|---|
| Acceso vial | `https://api.heigit.org/openrouteservice/v2/snap/driving-car` |
| Matriz de tiempos | `.../v2/matrix/driving-car` |
| Trazado de la ruta | `.../v2/directions/driving-car/geojson` |
| Optimización (VROOM) | `https://api.heigit.org/vroom/v0` |

Todos son editables en *Opciones avanzadas*. No se usa el dominio deprecado `api.openrouteservice.org`.

## 4. Usarla sin clave

- Pulse **"Continuar sin clave (usar OSRM)"**. Usa el servidor público de demostración de OSRM, comunitario y sin garantía de disponibilidad. Solo calcula rutas en automóvil.
- **Sin internet:** elija **"Sin conexión (aproximado)"**. Calcula el orden en línea recta (vecino más cercano + 2-opt + Or-opt). El orden queda marcado como **APROXIMADO, NO SIGUE VÍAS REALES** en la pantalla, el KMZ, el Excel y las incidencias.
- Cargar archivos, generar el KMZ y llenar el Excel funcionan igual en los tres modos.

## 5. Flujo de trabajo de cada jornada

1. **Archivos:**
   - Arrastre el Excel de la comisión (`.xlsx`).
   - **Detección de municipios (solo con el Excel):** la aplicación lee el código DANE contenido en cada NPN (5 primeros dígitos) y la columna `Municipio`. Por **cada municipio** detectado muestra dos casillas: capa **URBANA** (`u_terreno`) y capa **RURAL** (`r_terreno`).
     - Cada casilla indica si es **Requerida** (hay predios de ese tipo en el Excel) u **Opcional**.
     - Puede cargar capa por capa, o usar **"Cargar todas las capas de una vez"**: seleccione todos los `.shp/.shx/.dbf/.prj` (o `.zip`) de todos los municipios y se asignan solos por sus códigos.
     - Si una capa se suelta en la casilla equivocada (rural en la urbana, u otro municipio), la aplicación lo detecta por los `CODIGO` y la reasigna, avisando.
     - Si al calcular la ruta falta alguna capa requerida, la aplicación lo advierte y ofrece "Continuar sin esas capas" (esos predios quedan al final sin ORDEN).
   - Para una nueva jornada basta con cambiar **solo el Excel**: las capas ya cargadas de cada municipio se conservan.
   - **Problemas detectados:** el panel tiene los botones **Reducir / Normal / Ampliar** (pantalla completa; Esc para salir), se puede estirar arrastrando su borde inferior y filtrar por severidad (ALTA, MEDIA, INFO).
2. **Revisión automática:**
   - Detecta la columna `NPN` y conserva siempre los 30 dígitos como texto.
   - Detecta el campo `CODIGO` de las capas y valida el sistema de coordenadas: debe ser MAGNA-SIRGAS / CTM12 y las coordenadas deben caer dentro de Colombia.
   - Muestra el resumen y las incidencias.
3. **Punto de partida:** clic en el mapa, búsqueda de dirección, "mi ubicación" o URL de Google Maps.
4. **Parámetros de la jornada.** Por defecto: inicio 07:00, jornada de 8 h, 15 min por predio y automóvil tipo sedán. Se recuerdan entre sesiones.
   - **Criterios de la ruta** (paso 3, recuadro 🧭):
     - **Orden de trabajo:** sin preferencia (la más corta) · primero lo RURAL · primero lo URBANO · en bloques.
     - **De lo más lejano a lo más cercano al casco urbano** (desde la v1.2.1):
       - La cercanía se mide en **tiempo real por vía** hasta el casco, no en línea recta.
       - La ruta completa se optimiza de **una sola vez**. Cada vez que el recorrido se *aleja* del casco entre dos visitas se suma una penalización proporcional a lo que se aleja.
       - Así la ruta empieza por lo lejano y se va acercando, sin ir y volver por el mismo camino.
       - El nivel (Flexible, Normal, Estricto, Muy estricto) regula cuánto se prioriza el orden de lejos a cerca frente a los kilómetros. Se recomienda **Normal**.
       - Si varios predios están sobre un mismo camino sin salida, es normal entrar y salir por él una vez.
     - **Terminar cerca del casco urbano** (por hoteles y seguridad).
     - **¿Dónde termina la jornada?** En el último predio · de regreso a la partida · **en un punto que marco** (🏁 hotel u oficina; la ruta incluye el trayecto final hasta ese punto).
     - El botón **⭐ Recomendado zona rural** aplica de una vez: rural primero, de lejos a cerca y terminar cerca del casco.
     - **Casco urbano:** se calcula solo, como el centro de los predios urbanos de la cabecera (zona 01) de cada municipio, y se ve en el mapa con 🏙. Si marca un punto 🏁, se usa ese punto como referencia.
     - El resultado indica cuánto tiempo adicional cuesta seguir los criterios frente a la ruta más corta, y a cuántos km del casco queda la última visita. El itinerario trae la columna `DIST_CASCO_URBANO_KM`.
5. **Acceso vial y ubicación manual:**
   - Al calcular la ruta, una sola petición comprueba que cada punto esté a ≤ 350 m de una vía. Solo se validan los puntos nuevos o que cambiaron; los demás conservan su validación.
   - Los que no cumplen se excluyen de la ruta, pero siguen en el KMZ como "sin acceso vial directo".
   - **Reubicar** (botón en la lista o en la ventana del predio): arrastre el marcador hasta el acceso.
     - El punto que usted fija se toma como **acceso confirmado** y **no se vuelve a validar**. Los demás predios conservan su validación, incluso si recarga capas.
     - Con OpenRouteService, al calcular la ruta solo ese punto se lleva a la vía más cercana (hasta 2 km). Si queda lejos, el itinerario indica "continuar a pie ~X m".
     - Puede volver al punto original con "Volver al punto original".
   - **Ubicar a mano los predios sin geometría** (o con geometría inválida, o de una capa que no tiene):
     - Pulse "📍 Ubicar en el mapa" y haga clic donde cree que está el predio.
     - La aplicación **sugiere** una ubicación junto a los predios vecinos (número de terreno más cercano) de la **misma manzana o vereda**, según el NPN. Pulse "Usar la ubicación sugerida" para aceptarla.
     - El punto se ve en **morado**, se puede arrastrar y entra en la ruta, en el Excel (con ORDEN) y en el KMZ como "SIN GEOMETRÍA — ubicación MANUAL aproximada para buscar en terreno". Queda la incidencia `UBICACION_MANUAL`.
6. **Calcular ruta óptima.** Si la ruta pasa de la jornada, igual se programa como **una sola ruta** y se registra la novedad `JORNADA_EXCEDIDA`.
   - **Cambiar el número de una parada (v1.3):** si al ver la ruta prefiere otro orden (por ejemplo, que la 17, que queda junto a la 7, sea la 8):
     - **En el mapa:** haga clic en la parada y use "Pasar a [N.º] → Aplicar", las flechas ◀ ▶, o "Visitar justo después de la [N.º]".
     - **En el itinerario:** use ▲ ▼ o escriba el nuevo número en la columna "Cambiar orden" y pulse Enter.
     - Las demás paradas se **renumeran solas** (la 8 pasa a 9, la 9 a 10…). Se recalculan horas de llegada y salida, distancias, totales y el **trazado sobre vías**. **No** se repite la optimización ni la matriz; solo se pide el trazado nuevo.
     - Se muestra el efecto del cambio (minutos y km de más o de menos frente al orden óptimo), y las filas cambiadas quedan resaltadas.
     - **↶ Deshacer último cambio** y **Restaurar orden óptimo** están disponibles en todo momento.
     - El Excel (ORDEN y horas), el KMZ, el GPX, el GeoJSON y los enlaces de Google Maps usan el orden ajustado. Queda la incidencia `ORDEN_MODIFICADO_MANUAL` con la lista de cambios (p. ej. 17→8, 8→9…).
     - Si vuelve a pulsar **Calcular ruta óptima**, se recalcula todo y se pierden los ajustes manuales.
   - **📱 Jornada para celular (v1.4):** después de calcular la ruta, este botón verde del paso 6 descarga `…_jornada_celular.geojson`. Ábralo en el celular con la app **Ruta Catastral Campo** (Android), que lleva el tiempo, el GPS, el chequeo de visitas, las observaciones y las fotos en terreno. El archivo también va dentro de "Descargar todo". Para instalar la app, siga el README de su proyecto.
7. **Descargas:**
   - **Descargar todo (.zip)**, o cada archivo por separado.
   - **Abrir en Google Maps:** un enlace por cada tramo de 10 paradas.

**Reglas que aplica la aplicación:**
- **NPN repetido en el Excel:** se visita una sola vez. Todas sus filas reciben el mismo ORDEN, y en el KMZ los radicados y mutaciones se unen con ` | `.
- **Excel de salida:** se llena la columna `Orden` y se agregan al final las columnas `HORA_LLEGADA` y `HORA_SALIDA`, más la hoja **ITINERARIO**. El resto de columnas, filas y su orden quedan **intactos**.

## 6. Abrir el KMZ

- **Google Earth (escritorio o web):** *Archivo → Abrir* (o arrastrar el `.kmz`). El KMZ trae estas carpetas:
  - "Punto de partida"
  - "Ruta optimizada"
  - "Predios urbanos" (azul)
  - "Predios rurales" (verde)

  Al hacer clic en un predio se abre su ficha con 8 campos: `area_shape`, `radicado`, `tipo_mutacion`, `observaciones`, `numero_documento`, `nombre`, `orden_visita` y `numero_predial`.
- **Google My Maps:** https://mymaps.google.com → *Crear mapa* → *Importar* → seleccione el `.kmz`.
  - My Maps reemplaza los íconos numerados por los suyos, pero conserva el nombre (número de orden) y la tabla de atributos.

## 7. Si recibiste esta aplicación de otra persona

- **No pidas ni reutilices claves ajenas.** HeiGIT permite 1 cuenta y 1 clave gratuita por persona.
- Registra **tu propia cuenta** en https://account.heigit.org (ver sección 3), **o** usa el modo **OSRM**, que no necesita clave.
- La aplicación no trae claves ni datos de nadie. Si al abrirla aparece una clave, está guardada en *tu* navegador: pulsa "Olvidar clave guardada".

## 8. Solución de problemas

| Problema | Qué hacer |
|---|---|
| **Aviso "Tu navegador bloqueó la conexión…" (CORS en `file://`)** | Abra la aplicación desde su URL de GitHub Pages (modo B) o con `python -m http.server` (modo C), o use el modo sin conexión. |
| **"La clave no es válida" (401/403)** | Copie de nuevo la clave **completa** desde account.heigit.org, incluido el `=` final. No es su correo ni su contraseña. |
| **"Cuota diaria agotada" (429)** | Se restablece 24 h después de su primera petición. Pulse "Continuar con OSRM". |
| **"El servicio no responde"** | Puede haber mantenimiento en HeiGIT. Cambie a OSRM o al modo sin conexión. |
| **Punto a más de 350 m de una vía** ("Could not find point … within a radius of 350.0 meters") | El paso 4 detecta estos puntos **antes** de optimizar y los excluye para que el proceso no falle. Arrastre el marcador hasta el acceso vial más cercano y valide de nuevo. |
| **Shapefile sin `.prj`** | La aplicación no supone ningún sistema por su cuenta: pide elegirlo de la lista (CTM12/EPSG:9377, MAGNA Bogotá/Oeste/Este…). Si las coordenadas no caen en Colombia, lo avisa. |
| **Datum Bogotá 1975 (antiguo)** | Se detecta, se aplica el cambio de datum estándar y se advierte. Se recomienda reproyectar la capa a MAGNA-SIRGAS. |
| **NPN en notación científica** (`7.6036E+29`) | Excel guardó el código como número y perdió dígitos; no se puede recuperar. Formatee la columna como **Texto** y vuelva a pegar los NPN. Se reporta como `NPN_NOTACION_CIENTIFICA`. |
| **NPN sin geometría** | Se reporta en incidencias (`NPN_SIN_GEOMETRIA`) y queda al final del itinerario, sin ORDEN y con el motivo. |
| **Archivo `.xls` antiguo** | Ábralo en Excel y use *Guardar como → Libro de Excel (.xlsx)*. |
| **El mapa base no se ve** | Cambie el mapa base (CARTO o Satélite) en la esquina superior derecha. Los predios y la ruta se dibujan igual. |
| **No se recuerda la configuración ni la clave** | En `file://` algunos navegadores aíslan el almacenamiento. Use el modo B. |

## 9. Incidencias (archivo `*_incidencias.xlsx`)

La hoja **INCIDENCIAS** tiene las columnas TIPO, SEVERIDAD, NPN, FILAS_EXCEL, DETALLE y RECOMENDACION. Los tipos son:

| Tipo | Qué significa |
|---|---|
| `CAPA_FALTANTE` | Falta la capa urbana o rural de un municipio que tiene predios en el Excel |
| `MUNICIPIO_INCONSISTENTE` | La columna Municipio no coincide con el código DANE de los NPN |
| `NPN_SIN_GEOMETRIA` | El NPN no aparece en las capas de su municipio |
| `NPN_DUPLICADO_EXCEL` | El NPN se repite en el Excel |
| `NPN_DUPLICADO_CAPA` | El NPN tiene varios polígonos en una capa |
| `NPN_EN_AMBAS_CAPAS` | El NPN está en `u_terreno` y en `r_terreno` |
| `GEOMETRIA_INVALIDA` | Polígono inválido o vacío |
| `GEOMETRIA_AUTOINTERSECCION` | El polígono se cruza a sí mismo |
| `NPN_INVALIDO` | El NPN no es un código válido |
| `NPN_NOTACION_CIENTIFICA` | Excel convirtió el NPN a notación científica |
| `NPN_RELLENADO` | El NPN tenía menos de 30 dígitos y se completó con ceros |
| `AMBITO_INCONSISTENTE` | La zona del NPN no coincide con la capa donde está |
| `SIN_ACCESO_VIAL` | El punto está a más de 350 m de una vía |
| `PUNTO_REUBICADO` | El punto de visita se movió a mano (acceso confirmado por el usuario) |
| `UBICACION_MANUAL` | Predio sin geometría ubicado a mano de forma aproximada |
| `SIN_VIA_CERCANA` | Punto fijado a mano a más de 2 km de cualquier vía |
| `TRAMO_A_PIE` | El vehículo llega hasta la vía más cercana; el resto es a pie |
| `ORDEN_MODIFICADO_MANUAL` | El usuario cambió el número de una o más paradas; incluye los cambios y su costo frente al óptimo |
| `CRITERIOS_DE_RUTA` | Criterios aplicados y su costo en tiempo frente a la ruta más corta |
| `EXCLUIDO_POR_USUARIO` | El usuario sacó el predio de la ruta |
| `JORNADA_EXCEDIDA` | La ruta termina después del fin de la jornada |
| `LLEGADA_FUERA_DE_JORNADA` | Un predio queda con llegada después del fin de la jornada |
| `RUTA_APROXIMADA` | El orden se calculó en línea recta |
| `OPTIMIZACION_LOCAL` | El orden se calculó en el navegador porque /optimization no respondió |
| `TRAZADO_NO_DISPONIBLE` | No se pudo descargar el trazado de la ruta sobre vías |

La hoja **RESUMEN** trae los totales de la jornada.

## 10. Créditos y licencias

- Ruta Catastral: licencia MIT. Todo el código es propio y está incluido en el `.html`.
- Métodos de referencia: área geodésica según el método de Turf.js (MIT), punto interior con el algoritmo *polylabel* (Mapbox, ISC) y series de Krüger para la Transversa de Mercator.
- Servicios externos (solo reciben coordenadas):
  - openrouteservice © HeiGIT gGmbH
  - OSRM (BSD-2)
  - Nominatim
  - Datos de mapa © colaboradores de OpenStreetMap (ODbL)
  - Teselas: OSM, © CARTO, © Esri
