# Laboratorio srv-lab: procedimiento paso a paso (AlmaLinux 9, Podman y GLPI)

Última actualización: 17 de octubre de 2026

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
| Temporizadores de systemd | Archivos que le dicen a systemd cuándo ejecutar un servicio, como un despertador con una tarea | Lanzan la copia de seguridad de GLPI a las 23:00 y el apagado del servidor a las 23:30 | cron, más antiguo y sin el registro integrado de systemd |
| rtcwake y Wake-on-LAN | rtcwake programa una alarma en el reloj de la placa para encender el equipo; Wake-on-LAN lo enciende con un mensaje por la red | El servidor se enciende solo a las 06:00. Encenderlo desde el celular falta por activar en la BIOS | Un enchufe programable, que apaga en seco sin cerrar el sistema |

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

**Comprobación de la memoria desde el servidor (17 de octubre de 2026).** `dmidecode` es una herramienta que lee los datos que la placa informa a la BIOS; solo lee, no cambia nada. No estaba instalada.

```bash
sudo dnf install -y dmidecode
sudo dmidecode -t baseboard
sudo dmidecode -t 16
sudo dmidecode -t memory
```

- `-t baseboard` mostró la placa: ASUS P8H61-M LX2.
- `-t 16` mostró 2 ranuras y una capacidad máxima de 32 GiB. Es una cifra genérica de la BIOS; el límite que vale es el de la ficha de ASUS, 16 GB con módulos de 8 GB.
- `-t memory` mostró la ranura DIMM0 con un módulo de 4 GB DDR3 a 1333 MT/s y la ranura DIMM1 vacía.
- **Decisión:** comprar un solo módulo de 8 GB, DDR3 de escritorio, 1333 MT/s y 1,5 V, y ponerlo en DIMM1. Con el de 4 GB quedan 12 GB. Los módulos de distinto tamaño funcionan juntos.
- **Si el módulo es de segunda:** pasar memtest86+ una pasada completa antes de usarlo.

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

Estado: **ejecutado y probado el 14 de octubre de 2026.** La base de datos se restauró en un contenedor temporal y mostró los 6 usuarios con su estado correcto. El volumen de archivos está respaldado y verificado (contiene `config/glpicrypt.key`); el 17 de octubre de 2026 también se probó importarlo en un volumen de prueba.

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

**Copia automática diaria (hecho el 15 de octubre de 2026).** Una copia que hay que acordarse de hacer a mano termina por olvidarse. Se automatiza con dos piezas: un *script* (un archivo de texto con los comandos de la copia, uno detrás de otro) y un *temporizador* de systemd que lo ejecuta solo todos los días a las 23:00 y conserva 14 días de copias.

**Paso 1: crear el script** (como `fabian`). Se guarda en `~/bin`, una carpeta para tus propios programas.

```
mkdir -p ~/bin
nano ~/bin/backup-glpi.sh
chmod 700 ~/bin/backup-glpi.sh
```

`nano` es un editor de texto de terminal: pega el contenido de abajo, guarda con `Ctrl + O` y `Enter`, y sal con `Ctrl + X`. `chmod 700` lo deja ejecutable y solo para tu usuario.

```
#!/bin/bash
# Copia de seguridad de GLPI: base de datos + volumen de datos.
# Guarda en ~/backups/glpi y borra las copias de mas de 14 dias.
set -euo pipefail
umask 077

DEST="$HOME/backups/glpi"
FECHA=$(date +%F)
DIAS=14

mkdir -p "$DEST"
rm -f "$DEST/.db.tmp" "$DEST/.data.tmp"

# 1. Base de datos (comprimida), primero a un archivo temporal
podman exec glpi-db sh -c 'mariadb-dump -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" --single-transaction --no-tablespaces "$MARIADB_DATABASE"' | gzip > "$DEST/.db.tmp"
gzip -t "$DEST/.db.tmp"
[ "$(stat -c%s "$DEST/.db.tmp")" -gt 10000 ] || { echo "ERROR: la copia de la base de datos es demasiado pequena"; exit 1; }
mv "$DEST/.db.tmp" "$DEST/glpi-db-$FECHA.sql.gz"

# 2. Volumen de archivos y configuracion (incluye la clave de cifrado)
podman volume export systemd-glpi-data -o "$DEST/.data.tmp"
tar -tf "$DEST/.data.tmp" > /dev/null
mv "$DEST/.data.tmp" "$DEST/glpi-data-$FECHA.tar"

chmod 600 "$DEST"/glpi-*
# 3. Borrar copias de mas de $DIAS dias
find "$DEST" -maxdepth 1 -name 'glpi-*' -mtime +"$DIAS" -delete

echo "Copia $FECHA correcta"
```

- `set -euo pipefail` hace que el script se detenga ante el primer error, para que nunca dé por buena una copia incompleta.
- `umask 077` deja privados los archivos nuevos. El `chmod 600` final hace falta igual, porque `podman volume export` ignora el umask y dejaba el `.tar` legible por todos (permisos 644).
- Cada copia se escribe primero en un archivo temporal (`.db.tmp`, `.data.tmp`) y solo se renombra si pasa las comprobaciones: `gzip -t` prueba el comprimido, la base debe pesar más de 10 000 bytes y `tar -tf` debe poder listar el volumen. Así una copia dañada nunca reemplaza a una buena.
- `find ... -mtime +"$DIAS" -delete` borra las copias de más de 14 días; el número está en `DIAS=14`.

**Paso 2: crear el servicio y el temporizador** (también como `fabian`, sin `sudo`). Systemd separa *qué* se ejecuta (el servicio) de *cuándo* (el temporizador). Van en `~/.config/systemd/user/`, la carpeta de systemd para tu usuario, la misma de los archivos Quadlet de GLPI.

```
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/glpi-backup.service
nano ~/.config/systemd/user/glpi-backup.timer
```

Contenido de `glpi-backup.service`:

```
[Unit]
Description=Copia de seguridad de GLPI (base de datos y volumen)

[Service]
Type=oneshot
ExecStart=%h/bin/backup-glpi.sh
```

Contenido de `glpi-backup.timer`:

