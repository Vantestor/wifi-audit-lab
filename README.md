
# Laboratorio de Auditoría Wi-Fi

Este repositorio documenta mi proceso de aprendizaje en ciberseguridad, realizando un pentesting a mi propia red doméstica.

## 🎯 Objetivos del laboratorio
*   Capturar un Handshake WPA2 (Raspberry Pi / Debian).
*   Convertir el hash al formato de Hashcat.
*   Realizar un ataque de diccionario (`rockyou.txt`) usando GPU (Windows 11).
*   Documentar errores (RF-kill, CUDA, reglas de Hashcat) y sus soluciones.
*   Aplicar lecciones de Blue Team (defensa y contraseñas robustas).




📘 Manual de Laboratorio: Auditoría WPA2 y Ataque de Diccionario (Red Team vs Blue Team)
Entorno:

Atacante/Captura: Raspberry Pi (Linux) + Tarjeta Atheros AR9285 (Soporta 2.4GHz).

Fuerza Bruta: Windows 11 + GPU NVIDIA GTX 1050 + Hashcat.

Objetivo: Red doméstica propia (SSID: Familia Menjivar Castro).

🛑 FASE 1: Preparación y Captura (En la Raspberry Pi)
El objetivo de esta fase es capturar el "Handshake" (el apretón de manos donde el router y el móvil intercambian las claves de cifrado).

Evitar interferencias: Matar los procesos que intentan gestionar la tarjeta de red (NetworkManager, wpa_supplicant) para que no interfieran.

sudo airmon-ng check kill

Activar Modo Monitor: Poner la tarjeta en modo "escucha" para capturar todo el tráfico aéreo.

sudo airmon-ng start wlan0

Verificar: iwconfig (Debe decir wlan0mon y Mode:Monitor).

Reconocimiento: Escanear las redes cercanas para identificar el objetivo.

sudo airodump-ng wlan0mon

Anotar: El BSSID (MAC del router) y el CH (Canal) de la red objetivo.

Captura enfocada: Fijar la tarjeta en el canal del objetivo y guardar la captura en un archivo.

sudo airodump-ng -c [CANAL] --bssid [BSSID] -w captura_casa wlan0mon

Forzar el Handshake (El Truco): Mientras el comando anterior corre, apagar y encender el Wi-Fi del teléfono móvil. Esto obliga al móvil a reconectarse y genera el tráfico del Handshake.

Esperar a ver en la esquina superior derecha: WPA handshake: [BSSID].

Presionar Ctrl + C para detener y guardar.

Limpieza (Dejar la Raspberry como estaba):

sudo airmon-ng stop wlan0mon

sudo systemctl restart NetworkManager

⚙️ FASE 2: Procesamiento y Fuerza Bruta (En Windows 11)
El objetivo es tomar el archivo de captura y probar millones de contraseñas usando la tarjeta gráfica (GPU).

Conversión de formato: Hashcat no lee el formato .cap nativo de Linux. En la Raspberry, convertir el archivo:

hcxpcapngtool -o captura_casa.hc22000 captura_casa-01.cap

(Pasar el archivo .hc22000 a Windows usando scp o un servidor web temporal).

Preparar el entorno en Windows:

Descargar Hashcat (versión binaria para Windows) y descomprimir.

Descargar el diccionario rockyou.txt (14 millones de contraseñas).

Ataque de Diccionario Estándar:

.\hashcat.exe -m 22000 -a 0 captura_casa.hc22000 rockyou.txt

Resultado: Si la contraseña es común, aparecerá en verde (Cracked). Si no, dirá Exhausted.

Ataque con Reglas (El poder real): Usar reglas para modificar las palabras del diccionario (ej: poner mayúsculas, añadir números).

Crear un archivo mi_dic.txt con la palabra base ej: [CONTRASEÑA_DE_LABORATORIO].

.\hashcat.exe -m 22000 -a 0 captura_casa.hc22000 mi_dic.txt -j c (El -j c pone la primera letra en mayúscula).

🧠 LOS "CONTRAPIES" (Errores comunes y cómo solucionarlos)
Estos son los problemas reales que enfrentamos en el laboratorio y que fortalecen el conocimiento:

Requested device "wlan0" does not exist

Causa: Al apagar la radio con rfkill, el sistema renombró la interfaz.

Solución: Usar iwconfig o ip a para ver el nombre real (a veces es wlan1 o wlp2s0).

ioctl(SIOCSIFFLAGS) failed: Operation not possible due to RF-kill

Causa: El chip Wi-Fi está bloqueado por hardware/software.

Solución: sudo rfkill unblock all.

You are trying to stop a device that isn't in monitor mode

Causa: Intentar apagar wlan0 cuando el modo monitor se aplicó a wlan0mon.

Solución: Es solo una advertencia. Para limpiar de verdad, usar sudo iw wlan0mon del o reiniciar el servicio de red.

Avahi-daemon sigue apareciendo en airmon-ng

Causa: Es un servicio de descubrimiento de red (impresoras, Chromecast).

Solución: Es inofensivo para el modo monitor. Se puede ignorar.

Hashcat: Failed to initialize NVIDIA RTC library / CUDA SDK Toolkit not installed

Causa: Faltan las librerías de desarrollo de NVIDIA (CUDA Toolkit).

Solución: Hashcat hace un "Fallback" automático a OpenCL. La velocidad baja un poco, pero funciona perfectamente. Si se quiere máximo rendimiento, instalar el CUDA Toolkit de NVIDIA.

Hashcat: rules\best64.rule: No such file or directory

Causa: Error de tipeo (cortar la extensión) o el archivo no venía en esa versión.

Solución: Revisar la carpeta rules con ls rules y usar el nombre exacto, o usar reglas en línea de comandos con -j.

🛡️ BUENAS PRÁCTICAS Y LECCIÓN DE BLUE TEAM
¿Hay buenas prácticas? Absolutamente, y son el núcleo de la ciberseguridad defensiva.

1. Para el Atacante (Red Team)
Ética y Legalidad: NUNCA auditar redes ajenas sin permiso por escrito. El laboratorio es en casa propia.

Captura en campo, procesamiento en casa: No lleves equipo pesado a la casa del cliente. Captura con la Raspberry y procesa en tu PC con GPU.

Ahorro de tiempo: No uses el rockyou.txt a ciegas en una Raspberry. Usa reglas y diccionarios personalizados (OSINT) para atacar primero las contraseñas probables.

2. Para el Defensor (Blue Team)
Contraseñas robustas: Quedó demostrado que Pentateuco102222 cae en milisegundos con una regla (-j c). Una buena contraseña debe ser larga (más de 15 caracteres) y sin patrones predecibles (ej: Gato-Volador-88#Casa).

WPA3: Si el router lo soporta, migrar de WPA2 a WPA3. WPA3 (SAE) elimina el ataque de diccionario offline (el handshake capturado no se puede procesar de esta forma).

Deshabilitar WPS: El botón WPS es un agujero de seguridad enorme que permite saltarse la contraseña. Desactívalo en el router.

Filtrado MAC: Como viste en casa de tu amigo, da una falsa sensación de seguridad. Es fácilmente evadible clonando una MAC. No es una defensa real, solo un estorbo.

Segmentación: Si tienes invitados o dispositivos IoT (bombillas, cámaras), ponlos en una VLAN de invitados separada de tus computadoras principales.

Documento generado como resumen del laboratorio práctico de auditoría inalámbrica.
