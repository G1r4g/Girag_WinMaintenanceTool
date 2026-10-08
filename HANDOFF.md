# HANDOFF — leer primero si continúas este proyecto (otra IA o persona)

## Objetivo
Herramienta Python para Soporte (Girag/GIT) que haga mantenimiento preventivo en Windows 10/11 con botones y progreso visible. Debe verse como `G1r4g/Girag_GLPI_MobileForms` (paleta en `girag_maint/ui/theme.py`, copiada de las variables `:root` de su `index.html`; el logo del repo es oscuro y no se ve sobre el fondo, por eso se usa el texto "GIRAG" blanco como hace su `.mark-fallback`).

## Estado (v0.3.0)
Hecho y verificado **solo en Linux**: UI (captura con Xvfb), parseo de progreso, planificación de bloatware, helpers de registro, flujo completo de limpieza sobre carpeta temporal (12 pruebas OK).
**NO verificado en Windows real** — todo lo que llama a `DISM`, `sfc`, `compact`, `PowerShell Appx`, `winreg`, `reg export`, `Checkpoint-Computer`, servicios `wuauserv/bits` y la elevación UAC.

## Primer paso recomendado: validar en una VM Windows 10 y otra Windows 11
1. `python main.py` → ¿pide UAC y abre? ¿Badge "Administrador" en verde?
2. Activar **Modo simulación** y ejecutar las 5 tareas (no deben modificar nada).
3. Sin simulación, en la VM: limpieza → comprobar que la barra avanza y los MB liberados son coherentes.
4. DISM: ¿la barra sigue el `[=== 62.3% ===]`? ¿decodifica bien tildes (códepage OEM)? ¿SFC muestra su % (sale en UTF‑16, se eliminan los nulos)?
5. Bloatware: revisar lista detectada antes de confirmar; comprobar que Tienda/Calculadora siguen.
6. Registro: revisar `logs\registro_problemas_*.txt`, ejecutar, y probar restaurar un `.reg` de `backups\`.
7. Compact OS en VM con SSD; verificar texto de `/query` en español.
8. winget: ejecutar «Actualizar aplicaciones» con 2-3 apps pendientes; verificar lista, selección, ★ de aprobadas, progreso y que solo se toquen las marcadas.
9. Lote: marcar 3 tareas y ejecutar; comprobar orden, barra de lote, que un fallo no detiene las demás y que «Cancelar todo» funciona.
10. GLPI: con el agente instalado, ejecutar la tarea y comprobar en GLPI (Administración → Inventario) que el equipo aparece/actualiza su última fecha de contacto.

## Decisiones de diseño
- **Tkinter + stdlib** (sin pip) para que corra en cualquier equipo de soporte; `ttk` tema `clam` para colorear la barra.
- Cada tarea corre en un hilo y habla con la UI por `queue` (`core/context.py`): `log/status/progress/confirm`. La UI nunca bloquea.
- `core/process.py::stream_process` lee bytes y separa por `\r`/`\n` (DISM/SFC redibujan el % con `\r`). Líneas con `%` → progreso; resto → log.
- Barra DISM global ponderada (5/25/40/25/5). RestoreHealth se queda quieto en ~20 % y ~62 %: es normal.
- **Seguridad**: toda acción destructiva pide confirmación (`ctx.confirm`); `ctx.dry_run` evita cambios (DISM/SFC no se ven afectados por la simulación); limpieza no sigue enlaces/junctions (`collect()`); registro solo borra rutas **absolutas, sin variables sin resolver, que no existen** (`definitely_missing`) y omite MSI/SystemComponent; si falla el respaldo `.reg`, no borra.
- Bloatware: `protect` gana siempre sobre `remove`; se omiten paquetes `NonRemovable`, framework y firma `System`.

## Cambios v0.2.0
- Bloatware: se retiraron de `remove` Office Hub, Sway y Clipchamp; `protect` ahora blinda explícitamente Calculadora, Fotos, ZuneMusic/ZuneVideo (Reproductor/Películas y TV), MediaPlayer, Clipchamp, extensiones de video/imagen y todo lo `*Office*`. Decisión de Soporte; hay pruebas que lo garantizan (`ProtectedAppsTests`). Apps de streaming de terceros (Netflix, Disney, Prime Video, Hulu) siguen en `remove`: quitar de la lista si también deben conservarse.
- Nueva tarea `tasks/glpi_sync.py` (tarjeta "Sincronizar con GLPI"): busca el agente, hace GET al servidor (cualquier respuesta HTTP = alcanzable), y ejecuta `glpi-agent.bat --force --server=<URL>` en primer plano mostrando su salida. Progreso = estimación (`SyncProgress`) porque el agente no emite %. Éxito = código 0 y ninguna línea `[error]`/`failed`/`can't connect`. Config en `config/glpi.json`.
- UI: columna de tareas desplazable (con rueda del mouse) para pantallas pequeñas.
- **Por validar en Windows:** ruta real del agente, que `--force --server=` envíe al plugin `glpiinventory`, los textos de log que disparan los hitos de progreso, y el quoting de la ruta con espacios al invocar el `.bat`. Alternativa si falla: pedir inventario al servicio local con `http://127.0.0.1:62354/now`.
- El repo de GitHub recibió solo los archivos de la raíz (faltaban `girag_maint/`, `config/`, `tests/`): verificar que se subieron completos.