```
[Unit]
Description=Copia de seguridad diaria de GLPI a las 23:00

[Timer]
OnCalendar=*-*-* 23:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Actívalo:

```
systemctl --user daemon-reload
systemctl --user enable --now glpi-backup.timer
systemctl --user list-timers --no-pager
```

- `Type=oneshot` significa que el servicio se ejecuta una vez y termina. `%h` es tu carpeta personal.
- `Persistent=true`: si el servidor estaba apagado a las 23:00, la copia se hace al encender.
- `enable --now` lo activa ahora y en cada arranque. `list-timers` debe mostrar la próxima ejecución (columna NEXT) a las 23:00.
- Funciona sin que haya una sesión abierta porque tu usuario tiene *linger* activado, lo mismo que hace arrancar GLPI solo. Compruébalo con `loginctl show-user fabian | grep Linger`: debe decir `Linger=yes`.
- La copia tarda segundos, así que termina mucho antes del apagado de las 23:30.

**Paso 3: comprobar.** La primera vez conviene lanzarla a mano y ver el resultado:

```
~/bin/backup-glpi.sh
ls -l --time-style=long-iso ~/backups/glpi
systemctl --user status glpi-backup.service --no-pager
```

Resultado correcto: `Copia AAAA-MM-DD correcta` y dos archivos nuevos con permisos `-rw-------`. La noche del 15 de octubre de 2026 las copias aparecieron solas a las 23:00.

**Ver los registros.** `journalctl --user -u glpi-backup.service` puede responder `No journal files were found`. El registro (journal) de este servidor era volátil, se borraba en cada reinicio, y ahora el servidor se reinicia cada noche. Se hizo persistente con:

```
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

Para los servicios de usuario también sirve `journalctl _SYSTEMD_USER_UNIT=glpi-backup.service --no-pager`. Si la pantalla se queda congelada con un `:` o `(END)` abajo, es el paginador `less`: pulsa `q`. Con `--no-pager` no aparece.

**Seguridad de las copias.** El `.tar` contiene `config/glpicrypt.key`, `config/config_db.php` (datos de conexión a la base de datos) y las llaves `oauth.pem` y `oauth.pub`. Mantén los permisos en 600, no las subas a Git (el `.gitignore` lo evita) y, si las sacas de casa, que sea cifradas.

**Límite actual.** Desde el 17 de octubre de 2026 las copias se guardan en un disco aparte (bloque siguiente): protegen contra la falla del SSD, pero siguen dentro del mismo equipo, así que no protegen contra un incendio, un robo o una sobretensión. Una copia cifrada fuera del servidor sigue en la lista de pendientes.

**Copias en un disco aparte (hecho el 17 de octubre de 2026).** El servidor tiene un disco mecánico WD de 1 TB (`sdb`) con cuatro particiones NTFS de un Windows anterior: una reservada del sistema, una de 160,9 GB, `DATOS` (762,8 GB) y `Camara` (7,2 GB). Solo se formateó `sdb4` (`Camara`); `DATOS` y las otras quedaron intactas y sin montar. Las copias ocupan poco (17 MB en cuatro días, unos 60 MB en los 14 días que se conservan), así que 7,2 GB sobran.

**Paso 1: confirmar el objetivo** (solo lee). Debe mostrar `sdb4`, `7.2G`, `Camara` y ningún punto de montaje.

```
lsblk -o NAME,SIZE,LABEL,MOUNTPOINTS /dev/sdb4
```

**Paso 2: formatear solo `sdb4`** (borra su contenido para siempre). `-f` es necesario porque ya tenía NTFS; `-L backups` le pone nombre.

```
sudo mkfs.xfs -f -L backups /dev/sdb4
```

**Paso 3: montarlo y dejarlo fijo al reiniciar.** Se usa el UUID (el identificador único de la partición) porque `sdb4` puede cambiar de nombre entre arranques.

```
sudo mkdir -p /srv/backups
UUID=$(sudo blkid -s UUID -o value /dev/sdb4)
echo "UUID=$UUID /srv/backups xfs defaults,nofail,x-systemd.device-timeout=10s 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown fabian:fabian /srv/backups
chmod 700 /srv/backups
df -h /srv/backups
```

- `nofail` hace que el servidor arranque aunque el disco falle; `x-systemd.device-timeout=10s` evita esperar más de 10 segundos.
- `mount -a` prueba la línea de `/etc/fstab` ahora; un error aparece aquí y no al reiniciar.
- Resultado: 7,2 GB de tamaño, 84 MB usados por la estructura del sistema de archivos.

**Paso 4: pasar las copias y cambiar el script.** Las copias viejas se quedan en `~/backups/glpi` hasta confirmar que todo funciona.

```
cp -a ~/backups/glpi /srv/backups/
sed -i 's|^DEST="\$HOME/backups/glpi"|DEST="/srv/backups/glpi"|' ~/bin/backup-glpi.sh
sed -i '/^umask 077/a mountpoint -q /srv/backups || { echo "ERROR: el disco de copias no esta montado"; exit 1; }' ~/bin/backup-glpi.sh
sed -i 's|Guarda en ~/backups/glpi|Guarda en /srv/backups/glpi|' ~/bin/backup-glpi.sh
grep -n 'DEST=\|mountpoint\|Guarda' ~/bin/backup-glpi.sh
```

- `sed -i` edita el archivo en su sitio. El segundo comando inserta una comprobación después de `umask 077`: si el disco no está montado, el script se detiene en vez de llenar el SSD en silencio.
- El `grep` debe mostrar el comentario con la ruta nueva, la línea `mountpoint -q ...` y `DEST="/srv/backups/glpi"`.

**Paso 5: probar una copia manual.**

```
~/bin/backup-glpi.sh
ls -lh /srv/backups/glpi
```

Resultado: «Copia 2026-10-17 correcta» y los archivos de los días 14 al 17 en el disco nuevo. El temporizador de las 11:00 p. m. usa este mismo script, así que desde esa noche guarda allí.

**Problemas posibles.**

- *El script dice «ERROR: el disco de copias no esta montado»:* revisa con `lsblk` que el disco esté conectado y corre `sudo mount -a`.
- *El servidor tarda en arrancar:* revisa que la línea de `/etc/fstab` tenga `nofail` y el tiempo de espera.
- *`mkfs.xfs` se niega a formatear:* falta `-f`; verifica antes con el Paso 1 que el dispositivo sea el correcto.

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

