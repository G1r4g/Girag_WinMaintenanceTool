# Girag_WinMaintenanceTool

Aplicación de escritorio en **Python (Tkinter, sin dependencias externas)** para que el equipo de Soporte de Girag realice **mantenimiento preventivo en equipos con Windows 10 y 11**. Interfaz y colores tomados de [Girag_GLPI_MobileForms](https://github.com/G1r4g/Girag_GLPI_MobileForms) (fondo `#0c1526`, tarjetas `#141f36`, acento naranja `#e8823f`).

> **Estado:** v0.1.0 — prototipo funcional. Lógica y UI probadas en Linux (pruebas unitarias + captura con Xvfb). **Aún NO probado en un Windows real**: ver `HANDOFF.md` antes de usarlo en producción.

## Funciones

| # | Botón | Qué hace |
|---|-------|----------|
| 1 | **Limpiar caché y temporales** | Escanea y borra temporales de Windows y de cada perfil de usuario, descargas de Windows Update, miniaturas, informes de error (WER), volcados y caché de Chrome, Edge, Firefox, Teams (clásico) y Office. Barra de progreso por archivos y total liberado. |
| 2 | **Reparar Windows (DISM + SFC)** | `DISM /CheckHealth` → `/ScanHealth` → `/RestoreHealth` → `sfc /scannow` → `/StartComponentCleanup`. Barra global ponderada y resultado por paso. |
| 3 | **Eliminar bloatware** | Quita apps Appx preinstaladas (lista editable en `config/bloatware.json`) para todos los usuarios y del perfil base. Lista `protect` (Tienda, Calculadora, Fotos, Terminal, Office…) siempre se respeta. |
| 4 | **Reparar registro** | Estilo Glary Utilities: detecta entradas huérfanas (programas desinstalados, App Paths, SharedDLLs, inicio, MUICache). Antes de borrar crea **punto de restauración** y **respaldo .reg**; si el respaldo falla, no borra nada. |
| 5 | **Compact OS** | `compact /CompactOS:always` con estado antes/después y espacio recuperado. |

Extras: **modo simulación** (no modifica nada), confirmación antes de cualquier cambio, botón **Cancelar**, tiempo transcurrido y log en pantalla + archivo.

## Requisitos
- Windows 10/11, **Python 3.10+** (Tkinter viene incluido en el instalador oficial).
- Ejecutar como **administrador** (la app lo solicita sola por UAC).

## Ejecutar
```bat
python main.py
```
Para desarrollar sin elevar: `python main.py --no-elevate`.

## Generar el .exe
```bat
build.bat
```
Crea `dist\GiragMantenimiento.exe` (PyInstaller, pide administrador al abrir).

## Pruebas
```bash
python -m unittest discover -s tests
```
(Corren en cualquier sistema; cubren parseo de progreso, planificador de bloatware, ayudantes del registro y el flujo de limpieza.)

## Dónde guarda cosas
- Logs: `C:\ProgramData\GiragMaintenance\logs\`
- Respaldos del registro: `C:\ProgramData\GiragMaintenance\backups\registro_<fecha>\` (doble clic en un `.reg` para restaurar).

## Estructura
```
main.py                      # arranque + elevación UAC
config/bloatware.json        # lista editable remove / protect / optional
girag_maint/
  core/   system.py context.py process.py   # admin, rutas, cola de eventos, ejecución de comandos
  tasks/  cleanup.py dism_repair.py bloatware.py registry_repair.py compact_os.py
  ui/     theme.py app.py                    # paleta Girag + ventana Tkinter
tests/test_core.py
HANDOFF.md                   # estado, decisiones y pendientes (leer primero si continúas el trabajo)
```
Para añadir una tarea: crear clase que herede de `tasks/base.py::Task`, implementar `run(ctx)` y registrarla en `tasks/__init__.py`. La UI genera la tarjeta sola.

## Subir a GitHub
```bash
git remote add origin https://github.com/G1r4g/Girag_WinMaintenanceTool.git
git push -u origin main
```
