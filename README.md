# Live IR — Falsa Bandera en `4geeks-server-lab`

**Respuesta a incidentes en caliente sobre un servidor Linux en producción, donde lo que parecía una intrusión externa resultó ser un compromiso interno con evidencia fabricada.**

---

## Descripción

Este repositorio documenta la respuesta completa a un incidente de seguridad en `4geeks-server-lab`, un Ubuntu Server 20.04 LTS con servicios Apache, vsftpd, SSH y agente Wazuh. El requisito operativo del caso fue mantener el servidor encendido y en servicio durante toda la investigación, por lo que el trabajo se resolvió bajo metodología **Live Incident Response** en lugar de análisis forense sobre imagen de disco.

La superficie del incidente mostraba todos los indicadores de un ataque externo: dos cuentas backdoor, credenciales en texto plano, un dropper, tráfico de fuerza bruta por SSH y un cronjob de exfiltración corriendo cada quince minutos como root. El cruce de `auth.log`, registros de sesiones `sudo` y metadatos de archivo demostró otra cosa: la totalidad de los artefactos fue construida desde la cuenta administrativa `sysadmin`, operando en consola local, dentro de una ventana ciega de noventa y nueve minutos abierta con una shell interactiva de root.

Proyecto final (Fase 3) de la especialización Blue Team / SOC de 4Geeks Academy, en convenio con UTEC.

---

## Escenario

| Elemento | Detalle |
|---|---|
| Host | `4geeks-server-lab` — Ubuntu Server 20.04 LTS |
| Servicios | Apache, vsftpd, SSH, agente Wazuh |
| Modalidad | Live IR — servidor encendido y operativo |
| Fases | 1. Reconocimiento y recolección · 2. Remediación · 3. Informe |
| Cuenta administrativa | `sysadmin` (única con sudo) |

---

## Metodología

**Live IR frente al forense tradicional.** El forense clásico trabaja sobre una imagen bit a bit del disco con el sistema apagado, lo que maximiza la integridad de la evidencia a costa de interrumpir el servicio y perder toda la evidencia volátil. Live IR invierte la prioridad: se acepta que cada comando modifica el estado del equipo a cambio de preservar la disponibilidad y capturar procesos, conexiones y tareas programadas que el apagado destruiría. Ese criterio atraviesa todo el caso, incluidas las decisiones de hardening.

**Marcos aplicados:**

- **SANS PICERL** como ciclo operativo. Las fases 1 y 2 cubren de Identificación a Recuperación; el informe cumple la etapa de Lecciones Aprendidas.
- **NIST SP 800-61 Rev. 3** (vigente desde abril de 2025, fecha en que el NIST retiró la Rev. 2), que integra la respuesta a incidentes dentro de la gestión de riesgo alineada al CSF 2.0.
- **RFC 3227** para el orden de volatilidad: primero procesos activos, conexiones y tareas programadas; después cuentas, logs de autenticación y archivos en disco.

**Preservación.** Toda acción destructiva de la Fase 2 estuvo precedida por la copia del artefacto a `/root/evidencia-incidente/` con hash SHA-256 calculado antes de la eliminación.

---

## Hallazgos

### 1. Persistencia y exfiltración

Tarea programada no autorizada en `/etc/cron.d/sys-maintenance`, ejecutando `/usr/local/bin/backup2.sh` cada quince minutos como root. El script empaqueta `/etc/passwd` y lo envía por HTTP POST a `192.168.1.100:8080/upload`. Al vivir en `/etc/cron.d/` sobrevive reinicios y no depende de ninguna sesión de usuario, y el nombre imita al de una tarea legítima de mantenimiento. El contraste de fechas lo delata: el `logrotate.sh` legítimo es del 21 de junio, `backup2.sh` del 23.

### 2. Cuentas no autorizadas