**Probar también el volumen de archivos, con una copia nueva (hecho el 17 de octubre de 2026).** La prueba anterior solo comprobó la base de datos. Esta repite la base con una copia hecha ese mismo día (que ya incluye tu usuario administrador propio) y además comprueba que el volumen de archivos se puede importar. Todo ocurre en un contenedor y un volumen de prueba, sin tocar GLPI.

**Paso 1: hacer una copia nueva y comprobar que está sana.**

```
~/bin/backup-glpi.sh
FECHA=$(date +%F)
ls -l --time-style=long-iso ~/backups/glpi/*$FECHA*
gzip -t ~/backups/glpi/glpi-db-$FECHA.sql.gz && echo "BD comprimida OK"
tar -tf ~/backups/glpi/glpi-data-$FECHA.tar | head -5
```

Resultado esperado: `Copia AAAA-MM-DD correcta`, los dos archivos con `-rw-------`, `BD comprimida OK` y, en la lista del tar, `config/`, `config/config_db.php`, `config/glpicrypt.key` y las llaves `oauth`.

**Paso 2: crear una base de datos temporal.**

```
IMG=$(podman inspect glpi-db --format '{{.ImageName}}')
podman run -d --name restore-test-db -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=1 -e MARIADB_DATABASE=glpi_test $IMG --innodb-buffer-pool-size=64M --max-connections=10
until podman logs restore-test-db 2>&1 | grep -q "init process done"; do sleep 2; done
until podman exec restore-test-db mariadb -uroot -e "SELECT 1" >/dev/null 2>&1; do sleep 2; done
echo "BD de prueba lista"
```

- La primera línea toma el nombre de la imagen de tu base de datos real, para probar con la misma versión de MariaDB.
- `MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=1` deja la base sin contraseña. Es aceptable solo aquí, porque no se publica ningún puerto (no lleva `-p`), solo se alcanza con `podman exec` desde tu usuario y se borra a los pocos minutos.
- `--innodb-buffer-pool-size=64M --max-connections=10` limitan la memoria; el servidor tiene 4 GB y GLPI sigue corriendo.
- Los dos `until` esperan a que MariaDB termine de arrancar. En el primer arranque se reinicia sola una vez, y importar antes puede fallar. Si pasan más de 2 minutos, pulsa `Ctrl + C` y revisa `podman logs restore-test-db`.

**Paso 3: restaurar la base y revisarla.**

```
zcat ~/backups/glpi/glpi-db-$FECHA.sql.gz | podman exec -i restore-test-db mariadb -uroot glpi_test
podman exec restore-test-db mariadb -uroot glpi_test -e "SELECT COUNT(*) AS tablas FROM information_schema.tables WHERE table_schema='glpi_test'; SELECT name, is_active FROM glpi_users ORDER BY name;"
```

Resultado correcto (17 de octubre de 2026): 442 tablas y los 6 usuarios, con `Fabian` y `glpi-system` en 1 y `glpi`, `normal`, `post-only` y `tech` en 0. Es lo mismo que muestra GLPI en Administración → Usuarios.

**Paso 4: restaurar el volumen de archivos.** Es el proceso inverso al de la copia.

```
podman volume create restore-test-data
podman volume import restore-test-data ~/backups/glpi/glpi-data-$FECHA.tar
podman run --rm --entrypoint ls -v restore-test-data:/v:ro $IMG -la /v
```

Resultado correcto: aparecen las carpetas `config`, `files`, `logs` y `marketplace`, con dueño `www-data` (el usuario con el que corre GLPI dentro de su contenedor). `--entrypoint ls` cambia el programa de arranque de la imagen por `ls`, solo para listar, y `:ro` monta el volumen en solo lectura.

**Paso 5: limpiar.**

```
podman rm -f restore-test-db
podman volume rm restore-test-data
podman ps
```

`podman ps` debe mostrar solo `glpi` y `glpi-db`: el servidor queda como estaba.

**Qué no cubre esta prueba.** No arrancó una segunda copia de GLPI con estos datos, ni se probó la restauración real sobre producción: el bloque «Restaurar de verdad» sigue sin probarse en producción.

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

## Git y GitHub: versionar la documentación y las configuraciones

Estado: **ejecutado el 15 de octubre de 2026.** Hay dos repositorios en GitHub, uno público con la documentación y otro privado con las configuraciones funcionales del servidor.

**Qué es cada cosa.** Git es un programa que se instala en tu portátil y guarda el historial de cambios de tus archivos: quién cambió qué, cuándo y por qué. GitHub es el sitio web donde se publica ese historial. Con Git puedes volver a una versión anterior si algo sale mal; con GitHub tienes una copia fuera de tu casa y una vitrina que un reclutador puede abrir.

**Por qué dos repositorios.** Lo que se muestra al mundo y lo que describe cómo está armado tu servidor no deben mezclarse.

| Repositorio | Visibilidad | Qué contiene | Qué nunca contiene |
| --- | --- | --- | --- |
| `srv-lab-docs` | Público | README y la guía del laboratorio, ya revisada | Contraseñas, llaves, correos, direcciones MAC |
| `srv-lab-privado` | Privado | Archivos Quadlet, scripts y su README | Contraseñas, archivos `.env`, llaves, copias de seguridad |

**Paso 1: instalar Git** (en el portátil, con PowerShell, no dentro de `ssh srv-lab`).

```
git --version
winget install --id Git.Git -e --source winget
```

- El primer comando comprueba si ya está instalado. Si responde con un número de versión, omite el segundo.
- `winget` instala desde el almacén oficial de Microsoft. Si pide aceptar los términos de la fuente, escribe `Y`.
- Al terminar, **cierra PowerShell y ábrelo de nuevo**. La ventana vieja no se entera del programa nuevo.
- Resultado esperado: `git version 2.xx.x.windows.x`.

**Paso 2: decirle a Git quién eres.** Cada cambio queda firmado con un nombre y un correo. Como el repositorio público lo puede ver cualquiera, se usa el correo privado de GitHub (`noreply`) y no tu correo real.

