# NOTES.md — worldcup2026sim

Proyecto archivado en agosto de 2026, después del Mundial.
Notas para el Andrés del futuro (o para quien quiera hacer un simulador de otra
competencia partiendo de aquí). El código dice *qué* hace; esto dice *cómo* y
*qué está mal*.

---

## 1. Qué era

Simulador interactivo del Mundial 2026. Tres funciones:

1. **Simular** el torneo completo (botón "Simulate All") o fase por fase
2. **Elegir resultados a mano**, partido por partido, con marcador editable
3. **Coronar al campeón** con overlay, confeti y botón de compartir

Además cargaba **resultados reales** del torneo desde un feed público, así que
durante junio y julio funcionaba como bracket vivo, no solo como juguete.

- **En línea:** worldcupsim2026.com — de mayo a agosto de 2026
- **Alojado en:** Cloudflare Workers, desplegado desde este repo
- **Monetización:** Google AdSense — **nunca aprobado** (ver sección 6)
- **Analítica:** Google Analytics 4 (ID en el `<head>` de `index.html`)

---

## 2. Cómo levantarlo

Sitio estático puro. **No hay build, no hay `package.json`, no hay dependencias
instalables.** Abres `index.html` en el navegador y funciona.

Para publicarlo: conectar el repo a Cloudflare Pages, o arrastrar la carpeta.
Nada más.

Archivos: `index.html` es el simulador completo (HTML + CSS + JS en un solo
archivo). El resto son páginas de apoyo: `about`, `blog`, `contact`, `privacy`,
`terms`, `cookies`, más `robots.txt`, `sitemap.xml` y `ads.txt`.

**Dependencias externas por CDN** (si alguna muere, el sitio se degrada):
Google Fonts (Playfair Display + DM Sans), AdSense, GA4, y el feed de
openfootball en `raw.githubusercontent.com`.

---

## 3. Cómo funciona por dentro

Todo el JS está embebido en `index.html`, en el `<script>` grande antes del
`<footer>`.

### Estado
Cuatro objetos globales en memoria:
- `res` — resultados por número de partido: `{hg, ag, pen, manual, real}`
- `tab` — pestaña activa
- `pendPen` — empates de eliminatoria esperando que elijas ganador en penales
- `realKO` — equipos reales que llegaron a cada llave, cuando difieren del sim

**No hay persistencia.** Si recargas la página pierdes todo. `localStorage` solo
se usa para el banner de cookies. Ver sección 5.

### Datos del torneo (hardcodeados)
- `FC` — emoji de bandera por país
- `T` — rating de cada selección, número del 48 al 88, **puesto a mano y a ojo**
  (España y Argentina 88; Curaçao 48)
- `G` — los 12 grupos de 4
- `GM` — los 72 partidos de grupos con fecha, hora, estadio, ciudad y país sede
- `R32`, `R16`, `QF`, `SF`, `TH`, `FI` — los 32 partidos de eliminatoria

### El motor de simulación — `sim(h, a, ko)`
No es Poisson ni Elo. Es una transformación lineal del rating más ruido uniforme:

```
hS = rating_local + random*15 + 5      // el +5 es ventaja de local
aS = rating_visitante + random*15
goles = max(0, round(score/30 - 1 + (random - 0.3) * 2))
```

Un equipo de 60 produce ~1 gol base; uno de 88, ~1.9. El ruido va de −0.6 a
+1.4, o sea **sesgado hacia arriba** (mete más goles de los que quita). El
resultado: marcadores casi siempre entre 0 y 4, y **muy pocas sorpresas** — la
varianza es demasiado baja para que un equipo débil gane a uno fuerte con la
frecuencia con que pasa en la realidad.

Si es eliminatoria y hay empate, se van a penales: `Math.random() > .5`.
**Un volado. El rating no influye.**

### Tabla de grupos — `grpSt(g)`
Puntos → diferencia de goles → goles a favor. **No implementa el desempate por
enfrentamiento directo**, que es el criterio que FIFA usa antes que goles a
favor. Simplificación consciente.

### Los ocho mejores terceros — `allThirds()`
Toma los 12 terceros de grupo que ya jugaron sus 3 partidos, los ordena por
puntos/DG/GF y corta los primeros 8.

### El cruce de dieciseisavos — `r32T(n)`
Aquí está la trampa grande y hay que saberla. En la realidad, **el cruce de
dieciseisavos no es fijo**: FIFA tiene una tabla de asignación que depende de
*qué grupos* aportaron los ocho terceros, y hay decenas de combinaciones
posibles.

