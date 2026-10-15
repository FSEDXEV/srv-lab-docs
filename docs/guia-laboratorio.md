# Laboratorio srv-lab: procedimiento paso a paso (AlmaLinux 9, Podman y GLPI)

Última actualización: 14 de octubre de 2026

## Objetivo y alcance

Esta guía explica cómo se montó, paso a paso, un servidor de laboratorio con AlmaLinux 9 y GLPI sobre un equipo de escritorio antiguo, para que una persona con conocimientos básicos pueda repetirlo.

El propósito del proyecto es tener un entorno propio donde practicar administración de sistemas, seguridad, gestión de servicios (ITIL) y documentación, y dejar cada decisión registrada como evidencia de trabajo.

Al terminar se tiene:

- Un servidor Linux actualizado, con acceso remoto por llave SSH, firewall y SELinux activos.
- GLPI (inventario de equipos y tickets) corriendo en contenedores Podman que arrancan solos.
- Copias de seguridad y una guía reproducible.

Convenciones: los comandos se ejecutan como el usuario `fabian` y con `sudo` solo cuando hace falta. Cambia `fabian`, `srv-lab` y la IP `192.168.1.37` por los valores de tu red. Cada paso indica qué hace, para qué sirve, cómo comprobarlo y qué puede salir mal.

Fuera de alcance por ahora: acceso desde internet, HTTPS y alta disponibilidad. Son mejoras posteriores.

## Mapa del laboratorio

*(Diagrama: 1 servidor con 3 puertos abiertos: 22 SSH, 9090 Cockpit y 8080 GLPI. Disponible en la versión original de la guía.)*

Los equipos de la red local llegan al servidor por tres puertos; la base de datos de GLPI solo es accesible desde el contenedor de GLPI.

## Qué es cada herramienta y para qué sirve

El laboratorio es una cadena de piezas pequeñas, cada una con un trabajo. Esta sección explica cada una en lenguaje simple, para qué se usa aquí y qué alternativas existen.

**Cómo encajan:** tú abres el navegador y entras al puerto 8080 del servidor. El cortafuegos (firewalld) lo permite y Podman entrega la petición al contenedor de GLPI. GLPI consulta a MariaDB por una red interna de Podman. Los datos viven en dos volúmenes, `glpi-db` y `glpi-data`, que son justo lo que se respalda.

### Ya instaladas

| Herramienta | Qué es | Para qué la usamos aquí | Alternativas |
| --- | --- | --- | --- |
| AlmaLinux 9 | Sistema operativo Linux gratuito, compatible con Red Hat Enterprise Linux (RHEL) y pensado para servidores | Es la base de todo. Se eligió por ser estable, recibir parches de seguridad durante años y ser el tipo de sistema que usan muchas empresas | Ubuntu Server, Debian, Rocky Linux |
| Podman | Programa que ejecuta contenedores: aplicaciones empaquetadas con todo lo que necesitan para funcionar | Ejecuta GLPI y MariaDB aislados del sistema, sin instalar PHP, Apache ni la base de datos en el servidor. Se puede borrar o actualizar un servicio sin ensuciar el resto. Funciona sin permisos de administrador (rootless), lo que reduce el daño si algo falla | Docker, que hace lo mismo pero usa un servicio con privilegios; Podman es el estándar en AlmaLinux |
| Quadlet | Una forma de describir un contenedor en un archivo de texto (`.container`) para que systemd lo gestione | Hace que GLPI y MariaDB arranquen solos al encender el servidor y se reinicien si fallan. La configuración queda en archivos que se pueden guardar en Git | Comandos `podman run` manuales, que no sobreviven a un reinicio |
| systemd | El gestor de servicios de Linux: arranca, detiene y vigila programas | Con `systemctl` se enciende, apaga y consulta el estado de los servicios, incluido GLPI | Está en todas las distribuciones modernas |
| GLPI | Plataforma web gratuita de gestión de servicios de TI (ITSM) y de inventario | Inventario de equipos con su hoja de vida, tickets de soporte, gestión de cambios (ITIL), contratos y licencias. Es práctica real para el perfil de ingeniería de sistemas y se usa en clínicas y empresas | Snipe-IT (solo inventario), osTicket o Zammad (solo tickets), Jira Service Management (de pago) |
| MariaDB | Base de datos: un programa que guarda información en tablas y responde consultas | GLPI guarda aquí usuarios, equipos, tickets e historial. Es la memoria de GLPI, por eso es lo primero que se respalda | MySQL (casi idéntico) |
| SSH con llave | Conexión remota cifrada a la terminal del servidor | Administrar srv-lab desde el portátil sin teclado ni monitor. Una llave pública vive en el servidor y la privada en tu PC; sin la privada nadie entra | Escritorio remoto (RDP, VNC), innecesario en un servidor sin pantalla |
| firewalld | Cortafuegos: decide qué puertos aceptan conexiones desde la red | Solo están abiertos el 22 (SSH), el 9090 (Cockpit) y el 8080 (GLPI); todo lo demás está cerrado | nftables o iptables manuales |
| SELinux | Capa de seguridad que limita qué puede tocar cada programa, aunque lo vulneren | Se deja en modo Enforcing: si GLPI fuera atacado, no podría leer archivos ajenos. Es práctica habitual en RHEL y AlmaLinux | Desactivarlo es un mal hábito y no se hace |
| Cockpit | Panel web de administración del servidor, en el puerto 9090 | Ver CPU, memoria, discos, servicios y registros desde el navegador, sin memorizar comandos | Webmin, o solo la terminal |
| dnf-automatic | Servicio que instala solo las actualizaciones de seguridad | Mantiene el servidor parcheado sin intervención. No reinicia por su cuenta, para que los servicios no se caigan de repente | Actualizar a mano cada semana |
| Bitwarden | Gestor de contraseñas en la nube, con cifrado | Guarda contraseñas largas y únicas (router, GLPI, Cockpit, GitHub) y solo necesitas recordar la maestra | KeePassXC (archivo local), Proton Pass |
| Git y GitHub | Git guarda el historial de cambios de los archivos; GitHub aloja los repositorios en internet | Versionar la documentación y las configuraciones. El repositorio público muestra tu trabajo a reclutadores y el privado guarda lo funcional | GitLab, Forgejo (autoalojado) |

### Por instalar

| Herramienta | Qué es | Para qué la usaremos |
| --- | --- | --- |
| Nagios | Sistema de monitoreo | Vigilar el servidor y el router y avisar si algo se cae; práctica de supervisión de infraestructura |
| MeshCentral | Control remoto de equipos | Administrar equipos de la red desde el navegador, como en un servicio de soporte |
| Nextcloud | Almacenamiento y carpetas compartidas propias | Crear usuarios, carpetas y permisos, una tarea habitual en puestos de TI |
| OpenVAS | Escáner de vulnerabilidades | Revisar la seguridad del propio laboratorio (necesita 4 GB de RAM o más) |
| Proxy inverso con HTTPS | Puerta de entrada cifrada a los servicios web | Que GLPI y los demás se abran con candado y sin exponer cada puerto |
| Router multi-IA (LiteLLM) | Programa en Python que reparte cada tarea al modelo de IA más adecuado | Ahorrar tokens y orquestar varias IA; todavía no se prueba con claves reales |

## Hardware evaluado y decisiones

Se eligió el equipo de escritorio como servidor porque libera a los portátiles y tiene discos de sobra; su límite es la RAM (4 GB), que condiciona qué se instala.