1. En GitHub: foto de perfil → **Settings** → **Emails**. Activa **Keep my email addresses private** y **Block command line pushes that expose my email**.
2. Copia la dirección `noreply` que aparece, con la forma `NUMERO+USUARIO@users.noreply.github.com`.
3. En PowerShell, cambia los valores de ejemplo por los tuyos:

```
git config --global user.name "Tu Nombre"
git config --global user.email "NUMERO+USUARIO@users.noreply.github.com"
git config --global init.defaultBranch main
git config --global --list
```

- Los comandos de configuración no muestran nada; el último lista lo guardado.
- `init.defaultBranch main` hace que la rama principal se llame `main`, el nombre estándar de GitHub.

**Paso 3: crear una llave SSH solo para GitHub.** Con la llave, GitHub sabe que eres tú sin pedir contraseña. Es **distinta** de la llave `LAB` del servidor: si una se filtra, la otra sigue segura.

```
ssh-keygen -t ed25519 -C "github-usuario" -f $HOME\.ssh\github_ed25519
```

- `-t ed25519` es el tipo de llave, moderno y seguro. `-C` es una etiqueta para reconocerla. `-f` es el nombre del archivo, para no pisar otras llaves.
- Pide una **passphrase**: una contraseña que protege el archivo por si alguien lo copia. Al escribirla no se ve nada; es normal. Se pide dos veces. Guárdala en el gestor de contraseñas.
- Se crean dos archivos: `github_ed25519` (la **privada**, no se comparte con nadie) y `github_ed25519.pub` (la **pública**, la que va a GitHub).
- Para comprobar si quedó con passphrase: `ssh-keygen -y -f $HOME\.ssh\github_ed25519`. Si pide la passphrase, la tiene; imprime la llave pública.

**Paso 4: registrar la llave pública en GitHub.**

```
Get-Content $HOME\.ssh\github_ed25519.pub | Set-Clipboard
```

1. El comando copia la llave pública al portapapeles (no muestra nada).
2. En GitHub: foto → **Settings** → **SSH and GPG keys** → **New SSH key**.
3. **Title:** el nombre del equipo (por ejemplo `Portatil`). **Key type:** `Authentication Key`. **Key:** pega con `Ctrl + V` (empieza con `ssh-ed25519`).
4. **Add SSH key** y confirma con tu contraseña o con el 2FA si lo pide.

**Paso 5: probar la conexión y dejarla permanente.**

```
ssh -i $HOME\.ssh\github_ed25519 -T git@github.com
```

- La primera vez pregunta `Are you sure you want to continue connecting?`. Comprueba que la huella sea `SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU` (la publica GitHub en su documentación) y escribe `yes`.
- Respuesta correcta: `Hi USUARIO! You've successfully authenticated, but GitHub does not provide shell access.` No es un error: GitHub no da terminal, solo confirma quién eres.

Sin el `-i`, SSH no encuentra la llave porque solo prueba nombres estándar como `id_ed25519`. Se arregla una vez con una nota en el archivo de configuración (**ejecútalo una sola vez**, o quedará el bloque duplicado):

```
Add-Content $HOME\.ssh\config "`nHost github.com`n  HostName github.com`n  User git`n  IdentityFile ~/.ssh/github_ed25519`n  IdentitiesOnly yes"
ssh -T git@github.com
```

Desde ahora SSH usa esa llave solo con github.com, y la llave `LAB` sigue siendo la del servidor.

**Paso 6: crear los repositorios** (en la web: **+** arriba a la derecha → **New repository**).

- `srv-lab-docs`: visibilidad **Public**, con **Add README** activado.
- `srv-lab-privado`: visibilidad **Private** (verifica este campo antes de crear), con **Add README** activado.
- Deja sin marcar **.gitignore** y **license**; el `.gitignore` se crea a mano en el paso 8.

**Paso 7: clonar** (descargar una copia del repositorio a tu portátil, conectada con el original).

```
New-Item -ItemType Directory -Force $HOME\proyectos
cd $HOME\proyectos
git clone git@github.com:USUARIO/srv-lab-docs.git
git clone git@github.com:USUARIO/srv-lab-privado.git
```

Cada clonación pide la passphrase de la llave. Resultado esperado: `Cloning into ...` y una carpeta con `README.md`.

**Paso 8: crear el `.gitignore`.** Es la lista de archivos que Git nunca debe subir, y la red de seguridad contra subir una llave o una copia de seguridad por accidente. Se crea dentro de cada repositorio:

```
@"
*.env
.env
*.key
*.pem
LAB
LAB.pub
id_*
*.sql
*.sql.gz
*.tar
backups/
"@ | Set-Content .gitignore
```

**Publicación automática (17 de octubre de 2026).** Con GitHub vinculado a Claude (Ajustes, Conectores) y la aplicación instalada solo en el repositorio `srv-lab-docs`, Claude publica la guía después de cada avance. El proceso es siempre el mismo: exportar la guía, aplicar el saneamiento, correr la revisión de fugas (si encuentra algo grave, no se publica), copiar el archivo a `docs/guia-laboratorio.md`, hacer el commit y subirlo. Solo se publica la versión saneada; el repositorio `srv-lab-privado` no se toca sin que lo pidas.

**Paso 9: preparar el contenido.** Antes de publicar algo en el repositorio público se revisa buscando datos sensibles. En la copia pública de esta guía se aplicaron estos cambios:

- La dirección MAC se reemplazó por `00:14:d1:xx:xx:xx`.
- La contraseña de prueba del ejemplo de restauración se reemplazó por `CLAVE_DE_PRUEBA`.
- El diagrama no se exporta a Markdown; se reemplazó por una línea que lo describe.

Para el repositorio privado, antes de copiar los archivos de Quadlet se contaron coincidencias de las palabras `pass`, `secret`, `token` y `key` sin mostrar su contenido (`grep -c -i -E "pass|secret|token|key" archivos`); todos dieron `0`. Los archivos `.container` solo apuntan a los `.env` por su ruta (`EnvironmentFile=%h/lab/glpi/db.env`), por eso pueden subirse sin secretos. Se trajeron con:

```
scp "srv-lab:.config/containers/systemd/*" $HOME\proyectos\srv-lab-privado\quadlet\
```

**Paso 10: guardar y subir los cambios.** Git trabaja en etapas: preparas qué cambios entran (`add`), tomas la "foto" con un mensaje (`commit`) y la envías a GitHub (`push`).

```
git status
git add .
git status
git commit -m "Describe qué cambió"
git push
```

- El primer `git status` muestra en rojo lo que cambió. Después de `git add .`, debe verse en verde. **Revisa esa lista antes de confirmar:** si aparece un `.env`, un `.sql`, un `.tar` o una llave, no sigas.
- Los avisos `LF will be replaced by CRLF` son normales en Windows: Git solo ajusta los saltos de línea.
- `git push` pide la passphrase y termina con `main -> main`.

**Paso 11: verificar en el navegador.** Abre `https://github.com/USUARIO/srv-lab-docs`: debe verse el README y la carpeta `docs`. En `srv-lab-privado` debe verse la carpeta `quadlet` y el candado con la etiqueta **Private**.

