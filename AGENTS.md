# AGENTS.md

Aplicación de cálculo de absorción acústica para importar en EASE. App web **estática de un solo archivo** (`index.html`): CSS, HTML y JS inline, sin build, sin dependencias, sin `package.json`.

## Arquitectura (no obvio)

- Todo vive en un único `<script>` dentro de `index.html`. `index.html` es el entrypoint y toda la app.
- Al cargar se ejecuta `renderEstudios()` y luego, al final del script, `buildLib(); buildManual(); buildPersona(); onModeSel(); onPersonType(); calcular();`
- **Dos conceptos de "persona" que NO son lo mismo** (no confundir):
  - `A_ocup` = área física por persona (m²), se usa solo para ocupación/densidad/capacidad (`% = N×A_ocup/S`).
  - `A_persona(f)` = absorción acústica por persona, tabla de 21 bandas, **escalada por el factor `k` de tipo de persona**.
- Bandas: `FREQ` = 21 tercios de octava (100 Hz–10 kHz); `BL` = etiquetas para mostrar; `NRC_IDX=[4,7,10,13]` = índices de α250/α500/α1k/α2k para el NRC.
- `expandOct(oct)` interpola 6 valores de octava (125–4k) a las 21 bandas con interpolación logarítmica. Materiales de `LIBR` con `full:true` ya traen las 21 bandas reales y no deben pasar por `expandOct`.
- Tipos de persona: `PERS_TYPES` (niño/joven/adulto/mixto) con `{sit, stand, k}` donde `k` multiplica `A_persona` en el cálculo.
- **Dos formateadores**, usarlos según el valor, no `toFixed` directo:
  - `fmt(v)` = 3 decimales, elimina ceros finales → para % , densidad, A_ocup, escenarios.
  - `fmt2(v)` = **siempre 2 decimales** → para TODOS los valores de absorción (α_base, α_eff, NRC, A_persona) en pantalla, PDF y export. La tabla editable de A_persona usa `.toFixed(2)`.
- La página es de **una sola columna**: las entradas se apilan arriba y los resultados aparecen debajo de la sección de cálculo.

## Reglas de cálculo (código crítico en `calcular()`)

- Parámetro primario `primarySel`: `pct` / `pers` / `dens`.
- N **derivado** de otro parámetro (% o densidad) se redondea con `Math.floor` para no superar la capacidad física; N **directo** (`pers`) con `Math.round`.
- Se rastrean `pctReq` (solicitado) y `pctReal` (real tras redondeo); ambos se muestran en resultados y PDF.
- Tope de densidad 6 pers/m²: en modo `dens` el campo es un `<select>` 1–6; en modo `pers`, error si `N > S×6`.
- `lastReport` es la fuente del PDF: si agregas algo a la salida de pantalla, también debes guardarlo en `lastReport` o no saldrá en `generarPDF()` (bug real ya corregido). El bloque `@media print` llena `#printArea` y el PDF se genera vía `window.print()`.
- La antigua sección "Copiar formato EASE" fue eliminada. Hoy el único export es `copiarExcel()` (botón "Copiar a Excel"), que arma un TSV desde `lastReport`: bloque de configuración + tabla de coeficientes (frecuencias × escenarios) + resumen por escenario (N, densidad, % real, NRC). Leer siempre `ro.aEff[i]` con `i` de las 21 bandas; **nunca** indexar `r.rows[i]` por índice de banda: `rows.length` = nº de escenarios, no 21 (bug real ya corregido).

## Estudios (guardar/cargar versiones)

- Persistencia vía `localStorage`, clave `ease_estudios` (array de `{id, name, date, state}`).
- `captureState()` serializa todos los inputs; `applyState(s)` los restaura y llama `calcular()`. El JSON de estado usa claves abreviadas: `fonte` = fuente (lib/manual) y `st` = estado de persona (sit/stand).
- **Gotchas de `applyState` (orden crítico, dos bugs reales ya corregidos)**:
  - Setear `dataset.touched="1"` en `scenVals` y `densSel` **antes** de llamar a `onPrimarySel()`/`onModeSel()`, o esos handlers reescriben los valores guardados con los defaults.
  - Restaurar `aOcupSit`/`aOcupStand` **después** de `onPersonType()`, porque ese handler los sobrescribe con el valor estándar del tipo y se perdía el `A_ocup` editado a mano (los resultados cargados no coincidían con los guardados).
- `renderEstudios()` se invoca al cargar; los botones Cargar/Eliminar llaman `cargarEstudio(id)` / `borrarEstudio(id)` con ids numéricos.

## Verificación (no hay framework de test)

Valida sintaxis JS extrayendo el `<script>` y compilando sin ejecutar:

```powershell
$html = Get-Content -Raw -LiteralPath "index.html"
$code = [regex]::Match($html, '(?s)<script>(.*)</script>').Groups[1].Value
$code | node -e "try{ new Function(require('fs').readFileSync(0,'utf8')); console.log('OK') }catch(e){ console.log('ERROR '+e.message) }"
```

**`jsdom` está instalado en `C:\Users\stico\AppData\Local\Temp\opencode`** (con `npm i jsdom`). Es la forma de probar la app **completa** (flujos de estudios, export, decimales, validaciones) sin navegador: cargar el HTML con `runScripts:"dangerously"`, y para `copiarExcel`/PDF hay que hacer stub de `navigator.clipboard` y `window.print`. La sintaxis-check no alcanza: bugs reales se detectan ejecutando (p. ej. `r.rows[i]` con menos escenarios que 21 bandas, o `onPersonType` sobrescribiendo `A_ocup` al cargar un estudio). Para la matemática pura sin DOM, prototipa en node suelto replicando `expandOct`, N, α_eff, NRC.

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