| Equipo | Procesador | RAM | Disco | Virtualización | Papel |
| --- | --- | --- | --- | --- | --- |
| HP Pavilion 14-av005la | AMD A10-8700P, 4 núcleos | 7,4 GB utilizables | 1 SSD | Deshabilitada | Reserva y pruebas |
| HP 14-bs012la | Intel Core i3-6006U, 2 núcleos / 4 hilos | 15,9 GB (ya en el máximo de 16 GB) | 2 SSD | Habilitada | Equipo de trabajo |
| Desktop ASUS P8H61-M LX2 | Intel Core i3-2120, 2 núcleos / 4 hilos | 4 GB DDR3-1333 (la placa admite hasta 16 GB) | SSD Crucial BX500 240 GB | Habilitada en la BIOS | Servidor `srv-lab` |

Decisiones y motivos:

- **Contenedores en vez de máquinas virtuales.** Con 4 GB de RAM, un contenedor consume una fracción de lo que gasta una VM.
- **Un solo SSD para el sistema.** Se instala solo en el Crucial de 240 GB; el SSD de 1 TB y el disco duro de 1 TB quedan desconectados hasta que hagan falta, y así el instalador no puede borrarlos por error.
- **Ampliar la RAM después.** La placa tiene 2 ranuras DIMM y admite hasta 16 GB con módulos de 8 GB; la frecuencia real será 1333 MHz porque el i3-2120 es de segunda generación ([ficha de la placa](https://dlcdnet.asus.com/pub/ASUS/mb/LGA1155/P8H61-M_LX2_R20/E7377_P8H61-M_LX2_R2.pdf)).

Cómo averiguar la RAM máxima de cualquier equipo:

1. Administrador de tareas → Rendimiento → Memoria: muestra "ranuras usadas", tipo y velocidad.
2. PowerShell: `Get-CimInstance Win32_PhysicalMemoryArray | Select MaxCapacityEx, MemoryDevices` (el valor sale en KB; la BIOS a veces lo declara mal).
3. Buscar el modelo exacto en la ficha del fabricante y comparar con el límite del procesador (Intel ARK o AMD).

El máximo real es el menor entre procesador, placa y sistema operativo.

## Preparación de la BIOS

La BIOS se ajusta antes de instalar para que el equipo funcione como servidor: virtualización activa, disco en modo moderno y encendido automático tras un corte de luz.

Para entrar, pulsa `Supr` al encender; `F8` abre el menú de arranque y `F10` guarda y sale. Sin `F10` ningún cambio se aplica.

| Ajuste | Ruta | Valor | Para qué sirve |
| --- | --- | --- | --- |
| Intel Virtualization Technology | Advanced → CPU Configuration | Enabled | Permite máquinas virtuales (KVM) más adelante |
| SATA Mode | Advanced → SATA Configuration | AHCI Mode | Modo moderno del disco; mejor rendimiento y lectura de estado SMART |
| Restore AC Power Loss | Advanced → APM | Power On | El servidor enciende solo cuando vuelve la luz |
| Power On By PCIE | Advanced → APM | Enabled | Permite Wake on LAN por la red integrada de la placa |
| Power On By PCI | Advanced → APM | Enabled (pendiente) | Wake on LAN con la tarjeta de red PCI que se usa hoy |
| Wait For 'F1' If Error | Boot | Disabled | El equipo no se queda esperando una tecla si hay una advertencia |
| Boot Option #1 | Boot | USB (UEFI) | Arrancar el instalador desde el USB |
| System Date / System Time | Main | Hora real | Evita fallos de certificados y registros desordenados |

Se dejaron sin tocar:

- Todo lo de overclock (CPU Ratio, GPU Boost, DRAM Timing Control): aumentan calor y riesgo sin beneficio para un servidor.
- Initiate Graphic Adapter (PEG/PCI): decide qué tarjeta de video arranca primero; cambiarlo con el monitor en otra salida deja la pantalla en negro.
- La actualización de la BIOS: es la versión de 2012 y un corte de luz a mitad del proceso puede dañar la placa.

Problemas frecuentes:

- La opción no aparece: entra al submenú anterior (la pantalla "CPU Configuration" es solo un submenú de Advanced; la lista completa está un nivel arriba).
- Cambié algo y no se aplicó: faltó `F10`, o saliste con `Esc` sin guardar.
- El sistema no ve la tarjeta de red aunque la BIOS tenga el LAN en Enabled: ver la sección de red.

## Instalación de AlmaLinux 9.8

Se instaló AlmaLinux 9.8 en modo mínimo desde un USB con Ventoy, usando solo el SSD de 240 GB como destino, para obtener un sistema liviano y fácil de proteger.

Por qué la versión 9: tiene soporte más largo que la 8, y la 10 exige procesadores más recientes que el i3-2120 de este equipo.

Pasos:

1. Prepara un USB con Ventoy (programa que permite arrancar varias ISO desde un mismo USB) y copia en él la ISO `AlmaLinux-9.8-x86_64-minimal.iso`.
2. Apaga el equipo y deja conectado solo el SSD de 240 GB. Desconecta el disco duro y el SSD de 1 TB para que el instalador no pueda tocarlos.
3. Arranca desde el USB (`F8` → opción UEFI del USB) y en el menú de Ventoy elige la ISO.
4. En el menú de GRUB elige "Install AlmaLinux 9.8". No elijas el modo FIPS (seguridad estricta que complica todo) ni "Test this media" (revisa el USB y tarda).
5. Idioma: Español (Colombia). Teclado: Español latinoamericano (latam).
6. Destino de la instalación: marca solo el disco `sda` (el Crucial), configuración **Automática**, sin cifrado. Con cifrado el servidor pediría una frase en cada arranque y no podría reiniciarse solo tras un corte de luz.
7. Selección de software: **Instalación mínima**.
8. Contraseña de root: déjala deshabilitada. Creación de usuario: nombre en minúsculas (`fabian`), marca "Hacer administrador a este usuario" y usa una contraseña larga.
9. KDUMP: desmarca "Habilitar kdump". Reserva memoria para volcados de fallos y con 4 GB hace falta toda.
10. Red: debe aparecer una interfaz conectada (ver la sección de red). No pulses "Comenzar la instalación" mientras diga "No hay dispositivos de red".
11. Pulsa **Comenzar la instalación**, espera, **retira el USB** y reinicia.

Particionado automático resultante:

| Punto de montaje | Tamaño | Uso |
| --- | --- | --- |
| `/boot/efi` | 600 MB | Arranque UEFI |
| `/boot` | 1 GB | Núcleos del sistema |
| `/` | 70 GB | Sistema (usa unos 3 GB) |
| `/home` | 148 GB | Datos de usuario y contenedores |
| swap | 3,9 GB | Memoria de apoyo |

Mensajes que parecen errores pero no lo son:

- "el procesador tiene activado SMT": Hyper-Threading activo. Se deja así; desactivarlo dejaría solo 2 hilos.
- `ata6: link is slow to respond, please be patient` al arrancar: el sistema revisa un puerto SATA sin disco y tarda en rendirse. Si dura más de 3 o 4 minutos, revisa los cables SATA.

Problemas frecuentes:

- Al reiniciar vuelve al instalador: el USB sigue conectado.
- "Por favor complete los elementos marcados": falta crear el usuario.
- La tarjeta de red no aparece: ver la sección de red.

## Red: tarjeta PCI e IP fija

El instalador no detectó el chip de red de la placa, así que se instaló una tarjeta PCI TRENDnet (chip Realtek RTL8169), que Linux reconoce como `enp5s0`, y se fijó su dirección IP.

**Diagnóstico.** Durante la instalación, `Ctrl + Alt + F2` abre una consola. Comandos usados:

```
ip link
lspci -nn | grep -i -E "ethernet|network"
lsmod | grep r8169
```

- `ip link` lista las interfaces de red. Si solo aparece `lo` (la interfaz interna), el sistema no ve ninguna tarjeta.
- `lspci` lista los dispositivos PCI; sirve para saber si la placa ve el chip.
- `lsmod | grep r8169` comprueba si el driver de Realtek está cargado.

Con `Ctrl + Alt + F6` (o `F1`) se vuelve a la pantalla gráfica.

**Instalación de la tarjeta.**

1. Apaga el equipo y desenchufa el cable de corriente.
2. Toca una parte metálica de la carcasa para descargarte.
3. Inserta la tarjeta en la ranura **PCI** (larga, con una muesca hacia el final). No va en las ranuras pequeñas PCIe x1 ni en la larga de la tarjeta de video.
4. Atornilla el soporte, conecta el cable de red **a la tarjeta** y deja el adaptador Wi-Fi USB desconectado (el instalador no trae sus drivers).
5. Enciende. Al reentrar al instalador, `enp5s0` aparece como conectada.

**IP fija.** El router entrega por defecto una IP que puede cambiar; para un servidor conviene una fija:

```
nmcli connection show
sudo nmcli connection modify enp5s0 ipv4.method manual ipv4.addresses 192.168.1.37/24 ipv4.gateway 192.168.1.1 ipv4.dns "192.168.1.1 8.8.8.8" ipv4.ignore-auto-dns yes
nmcli -g ipv4.method,ipv4.addresses,ipv4.gateway connection show enp5s0
sudo reboot
```

- `nmcli connection show` muestra el nombre de la conexión (aquí `enp5s0`).
- `ipv4.method manual` desactiva la IP automática; `ipv4.addresses` fija la dirección y la máscara (`/24`); `ipv4.gateway` es el router; `ipv4.dns` son los servidores de nombres.
- La tercera línea verifica el resultado antes de reiniciar.

Comprobación tras reiniciar: `ip a show enp5s0`, `ping -c 3 8.8.8.8` (hay internet) y `ping -c 3 google.com` (el DNS resuelve nombres).

**Velocidad de 100 Mb/s.** La tarjeta es Gigabit, pero el enlace negoció 100:

```
sudo ethtool enp5s0 | grep -E "Speed|Duplex"
sudo ethtool enp5s0 | grep -i -A4 "link partner"
```

La segunda salida mostró que el otro extremo (router o switch) solo anuncia 10 y 100 Mb/s. La causa es el puerto del equipo de red o un dispositivo intermedio, no la tarjeta. Pruebas: otro puerto del router, conexión directa sin switch y un cable categoría 5e o 6 en buen estado. A 100 Mb/s el laboratorio funciona; solo se nota con respaldos grandes.

**Hecho (14 de octubre de 2026).** La IP `192.168.1.37` quedó reservada en el router VSOL para la MAC `00:14:d1:xx:xx:xx`, en la lista de direcciones IP reservadas del apartado LAN. Con eso el router no entrega esa dirección a otro equipo.

Problemas frecuentes:

- Aparece solo `lo`: la placa no ve la tarjeta. Revisa que esté bien asentada (limpia los contactos con alcohol isopropílico) o usa un adaptador USB-Ethernet con chip Realtek RTL8153 o ASIX AX88179.
- `ping` a una IP funciona pero a un nombre no: falla el DNS; revisa `ipv4.dns`.
- Tras reiniciar no responde por SSH: conecta teclado y monitor y revisa `nmcli connection show` e `ip a`.

## Primeros ajustes del sistema

Tras la instalación se actualiza el sistema, se corrige el reloj y se instalan las herramientas básicas, trabajando siempre desde el portátil por SSH y no desde el teclado del servidor.

**Conectarse.** Desde PowerShell en el portátil: `ssh fabian@192.168.1.37`. La primera vez pregunta si confías en el servidor: escribe `yes`. La contraseña no se muestra al escribirla.

**Actualizar.**

```
sudo dnf update -y
sudo reboot
uname -r
```

- `dnf update` descarga e instala las actualizaciones (aquí fueron 113 paquetes, con kernel nuevo). Se hace primero para cerrar vulnerabilidades conocidas.
- Se reinicia para cargar el kernel nuevo; `uname -r` debe mostrar la versión nueva.

**Hora y zona horaria.**

```
timedatectl
sudo timedatectl set-local-rtc 0
```

- Debe decir `Time zone: America/Bogota` y `System clock synchronized: yes` (el sistema se sincroniza solo por internet).
- Si aparece un aviso amarillo sobre "RTC in local TZ", el reloj de hardware está en hora local; Linux lo guarda mejor en UTC. `set-local-rtc 0` lo corrige sin cambiar la hora que ves.
- Una hora correcta importa: los registros, los certificados y las tareas programadas dependen de ella.

**Herramientas básicas.**

```
sudo dnf install -y cockpit cockpit-storaged cockpit-podman vim tmux git bind-utils
```

- `vim`: editor de texto. `tmux`: mantiene sesiones abiertas aunque se corte la conexión. `git`: control de versiones. `bind-utils`: comandos de diagnóstico de nombres (`dig`, `nslookup`).
- `htop` (monitor de procesos) no está en los repositorios base. Si lo quieres, activa el repositorio extra EPEL con `sudo dnf install -y epel-release` y luego `sudo dnf install -y htop`. Es opcional: `top` viene instalado y Cockpit muestra CPU y memoria.

**Hábito de seguridad.** Trabaja como `fabian` y antepón `sudo` solo a los comandos que lo necesiten. `sudo su` te deja como administrador todo el rato, y un error de tecleo puede borrar archivos del sistema. Además, los servicios de usuario de Podman no funcionan bien desde una sesión abierta con `sudo su`.

Problemas frecuentes:

- `No match for argument: htop` y todo se cancela: DNF aborta la instalación completa si un paquete no existe. Repite el comando sin `htop`.
- La actualización tarda: es normal en la primera vez; el archivo más grande (firmware) pesa cientos de MB.
- Tras reiniciar no entra por SSH: espera 1 o 2 minutos; el puerto SATA vacío retrasa el arranque unos segundos.

## Acceso SSH seguro con llave

Se configuró acceso por llave SSH y se desactivó el inicio de sesión con contraseña y como root, porque una contraseña se puede adivinar o robar y una llave privada no viaja por la red.

Una llave SSH son dos archivos: la **privada** (se queda en tu portátil y no se comparte nunca) y la **pública** (se copia al servidor).

**1. Crear la llave (en el portátil, PowerShell).**

```
ssh-keygen -t ed25519
```

- `ed25519` es un tipo de llave moderno, corto y seguro.
- Te pregunta el nombre del archivo y una frase de paso. Pulsa Enter para aceptar la ruta por defecto (`C:\Users\<usuario>\.ssh\id_ed25519`). Si escribes solo un nombre (aquí `LAB`), el archivo se crea en la carpeta actual y hay que moverlo a `.ssh`:

```
New-Item -ItemType Directory -Force $env:USERPROFILE\.ssh
Move-Item $env:USERPROFILE\LAB $env:USERPROFILE\.ssh\
Move-Item $env:USERPROFILE\LAB.pub $env:USERPROFILE\.ssh\
```

- Una frase de paso protege la llave si alguien copia el archivo. Se puede añadir después: `ssh-keygen -p -f $env:USERPROFILE\.ssh\LAB`.

**2. Mostrar la llave pública.**

```
type $env:USERPROFILE\.ssh\LAB.pub
```

Copia la línea completa (empieza con `ssh-ed25519`).

**3. Instalarla en el servidor (sesión SSH abierta, como `fabian`).**

```
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA...tu_llave... usuario@PC' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
restorecon -R ~/.ssh
cat ~/.ssh/authorized_keys
```

- `chmod 700` y `600` restringen los permisos: si son más abiertos, SSH ignora la llave.
- `restorecon` restablece las etiquetas de SELinux de esos archivos.
- `cat` comprueba que la llave quedó en una sola línea.

**4. Atajo de conexión (en el portátil).**

```
Add-Content $env:USERPROFILE\.ssh\config "Host srv-lab`n  HostName 192.168.1.37`n  User fabian`n  IdentityFile ~/.ssh/LAB"
```

Desde ahora basta con `ssh srv-lab`.

**5. Probar la llave** (en una ventana nueva, sin cerrar la que ya está abierta):

```
ssh srv-lab
ssh -v srv-lab 2>&1 | findstr /i "Offering Authenticated"
```

La segunda línea debe mostrar `Authenticated to 192.168.1.37 ... using "publickey"`. Si no funciona, **no sigas**: resuélvelo antes de desactivar la contraseña.

**6. Endurecer el servidor** (con una sesión de respaldo abierta):

```
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf > /dev/null << 'EOF'
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
MaxAuthTries 3
EOF
sudo sshd -t
sudo sshd -T | grep -Ei "^(passwordauthentication|permitrootlogin|pubkeyauthentication)"
sudo systemctl reload sshd
```

- El archivo se llama `00-...` para que se lea primero: en AlmaLinux 9 SSH usa el primer valor que encuentra y la configuración del sistema también tiene entradas.
- `sshd -t` valida la sintaxis (sin salida = correcto). `sshd -T` muestra los valores efectivos: deben ser `passwordauthentication no`, `permitrootlogin no`, `pubkeyauthentication yes`.
- `reload` aplica el cambio sin cortar las sesiones abiertas. Hasta ejecutarlo, la contraseña sigue funcionando.

**7. Probar que la contraseña ya no sirve:**

```
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no fabian@192.168.1.37
```

Debe responder `Permission denied (publickey,...)`. Y `ssh srv-lab` debe seguir entrando.

Problemas frecuentes:

- `-bash: ssh-ed25519: command not found`: pegaste la llave sin el `echo '...' >> ~/.ssh/authorized_keys`; no pasó nada, repite el paso 3.
- Sigue pidiendo contraseña con la llave: revisa permisos (`ls -ld ~/.ssh`, `ls -l ~/.ssh/authorized_keys`), `restorecon` y el registro con `sudo journalctl -u sshd -n 20`.
- Sigue entrando con contraseña tras endurecer: falta `sudo systemctl reload sshd` o el valor no se aplicó; compáralo con `sshd -T`.
- Pregunta "authenticity of host ... can't be established": aparece al conectar por IP después de haber usado el alias; es el mismo servidor, responde `yes`.
- Perdiste la llave privada: sin contraseña de respaldo, solo se recupera con teclado y monitor conectados al servidor. Guarda una copia del archivo `LAB` en un lugar seguro (no en la nube pública).

## Firewall y SELinux

El servidor solo deja pasar los servicios necesarios (SSH, Cockpit y, más adelante, GLPI), y SELinux queda activo como segunda barrera.

**Firewall (`firewalld`).** Se ve el estado con:

```
sudo firewall-cmd --list-all
```

La zona `public` tenía activos `cockpit`, `dhcpv6-client` y `ssh`. Como la IP es fija en IPv4, se quitó el cliente DHCPv6 y se abrió el puerto de GLPI:

```
sudo firewall-cmd --permanent --remove-service=dhcpv6-client
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
sudo firewall-cmd --list-ports
```

- `--permanent` guarda la regla para después de reiniciar; sin él, solo vale hasta el próximo reinicio.
- `--reload` aplica las reglas permanentes ahora.
- Resultado esperado: servicios `cockpit ssh` y puerto `8080/tcp`.
- Regla general: abre solo lo que uses y cierra lo demás. Nunca abras Cockpit (9090) ni SSH al internet; para acceder desde fuera se usa una VPN.

**SELinux.** Es un control de acceso que limita lo que cada proceso puede tocar, incluso si lo ejecuta un administrador:

```
getenforce
```

Debe responder `Enforcing`. No lo desactives para "arreglar" un error: casi siempre se corrige con la etiqueta correcta. Los volúmenes de Podman con nombre ya reciben la etiqueta adecuada; si montas una carpeta del disco en un contenedor, agrega `:Z` al montaje.

Problemas frecuentes:

- No abre la página de un servicio nuevo: falta abrir su puerto en el firewall (`--add-port` y `--reload`).
- Una regla desaparece tras reiniciar: se agregó sin `--permanent`.
- Un servicio falla con "permission denied" aunque los permisos de archivo estén bien: puede ser SELinux; revisa los registros de auditoría y corrige la etiqueta en lugar de desactivarlo.

## Actualizaciones automáticas de seguridad

Un temporizador instala cada día los parches de seguridad sin intervención, y deja el reinicio a tu criterio, para que el servidor no quede desactualizado entre revisiones.

```
sudo dnf install -y dnf-automatic
sudo sed -i 's/^upgrade_type.*/upgrade_type = security/; s/^apply_updates.*/apply_updates = yes/' /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer
systemctl list-timers dnf-automatic.timer
grep -E "^(upgrade_type|apply_updates)" /etc/dnf/automatic.conf
```

- `dnf-automatic` es el paquete que automatiza las actualizaciones.
- `sed` cambia dos valores del archivo de configuración: `upgrade_type = security` (solo parches de seguridad, no todo) y `apply_updates = yes` (los instala, en vez de solo descargarlos).
- `enable --now` activa el temporizador ahora y en cada arranque. `list-timers` muestra la próxima ejecución (aquí, a las 06:13 del día siguiente).
- La última línea comprueba que los dos valores quedaron aplicados.

El equipo **no se reinicia solo**. Si se actualiza el kernel, reinicia tú cuando convenga (Cockpit lo avisa).

Problemas frecuentes:

- Una actualización rompe un servicio: por eso existen las copias de seguridad; revisa `sudo journalctl -u dnf-automatic -n 50` y restaura si hace falta.
- El temporizador no aparece: repite `enable --now` y revisa con `systemctl status dnf-automatic.timer`.

## Cockpit: administración desde el navegador

Cockpit es un panel web para administrar el servidor (servicios, almacenamiento, contenedores, actualizaciones) desde cualquier navegador de la red local, incluido el celular o la tablet.

```
sudo dnf install -y cockpit cockpit-storaged cockpit-podman
sudo systemctl enable --now cockpit.socket
sudo systemctl status cockpit.socket
```

- `cockpit.socket` arranca el panel solo cuando alguien lo abre, y así gasta casi nada en reposo.
- El estado debe decir `active (listening)`. El firewall ya permitía el servicio `cockpit`.

**Entrar.** Abre `https://192.168.1.37:9090`. El navegador avisa que el certificado no es de confianza porque es autofirmado; en una red local propia es normal, acepta continuar. Entra con el usuario `fabian`.

**Acceso administrativo.** Arriba a la derecha aparece "Acceso limitado" (un candado). Púlsalo y escribe tu contraseña para poder instalar, reiniciar y gestionar servicios.

Precauciones:

- Mantén Cockpit solo en la red local; no abras el puerto 9090 al internet.
- Los contenedores del usuario aparecen en la sección de Podman.
- Con 4 GB de RAM, cierra la pestaña cuando no la uses.

Problemas frecuentes:

- `Unit file cockpit.socket does not exist`: la instalación de paquetes falló antes de terminar (por ejemplo, por pedir `htop`, que no existe en los repositorios base). Repite la instalación sin ese paquete.
- No carga la página: confirma `systemctl status cockpit.socket` y `sudo firewall-cmd --list-services`.
- Aviso de certificado cada vez: es esperable; más adelante se puede poner HTTPS con un certificado propio.

## Contenedores con Podman y Quadlet

Los servicios corren en contenedores Podman sin privilegios de administrador, definidos con archivos de texto (Quadlet) que systemd convierte en servicios que arrancan solos.

Conceptos:

- **Contenedor:** un programa empaquetado con sus dependencias, aislado del resto del sistema. Gasta mucha menos RAM que una máquina virtual.
- **Podman:** herramienta de contenedores que no necesita un servicio central (daemon). Aquí es la versión 5.8.2.
- **Rootless:** los contenedores corren como el usuario `fabian`, no como administrador. Si alguien rompe uno, tiene menos poder sobre el sistema.
- **Quadlet:** archivos `.container`, `.volume` y `.network` en `~/.config/containers/systemd/`. Systemd los traduce a servicios, con reinicio automático.
- **Linger:** hace que los servicios de un usuario sigan funcionando cuando cierra la sesión y arranquen al encender el equipo.

Preparación:

```
podman --version || sudo dnf install -y podman
sudo loginctl enable-linger fabian
podman info | grep -i graphroot
mkdir -p ~/lab/{glpi,nagios,mesh,docs}
```

- La primera línea comprueba que Podman existe (y lo instala si no).
- `enable-linger` activa el modo descrito arriba.
- `graphRoot` debe apuntar a `/home/fabian/.local/share/containers/storage`: los contenedores usan el disco grande (`/home`, 148 GB) y no los 70 GB del sistema.
- `~/lab` ordena el trabajo: una carpeta por servicio y `docs` para la documentación.

Comandos de uso diario:

| Comando | Para qué sirve |
| --- | --- |
| `podman ps` | Lista los contenedores en marcha |
| `podman logs --tail 20 <nombre>` | Muestra los últimos mensajes de un contenedor |
| `podman stats --no-stream` | Consumo de CPU y RAM por contenedor |
| `systemctl --user status <servicio>` | Estado de un servicio de usuario |
| `systemctl --user restart <servicio>` | Reinicia un servicio |
| `systemctl --user daemon-reload` | Relee los archivos Quadlet tras editarlos |

Límites que conviene conocer:

- Un usuario normal no puede abrir puertos menores a 1024: los servicios web salen por puertos altos (por ejemplo 8080).
- Con SELinux activo, las carpetas del disco que se montan en un contenedor necesitan `:Z`; los volúmenes con nombre no.
- Con 4 GB de RAM, instala los servicios por fases y revisa el consumo con `free -h` y `podman stats` antes de añadir otro.

Problemas frecuentes:

- `systemctl --user` da error: abriste la sesión con `sudo su` o sin una sesión real; entra con `ssh srv-lab` como `fabian`.
- Los contenedores no arrancan tras reiniciar: falta `enable-linger`.
- Cambiaste un archivo `.container` y no se nota: ejecuta `systemctl --user daemon-reload` y reinicia el servicio.

## GLPI en contenedores

GLPI (inventario de equipos y gestión de tickets) corre en dos contenedores, la aplicación y su base de datos, fijados a una versión concreta para que el laboratorio sea reproducible.

Decisiones:

- **Imagen oficial `glpi/glpi`**, publicada por el proyecto en Docker Hub y GitHub ([Run GLPI with Docker](https://www.glpi-project.org/en/run-glpi-with-docker/)).
- **Versión fija `11.0.11`.** GLPI 12.0.0 salió el 7 de octubre de 2026 ([versiones](https://github.com/glpi-project/glpi/releases)) y la etiqueta `latest` ya apunta a ella ([etiquetas](https://hub.docker.com/r/glpi/glpi/tags)). Se prefirió la 11 por madurez; pasar de la 11 a la 12 queda como ejercicio de gestión de cambios (respaldo, plan, ejecución, verificación).
- **MariaDB 10.11** en vez de MySQL, porque consume menos RAM. No se verificaron los requisitos de base de datos de GLPI 12.
- **Contraseñas aleatorias**, no las de ejemplo del repositorio oficial.

**Paso 1: contraseñas y archivos de entorno** (como `fabian`).

```
mkdir -p ~/lab/glpi ~/.config/containers/systemd && cd ~/lab/glpi
DBPASS=$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 24)
cat > glpi.env << EOF
GLPI_DB_HOST=glpi-db
GLPI_DB_PORT=3306
GLPI_DB_NAME=glpi
GLPI_DB_USER=glpi
GLPI_DB_PASSWORD=$DBPASS
EOF
cat > db.env << EOF
MARIADB_RANDOM_ROOT_PASSWORD=yes
MARIADB_DATABASE=glpi
MARIADB_USER=glpi
MARIADB_PASSWORD=$DBPASS
EOF
chmod 600 glpi.env db.env
unset DBPASS
```

- Se genera una contraseña aleatoria de 24 caracteres y se escribe en dos archivos de entorno: uno para la aplicación y otro para la base de datos.
- `chmod 600` hace que solo `fabian` pueda leerlos. `unset DBPASS` la borra de la sesión.
- La contraseña de administrador de MariaDB se genera sola y no se usa.

**Paso 2: archivos Quadlet** (en `~/.config/containers/systemd`).

```
cd ~/.config/containers/systemd
printf '[Network]\nNetworkName=glpi\n' > glpi.network
printf '[Volume]\n' > glpi-db.volume
printf '[Volume]\n' > glpi-data.volume
```

Base de datos, archivo `glpi-db.container`:

```
[Unit]
Description=GLPI - base de datos

[Container]
ContainerName=glpi-db
Image=docker.io/library/mariadb:10.11
Network=glpi.network
Volume=glpi-db.volume:/var/lib/mysql
EnvironmentFile=%h/lab/glpi/db.env
Exec=--innodb-buffer-pool-size=128M --max-connections=30

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Aplicación, archivo `glpi.container`:

```
[Unit]
Description=GLPI - aplicacion web
Requires=glpi-db.service
After=glpi-db.service

[Container]
ContainerName=glpi
Image=docker.io/glpi/glpi:11.0.11
Network=glpi.network
PublishPort=8080:80
Volume=glpi-data.volume:/var/glpi
EnvironmentFile=%h/lab/glpi/glpi.env

[Service]
Restart=always

[Install]
WantedBy=default.target
```

- `Network=glpi.network`: ambos contenedores comparten una red privada y se ven por su nombre (`glpi-db`).
- `Volume=...`: los datos viven en volúmenes con nombre dentro de `/home`, y sobreviven a que se recreen los contenedores.
- `Exec=--innodb-buffer-pool-size=128M`: limita la memoria que usa la base de datos.
- `Requires` y `After`: GLPI arranca después de la base de datos.
- `PublishPort=8080:80`: el puerto 80 del contenedor se expone en el 8080 del servidor.
- `Restart=always` y `WantedBy=default.target`: se reinician si fallan y arrancan con el sistema (con linger activo).

**Paso 3: abrir el puerto, arrancar y comprobar.**

```
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
systemctl --user daemon-reload
systemctl --user start glpi-db.service glpi.service
podman ps
podman logs --tail 20 glpi
podman stats --no-stream
free -h
```

La primera vez descarga unos 350 MB de imágenes. `podman ps` debe mostrar la imagen `glpi/glpi:11.0.11` y el puerto `0.0.0.0:8080->80/tcp`. La imagen instala la base de datos sola; en los registros aparecen las tareas automáticas de GLPI ejecutándose cada minuto.

Consumo medido: `glpi-db` unos 200 MB de RAM y `glpi` unos 43 MB. El servidor quedó con unos 988 MiB usados y 2,6 GiB disponibles. Ojo: `systemctl status` muestra cifras más altas porque incluyen la memoria de caché.

**Paso 4: primer acceso y seguridad.** Abre `http://192.168.1.37:8080` (cuando termine la instalación, 1 a 3 minutos).

- GLPI trae cuatro usuarios de ejemplo (`glpi`, `tech`, `normal`, `post-only`) con contraseña igual al nombre. Un aviso naranja lo recuerda.
- Orden seguro: crea primero tu propio usuario Super-Admin (Administration → Users), **entra con él para comprobarlo**, y solo entonces cambia la contraseña de los otros por una larga aleatoria y márcalos como inactivos.
- El panel muestra "datos de demostración": equipos y programas que no existen. Sirven para explorar. Tras desactivarlos, verifica si desaparecieron; si no, como aún no hay datos reales, se pueden borrar los volúmenes y reinstalar en minutos.
- La barra dice "No seguro" porque la conexión es HTTP. Es aceptable en la red local del laboratorio, pero la contraseña viaja sin cifrar; más adelante se añade HTTPS con un proxy inverso.

Problemas frecuentes:

- La página no carga: comprueba `podman ps`, el puerto abierto en el firewall y los registros (`podman logs glpi`); la primera instalación tarda.
- `glpi` falla por no encontrar la base de datos: revisa `journalctl --user -u glpi-db -n 30` y que ambos contenedores estén en la red `glpi`.
- El texto del archivo Quadlet quedó mezclado al pegar un bloque largo: revisa con `cat` y vuelve a pegarlo en trozos más cortos.
- Las horas de los registros aparecen en UTC: no afecta el funcionamiento; se ajusta después.

**Paso 5: crear tu usuario administrador y desactivar los de fábrica** (hecho el 14 de octubre de 2026).

GLPI se instala con cuatro usuarios de ejemplo cuyas contraseñas son públicas (el mismo nombre que el usuario). Cualquiera que llegue a la pantalla de inicio podría entrar con ellos, así que se reemplazan por un usuario propio.

1. Entra a `http://192.168.1.37:8080` con el usuario de fábrica `glpi`.
2. Ve a **Administración > Usuarios** y pulsa **+ Agregar**.
3. Rellena el identificador (por ejemplo `Fabian`), una contraseña larga generada por el gestor de contraseñas, el correo, el perfil **Super-Admin** y la entidad raíz con la opción recursiva activada.
4. Cierra sesión y entra con tu usuario nuevo. **No sigas hasta confirmar que funciona.**
5. En la lista de usuarios, desactiva `glpi`, `tech`, `normal` y `post-only` (campo *Activo* en *No*).
6. Comprueba la lista: tu usuario y `glpi-system` en *Sí*, los otros cuatro en *No*.

| Usuario de fábrica | Perfil | Qué se hizo |
| --- | --- | --- |
| `glpi` | Super-Admin | Desactivado |
| `tech` | Technician | Desactivado |
| `normal` | Observer | Desactivado |
| `post-only` | Self-Service | Desactivado |
| `glpi-system` | Cuenta interna | Se deja activo: GLPI la usa para sus tareas automáticas |

Se desactiva en lugar de borrar para que sea reversible y no se pierda el historial. Una cuenta con un perfil de administrador y contraseña conocida es de las primeras cosas que revisa un auditor de seguridad.

## Copias de seguridad

Una copia de GLPI necesita dos piezas, la base de datos y el volumen de datos, y debe guardarse también fuera del servidor; un respaldo en el mismo disco no protege si el disco falla.

Estado: **ejecutado y probado el 14 de octubre de 2026.** La base de datos se restauró en un contenedor temporal y mostró los 6 usuarios con su estado correcto. El volumen de archivos está respaldado y verificado (contiene `config/glpicrypt.key`), pero su restauración todavía no se probó.

**Hacer la copia** (como `fabian`):

```
mkdir -p ~/backups/glpi && chmod 700 ~/backups
podman exec glpi-db sh -c 'mariadb-dump -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" --single-transaction --no-tablespaces "$MARIADB_DATABASE"' > ~/backups/glpi/glpi-db-$(date +%F).sql
gzip ~/backups/glpi/glpi-db-$(date +%F).sql
podman volume export systemd-glpi-data -o ~/backups/glpi/glpi-data-$(date +%F).tar
chmod 600 ~/backups/glpi/*
ls -lh ~/backups/glpi
tar -tf ~/backups/glpi/glpi-data-$(date +%F).tar | grep glpicrypt
```

- `mariadb-dump` exporta la base de datos a texto SQL. Se usa el usuario de GLPI (`MARIADB_USER`) y no root, porque la clave de root es aleatoria (`MARIADB_RANDOM_ROOT_PASSWORD`) y no se guardó en ningún archivo.
- Los comandos con `$MARIADB_USER` y `$MARIADB_PASSWORD` van entre comillas simples para que los lea el contenedor y la clave no aparezca en pantalla ni en el historial.
- `--single-transaction` hace la copia sin detener GLPI; `--no-tablespaces` evita un error de permisos del usuario de GLPI; `$(date +%F)` agrega la fecha al nombre.
- `podman volume ls` muestra el nombre real del volumen de datos; aquí es `systemd-glpi-data`.
- El volumen de datos guarda la configuración de GLPI, incluida su clave de cifrado. Sin ella, la copia de la base de datos no sirve para recuperar las contraseñas que GLPI guarda cifradas. Por eso hacen falta **los dos archivos**.
- `chmod 600` deja los archivos legibles solo por tu usuario. Resultado esperado: un `.tar` de unos 3,8 MB y un `.sql.gz` de unos 130 KB.

**Sacar la copia del servidor** (desde el portátil, PowerShell):

```
mkdir C:\Users\USUARIO\backups-srv-lab
scp srv-lab:backups/glpi/* C:\Users\USUARIO\backups-srv-lab\
dir C:\Users\USUARIO\backups-srv-lab
```

Estos comandos son de Windows: se ejecutan en una ventana nueva, **fuera de la sesión `ssh srv-lab`**. El prompt `[fabian@srv-lab ~]$` es el servidor y `C:\Users\USUARIO>` es el portátil.

**Probar la restauración sin tocar GLPI** (hecho). Se importa la copia en una base temporal con una contraseña de prueba que nunca se reutiliza:

```
podman run -d --name test-restore --memory 300m -e MARIADB_RANDOM_ROOT_PASSWORD=yes -e MARIADB_DATABASE=glpitest -e MARIADB_USER=test -e MARIADB_PASSWORD=CLAVE_DE_PRUEBA docker.io/library/mariadb:10.11
sleep 25
zcat ~/backups/glpi/glpi-db-FECHA.sql.gz | podman exec -i test-restore sh -c 'mariadb -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" glpitest'
podman exec test-restore sh -c 'mariadb -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" glpitest -e "SELECT name, is_active FROM glpi_users;"'
podman rm -fv test-restore
```

Cambia `FECHA` por la de tu archivo. Resultado correcto: la consulta lista los 6 usuarios, con `Fabian` y `glpi-system` en 1 y los otros cuatro en 0. `podman rm -fv` borra la base temporal y su volumen.

**Restaurar de verdad** (solo en una emergencia; esta variante aún no se ha probado en producción):

```
systemctl --user stop glpi.service
zcat ~/backups/glpi/glpi-db-FECHA.sql.gz | podman exec -i glpi-db sh -c 'mariadb -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" "$MARIADB_DATABASE"'
podman volume import systemd-glpi-data ~/backups/glpi/glpi-data-FECHA.tar
systemctl --user start glpi.service
```

Probar la restauración una vez, en calma, es lo que convierte la copia en un respaldo real.

Regla útil: 3 copias, en 2 lugares distintos, 1 fuera de casa (la llamada regla 3-2-1). Para este laboratorio basta con empezar por servidor + portátil.

Problemas frecuentes:

- `Error: no such volume`: el nombre del volumen es otro; revisa `podman volume ls`.
- El archivo de la base de datos pesa 0 bytes: falló el comando; revisa que `glpi-db` esté en marcha y que las variables `MARIADB_USER` y `MARIADB_PASSWORD` existan dentro del contenedor (`cut -d= -f1 ~/lab/glpi/db.env` lista los nombres sin mostrar las claves).
- `scp: Could not resolve hostname c`: el comando de Windows se ejecutó dentro del servidor. Sal con `exit` y vuelve a ejecutarlo en el portátil. Si se creó una carpeta de nombre raro en el servidor, bórrala con `rmdir ~/'C:UsersUSUARIObackups-srv-lab'`.
- La copia de seguridad contiene contraseñas y claves: trátala como un secreto, no la subas a repositorios ni a nubes públicas.

## Gestión de contraseñas y secretos

Usa un gestor de contraseñas con una contraseña maestra larga y verificación en dos pasos; Bitwarden en su plan gratuito es la opción recomendada porque sincroniza portátil, celular y tablet.

| Opción | Costo | A favor | Límites |
| --- | --- | --- | --- |
| Bitwarden Free | Gratis | Contraseñas ilimitadas en dispositivos ilimitados, apps móviles y extensión de navegador, código abierto y auditado | Funciones avanzadas (autenticador integrado, reportes de salud de la bóveda, adjuntos) requieren Premium, unos 1,65 USD al mes según una comparación de junio de 2026 |
| KeePassXC | Gratis | Archivo local, sin cuenta ni nube | Tú resuelves la sincronización, las copias y el celular; sin recuperación si pierdes la contraseña maestra |

Fuentes: [guía de gestores gratuitos](https://itechguides.com/best-free-password-managers-for-2026-bitwarden-proton-pass-apple-passwords-and-more), [Bitwarden frente a KeePassXC](https://toolradar.com/compare/bitwarden-vs-keepassxc). Los planes y límites cambian por país y fecha: revisa la página oficial del proveedor antes de registrarte. 1Password ofrece solo una prueba de 14 días y Dashlane retiró su plan gratuito el 1 de julio de 2026.

Reglas del laboratorio:

1. **Contraseña maestra:** una frase larga (cuatro palabras al azar o más) que no uses en ningún otro sitio. No dependas de recuperarla: anótala en papel y guárdala en un lugar seguro.
2. **Verificación en dos pasos** en la cuenta del gestor, con una app autenticadora; imprime los códigos de recuperación y guárdalos aparte.
3. **Una entrada por servicio:** router, Cockpit, GLPI (con tu usuario administrador propio), claves de API y cuentas de empleo. Deja que el gestor genere contraseñas largas y únicas.
4. **Archivos con secretos en el servidor:** los `.env` de `~/lab/glpi` llevan `chmod 600`, y los del router de IA igual; nunca se suben a Git (van en `.gitignore`).
5. **Llave privada SSH (`LAB`):** no va a repositorios ni a la nube pública. Guarda una copia en un USB o lugar seguro y ponle frase de paso.
6. **Claves de API:** una por proyecto y con límite de gasto si el proveedor lo permite; si se filtra, se revoca y se crea otra.
7. **Nunca pegues contraseñas, claves ni datos sensibles en el chat de una IA.**

## Problemas frecuentes y cómo corregirlos

Estos son los fallos que aparecieron o que conviene esperar en este montaje, con su causa y su arreglo; cada sección anterior detalla más los suyos.

| Síntoma | Causa probable | Solución |
| --- | --- | --- |
| El instalador dice "No hay dispositivos de red disponibles" | El sistema no ve el chip de red | Comprobar con `ip link` y `lspci`; instalar la tarjeta PCI o usar un adaptador USB-Ethernet con chip Realtek RTL8153 o ASIX AX88179 |
| `ata6: link is slow to respond` al arrancar | Puerto SATA sin disco | Es normal; si pasa de 3 o 4 minutos, revisar cables SATA |
| Al reiniciar vuelve el instalador | El USB sigue conectado | Retirarlo y reiniciar |
| `Unit file cockpit.socket does not exist` | La instalación de paquetes se canceló (por pedir `htop`) | Repetir la instalación sin `htop` |
| `-bash: ssh-ed25519: command not found` | Se pegó la llave sin el comando `echo` | Repetir el paso con `echo '...' >> ~/.ssh/authorized_keys` |
| SSH sigue pidiendo contraseña con la llave | Permisos, etiqueta SELinux o llave mal pegada | `chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys`, `restorecon -R ~/.ssh`, `sudo journalctl -u sshd -n 20` |
| La contraseña SSH sigue funcionando tras endurecer | No se recargó el servicio | `sudo sshd -T` para ver los valores y `sudo systemctl reload sshd` |
| No se puede entrar por SSH y se perdió la llave | Sin acceso por contraseña | Teclado y monitor en el servidor; por eso se guarda una copia de la llave |
| Enlace a 100 Mb/s | Puerto del router o switch Fast Ethernet, o cable | Otro puerto, conexión directa, cable categoría 5e o 6 |
| Aviso "RTC in local TZ" en `timedatectl` | Reloj de hardware en hora local | `sudo timedatectl set-local-rtc 0` |
| `systemctl --user` da error | Sesión abierta con `sudo su` | Entrar con `ssh srv-lab` como `fabian` |
| Los contenedores no arrancan tras reiniciar | Linger sin activar | `sudo loginctl enable-linger fabian` |
| GLPI no carga en el navegador | Instalación en curso o puerto cerrado | `podman logs glpi`, esperar unos minutos y revisar `sudo firewall-cmd --list-ports` |
| Un archivo Quadlet quedó mezclado al pegarlo | Pegar bloques largos en la terminal | Revisar con `cat` y volver a pegar por trozos |
| Poca RAM disponible | Demasiados servicios para 4 GB | `free -h` y `podman stats --no-stream`; no instalar más hasta ampliar la RAM |

Antes de pedir ayuda, reúne: qué comando ejecutaste, el mensaje de error completo y la salida de `journalctl --user -u <servicio> -n 30` (o `sudo journalctl -u <servicio> -n 30` si es un servicio del sistema).

## Integración multi-IA y uso seguro de skills

El router en Python envía cada tarea al modelo más adecuado y barato, empieza con modelos gratuitos y deja Claude y ChatGPT por API como opción de pago, para ahorrar tokens sin perder calidad.

Estado: el proyecto `multi-ia-router` (versión 2) está entregado y verificado con respuestas simuladas, pero **aún no se probó con claves reales** ni se instaló en el portátil o en el servidor.

| Pieza | Qué hace |
| --- | --- |
| `config.yaml` | Lista de modelos, cadenas de respaldo por tipo de tarea y ajustes |
| `router.py` | Clasifica la tarea (simple, compleja, larga, crítica), elige el modelo y prueba el siguiente si uno falla |
| `historial.py` | Guarda la conversación en archivos de texto; a la IA solo se envían las notas, un resumen y los últimos 6 mensajes |
| `costs.py` | Registra llamadas, tokens y costo en `logs/usage.jsonl` |
| `main.py` | Comandos: `python main.py chat`, `--estado`, `--costos`, `--revisar`, `--critica` |
| `.env` | Claves de API (secreto, fuera de Git) |

Puntos clave:

- Las suscripciones de chat (ChatGPT Go, Claude Pro) **no sirven para programas**. Los programas usan APIs, un servicio aparte que se paga por uso.
- Los planes gratuitos (Gemini y Groq) tienen límites que cambian; revisa su documentación antes de depender de ellos. Pueden usar las consultas para mejorar sus modelos, así que **no envíes** contraseñas, direcciones IP internas, cédulas ni datos de terceros.
- Instalación en Windows: Python 3.12 o superior, `python -m venv venv`, `venv\Scripts\activate`, `pip install -r requirements.txt` y copiar `.env.example` a `.env` con las claves. Si falla con Python 3.14, instala 3.12 y crea el entorno con `py -3.12 -m venv venv`.

**Skills y plugins de Claude.** Una skill es una carpeta con instrucciones (`SKILL.md`) y, a veces, scripts que Claude carga para una tarea concreta. Se instalan en Claude.ai desde Customize → Skills y en Claude Code en `~/.claude/skills/`. Para tus objetivos (postulaciones, priorizar cursos y documentación) bastan las skills incluidas de Word, PDF, Excel y PowerPoint, más skills **propias** y sencillas.

Antes de instalar una skill de terceros, aplica esta lista (basada en la [guía de seguridad de Anthropic para Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise), que pide tratarlas con el mismo rigor que instalar software en un servidor de producción):

1. **Origen:** prefiere las de Anthropic o repositorios oficiales; desconfía de autores anónimos.
2. **Lee todo:** abre `SKILL.md` y cada script; entiende qué archivos y comandos toca.
3. **Señales de alerta:** instrucciones de ignorar reglas de seguridad, de ocultarte acciones, de enviar datos a servicios externos o de usar tus cuentas conectadas sin preguntar; código ofuscado.
4. **Prueba aislada:** primero con datos de ejemplo, sin conectores ni claves.
5. **Sin secretos dentro:** nunca pongas claves ni contraseñas en una skill; viajan con ella al compartirla.
6. **Versión fija y registro:** guarda una copia local y anota nombre, origen, fecha y versión de cada skill aprobada en GLPI, como cualquier otro software del laboratorio.

Contexto: el escaneo automático de skills existe solo en organizaciones Enterprise, y las skills subidas por API no se escanean; por eso la revisión manual es la defensa principal. Una [guía de verificación](https://woolen-browser-ca4.notion.site/SOP-How-to-Check-If-a-Claude-Skill-Is-Safe-Before-You-Install-It-381d989200bc802b929bf305467bb4a2) cita investigaciones según las cuales más de una de cada cuatro skills públicas tiene al menos una vulnerabilidad; trátalo como orientación, no como cifra exacta.

**Conectores** (acceso a correo, Drive, calendario): conecta solo los que uses, con el mínimo permiso (solo lectura si basta) y revísalos o revócalos desde los ajustes de la cuenta.

**Postulaciones y cursos.** La IA puede analizar una oferta, señalar qué te falta y proponer cómo adaptar tu hoja de vida, y priorizar tus cursos según las brechas detectadas. Revisa siempre el texto final antes de enviarlo y no incluyas datos personales sensibles en planes gratuitos. Para priorizar cursos falta la lista (ver pendientes).

## Pendientes, próximos pasos y datos por completar

La primera etapa está casi cerrada: GLPI ya tiene usuario propio y la copia de la base de datos se restauró con éxito; faltan automatizar las copias, versionar la documentación y el resto de pendientes, por fases según la RAM disponible.

Pendientes inmediatos:

- [x] Crear un usuario administrador propio en GLPI, comprobarlo y desactivar `glpi`, `tech`, `normal` y `post-only`.
- [x] Hacer la primera copia de seguridad, probar la restauración de la base de datos y copiar los archivos al portátil.
- [x] Reservar la IP `192.168.1.37` en el router para la MAC `00:14:d1:xx:xx:xx`.
- [ ] Probar la restauración del volumen de archivos de GLPI (con la clave de cifrado) en un volumen de prueba.
- [ ] Automatizar la copia de seguridad con un temporizador de systemd.
- [ ] Crear los repositorios de GitHub (documentación pública y privado para lo funcional) y subir la documentación.
- [ ] Activar `Power On By PCI` en la BIOS (Wake on LAN con la tarjeta PCI).
- [ ] Probar otro cable o puerto para subir de 100 Mb/s a 1 Gb/s.
- [ ] Gestor de contraseñas: configurarlo con verificación en dos pasos y guardar las entradas del laboratorio.

Siguientes fases:

- [ ] Registrar los equipos del laboratorio como activos en GLPI y crear un ticket de cambio real (por ejemplo, "cambio a IP fija").
- [ ] Instalar el router multi-IA en el portátil y probarlo con claves gratuitas.
- [ ] Subir la RAM a 8 GB o más (módulo DDR3 de escritorio de 1,5 V).
- [ ] Poner HTTPS con un proxy inverso.
- [ ] Instalar por fases Nagios, MeshCentral, Nextcloud y, al final, OpenVAS (necesita 4 GB de RAM o más).
- [ ] Subir GLPI de la versión 11 a la 12 como ejercicio de gestión de cambios.

Datos por completar para que la guía sea autosuficiente:

- Modelo del router y de sus puertos LAN (para saber si el enlace a 1 Gb/s es posible).
- Capturas de pantalla de cada paso importante (BIOS, instalador, Cockpit, GLPI).
- Lista de cursos en los que estás inscrito: nombre, plataforma, horas, avance y fecha límite, para priorizarlos con ayuda de IA.
- Política de contraseñas y de copias que quieras adoptar (frecuencia y dónde se guardan).

Fuentes consultadas:

- [Ficha de la placa ASUS P8H61-M LX2 R2.0](https://dlcdnet.asus.com/pub/ASUS/mb/LGA1155/P8H61-M_LX2_R20/E7377_P8H61-M_LX2_R2.pdf)
- [Run GLPI with Docker (glpi-project.org)](https://www.glpi-project.org/en/run-glpi-with-docker/)
- [Versiones de GLPI](https://github.com/glpi-project/glpi/releases)
- [Etiquetas de la imagen glpi/glpi](https://hub.docker.com/r/glpi/glpi/tags)
- [Guía de gestores de contraseñas gratuitos](https://itechguides.com/best-free-password-managers-for-2026-bitwarden-proton-pass-apple-passwords-and-more)
- [Bitwarden frente a KeePassXC](https://toolradar.com/compare/bitwarden-vs-keepassxc)
- [Skills for enterprise: revisión de seguridad (Anthropic)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise)