**Ciclo de trabajo.** Cada vez que cambie la guía o un archivo: exportar la guía y limpiarla de datos sensibles, copiarla a la carpeta del repositorio y repetir el paso 10. La copia pública de la guía es una foto del día en que se exportó; no se actualiza sola.

**Reglas de seguridad de los repositorios:**

1. Nunca se suben contraseñas, llaves privadas, archivos `.env`, copias de seguridad ni bases de datos.
2. Siempre se revisa `git status` en verde antes de hacer `commit`.
3. El repositorio público solo contiene lo que no te importaría que lea cualquiera, incluido un empleador.
4. Se activa la verificación en dos pasos de GitHub y los códigos de recuperación se guardan en el gestor de contraseñas.
5. Si algún secreto llega a subirse, no basta con borrarlo en un commit nuevo: queda en el historial. Hay que cambiar esa contraseña o llave de inmediato.

Problemas frecuentes de esta sección:

- `git : El término 'git' no se reconoce`: no está instalado o la ventana de PowerShell es anterior a la instalación. Instálalo y abre una ventana nueva.
- `Permission denied (publickey)`: SSH no usó la llave o GitHub no la tiene. Prueba con `ssh -i $HOME\.ssh\github_ed25519 -T git@github.com`; si funciona, falta la nota del archivo `config` (paso 5); si no, revisa en GitHub que la llave pública esté guardada (paso 4).
- `fatal: Could not read from remote repository`: la misma causa de arriba, o el nombre del repositorio está mal escrito.
- La passphrase se pide en cada operación: es esperado. Se puede evitar con el agente de SSH de Windows, pero es una mejora opcional.
- El archivo de configuración tiene el bloque `Host github.com` repetido: ábrelo con `notepad $HOME\.ssh\config` y borra uno de los dos, sin tocar el bloque de `srv-lab`.

## Apagado nocturno y encendido programado

Estado: **ejecutado el 15 de octubre de 2026 y comprobado la noche siguiente.** El servidor se apaga solo a las 23:30 y se enciende solo a las 06:00; en la primera noche real arrancó a las 06:00:20 del 16 de octubre.

El laboratorio no necesita estar encendido de madrugada, y apagarlo ahorra electricidad y desgaste. El horario de cada noche queda así: 23:00 copia de seguridad, 23:30 apagado (con un aviso de 60 segundos) y 06:00 encendido.

**Cómo se enciende un equipo apagado.** La placa tiene su propio reloj, el *RTC* (reloj de tiempo real), que sigue funcionando aunque el servidor esté apagado. El comando `rtcwake` programa una alarma en ese reloj y apaga el equipo; cuando suena la alarma, la placa enciende el servidor. Esta placa lo hizo sin cambiar nada en la BIOS.

El RTC guarda la hora en UTC, cinco horas por delante de Colombia. Por eso el registro dice `11:00:00` cuando la alarma es a las 06:00 de aquí. No es un error.

**Paso 1: crear el script** (con `sudo`, porque apagar el equipo exige permisos de administrador). Se guarda en `/usr/local/sbin`, la carpeta para programas de administración propios.

```
sudo nano /usr/local/sbin/apagar-con-alarma.sh
sudo chmod 755 /usr/local/sbin/apagar-con-alarma.sh
```

Contenido del script:

```
#!/bin/bash
# Apaga el servidor y programa su encendido a las 06:00 con la alarma del reloj (RTC).
set -euo pipefail

SALTAR=/home/fabian/no-apagar
HORA=06:00

AHORA=$(date +%s)
OBJETIVO=$(date -d "$HORA" +%s)
if [ "$OBJETIVO" -le "$AHORA" ]; then
  OBJETIVO=$(date -d "tomorrow $HORA" +%s)
fi

echo "Encendido programado para: $(date -d @"$OBJETIVO" '+%F %T')"

if [ "${1:-}" = "--prueba" ]; then
  echo "Modo prueba: no se apaga nada."
  exit 0
fi

if [ -e "$SALTAR" ]; then
  rm -f "$SALTAR"
  echo "Apagado omitido esta noche (existia $SALTAR)."
  exit 0
fi

wall "srv-lab se apaga en 60 segundos por horario. Para cancelar: sudo systemctl stop apagado-nocturno.service"
sleep 60
exec /usr/sbin/rtcwake -m off -t "$OBJETIVO"
```

- Calcula la próxima 06:00: si la de hoy ya pasó, usa la de mañana.
- Con `--prueba` solo muestra cuándo encendería, sin apagar nada.
- Si existe el archivo `/home/fabian/no-apagar`, se salta el apagado esa noche y borra el archivo, así que solo vale una vez.
- `wall` manda un aviso a todas las sesiones abiertas y `sleep 60` deja un minuto para cancelar.
- `exec rtcwake -m off -t ...` deja la alarma puesta y apaga el equipo.

**Paso 2: probar sin apagar nada.**

```
sudo /usr/local/sbin/apagar-con-alarma.sh --prueba
```

Debe responder `Encendido programado para: AAAA-MM-DD 06:00:00` y `Modo prueba: no se apaga nada.`

**Paso 3: probar que la alarma funciona de verdad.** Antes de fiarse del horario real se hizo una prueba corta, con una alarma a 3 minutos. El servidor se apaga en el acto y tiene que encenderse solo:

```
sudo rtcwake -m off -s 180
```

El servidor se encendió solo a los 3 minutos: `last reboot` mostró un arranque a las 22:49, tras apagarse a las 22:46. Si no se enciende en unos 5 minutos, pulsa el botón de encendido.

**Paso 4: crear el servicio y el temporizador** (con `sudo`, en `/etc/systemd/system/`, porque son del sistema y no de tu usuario; la copia de seguridad va en tu usuario porque Podman corre sin permisos de administrador).

```
sudo nano /etc/systemd/system/apagado-nocturno.service
sudo nano /etc/systemd/system/apagado-nocturno.timer
```

Contenido de `apagado-nocturno.service`:

```
[Unit]
Description=Apagado nocturno con encendido programado

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/apagar-con-alarma.sh
```

Contenido de `apagado-nocturno.timer`:

```
[Unit]
Description=Apagado nocturno a las 23:30

[Timer]
OnCalendar=*-*-* 23:30:00

[Install]
WantedBy=timers.target
```

Actívalo:

```
sudo systemctl daemon-reload
sudo systemctl enable --now apagado-nocturno.timer
systemctl list-timers apagado-nocturno.timer --no-pager
```

Este temporizador no lleva `Persistent=true`, a diferencia del de la copia: si el servidor estaba apagado a las 23:30, no debe apagarse apenas se encienda.

**Paso 5: comprobar a la mañana siguiente.**

```
uptime -s
last reboot | head -3
journalctl -u apagado-nocturno.service --no-pager | tail -5
ls -l --time-style=long-iso ~/backups/glpi
```

Resultado del 16 de octubre de 2026: `uptime -s` dio `2026-10-16 06:00:20`; el registro mostró `Encendido programado para: 2026-10-16 06:00:00` y `rtcwake: wakeup from "off" using /dev/rtc0 at Fri Oct 16 11:00:00 2026` (esa hora está en UTC); y la copia de las 23:00 estaba en su carpeta.

**Controlar el apagado:**

- Saltarse una noche: `touch ~/no-apagar`. El script borra el archivo después de usarlo.
- Cancelar durante el aviso de 60 segundos: `sudo systemctl stop apagado-nocturno.service`.
- Desactivar el horario: `sudo systemctl disable --now apagado-nocturno.timer`.
- Cambiar las horas: edita `HORA` en el script y `OnCalendar` en el temporizador; después `sudo systemctl daemon-reload` y `sudo systemctl restart apagado-nocturno.timer`.
- Apagarlo a mano durante el día y que se encienda a las 06:00 del día siguiente: `sudo /usr/local/sbin/apagar-con-alarma.sh` (sin `--prueba`). Un apagado normal, por Cockpit o con `poweroff`, no programa ninguna alarma: el servidor queda apagado hasta que pulses el botón o uses Wake-on-LAN.

Problemas frecuentes:

- El servidor no se encendió a las 06:00: pulsa el botón de encendido y mira `last reboot` y `journalctl -u apagado-nocturno.service --no-pager | tail -5`. Si el registro muestra la línea `rtcwake: wakeup from "off"`, la alarma sí se programó y el fallo está en el equipo o en la BIOS; revísala la próxima vez que conectes un monitor.
- El registro muestra la hora cinco horas adelantada: es el RTC, que trabaja en UTC. La hora de Colombia es UTC menos 5.
- La terminal se queda con `(END)` o `:` al final: es el paginador; pulsa `q`.

## Wake-on-LAN y control desde el celular

Estado: **configurado en el sistema el 15 de octubre de 2026; la primera prueba desde el celular falló y falta revisar la BIOS.** El servidor ya se enciende solo a las 06:00 (sección anterior); esto añade poder encenderlo a cualquier hora desde el celular, dentro de la red de casa.

**Qué es.** Wake-on-LAN (WoL) enciende un equipo apagado con un «paquete mágico»: un mensaje por la red local que lleva la dirección MAC del equipo repetida 16 veces. La tarjeta de red lo escucha aunque el servidor esté apagado. Para eso hacen falta tres cosas: que la placa le siga dando corriente, que el sistema operativo deje la función activada y que la BIOS permita encender por esa tarjeta. Solo funciona dentro de la red de casa. No hay que abrir ningún puerto del router para esto; para encenderlo desde fuera se usaría una VPN, más adelante.

**Paso 1: ver el estado de la tarjeta** (en el servidor).

```
sudo ethtool enp5s0 | grep Wake-on
```

`Wake-on: d` significa desactivado y `Wake-on: g` significa activado con paquete mágico. Al principio estaba en `d`.

**Paso 2: activarlo.** Se usan dos comandos: uno para ahora y otro para que no se pierda al reiniciar.

```
sudo ethtool -s enp5s0 wol g
sudo nmcli connection modify enp5s0 802-3-ethernet.wake-on-lan magic
nmcli -g 802-3-ethernet.wake-on-lan connection show enp5s0
```

- `ethtool -s ... wol g` cambia la tarjeta de inmediato, pero el cambio se pierde al reiniciar.
- `nmcli connection modify` guarda el ajuste en NetworkManager, que lo aplica en cada arranque. `enp5s0` es también el nombre de la conexión (se ve con `nmcli -t -f NAME,DEVICE connection show`).
- El último comando debe responder `magic`. Después de un reinicio, `ethtool` debe seguir mostrando `Wake-on: g`.

**Paso 3: BIOS (pendiente).** La placa corta la corriente de la tarjeta PCI al apagarse, salvo que la BIOS lo permita. Con un monitor conectado, entra con `Supr` al encender, ve a Advanced → APM y pon `Power On By PCI Devices` (a veces llamado PCIE) en Enabled. Si existe `ErP Ready`, déjalo en Disabled, porque corta la corriente de reposo. Guarda con `F10`. Se revisará en el próximo apagado.