Este simulador **no implementa esa tabla**. Asigna los terceros por orden de
clasificación a las llaves 74, 77, 79, 80, 81, 82, 85 y 87 en secuencia fija.
Es una aproximación razonable para un juguete, pero es incorrecta contra el
reglamento real.

### El bracket — `PAIR` y `DEPS`
`PAIR` define qué dos partidos alimentan cada llave siguiente (89 sale de 74 y
77, etc.). `koPair(n)` y `W(n)`/`Ls(n)` recorren esa estructura recursivamente
para saber quién juega y quién ganó.

`DEPS` es el mapa inverso, y `clearDown(n)` lo usa para **invalidar en cascada**:
si editas un partido de octavos, borra automáticamente cuartos, semis y final,
porque dejaron de ser válidos. Esta es la parte mejor resuelta del proyecto y
la que vale la pena copiar tal cual.

### Carga de resultados reales — `loadReal()`
Hace `fetch` al JSON de **openfootball/worldcup.json** en GitHub. Sin API key,
sin límite de cuota. Lo interesante son los tres problemas que hubo que
resolver:

1. **Nombres distintos.** El feed dice "Czech Republic", el sim dice "Czechia".
   Se resuelve con `NAMEMAP` más `normT()`, que quita acentos y puntuación para
   que "Türkiye" y "Turkiye" empaten.
2. **Orientación local/visitante.** El feed puede listar los equipos al revés
   que mi calendario. Los partidos de grupo se emparejan por el *par* de
   equipos sin orden, y luego se reorienta el marcador a mi `GM`.
3. **Eliminatoria.** Ahí no se puede emparejar por nombre, porque los equipos
   cambian. Se empareja por número de partido (73–104), y `realKO` sobrescribe
   quién jugó realmente esa llave.

También lee `score.p` (penales), `score.et` (prórroga) y `score.ft` en ese
orden de prioridad, e ignora placeholders tipo "W74" cuando el partido todavía
no tiene equipos definidos.

**Guard de bracket:** compara las referencias `W##` del feed contra mi objeto
`PAIR` y lanza un `console.warn('BRACKET MISMATCH')` si la topología oficial no
coincide con la mía.

> **Este guard atrapó un bug real en producción.** Ver el commit
> *"Fix swapped R16/QF bracket feeders in PAIR and DEPS"* (julio 2026): los
> alimentadores de octavos y cuartos estaban invertidos. Sin la validación
> automática habría implicado revisar 32 llaves a mano, o peor, nunca
> enterarse. Es el mejor argumento a favor de escribir validaciones baratas
> contra una fuente externa de verdad.

---

## 4. Lo que sí quedó bien

- **La invalidación en cascada** (`DEPS` + `clearDown`). Editar un resultado a
  media eliminatoria y que el bracket se limpie solo es lo que hace usable el
  modo manual.
- **La integración con openfootball.** Feed gratis, sin key, sin cuota, y el
  emparejamiento normalizado aguantó todo el torneo.
- **El guard de bracket.** Validación barata que atrapó un error caro.
- **Todo el andamiaje legal y de SEO.** Canonical, Open Graph, JSON-LD de
  `WebApplication`, sitemap, robots, `ads.txt`, páginas de privacy/terms/
  cookies, y el disclaimer de no afiliación con FIFA en el footer. Eso son
  varias horas de trabajo aburrido que se copian tal cual a cualquier proyecto
  parecido.

## 5. Lo que está mal (leer antes de reutilizar)

1. **No hay persistencia.** Recargas y pierdes las 104 predicciones. Es el peor
   defecto de UX del proyecto y la solución es trivial: serializar `res` a
   `localStorage`, que ya se usa para el consentimiento de cookies. **Si haces
   otro simulador, esto primero.**
2. **Ventaja de local en eliminatoria.** El `+5` se aplica a `m.h`, que en las
   llaves no es un local real sino la posición en el bracket. Sesgo gratuito.
3. **Penales por volado.** Debería pesar por rating, o al menos por quién
   dominó el partido.
4. **Sin desempate por enfrentamiento directo** en la tabla de grupos.
5. **Cruce de terceros simplificado** (ver sección 3). Es el error más
   "incorrecto" del simulador frente al reglamento.
6. **Ratings a ojo.** Vale más jalar Elo público (eloratings.net) o el ranking
   FIFA que inventar 48 números.
7. **Varianza demasiado baja.** El modelo casi nunca produce sorpresas, que es
   justo lo divertido de un Mundial. Un Poisson con lambda derivada de la
   diferencia de ratings daría resultados mucho más creíbles.
