# Casos de Estudio — instrucciones para Claude

Web comunitaria de casos de estudio de bolsa. Un único `index.html` (sin compilación) + Firestore
(`comunidadinversores-b9046`). Todo lo que hay que saber está en `README.md`.

- **En producción:** https://grubiogar.github.io/casos-estudio/ (GitHub Pages, rama `main`, raíz).
  **Push a `main` = publicar.** El repo es **público** (Pages gratis lo exige): nunca subas datos de
  miembros ni claves privadas. La config de Firebase del navegador es pública por diseño.
- Se queda en la cuenta `grubiogar` para no cambiar el enlace que usan los miembros; `gloriarubio` es
  colaboradora y trabaja con su sesión de siempre.
- Las copias de Firestore (`copias-seguridad/`) viven en OneDrive
  (`Documentos/ClaudeCode/Negocio/casosestudio/copias-seguridad`), no en git.

## Setup en máquina nueva

1. **Requisitos:** Git y GitHub CLI (`gh`, sesión `gloriarubio`). No hay secretos ni dependencias.
2. **Clonar** en la ruta de siempre (así coincide la memoria de Claude del proyecto):
   ```bash
   gh repo clone grubiogar/casos-estudio C:/dev/negocio/casos-estudio
   ```
3. **Probar en local:** abrir `index.html` en el navegador (o un servidor estático cualquiera).