**Paso 4: la aplicación en el celular.** En Android (vivo Y33s) se eligió *Wake On Lan* del desarrollador MR-Webb, de Google Play. En su ficha, el desarrollador declara que no comparte datos con terceros; pesa 3,8 MB, tiene más de 1 millón de descargas y 3,9 estrellas. Se descartó *WoLOn*, que tiene mejor nota (4,7) pero declara compartir con terceros el identificador del dispositivo, los registros de fallas y la actividad en la app, y ofrece compras dentro de la app. Con la regla de seguridad primero, pesa más lo que declara que la nota.

Se agrega un dispositivo con estos datos (en la app, botón + y «Añadir manualmente»):

| Campo en la app | Valor |
| --- | --- |
| Nombre del dispositivo | El que quieras, por ejemplo `srv-lab` |
| Dirección MAC | `00:14:d1:xx:xx:xx` (la de la tarjeta TRENDnet) |
| Hostname/IP/Dirección Broadcast | `192.168.1.255` |
| IP del dispositivo | `192.168.1.37` (sirve para que la app muestre si está encendido) |
| Puerto | `9` |
| Contraseña SecureOn | Vacía; la tarjeta no la usa |
| Red WiFi | El nombre de tu WiFi de casa |
| Comprobación de ping y NetBIOS | Enabled |

En el escaneo de red, la app muestra «Dirección MAC no encontrada» para casi todos los equipos. Es normal: Android reciente no deja leer la MAC de otros equipos. Por eso se escribe a mano.

**Paso 5: probar.** El celular debe estar en el WiFi de casa, no en datos móviles, y el servidor ya apagado (por ejemplo, después de las 23:30). Toca el botón de despertar de la app, espera 1 o 2 minutos y comprueba con `ping 192.168.1.37 -t` desde el portátil o mirando el punto de estado de la app: verde es encendido, rojo es apagado.

Resultado del 15 de octubre de 2026 (23:34): el servidor no se encendió y la app lo mostró en rojo. La causa más probable es la BIOS del paso 3, que aún no se revisa. Cuando se resuelva, se anotará aquí el resultado.

**Controlar el servidor desde el celular:**

- Apagarlo: abre Cockpit (`https://192.168.1.37:9090`) en el navegador del celular, con el celular en el WiFi de casa, y usa el botón de reinicio de arriba a la derecha → Apagar. Recuerda que un apagado así no programa la alarma de las 06:00.
- Encenderlo: con la alarma de las 06:00, con Wake-on-LAN (cuando la BIOS lo permita) o con el botón físico.

Problemas frecuentes:

- La app no enciende el servidor: confirma que `Wake-on` es `g`, que el celular está en el WiFi de casa, que la MAC y la dirección `192.168.1.255` están bien escritas y, si todo eso está bien, la BIOS (paso 3).
- Tras reiniciar, `Wake-on` vuelve a `d`: revisa que `nmcli -g 802-3-ethernet.wake-on-lan connection show enp5s0` diga `magic` y reactiva la conexión con `sudo nmcli connection up enp5s0`; la sesión SSH se corta unos segundos.
- El escaneo de la app no encuentra la MAC: es normal en Android reciente; escríbela a mano.

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

## Multi-IA para la búsqueda de empleo: diseño y bot de Telegram (17 de octubre de 2026)

Estado: **diseño definido, bot creado y token guardado; todavía no se instaló nada en el servidor** porque tiene 4 GB de RAM. La regla del sistema es que la IA prepara y tú apruebas: evalúa ofertas, adapta la hoja de vida y avisa, pero cada postulación la envías tú.

**Por qué no postula sola.** Muchos portales, entre ellos LinkedIn, restringen la automatización en sus condiciones de uso (verifica las de cada uno), y una cuenta bloqueada o un dato equivocado enviado a muchas empresas no se corrige. Con aprobación manual se conserva la calidad y se reduce el riesgo.

**Revisión del router v2 (zip).** Se leyó sin ejecutar nada, en una carpeta aislada. No contiene claves; las toma de `.env`, que Git ignora. Hallazgo de seguridad: `requirements.txt` pide `litellm>=1.50.0`, sin versión fija, y ese rango admite las versiones 1.82.7 y 1.82.8, que según lo investigado se comprometieron en marzo de 2026. Antes de instalarlo hay que fijar una versión exacta y revisada (`litellm==X.Y.Z`). Tampoco trae tareas programadas, lector de correo ni alertas: hoy solo responde cuando se le escribe.

| Pieza | Qué hace | Estado |
| --- | --- | --- |
| Lector de correo | Lee en solo lectura los correos de alertas de empleo y extrae los enlaces (API de Gmail) | Pendiente |
| Evaluador (router) | Puntúa cada oferta de 0 a 100, indica qué falta y elige el perfil de hoja de vida | Pendiente |
| Lista web | Página en el servidor, solo desde la red local o Tailscale y con usuario y contraseña; botones Postulé, Descartar y Seguimiento | Pendiente |
| Bot de Telegram | Envía avisos de ofertas, respuestas, seguimientos y alertas de estudio | Creado; falta el programa que lo atienda |
| Seguimiento | Recuerda escribir a los 5 días sin respuesta y propone cursos para cerrar brechas | Pendiente |

**Fuentes de ofertas.** Los correos de algunos portales traen solo el listado y los enlaces, sin el detalle. El sistema abrirá los enlaces públicos con pocas consultas; si un portal los bloquea, la lista web pedirá pegar el texto de la oferta. Un portal que solo funciona con sesión iniciada no se automatiza: el texto se pega a mano. No se guardan en el servidor las claves de ningún portal de empleo.

**Filtros y datos personales.** La zona, la modalidad (presencial, híbrida o remota) y el salario mínimo se configuran en un archivo local que no se publica. Una oferta por debajo del mínimo no se descarta: se marca y queda al final. Las hojas de vida son datos personales y no se suben a GitHub.

**Reglas de seguridad del bot.**

- El token del bot es una contraseña: solo vive en el servidor, nunca se pega en chats ni aparece en capturas.
- Telegram no cifra los mensajes de extremo a extremo; por el bot solo viajan títulos, puntajes y enlaces, nunca la hoja de vida ni datos personales.
- El bot responderá únicamente al identificador de chat autorizado e ignorará a cualquier otra cuenta.
- Las claves de las IAs también viven solo en el servidor; el navegador y la página web no las ven.