## Cambios v0.3.0
- La app ya fue abierta por el usuario en Windows 11 (build 26300) como administrador: la UI y la elevación UAC funcionan. Las tareas siguen sin validación de extremo a extremo.
- **Ejecución en lote:** casilla por tarjeta (`CheckBox` propio ☐/☑ naranja) + botón «Ejecutar seleccionadas (N)». Lógica en `core/runner.py::run_tasks` (sin Tk, con pruebas): orden = orden de `ALL_TASKS`, continúa tras fallos, `Cancelar todo` detiene el resto. Eventos nuevos: `begin/end/done`. Cada tarea mantiene sus confirmaciones (decisión de seguridad). Idea pendiente: modo desatendido.
- **Nueva tarea `tasks/winget_upgrade.py`:** `parse_upgrade_table` interpreta `winget upgrade` por posición de columnas (EN/ES); diálogo `ChooseDialog` vía `ctx.choose()`; actualiza por `--id --exact --silent --accept-*-agreements --disable-interactivity` (+`--source` winget/msstore). Código 0x8A15002B = ya al día (no es error). Lista de aprobadas en `%ProgramData%\GiragMaintenance\winget_approved.json`.
- **Por validar en Windows:** (1) que la tabla de winget se lea bien con la salida redirigida y la codificación UTF-8; (2) IDs truncados con «…» (se muestran deshabilitados; si es frecuente, resolver por nombre o ensanchar la salida); (3) apps abiertas que impiden actualizar (error reportado por app); (4) comportamiento de winget cuando se ejecuta elevado (paquetes msstore/por usuario).
- La sincronización con GLPI quedó como tarea 7 (última del orden) para reportar el estado final del equipo.

## Limitaciones / riesgos conocidos
- Caché de Teams solo "clásico"; Teams nuevo no incluido.
- La papelera de reciclaje **no** se vacía (puede tener datos del usuario).
- Registro: no repara asociaciones de archivo, ActiveX/COM, fuentes, rutas de ayuda (sí hace Glary: pendiente).
- `Remove-AppxPackage -AllUsers` requiere PS 5.1 y admin; algunas apps pueden fallar y se reportan como fallidas.
- Compact OS en HDD puede ser contraproducente (advertido en el diálogo).
- Sin firma de código: SmartScreen/antivirus pueden avisar con el `.exe`.

## Roadmap sugerido
1. Validación en Windows real y corrección de lo que aparezca.
2. Botón **"Mantenimiento completo"** que encadene las 5 tareas con un resumen final.
3. Reporte exportable (HTML/PDF/CSV) por equipo; integración opcional con GLPI (crear ticket/seguimiento con el resultado) reutilizando la API del proyecto Girag_GLPI_MobileForms.
4. Opción de restaurar respaldos de registro desde la propia app.
5. Más cachés (Teams nuevo, Zoom, Slack, Adobe) y vaciado opcional de papelera.
6. Ícono propio, versión en el `.exe`, firma de código.
7. Registro en Windows Event Log / ejecución silenciosa por línea de comandos (`--task cleanup --silent`) para uso con GPO/Intune.

## Convenciones
- Código y textos de UI en español; comentarios cortos.
- Nueva tarea: heredar de `tasks/base.py::Task`, implementar `run(ctx)->TaskResult`, registrar en `tasks/__init__.py`.
- Probar lógica pura en `tests/` (no depender de Windows); usar `unittest`.
