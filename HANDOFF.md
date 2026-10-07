# HANDOFF — leer primero si continúas este proyecto (otra IA o persona)

## Objetivo
Herramienta Python para Soporte (Girag/GIT) que haga mantenimiento preventivo en Windows 10/11 con botones y progreso visible. Debe verse como `G1r4g/Girag_GLPI_MobileForms` (paleta en `girag_maint/ui/theme.py`, copiada de las variables `:root` de su `index.html`; el logo del repo es oscuro y no se ve sobre el fondo, por eso se usa el texto "GIRAG" blanco como hace su `.mark-fallback`).

## Estado (v0.1.0)
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

## Decisiones de diseño
- **Tkinter + stdlib** (sin pip) para que corra en cualquier equipo de soporte; `ttk` tema `clam` para colorear la barra.
- Cada tarea corre en un hilo y habla con la UI por `queue` (`core/context.py`): `log/status/progress/confirm`. La UI nunca bloquea.
- `core/process.py::stream_process` lee bytes y separa por `\r`/`\n` (DISM/SFC redibujan el % con `\r`). Líneas con `%` → progreso; resto → log.
- Barra DISM global ponderada (5/25/40/25/5). RestoreHealth se queda quieto en ~20 % y ~62 %: es normal.
- **Seguridad**: toda acción destructiva pide confirmación (`ctx.confirm`); `ctx.dry_run` evita cambios (DISM/SFC no se ven afectados por la simulación); limpieza no sigue enlaces/junctions (`collect()`); registro solo borra rutas **absolutas, sin variables sin resolver, que no existen** (`definitely_missing`) y omite MSI/SystemComponent; si falla el respaldo `.reg`, no borra.
- Bloatware: `protect` gana siempre sobre `remove`; se omiten paquetes `NonRemovable`, framework y firma `System`.

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
