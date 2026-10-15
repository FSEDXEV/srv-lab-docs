# srv-lab-docs

Documentación de mi laboratorio personal de infraestructura TI: un servidor Linux montado desde cero sobre un equipo de escritorio antiguo, con servicios en contenedores, seguridad básica aplicada y cada decisión documentada paso a paso.

> Proyecto en curso. La documentación se actualiza a medida que avanzo.

## Para qué existe este laboratorio

- Practicar administración de sistemas, seguridad, gestión de servicios (ITIL) y documentación técnica.
- Dejar evidencia verificable de lo que sé hacer, no solo de lo que dice mi hoja de vida.
- Tener un entorno propio donde probar herramientas de TI sin riesgo para producción.

## Qué hay montado

| Componente | Detalle |
|---|---|
| Hardware | Equipo de escritorio reutilizado (i3, 4 GB de RAM, SSD de 240 GB) |
| Sistema operativo | AlmaLinux 9 |
| Contenedores | Podman rootless con Quadlet (servicios gestionados por systemd) |
| Aplicación | GLPI 11 (inventario y mesa de ayuda) con MariaDB 10.11 |
| Acceso remoto | SSH solo con llave, sin contraseña ni acceso directo como root |
| Seguridad | Firewall (firewalld) con puertos mínimos, SELinux en modo Enforcing, actualizaciones automáticas de seguridad |
| Administración | Cockpit (panel web) |
| Red | IP fija reservada en el router |

## Contenido

- [`docs/guia-laboratorio.md`](docs/guia-laboratorio.md): guía completa y reproducible, desde la instalación hasta GLPI, con explicación de qué hace cada herramienta y por qué se usa.

La guía cubre: instalación y red, acceso SSH por llave, firewall y SELinux, Cockpit, Podman y Quadlet, despliegue de GLPI con MariaDB, copias de seguridad con prueba de restauración, gestión de contraseñas y problemas frecuentes con su solución.

## Estado

- [x] Servidor instalado y endurecido
- [x] GLPI en contenedores, con usuario administrador propio y cuentas de fábrica desactivadas
- [x] Copia de seguridad de la base de datos y prueba de restauración
- [ ] Automatizar las copias con un temporizador de systemd
- [ ] Inventario de equipos y tickets de cambio en GLPI
- [ ] HTTPS con proxy inverso
- [ ] Monitoreo (Nagios), administración remota (MeshCentral) y almacenamiento (Nextcloud)

## Seguridad de este repositorio

Este repositorio es público y **no contiene** contraseñas, claves privadas, archivos `.env` ni copias de seguridad. Los ejemplos usan valores de relleno. Las direcciones IP son de red privada y solo tienen sentido dentro de mi red local.

## Autor

Fabian Sequeda, ingeniero de sistemas. Perfil: <https://github.com/FSEDXEV>
