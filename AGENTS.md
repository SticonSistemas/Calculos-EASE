# AGENTS.md

Aplicación de cálculo de absorción acústica para importar en EASE. App web **estática de un solo archivo** (`index.html`): CSS, HTML y JS inline, sin build, sin dependencias, sin `package.json`.

## Arquitectura (no obvio)

- Todo vive en un único `<script>` dentro de `index.html`. `index.html` es el entrypoint y toda la app.
- En init se ejecutan en orden: `buildLib(); buildManual(); buildPersona(); onModeSel(); onPersonType(); calcular();`
- **Dos conceptos de "persona" que NO son lo mismo** (no confundir):
  - `A_ocup` = área física por persona (m²), se usa solo para ocupación/densidad/capacidad (`% = N×A_ocup/S`).
  - `A_persona(f)` = absorción acústica por persona, tabla de 21 bandas, **escalada por el factor `k` de tipo de persona**.
- Bandas: `FREQ` = 21 tercios de octava (100 Hz–10 kHz); `BL` = etiquetas para mostrar; `NRC_IDX=[4,7,10,13]` = índices de α250/α500/α1k/α2k para el NRC.
- `expandOct(oct)` interpola 6 valores de octava (125–4k) a las 21 bandas con interpolación logarítmica. Materiales de `LIBR` con `full:true` ya traen las 21 bandas reales y no deben pasar por `expandOct`.
- Tipos de persona: `PERS_TYPES` (niño/joven/adulto/mixto) con `{sit, stand, k}` donde `k` multiplica `A_persona` en el cálculo.
- `fmt(v)` es el formateador canónico (3 decimales, elimina ceros finales). Usarlo en lugar de `toFixed` directo.

## Reglas de cálculo (código crítico en `calcular()`)

- Parámetro primario `primarySel`: `pct` / `pers` / `dens`.
- N **derivado** de otro parámetro (% o densidad) se redondea con `Math.floor` para no superar la capacidad física; N **directo** (`pers`) con `Math.round`.
- Se rastrean `pctReq` (solicitado) y `pctReal` (real tras redondeo); ambos se muestran en resultados y PDF.
- Tope de densidad 6 pers/m²: en modo `dens` el campo es un `<select>` 1–6; en modo `pers`, error si `N > S×6`.
- `lastReport` es la fuente del PDF: si agregas algo a la salida de pantalla, también debes guardarlo en `lastReport` o no saldrá en `generarPDF()` (bug real ya corregido). El bloque `@media print` llena `#printArea` y el PDF se genera vía `window.print()`.

## Verificación (no hay framework de test)

Valida sintaxis JS extrayendo el `<script>` y compilando sin ejecutar:

```powershell
$html = Get-Content -Raw -LiteralPath "index.html"
$code = [regex]::Match($html, '(?s)<script>(.*)</script>').Groups[1].Value
$code | node -e "try{ new Function(require('fs').readFileSync(0,'utf8')); console.log('OK') }catch(e){ console.log('ERROR '+e.message) }"
```

Para la matemática, prototipa en un script `node` suelto en `C:\Users\stico\AppData\Local\Temp\opencode` (replica `expandOct`, N, α_eff, NRC) en lugar de depender del DOM.

## Git / despliegue

- El repo git de referencia es **este** directorio (`Calculos-EASE`), NO el repo padre `C:\Users\stico` (que también es un repo git). Opera git aquí; en el padre se ve un aviso de "embedded repository".
- GitHub Pages publica automáticamente desde `main` (el HTML es estático; el archivo debe seguir llamándose `index.html` para servirse en la raíz). El build tarda ~1 min.
- Flujo de commit+push (el repo usa identidad `SticonSistemas`; se fija con `-c` para no tocar la config global):

  ```powershell
  git add -A; git -c user.name="SticonSistemas" -c user.email="sticonsistemas@users.noreply.github.com" commit -m "mensaje" && git push origin main
  ```

- No hay release/CI. Convención de mensajes: `feat:` / `fix:` en español (ver `git log --oneline -20`).

## Convenciones de UI

- Tema oscuro vía variables CSS (`--bg`, `--panel`, etc.). Mensajes en `#outmsg`; los avisos usan `var(--err)`.
- `onModeSel` deshabilita la opción de densidad en modo "Butacas". `onPrimarySel` alterna entre el campo de texto de la lista de valores y el selector de densidad.