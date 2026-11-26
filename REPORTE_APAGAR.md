# Reporte: `DESPERTAR/APAGAR` no detiene `DORMIR` / `entrenar_lora.py` entre sesiones MSYS2/Windows

## Problema detectado (2026-11-25)
Al ejecutar [DESPERTAR/APAGAR](file:///d:/PROYECTOS/modelo3/IAUTONOMA/DESPERTAR/APAGAR), el script informó `IAUTONOMA apagada con seguridad`, pero quedaron activos:
- `bash.exe` (`DESPERTAR/DORMIR`), `nice.exe` y `python.exe` (`entrenar_lora.py`).
- `llama-server.exe` en el puerto `8099` (servidor de pruebas de `CLUSTER/`).

## Causa
1. `dormir.pid` guarda `$$` (PID POSIX de MSYS2/Cygwin), que no es visible con `kill -0` desde otro runtime Bash (p. ej. Git Bash vs MSYS2) y tampoco cierra el subárbol de `python.exe` en Windows si muere el padre.
2. `APAGAR` borra `$SENAL_DESPERTAR` tras esperar `10 s` (`50 * 0.2`), antes de que `entrenar_lora.py` termine un paso lento en CPU para leer el stop-file.

## Propuesta de solución
1. En `DESPERTAR/DORMIR`, guardar también el PID real de Windows (o localizar `DORMIR` / `entrenar_lora.py` por línea de comandos al apagar).
2. En `DESPERTAR/APAGAR`, si `kill` POSIX no detiene el proceso tras esperar el stop-file, rematar por árbol de procesos (`taskkill /T /F` en Windows o `pkill -f` en Linux) sobre `DORMIR` y `entrenar_lora.py` que apunten a `$DIR`.