Dos cuentas ajenas a la operación legítima, ambas con `/bin/bash`: `hacker` (UID 1002) y `reports` (UID 1001). Ninguna pertenece al grupo sudo ni figura en sudoers. La cuenta `hacker` nunca registró un login exitoso y fue el blanco declarado de una campaña de fuerza bruta desde `192.168.1.103`. El home de `reports` apareció poblado de señuelos: un `.note` propiedad de root, un `.bash_history` fabricado, un dropper `install.sh`, y archivos `backup.log` y `chat.txt` con contenido falso, más credenciales en texto plano duplicadas en `/opt/.archive/` y `/var/backups/.logs/`.

### 3. Causa raíz: compromiso interno con fabricación de evidencia

La revisión de `auth.log` y de las sesiones sudo ubicó todo el montaje en la cuenta `sysadmin` desde `tty1`, sin exploit ni escalada de privilegios. A las 15:02:15 del 23 de junio se abre una sesión sudo cuyo comando es directamente `/bin/bash`, que permanece abierta hasta las 16:41:47: noventa y nueve minutos sin registro individual de comandos. Los metadatos `stat` ubican la creación del cronjob y del script de exfiltración dentro de esa ventana ciega.

---

## Las inconsistencias que delataron el montaje

Esta es la parte interesante del caso, y el motivo por el que la hipótesis de intrusión externa no sobrevivió al análisis:

- **Cronología invertida.** La cuenta `hacker` ya existía antes del ataque de fuerza bruta que supuestamente la tenía como objetivo.
- **El log fabricado se contradice con el script real.** `backup.log` afirma haber comprimido `/etc/shadow` y subido los datos a `102.168.1.100`; el `backup2.sh` real empaqueta `/etc/passwd` y exfiltra a `192.168.1.100`. Un atacante que documenta su propio script no se equivoca de archivo ni de dirección.
- **El dropper nunca corrió.** El directorio `/tmp/.temp/` que `install.sh` crearía no existe, pese a que el historial fabricado simula haberlo ejecutado.
- **Los señuelos son propiedad de root.** El `.note` dentro del home de `reports` no fue escrito por esa cuenta sino plantado por un tercero con privilegios.
- **La técnica anti-forense se volvió autoincriminatoria.** La primera acción registrada en el historial de `sysadmin` fue `rm ~/.bash_history`. El borrado eliminó el historial previo, pero la shell reescribe el archivo al cerrar sesión: todo lo ejecutado después quedó grabado, incluido el guion completo de la fabricación.
- **El "atacante remoto" era la estación del operador.** El 21 de junio a las 19:38 falla una contraseña de root desde `192.168.1.50`; veintisiete segundos después esa misma dirección autentica correctamente como `sysadmin`.

---

## Vulnerabilidades detectadas

1. **Ausencia de auditoría sobre sesiones administrativas** — la vulnerabilidad central. Sudo registraba la invocación de cada comando pero no la actividad interna de una shell interactiva de root.
2. **Autenticación por contraseña habilitada en SSH** — superficie que hizo posible la campaña de fuerza bruta.
3. **Sistema operativo en fin de soporte** — Ubuntu 20.04 LTS, soporte estándar finalizado el 31 de mayo de 2025.
4. **Permisos de escritura sobre artefactos sensibles** — cualquier cuenta con sudo instala persistencia en `/etc/cron.d/` de forma trivial.

---

## Remediación

**Contención.** Reglas de firewall cortando los dos canales del incidente: `DENY OUT` hacia `192.168.1.100` (destino de la exfiltración) y `DENY IN` desde `192.168.1.103` (origen de la fuerza bruta).

**Erradicación.** Preservación con hash SHA-256 previa a cada eliminación:

| Artefacto | SHA-256 |
|---|---|
| `sys-maintenance` | `7efbc8efa6b30b48d276377abeb1da24c7c09d8866b4614db022cd18ee4286a6` |
| `backup2.sh` | `f679152f3b3dcea39f0df7037cd28028b353473ae1de6706c2be78d3455aa969` |

