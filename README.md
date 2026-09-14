# Casos de Estudio — Estrategia de Inversión

Web comunitaria donde los miembros publican casos de estudio de operaciones
de bolsa, con su gráfico, su entrada, su stop y sus salidas.

- **En producción:** https://grubiogar.github.io/casos-estudio/
- **Repositorio:** `grubiogar/casos-estudio` (GitHub Pages, rama `main`, raíz)
- **Base de datos:** Firebase Firestore, proyecto `comunidadinversores-b9046`

Toda la aplicación es un único `index.html` sin dependencias ni compilación.
Firebase se carga por CDN en tiempo de ejecución.

---

## Cómo se guardan los datos

Firestore es la fuente de verdad. `localStorage` es **solo una caché de
arranque** para que la web pinte algo antes de que llegue la red: nunca debe
tratarse como almacenamiento real.

Hay dos colecciones, y están separadas a propósito:

| Colección | Qué guarda | Tamaño |
|---|---|---|
| `cases/{id}` | Metadatos del caso + `thumb` (miniatura) | ~19 KB por caso |
| `caseImages/{id}` | Solo `{ image }`, el gráfico completo | ~200 KB por caso |

**Por qué están separadas:** la lista escucha `cases` entera con un
`onSnapshot`, así que todo lo que viva ahí se descarga en cada carga de la
web. Con las imágenes dentro eran ~5 MB por visita. La imagen completa se
pide solo al abrir un caso concreto (`firestoreLoadImage`), y queda cacheada
en memoria en `imgCache`.

La miniatura sí viaja dentro del caso porque la rejilla de fichas la
necesita de inmediato. Se genera con `makeThumb()` y ronda los 15 KB.

### Reglas que conviene no romper

- **`persistLocal()` nunca debe lanzar una excepción.** Si `localStorage` se
  llena, degrada (suelta miniaturas, y si hace falta se desactiva), pero
  jamás corta el flujo de guardado. Esto causó una pérdida de datos real
  (ver más abajo).
- **Firestore se escribe ANTES que la caché local.** Un fallo de la caché no
  puede impedir que el dato llegue a la nube.
- **Las imágenes no se guardan en `localStorage`.** `persistLocal()` las
  quita siempre.
- **Nada borra documentos.** Ver la sección de seguridad.

---

## Seguridad (`firestore.rules`)

**Límite importante y consciente:** la app no tiene sistema de login. Sin
autenticación, las reglas **no pueden distinguir a un miembro de un
desconocido**: todos llegan igual de anónimos. El PIN de administrador
(`ADMIN_PIN`) vive en el código del navegador, así que tampoco es una
barrera real — cualquiera que abra el código fuente lo ve.

Por eso las reglas no intentan decidir *quién* escribe. Lo que hacen es
volver **imposibles las operaciones destructivas** para cualquiera:

- `allow delete: if false` en las dos colecciones. Nadie borra nada, nunca.
- `createdAt` es inmutable: no se puede reescribir la identidad de un caso.
- Validación de forma y topes de tamaño (imagen 600 KB, miniatura 60 KB,
  notas 8000, `status` dentro de un enum cerrado).
- El resto de colecciones, cerradas, para que nadie llene el proyecto.

Lo peor que puede hacer un atacante es **añadir casos basura**, que se
limpian. Lo que ya no puede es destruir los casos reales.

Si algún día se quiere seguridad de verdad, hace falta Firebase Auth y una
pantalla de acceso.

### Borrar de verdad

La app **no borra**: usa descarte (`status: 'discarded'`), que es reversible
y conserva el caso con su motivo y su fecha. Un administrador puede volver a
publicarlo. Si de verdad hay que eliminar algo, se hace a mano desde la
consola de Firebase.

---

## Despliegue

### La app

Push a `main`. GitHub Pages reconstruye solo (~40 s).

Aviso: el repo es de la cuenta **`grubiogar`**, pero el gestor de
credenciales de Windows suele tener la sesión de otra cuenta, y entonces el
push falla con `403 denied`. Se resuelve vaciando la lista de helpers para
ese push:

```bash
git -c credential.helper= -c credential.helper='!gh auth git-credential' push origin main
```

Si Pages se queda encolado sin avanzar (`updated_at` igual a `created_at`),
se fuerza con `gh api -X POST repos/grubiogar/casos-estudio/pages/builds`.

### Las reglas

**No se despliegan con el push.** Hay que pegarlas a mano en la consola:

https://console.firebase.google.com/project/comunidadinversores-b9046/firestore/rules

Requiere la cuenta de Google dueña del proyecto de Firebase, que no es la
misma que la de GitHub.

---

## Utilidades

`migrarImagenes()` — función global, se lanza desde la consola del
navegador. Mueve a `caseImages` las imágenes que todavía viajen dentro del
caso y les crea la miniatura. Escribe la copia aparte **antes** de quitarla
del caso, así que no puede perder nada a mitad. Es idempotente. Se ejecuta
sola tras cada importación.

---

## Historial de incidencias

### Pérdida silenciosa de casos (2 julio – 14 septiembre 2026)

Durante dos meses y medio, los casos nuevos **no se guardaban y no daban
ningún error**. La gente creía haber guardado, recargaba, y el caso no
estaba.

Causa: las imágenes se guardaban como base64 dentro de cada caso, y la
caché de `localStorage` llegó al 99% de su cuota (~5 MB). En `saveCase()`,
`persist()` lanzaba `QuotaExceededError` **antes** de llamar a
`firestoreSave()`, abortando la función. Como `toast()` y `render()`
tampoco llegaban a ejecutarse, el fallo era invisible.

Los casos de ese periodo **no se pudieron recuperar**: nunca llegaron a
escribirse en ningún sitio.

Arreglos: `persistLocal()` a prueba de fallos, Firestore antes que la caché,
errores visibles, y un tope real de 180 KB por imagen (antes un comentario
decía "<900 KB" pero no se comprobaban bytes, solo se redimensionaba).

### Base de datos abierta (hasta el 14 septiembre 2026)

Las reglas permitían lectura y escritura a cualquiera. Con la API key que
está a la vista en el código fuente se podía vaciar la colección entera. Se
verificó escribiendo y borrando un documento de forma anónima.

Resuelto con `firestore.rules` y sustituyendo el borrado por descarte.

### 5 MB por visita (hasta el 14 septiembre 2026)

La lista se descargaba todas las imágenes en cada carga. Resuelto separando
`caseImages` y dejando solo miniaturas en el caso: de ~5.083 KB a 483 KB
(10,5x menos). Efecto secundario valioso: la caché local bajó de 5,19 MB a
404 KB, con lo que la cuota que causó la pérdida de datos queda con mucho
margen.

---

## Copias de seguridad

En `copias-seguridad/` (fuera del repositorio, en OneDrive):

- `cases-completo-2026-09-14.jsonl` — los 25 casos con sus imágenes
  originales, en formato REST de Firestore, tal y como estaban antes de la
  migración. Restaurable con `PATCH /cases/<id>` enviando `{"fields": ...}`.
- `index-antes-de-arreglos-2026-09-14.html` — la app antes de los arreglos.

Para exportar desde la propia web, el botón **Exportar** descarga un JSON
con las imágenes completas incluidas.