**Paso 1: crear el bot** (en Telegram, desde el celular). Abre un chat con `@BotFather`, escribe `/newbot`, elige un nombre para mostrar y un nombre de usuario único que termine en `bot`. BotFather entrega un token (números, dos puntos y letras): trátalo como una contraseña.

**Paso 2: si el token se muestra por error, revócalo.** El 17 de octubre de 2026 el token apareció completo en una captura de pantalla. Se corrigió escribiendo `/revoke` en BotFather, eligiendo el bot y guardando el token nuevo: el anterior deja de funcionar al instante. En adelante se recortan o tapan las capturas antes de enviarlas.

**Paso 3: guardar el token en el servidor** (como `fabian`, por `ssh srv-lab`, ejecutando cada bloque por separado). Primero una carpeta que solo tu usuario puede abrir:

```bash
mkdir -p ~/.config/srv-lab && chmod 700 ~/.config/srv-lab
```

Luego se lee el token con la entrada oculta (`-s`), para que no se vea en pantalla ni quede en el historial de comandos. Se pega el token y se pulsa Enter:

```bash
read -rsp "Pega el token y pulsa Enter: " T; echo
```

Por último se escribe en un archivo protegido y se borra la variable:

```bash
umask 077; printf 'TELEGRAM_BOT_TOKEN=%s\n' "$T" > ~/.config/srv-lab/telegram.env; unset T
wc -c ~/.config/srv-lab/telegram.env
```

- `umask 077` hace que el archivo nazca con permisos `-rw-------`; con `ls -l` debe verse así y con dueño `fabian`.
- `wc -c` cuenta los bytes: con un token completo da unos 66; si da **20**, el token quedó vacío (solo está el texto `TELEGRAM_BOT_TOKEN=` y el salto de línea).
- No abras el archivo con `cat` ni lo muestres en capturas.
- El archivo está fuera de cualquier repositorio, así que Git no puede subirlo por accidente.

Resultado del 17 de octubre de 2026: a la primera el archivo salió de 20 bytes porque el comando de lectura se ejecutó junto con el siguiente y no recibió el token; al repetir cada bloque por separado salió de 66 bytes. Después de escribir `/start` al bot, el chat queda registrado.

**Orden de construcción** (bloques de una a dos horas, con el servidor encendido entre las 6:00 a. m. y las 11:30 p. m.): 1) hojas de vida por perfil (hecho, sin servidor); 2) el programa que atiende al bot y la lista de chats autorizados; 3) lista web y evaluador, primero con ofertas pegadas a mano; 4) lector de correo con la API de Gmail en solo lectura; 5) seguimiento y alertas de estudio. El lector de correo y las consultas corren de día, porque el servidor está apagado de 11:30 p. m. a 6:00 a. m. y los avisos pueden llegar con retraso.

Problemas posibles:

- *`wc -c` da 20 bytes:* el token no entró; repite el Paso 3 con cada bloque por separado.
- *El token apareció en una captura o en un chat:* `/revoke` en BotFather y token nuevo.
- *El bot no responde al `/start`:* es normal hasta que exista el programa que lo atienda.

## Pendientes, próximos pasos y datos por completar

La primera etapa está casi cerrada: GLPI ya tiene usuario propio y la copia de la base de datos se restauró con éxito; faltan automatizar las copias, versionar la documentación y el resto de pendientes, por fases según la RAM disponible.

Pendientes inmediatos:

- [x] Crear un usuario administrador propio en GLPI, comprobarlo y desactivar `glpi`, `tech`, `normal` y `post-only`.
- [x] Hacer la primera copia de seguridad, probar la restauración de la base de datos y copiar los archivos al portátil.
- [x] Reservar la IP `192.168.1.37` en el router para la MAC `00:14:d1:xx:xx:xx`.
- [x] Probar la restauración del volumen de archivos de GLPI (con la clave de cifrado) en un volumen de prueba (hecho el 17 de octubre de 2026).
- [x] Automatizar la copia de seguridad con un temporizador de systemd (hecho el 15 de octubre de 2026: todos los días a las 23:00, con 14 días de copias).
- [x] Crear los repositorios de GitHub (documentación pública y privado para lo funcional) y subir la documentación y los archivos Quadlet (hecho el 15 de octubre de 2026).
- [ ] Activar `Power On By PCI` en la BIOS (Wake on LAN con la tarjeta PCI). La primera prueba desde el celular falló el 15 de octubre de 2026; se revisará en el próximo apagado.
- [ ] Probar otro cable o puerto para subir de 100 Mb/s a 1 Gb/s.
- [ ] Gestor de contraseñas: configurarlo con verificación en dos pasos y guardar las entradas del laboratorio.

* [x] Programar el apagado a las 23:30 y el encendido a las 06:00 con la alarma del reloj de la placa (hecho el 15 de octubre de 2026).
* [ ] Guardar una copia de las copias de seguridad fuera del servidor (en el portátil o en un disco externo), con cifrado, porque hoy todas están en el mismo SSD.
* [x] Vincular GitHub con Claude, dando acceso solo al repositorio `srv-lab-docs` (hecho el 17 de octubre de 2026). La guía saneada ya se publicó; en adelante se publica después de cada avance.

- [ ] Comprar un módulo de 8 GB DDR3 de 1333 MT/s y 1,5 V, probarlo con memtest86+ e instalarlo en la ranura DIMM1.

* [ ] Confirmar que la copia de las 11:00 p. m. del 17 de octubre aparece en `/srv/backups/glpi` y, después, borrar la carpeta vieja `~/backups/glpi` del SSD.
* [ ] Repetir la prueba de restauración usando las copias del disco nuevo.
* [ ] Revisar en solo lectura qué hay en la partición `DATOS` (762,8 GB) y decidir qué se rescata y a dónde.

- [ ] Probar el token del bot (consulta `getMe`), registrar el chat autorizado y enviar un mensaje de prueba al celular.
- [ ] Fijar una versión exacta y revisada de `litellm` en el `requirements.txt` del router antes de instalarlo.
- [ ] Crear el proyecto gratuito de Google Cloud con la API de Gmail en solo lectura para el lector de correo.
- [ ] Construir la lista web de ofertas y el evaluador, primero con ofertas pegadas a mano.

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