8. **El banner de cookies es cosmético.** Los scripts de AdSense y GA4 se
   cargan en el `<head>`, *antes* de que el usuario acepte. Rechazar solo apaga
   GA después del hecho. Si vuelves a monetizar con tráfico europeo, esto hay
   que rehacerlo: cargar los scripts solo tras el consentimiento.
9. **Todo en un archivo.** ~700 líneas de HTML, CSS y JS mezclados. Funcionó
   para el alcance que tenía, pero no escala.

---

## 6. Resultados reales del proyecto

### Tráfico (25 may – 22 ago 2026)

- **400 usuarios activos**, ~1,900 eventos
- **Canales:** Direct 257 · Organic Social 179 · Organic Search 91 ·
  Referral 8 · Unassigned 1
- **Países:** US 100 · MX 85 · CO 21 · AR 17 · RU 17 · DE 14 · PE 12
- **Páginas:** home 617 vistas. Todo el resto del sitio (about, blog, privacy,
  terms, cookies, contact) sumó menos de 15 vistas.
- **Forma de la curva:** un pico de ~230 usuarios en un solo día a finales de
  mayo, y después casi nada durante junio y julio — justo cuando se jugaba el
  Mundial.

### Monetización

**AdSense nunca aprobó el sitio.** Rechazo por *"contenido de bajo valor"* el
12 de abril de 2026, es decir *antes* de que empezara el torneo. El `ads.txt`
figuraba como "No encontrado". **Ingresos totales: $0.**

### Las tres lecciones

1. **El tráfico fue social, no de búsqueda.** Direct + Organic Social =
   436 sesiones contra 91 de Organic Search. Todo el andamiaje de SEO
   (canonical, JSON-LD, sitemap) apuntó al canal que menos aportó. Para un
   producto de temporada es lógico: competir por "world cup 2026 simulator"
   contra FIFA y ESPN no es viable. **La palanca real era que el usuario
   compartiera su bracket** — y el botón de compartir ya existía, pero solo
   generaba texto. Con una imagen generada del bracket habría sido otra cosa.

2. **La falta de persistencia rompió el ciclo justo donde debía cerrarse.**
   Llegaba gente por un link compartido, llenaba predicciones, y al recargar lo
   perdía todo. Ninguna razón para volver. Eso explica el pico único seguido de
   silencio, mejor que cualquier teoría sobre el contenido.

3. **AdSense rechaza herramientas interactivas sin contenido editorial real.**
   Las páginas de about/blog/privacy existían justo para pasar la revisión, y
   no bastaron. Si el plan de monetización depende de AdSense, hay que resolver
   la aprobación **meses antes** del evento y con contenido escrito de verdad —
   o elegir otro modelo desde el principio.

**Diagnóstico corto:** ninguna de las tres fallas fue técnica. Todas fueron de
producto y de secuencia.

---

## 7. Referencia: formato del Mundial

**2030 usa el mismo formato de 48 equipos que 2026**, así que la estructura de
este proyecto sigue vigente:

- 48 selecciones en 12 grupos de 4, tres partidos por equipo
- Clasifican los 2 primeros de cada grupo (24) más los 8 mejores terceros = 32
- Dieciseisavos → octavos → cuartos → semis → final. **104 partidos**
- El cruce de dieciseisavos **no es un bracket fijo** (ver sección 3)

**Antes de reutilizar nada en 2030: confirma que FIFA no cambió el formato.**
No lo asumas. Y considera que 2030 se reparte entre varios países y
continentes, así que la parte de sedes, husos horarios y horarios de partido
hay que rehacerla completa.

---

## 8. Si haces otro simulador

Sirve para Liga MX, Champions, Copa Oro, lo que sea. Ruta corta:

**Copiar tal cual:** la estructura `PAIR`/`DEPS` para el bracket, `loadReal()`
adaptado a otro feed, el guard de validación contra la fuente externa, las
páginas legales y el CSS (es independiente del deporte).

**Rehacer:** el motor `sim()` — con Poisson y ratings reales esta vez. Y meter
persistencia desde el primer commit.

**Cambiar de raíz:** `G`, `GM`, `T`, `FC` y toda la estructura de fases. Están
hardcodeadas para 48 equipos y 12 grupos. Una liga con liguilla no se parece en
nada.

**Priorizar distinto:** persistencia primero, flujo de compartir segundo,
monetización resuelta *antes* de construir, SEO al final.

**No olvidar:** los IDs de AdSense y Analytics están en el `<head>` de
`index.html`. Reemplázalos o quítalos según el proyecto nuevo.

---

Dominio vence 	Apr 1, 2027

*Archivado por Andrés Rosas — agosto 2026*