El home completo de `reports` se preservó archivo por archivo antes de eliminar la cuenta. Las dos copias de credenciales arrojaron el mismo hash (`24bfa4c4…69268e`), lo que prueba criptográficamente que eran duplicados idénticos del mismo contenido plantado y no dos hallazgos independientes.

**Recuperación.** Repetición de los chequeos clave de la Fase 1 con resultado limpio: `/etc/passwd` solo con `sysadmin` y cuentas de sistema, `/etc/cron.d/` sin artefactos, sin procesos anómalos, reglas de contención activas y los cuatro servicios legítimos respondiendo.

Dos errores propios quedaron documentados en el informe en lugar de disimularse, porque ilustran el valor de verificar cada acción destructiva: un tipeo que bloqueó dos veces la misma IP en el firewall, y una ruta en singular que dejó un directorio de señuelos sin borrar con fallo silencioso. Ambos se detectaron en la verificación posterior y se corrigieron.

---

## Fortalecimiento

**Aplicado:**

- Rotación de la contraseña de `sysadmin`, cuenta desde la que se construyó todo el compromiso.
- **Auditoría de entrada y salida en sesiones sudo** — respuesta directa a la causa raíz. Se configuró `Defaults log_input,log_output` en un archivo dedicado dentro de `/etc/sudoers.d/`, validado con `visudo -c` y con permisos `0440`. Cualquier shell interactiva de root abierta vía sudo queda grabada en su totalidad y se reproduce con `sudoreplay`.

**Recomendado, no aplicado:**

- **Deshabilitar autenticación por contraseña en SSH** (`PasswordAuthentication no`, `PermitRootLogin no`) — prioridad alta. Se documenta expresamente como recomendación y no como acción ejecutada: cortar el acceso por contraseña en un servidor que se presume en producción puede dejar sin trabajar a usuarios legítimos que aún no tengan su clave configurada, lo que contradice el mismo principio de disponibilidad que justifica el enfoque de Live IR. Se incluye la advertencia sobre precedencia de configuración en `/etc/ssh/sshd_config.d/` en instalaciones con cloud-init.
- **Actualización a 22.04 LTS** vía `do-release-upgrade`, en ventana de mantenimiento y con respaldo previo.
- Reglas de correlación en Wazuh para creación de cuentas, modificación de `/etc/cron.d/` y apertura de shells de root.

---

## Contenido del repositorio

```
.
├── README.md
├── informe-live-ir-4geeks.pdf     # Informe técnico final (Fase 3), 24 páginas
└── capturas/                      # Evidencia visual, Fases 1 y 2 (Figuras 1 a 23)
```

El informe completo desarrolla la metodología, la reconstrucción cronológica minuto a minuto y el anexo curado de veintitrés capturas de evidencia.

---

## Herramientas y comandos

`cron` · `stat` · `auth.log` · `last` · `ss` · `systemctl` · `sha256sum` · `userdel` · `ufw` · `visudo` · `sudoreplay` · Wazuh agent · VirtualBox

---

## Lección operativa

Un privilegio administrativo sin auditoría de su uso es un punto ciego capaz de ocultar las acciones más graves de un incidente. La ventana de noventa y nueve minutos de este caso no fue un descuido del atacante: fue el espacio que el propio control de auditoría dejó abierto. El caso también deja una segunda lección, menos evidente: los indicadores de compromiso no se leen de a uno, se leen cruzados. Cada artefacto por separado apuntaba a un intruso externo; la cronología entre ellos apuntaba a otra cosa.

---

## Aviso

Entorno de laboratorio controlado con fines formativos. Las direcciones IP son de rango privado y las credenciales que aparecen en la documentación corresponden a artefactos plantados dentro del escenario, sin validez fuera de él.

---

## Autor

**Gonzalo Rodríguez** — Especialización Blue Team / SOC, 4Geeks Academy en convenio con UTEC.

GitHub: [@GonzaBot](https://github.com/GonzaBot) · LinkedIn: [gonzardem](https://www.linkedin.com/in/gonzardem)
