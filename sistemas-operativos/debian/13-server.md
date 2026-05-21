# Despliegue de Debian 13 Trixie server

Este documento describe las acciones necesarias para dejar listo un sistema Debian 13.5 Trixie server para las operaciones cotidianas, haciendo énfasis en la seguridad (hardening). El manual tiene un enfoque generalista, sirve como base para cualquier servidor, y como punto de partida. Luego, cada servidor específico deberá desplegar las acciones según su particularidad.

<p style="text-align: center"><img src="./assets/logo debian.png" style="width: 25%;" alt="Debian" /></p>

**Autor:** [Calú](https://github.com/calu777)  
**Fecha de inicio:** 2026-04-19  
**Última actualización:** 2026-05-20  
**Versión:** 0.3

## a. Historial de cambios

- [0.3] – 2026-05-20
  - Seguridad: URLs de plantillas migradas a SHA anclado + verificación sha256sum; nuevo hardening de sudo/pam_faillock; refuerzo SSH con   DisableForwarding; passphrase de Borg protegida con read -s.
  - Correcciones: SystemMaxUse de journald reducido a 1 G; nota sobre PASS_MIN_LEN ignorado con pam_pwquality; variable $MIPUERTO_SMTP para evitar colisión; LUKS2/Argon2id/GRUB aclarado; sed de dropbear reemplazado por secuencia robusta; fixes de numeración y typo.
  - Mejoras: nueva §12.6 con auditoría lynis; soporte NTS en chrony; pool NTP por país; nota de egreso en UFW; referencias cruzadas entre MTA, fail2ban y pam_faillock.
- [0.2] – 2026-05-16
  - Actualización de Debian 13.4 a 13.5
  - Inclusión del override en C.6.1
- [0.1] – 2026-04-28
  - Manual terminado para la versión 13.4.0 de Debian

## b. To-do

- Nuevo manual de reenvío de logs a un servidor central con `systemd-journal-upload` y `systemd-journal-remote`
- Más precisión y profundidad en el uso de `borg` para respaldos
- Inclusión de Ansible

## c. Requerimientos previos

- Llave SSH Ed25519 para el punto "4.2 Copia de la llave pública del administrador". Manual para generarla disponible [aquí](https://github.com/noggalito/manuales/blob/main/ssh-ed25519.md)
- Una cuenta de correo electrónico (a veces con contraseña por aplicación) desde donde se enviarán las notificaciones del servidor

---

## Índice

### Etapa I — Preparación

#### 1. Descarga y validación de la ISO

- [1.1 Variables de trabajo](#11-variables-de-trabajo)
- [1.2 Importación y verificación de la clave de firmado](#12-importación-y-verificación-de-la-clave-de-firmado)
- [1.3 Descarga de ISO y archivos de verificación](#13-descarga-de-iso-y-archivos-de-verificación)
- [1.4 Verificación de la firma GPG de los checksums](#14-verificación-de-la-firma-gpg-de-los-checksums)
- [1.5 Verificación del hash de la ISO](#15-verificación-del-hash-de-la-iso)
- [1.6 Verificación integral (opcional, para scripts)](#16-verificación-integral-opcional-para-scripts)

#### 2. Instalación del sistema base

- [2.1 Consideraciones previas al instalador](#21-consideraciones-previas-al-instalador)
- [2.2 Estrategia de particionado](#22-estrategia-de-particionado)
- [2.3 Selección de paquetes durante la instalación](#23-selección-de-paquetes-durante-la-instalación)
- [2.4 Unlock remoto de LUKS (opcional, solo si se usó cifrado)](#24-unlock-remoto-de-luks-opcional-solo-si-se-usó-cifrado)
- [2.5 Primer arranque y verificación básica](#25-primer-arranque-y-verificación-básica)
- [2.6 Checklist de cierre de la Etapa I](#26-checklist-de-cierre-de-la-etapa-i)

### Etapa II — Configuración base del sistema

#### 3. Preparación mínima para administración

- [3.1 Diagnóstico de instalación de `sudo`](#31-diagnóstico-de-instalación-de-sudo)
- [3.2 Instalación de `sudo`](#32-instalación-de-sudo)
- [3.3 Activación de la pertenencia al grupo](#33-activación-de-la-pertenencia-al-grupo)
- [3.4 Checklist de cierre](#34-checklist-de-cierre)

#### 4. Acceso remoto inicial

- [4.1 Verificación del servidor SSH](#41-verificación-del-servidor-ssh)
- [4.2 Copia de la llave pública del administrador](#42-copia-de-la-llave-pública-del-administrador)
- [4.3 Verificación del acceso por llave](#43-verificación-del-acceso-por-llave)
- [4.4 Mantener el canal de rescate abierto](#44-mantener-el-canal-de-rescate-abierto)
- [4.5 Checklist de cierre](#45-checklist-de-cierre)

#### 5. Configuración regional y de identidad

- [5.1 Hostname](#51-hostname)
- [5.2 Archivo `/etc/hosts`](#52-archivo-etchosts)
- [5.3 Zona horaria](#53-zona-horaria)
- [5.4 Locales](#54-locales)
- [5.5 Red](#55-red)
- [5.6 `machine-id`](#56-machine-id)
- [5.7 Checklist de cierre](#57-checklist-de-cierre)

#### 6. Repositorios y primera actualización completa

- [6.1 Respaldo del `sources.list` original](#61-respaldo-del-sourceslist-original)
- [6.2 Edición del `sources.list`](#62-edición-del-sourceslist)
- [6.3 Actualización completa](#63-actualización-completa)
- [6.4 Reinicio si fue actualizado el kernel](#64-reinicio-si-fue-actualizado-el-kernel)
- [6.5 Checklist de cierre](#65-checklist-de-cierre)

#### 7. Usuarios y privilegios

- [7.1 Verificación del usuario administrativo](#71-verificación-del-usuario-administrativo)
- [7.2 Grupos recomendados para el usuario administrativo](#72-grupos-recomendados-para-el-usuario-administrativo)
- [7.3 Hardening de sudo](#73-hardening-de-sudo)
- [7.4 Política de contraseñas](#74-política-de-contraseñas)
- [7.5 Creación de usuarios adicionales (si aplica)](#75-creación-de-usuarios-adicionales-si-aplica)
- [7.6 Bloqueo de la cuenta de root](#76-bloqueo-de-la-cuenta-de-root)
- [7.7 Checklist de cierre de la Etapa II](#77-checklist-de-cierre-de-la-etapa-ii)

### Etapa III — Hardening

#### 8. Hardening de SSH

- [8.1 Archivo modular `/etc/ssh/sshd_config.d/99-hardening.conf`](#81-archivo-modular-etcsshsshd_configd99-hardeningconf)
- [8.2 Cambio de puerto, password auth y solo pubkey](#82-cambio-de-puerto-password-auth-y-solo-pubkey)
- [8.3 Restricción de usuarios (`AllowUsers`)](#83-restricción-de-usuarios-allowusers)
- [8.4 Límites de sesión y timeouts](#84-límites-de-sesión-y-timeouts)
- [8.5 Algoritmos criptográficos modernos](#85-algoritmos-criptográficos-modernos)
- [8.6 Validación y recarga segura](#86-validación-y-recarga-segura)
- [8.7 Checklist de cierre](#87-checklist-de-cierre)

#### 9. Firewall (UFW)

- [9.1 Instalación y políticas por defecto](#91-instalación-y-políticas-por-defecto)
- [9.2 IPv6 (`/etc/default/ufw`)](#92-ipv6-etcdefaultufw)
- [9.3 Apertura selectiva de puertos según rol](#93-apertura-selectiva-de-puertos-según-rol)
- [9.4 Inspección antes de activar](#94-inspección-antes-de-activar)
- [9.5 Activación final y verificación](#95-activación-final-y-verificación)
- [9.6 Checklist de cierre](#96-checklist-de-cierre)

#### 10. Protección contra fuerza bruta (fail2ban)

- [10.1 Instalación](#101-instalación)
- [10.2 Configuración local (`jail.local`)](#102-configuración-local-jaillocal)
- [10.3 Activación](#103-activación)
- [10.4 Monitoreo de bans](#104-monitoreo-de-bans)
- [10.5 Checklist de cierre](#105-checklist-de-cierre)

#### 11. Hardening del kernel y del sistema

- [11.1 Parámetros sysctl de red y de kernel](#111-parámetros-sysctl-de-red-y-de-kernel)
- [11.2 Límites de recursos (`/etc/security/limits.conf`)](#112-límites-de-recursos-etcsecuritylimitsconf)
- [11.3 Deshabilitar módulos de kernel innecesarios](#113-deshabilitar-módulos-de-kernel-innecesarios)
- [11.4 Protección de GRUB (solo bare metal)](#114-protección-de-grub-solo-bare-metal)
- [11.5 Checklist de cierre](#115-checklist-de-cierre)

#### 12. Auditoría y detección

- [12.1 `auditd` — reglas mínimas](#121-auditd--reglas-mínimas)
- [12.2 `rkhunter` — escaneo de rootkits](#122-rkhunter--escaneo-de-rootkits)
- [12.3 `debsums` — integridad de paquetes](#123-debsums--integridad-de-paquetes)
- [12.4 AIDE — integridad de archivos del sistema](#124-aide--integridad-de-archivos-del-sistema)
- [12.5 Revisión periódica de logs y registros](#125-revisión-periódica-de-logs-y-registros)
- [12.6 `lynis` — auditoría periódica del baseline de seguridad](#126-lynis--auditoría-periódica-del-baseline-de-seguridad)
- [12.7 Checklist de cierre](#127-checklist-de-cierre)

#### 13. AppArmor

- [13.1 Verificación del estado](#131-verificación-del-estado)
- [13.2 Perfiles en `enforce` vs `complain`](#132-perfiles-en-enforce-vs-complain)
- [13.3 Añadir perfiles adicionales](#133-añadir-perfiles-adicionales)
- [13.4 Checklist de cierre](#134-checklist-de-cierre)

#### [Cierre de la Etapa III](#cierre-de-la-etapa-iii)

### Etapa IV — Mantenimiento y actualizaciones

#### 14. Actualizaciones automáticas

- [14.1 Instalación](#141-instalación)
- [14.2 Configuración de orígenes](#142-configuración-de-orígenes)
- [14.3 Notificación por correo](#143-notificación-por-correo)
- [14.4 Reinicio automático controlado](#144-reinicio-automático-controlado)
- [14.5 Habilitación del flujo periódico](#145-habilitación-del-flujo-periódico)
- [14.6 Verificación de los timers](#146-verificación-de-los-timers)
- [14.7 Simulación antes de la primera ejecución real](#147-simulación-antes-de-la-primera-ejecución-real)
- [14.8 Revisión de logs tras la primera ejecución](#148-revisión-de-logs-tras-la-primera-ejecución)
- [14.9 Checklist de cierre](#149-checklist-de-cierre)

#### 15. Sincronización de tiempo

- [15.1 Migración de `systemd-timesyncd` a `chrony`](#151-migración-de-systemd-timesyncd-a-chrony)
- [15.2 Configuración de servidores NTP](#152-configuración-de-servidores-ntp)
- [15.3 Verificación](#153-verificación)
- [15.4 Comportamiento ante desviaciones grandes](#154-comportamiento-ante-desviaciones-grandes)
- [15.5 Verificación cruzada con `timedatectl`](#155-verificación-cruzada-con-timedatectl)
- [15.6 Checklist de cierre](#156-checklist-de-cierre)

#### 16. Logs y rotación

- [16.1 Configuración de `journald`](#161-configuración-de-journald)
- [16.2 `logrotate` para logs de aplicaciones](#162-logrotate-para-logs-de-aplicaciones)
- [16.3 Reenvío a un servidor central (opcional)](#163-reenvío-a-un-servidor-central-opcional)
- [16.4 Verificación de uso de disco](#164-verificación-de-uso-de-disco)
- [16.5 Checklist de cierre](#165-checklist-de-cierre)

#### 17. Monitoreo básico

- [17.1 Herramientas de consulta puntual](#171-herramientas-de-consulta-puntual)
- [17.2 Alertas mínimas vía cron](#172-alertas-mínimas-vía-cron)
- [17.3 Estado de servicios críticos](#173-estado-de-servicios-críticos)
- [17.4 Checklist de cierre](#174-checklist-de-cierre)

#### 18. Respaldos

- [18.1 Estrategia 3-2-1](#181-estrategia-3-2-1)
- [18.2 Qué respaldar](#182-qué-respaldar)
- [18.3 BorgBackup](#183-borgbackup)
- [18.4 Verificación del repositorio](#184-verificación-del-repositorio)
- [18.5 Prueba de restauración](#185-prueba-de-restauración)
- [18.6 Restauración del servidor desde cero](#186-restauración-del-servidor-desde-cero)
- [18.7 Checklist de cierre](#187-checklist-de-cierre)

#### 19. Documentación operativa

- [19.1 Inventario del servidor](#191-inventario-del-servidor)
- [19.2 Diagrama lógico](#192-diagrama-lógico)
- [19.3 Runbook básico](#193-runbook-básico)
- [19.4 Mantenimiento de la documentación](#194-mantenimiento-de-la-documentación)
- [19.5 Checklist de cierre](#195-checklist-de-cierre)

#### [Cierre de la Etapa IV](#cierre-de-la-etapa-iv)

### Anexos

#### Anexo A — Índice consolidado de plantillas

- [A.1 Convenciones del sistema de plantillas](#a1-convenciones-del-sistema-de-plantillas)
- [A.2 Inventario de plantillas](#a2-inventario-de-plantillas)
- [A.3 Resumen de variables del manual](#a3-resumen-de-variables-del-manual)
- [A.4 Modificación local de plantillas](#a4-modificación-local-de-plantillas)

#### Anexo B — Diagnóstico rápido post-incidente

- [B.1 Imagen general del sistema](#b1-imagen-general-del-sistema)
- [B.2 Síntoma: el servidor está lento o no responde](#b2-síntoma-el-servidor-está-lento-o-no-responde)
- [B.3 Síntoma: un servicio no funciona](#b3-síntoma-un-servicio-no-funciona)
- [B.4 Síntoma: el disco está lleno](#b4-síntoma-el-disco-está-lleno)
- [B.5 Síntoma: no puedo conectar por SSH](#b5-síntoma-no-puedo-conectar-por-ssh)
- [B.6 Síntoma: red sin conectividad](#b6-síntoma-red-sin-conectividad)
- [B.7 Síntoma: alerta de fail2ban con muchas IPs baneadas](#b7-síntoma-alerta-de-fail2ban-con-muchas-ips-baneadas)
- [B.8 Síntoma: el sistema no arrancó del todo](#b8-síntoma-el-sistema-no-arrancó-del-todo)
- [B.9 Recolección de evidencia para análisis posterior](#b9-recolección-de-evidencia-para-análisis-posterior)
- [B.10 Cuándo escalar](#b10-cuándo-escalar)

#### Anexo C — Reenvío de correo del sistema con `msmtp`

- [C.1 Decisiones de diseño](#c1-decisiones-de-diseño)
- [C.2 Requisitos previos](#c2-requisitos-previos)
- [C.3 Instalación](#c3-instalación)
- [C.4 Configuración global de msmtp](#c4-configuración-global-de-msmtp)
- [C.5 Almacenamiento seguro de la contraseña](#c5-almacenamiento-seguro-de-la-contraseña)
- [C.6 Aliases para destinatarios locales](#c6-aliases-para-destinatarios-locales)
- [C.7 Configuración de mailutils](#c7-configuración-de-mailutils)
- [C.8 Acceso del usuario administrativo al grupo `msmtp`](#c8-acceso-del-usuario-administrativo-al-grupo-msmtp)
- [C.9 Prueba de envío directo](#c9-prueba-de-envío-directo)
- [C.10 Prueba de redirección de aliases](#c10-prueba-de-redirección-de-aliases)
- [C.11 Prueba con el flujo completo del sistema](#c11-prueba-con-el-flujo-completo-del-sistema)
- [C.12 Consideraciones de seguridad](#c12-consideraciones-de-seguridad)
- [C.13 Logs](#c13-logs)
- [C.14 Checklist de cierre](#c14-checklist-de-cierre)

---

## Etapa I — Preparación

### 1. Descarga y validación de la ISO

Directorio de [netinst](https://cdimage.debian.org/debian-cd/current/amd64/bt-cd/)

#### 1.1 Variables de trabajo

```bash
DEBIAN_VER="13.5.0"
DEBIAN_ARCH="amd64"
ISO_NAME="debian-${DEBIAN_VER}-${DEBIAN_ARCH}-netinst.iso"
WORKDIR="/tmp/iso-debian"
MIRROR="https://cdimage.debian.org/debian-cd/${DEBIAN_VER}/${DEBIAN_ARCH}/iso-cd"
CD_SIGNING_KEY="DF9B9C49EAA9298432589D76DA87E80D6294BE9B"

mkdir -p "${WORKDIR}"
cd "${WORKDIR}"
```

> **Nota:** se usa la ruta versionada (`/${DEBIAN_VER}/`) en lugar de `current/` para que la verificación sea reproducible y no cambie bajo los pies si Debian publica una nueva versión puntual durante el proceso.

#### 1.2 Importación y verificación de la clave de firmado

La clave de firmado de las imágenes de CD/DVD de Debian se obtiene directamente del keyserver

```bash
gpg --keyserver hkps://keyring.debian.org --recv-keys "${CD_SIGNING_KEY}"
gpg --fingerprint "${CD_SIGNING_KEY}"
```

Confirmar que la huella mostrada coincide **exactamente** con la publicada en la página oficial: <https://www.debian.org/CD/verify>. Huella esperada:

```
DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

#### 1.3 Descarga de ISO y archivos de verificación

```bash
wget -c "${MIRROR}/SHA512SUMS"
wget -c "${MIRROR}/SHA512SUMS.sign"
wget -c "${MIRROR}/${ISO_NAME}"
```

La opción `-c` permite reanudar descargas interrumpidas sin volver a empezar.

#### 1.4 Verificación de la firma GPG de los checksums

```bash
gpg --verify SHA512SUMS.sign SHA512SUMS
```

Resultado esperado (la fecha variará según la publicación del release):

```
gpg: Firmado el sáb 14 mar 2026 12:41:59 -05
gpg:                usando RSA clave DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Firma correcta de "Debian CD signing key <debian-cd@lists.debian.org>" [desconocido]
gpg: ATENCIÓN: ¡Esta clave no está certificada por una firma de confianza!
gpg:          No hay indicios de que la firma pertenezca al propietario.
Huellas dactilares de la clave primaria: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

> La advertencia "no está certificada por una firma de confianza" es normal y se resuelve manualmente comparando la huella con la web oficial (paso 1.2). Lo crítico es que diga "Firma correcta".

#### 1.5 Verificación del hash de la ISO

```bash
sha512sum --ignore-missing -c SHA512SUMS
```

Resultado esperado:

```
debian-13.5.0-amd64-netinst.iso: La suma coincide
```

La opción `--ignore-missing` evita mensajes de error por las decenas de otras
ISOs listadas en `SHA512SUMS` que no se descargaron (DVD completo, arquitecturas
alternativas, etc.).

#### 1.6 Verificación integral (opcional, para scripts)

Si se automatiza la descarga, conviene validar todo el flujo con códigos de
salida explícitos:

```bash
gpg --verify SHA512SUMS.sign SHA512SUMS \
  && sha512sum --ignore-missing -c SHA512SUMS \
  && echo "✓ ISO verificada y lista para usarse" \
  || echo "✗ FALLÓ la verificación — NO usar esta ISO"
```

Finalmente, reubicar la ISO en la carpeta deseada y borrar el directorio $WORKDIR

```bash
cd
rm -fR "$WORKDIR"
```

### 2. Instalación del sistema base

Esta sección cubre la instalación inicial de Debian 13 Trixie como servidor, con un enfoque generalista aplicable a tres escenarios:

- Hardware físico (bare metal): control total sobre particionado, firmware, consola.
- Máquina virtual local (KVM/libvirt, VirtualBox, VMware Workstation): instalación desde ISO montada virtualmente.
- VPS en la nube (DigitalOcean, Hetzner, AWS, etc.): generalmente el proveedor ya entrega un sistema preinstalado; esta sección indica qué pasos se omiten y qué se verifica.

El objetivo es llegar a un sistema arrancable, accesible por consola (o SSH en caso de VPS), y listo para la configuración posterior descrita en secciones siguientes.

#### 2.1 Consideraciones previas al instalador

##### 2.1.1 Modo de arranque: UEFI vs BIOS legacy

Debian 13 soporta ambos, pero para despliegues nuevos se recomienda UEFI por las siguientes razones:

- Compatibilidad nativa con discos grandes (>2 TiB) sin recurrir a trucos.
- Posibilidad futura de Secure Boot si el rol lo exige.
- Mejor soporte en hypervisores modernos (KVM con OVMF, VirtualBox EFI, cloud).

En VPS este parámetro suele estar fijado por el proveedor y no es negociable. En VM local, habilitar firmware EFI en la definición (`virt-manager` → *Customize before install* → *Overview* → *Firmware: UEFI*). En bare metal, verificar en la BIOS/UEFI que el modo sea UEFI puro (no CSM/Legacy).

##### 2.1.2 Medio de instalación

- Físico: pendrive flasheado con la ISO `netinst` verificada en la sección 1. Usar `dd` o herramientas equivalentes; evitar Rufus en modo ISO + partición (puede introducir ambigüedades en GPT).
- VM local: la ISO se conecta como CD-ROM virtual. No es necesario flashearla.
- VPS: casi siempre no aplica. El proveedor ofrece una plantilla de Debian 13 preinstalada. Si el VPS permite ISO personalizada (Hetzner, algunos OVH), se puede seguir el flujo de instalación manual; si no, se parte del sistema entregado y se salta a la sección 2.5.

##### 2.1.3 Recursos mínimos recomendados

Para un servidor base Debian 13 sin servicios adicionales:

| Recurso | Mínimo     | Recomendado base |
|---------|------------|------------------|
| CPU     | 1 vCPU     | 2 vCPU           |
| RAM     | 512 MB     | 1–2 GB           |
| Disco   | 8 GB       | 20 GB            |
| Red     | 1 interfaz | 1 interfaz       |

El dimensionamiento real depende del rol final (mail, web, Odoo, etc.) y se ajusta en los manuales específicos que extiendan éste.

#### 2.2 Estrategia de particionado

El instalador de Debian ofrece esquemas guiados y manuales. Para un servidor, lo razonable es siempre manual (o guiado con revisión posterior), porque el particionado es una decisión difícil de revertir.

##### 2.2.1 Esquema base recomendado (sin cifrado)

Para servidores sencillos donde el cifrado no es requisito:

```
/dev/sda1  →  /boot/efi   (EFI System Partition, 1 GiB, FAT32)
/dev/sda2  →  /boot       (1 GiB, ext4)
/dev/sda3  →  LVM PV
            ├─ vg-sistema/lv-root   →  /      (ext4, 10–20 GiB)
            ├─ vg-sistema/lv-var    →  /var   (ext4, 5–10 GiB)
            ├─ vg-sistema/lv-log    →  /var/log (ext4, 2–5 GiB)
            ├─ vg-sistema/lv-tmp    →  /tmp   (ext4, 2 GiB, opciones nodev,nosuid,noexec)
            ├─ vg-sistema/lv-home   →  /home  (ext4, el resto o según necesidad)
            └─ vg-sistema/lv-swap   →  swap   (igual a RAM hasta 4 GB; 4–8 GB para más)
```

**Motivación del diseño:**

- `/boot` separado y sin LVM: garantiza que el bootloader pueda leer el kernel sin depender de LVM/cifrado.
- `/var`, `/var/log` y `/tmp` separados: evita que logs desbordados o `/tmp` lleno tumben el sistema.
- LVM: permite redimensionar, añadir snapshots, y ampliar el VG si se agregan discos.
- Opciones de montaje restrictivas en `/tmp` (`nodev,nosuid,noexec`): hardening básico gratuito.

##### 2.2.2 Esquema con cifrado LUKS (recomendado para datos sensibles o servidores físicos)

Cuando el servidor aloja datos sensibles, o su ubicación física no está plenamente controlada (oficina, colocation, laptop reutilizada como server), se añade LUKS2:

```
/dev/sda1  →  /boot/efi   (ESP, 512 MiB, FAT32, sin cifrar)
/dev/sda2  →  /boot       (1 GiB, ext4, sin cifrar)
/dev/sda3  →  LUKS2 → LVM PV
            └─ vg-sistema/[mismos LVs que arriba]
```

**Puntos importantes sobre LUKS en servidores:**

1. `/boot` no se cifra en el esquema estándar de Debian. GRUB 2.12 (incluido en Debian 13 Trixie) soporta LUKS2 con PBKDF2 como KDF, pero **no** con Argon2id, que es precisamente el KDF por defecto cuando el instalador de Debian 13 crea un contenedor LUKS2. Por eso si `/boot` quedara dentro del contenedor LUKS2 con Argon2id, GRUB no podría leerlo y el sistema no arrancaría. Mantener `/boot` fuera del contenedor LUKS simplifica el arranque y es un compromiso aceptable: el contenido de `/boot` no es sensible (kernels e initramfs), y el secreto real reside en los datos del volumen cifrado.

2. Frase de paso vs archivo de clave: el instalador pide una frase de paso. Esta frase se solicitará en cada arranque, lo cual es problemático en servidores remotos. La solución es habilitar unlock remoto (sección 2.4).

3. LUKS2 vs LUKS1: el instalador de Debian 13 usa LUKS2 por defecto (Argon2id como KDF). Esto es correcto y se mantiene.

4. Swap cifrado: el swap debe ir dentro del contenedor LUKS (como un LV más del VG cifrado), nunca fuera. Si queda fuera, filtra datos de RAM.

##### 2.2.3 En VPS

El proveedor entrega un particionado simple (usualmente una sola partición `/` en el disco virtual, sin LVM ni cifrado). Modificarlo en caliente es arriesgado y rara vez justificado. Lo razonable es:

- Aceptar el particionado del proveedor para el disco raíz.
- Si se añaden volúmenes adicionales para datos (bases de datos, backups), esos sí se pueden cifrar con LUKS sin afectar el boot.

##### 2.2.4 Sistema de archivos

`ext4` sigue siendo la opción por defecto y la más predecible para servidores generalistas. `btrfs` y `xfs` tienen sus usos pero introducen complejidad que no siempre se justifica en una base común. Este manual asume `ext4` salvo que un manual derivado especifique lo contrario.

#### 2.3 Selección de paquetes durante la instalación

En la pantalla de selección de software (`tasksel`), la recomendación para un servidor base es dejar marcado solo lo estrictamente necesario:

| Tarea                                | Marcar | Motivo                                                |
|--------------------------------------|--------|-------------------------------------------------------|
| Debian desktop environment           | ❌ No  | Un servidor no necesita GUI                           |
| ...y subvariantes (GNOME, KDE, etc.) | ❌ No  | Ídem                                                  |
| web server                           | ❌ No  | Se instala según rol en manuales específicos          |
| SSH server                           | ✅ Sí  | Acceso remoto desde el primer arranque                |
| standard system utilities            | ✅ Sí  | Herramientas básicas (`less`, `bzip2`, `wget`, etc.)  |

No se marca `web server`, `print server`, ni ningún rol específico aunque se sepa que el servidor será, por ejemplo, un web server. La razón es que `tasksel` instala versiones y combinaciones de paquetes que luego complican la instalación específica (por ejemplo, Apache cuando se quería Nginx). Los roles se instalan limpios en los manuales que extienden éste.

El momento de escoger el kernel -en caso de que el instalador permita esta opción- siempre utilizar el genérico `linux-image-amd64`. Es un metapaquete que siempre depende de la última versión del kernel de la serie estable en Debian 13. Las actualizaciones de seguridad del kernel llegan automáticamente vía `unattended-upgrades` (sección 14), no hay necesidad de recordar cambiar manualmente de paquete cuando salga una nueva versión del kernel, y `apt autoremove` limpia correctamente los kernels viejos (el metapaquete sabe cuál es el "actual" y preserva lo correcto).

En tipo de initrd seleccionar `genérico`. Garantiza que el sistema arranque si se migra a otro hipervisor, se convierte en imagen portable, o se modifica el tipo de almacenamiento virtual. La opción `dirigido` solo se justifica en hardware fijo donde el tamaño del initrd es crítico.

En VPS, esta pantalla no aparece (el sistema ya viene instalado). Se verifica en la sección 2.5 que no haya servicios innecesarios corriendo y se purgan si es necesario.

#### 2.4 Unlock remoto de LUKS (opcional, solo si se usó cifrado)

Si el servidor usa LUKS, por defecto pedirá la frase de paso en la consola física en cada arranque. Para servidores remotos esto es inviable: se configura un SSH mínimo dentro del initramfs que permite desbloquear el disco desde otra máquina.

Esta configuración se hace inmediatamente después de la primera instalación, antes de cualquier reinicio programado o movimiento del servidor a su ubicación final.

##### 2.4.1 Instalación del paquete

```bash
sudo apt update
sudo apt install -y dropbear-initramfs
```

##### 2.4.2 Configuración de la llave pública autorizada

El SSH del initramfs acepta solo autenticación por llave pública (no password). Hay que cargar la llave pública del operador que hará el unlock:

```bash
sudo mkdir -p /etc/dropbear/initramfs
sudo cp ~/.ssh/id_ed25519.pub /etc/dropbear/initramfs/authorized_keys
sudo chmod 600 /etc/dropbear/initramfs/authorized_keys
```

Ésta debe ser una llave específica para unlock remoto, no la llave personal habitual del administrador. Se genera una llave dedicada, idealmente con passphrase diferente, y se guarda en un gestor de contraseñas.

##### 2.4.3 Parámetros de red del initramfs

El initramfs necesita levantar la red para que SSH escuche. En DHCP (común en VPS y redes de oficina) no hace falta configuración adicional. En IP estática, editar `/etc/initramfs-tools/initramfs.conf`:

```
IP=192.168.1.50::192.168.1.1:255.255.255.0:servidor::off
```

Formato: `IP=cliente:server:gateway:netmask:hostname:interfaz:autoconf`. El campo `server` queda vacío (NFS root, no aplica).

##### 2.4.4 Puerto SSH del initramfs

Por defecto, dropbear escucha en el puerto 22 del initramfs. Conviene cambiarlo para que no colisione con el SSH del sistema ya arrancado (distinta sesión, misma IP, evita confusiones en `known_hosts`):

```bash
# Secuencia robusta: funciona tanto si la línea está comentada
# como si ya existe descomentada o si la directiva no está en el archivo
if grep -q 'DROPBEAR_OPTIONS' /etc/dropbear/initramfs/dropbear.conf; then
    sudo sed -i 's/.*DROPBEAR_OPTIONS=.*/DROPBEAR_OPTIONS="-p 2222"/' \
        /etc/dropbear/initramfs/dropbear.conf
else
    echo 'DROPBEAR_OPTIONS="-p 2222"' \
        | sudo tee -a /etc/dropbear/initramfs/dropbear.conf > /dev/null
fi

# Verificar que la directiva quedó activa
grep 'DROPBEAR_OPTIONS' /etc/dropbear/initramfs/dropbear.conf
# Resultado esperado: DROPBEAR_OPTIONS="-p 2222"
```

##### 2.4.5 Regenerar el initramfs y probar

```bash
sudo update-initramfs -u
```

**Prueba antes de que sea crítico:** reiniciar el servidor. Desde otra máquina:

```bash
ssh -p 2222 -i ~/.ssh/unlock_key root@ip.del.servidor
# Al entrar, ejecutar dentro del initramfs:
cryptroot-unlock
# Introducir la frase de paso de LUKS
# La conexión se cierra sola; el sistema continúa arrancando
```

Si el unlock funciona, el servidor termina de arrancar y se puede acceder por SSH normal al sistema completo.

#### 2.5 Primer arranque y verificación básica

Tras la instalación (o entrega del VPS), verificar que el sistema está correctamente desplegado antes de proceder al hardening.

##### 2.5.1 Identificación del sistema

```bash
# Versión y nombre de release
cat /etc/os-release

# Kernel y arquitectura
uname -a

# machine-id (importante en VMs clonadas: debe ser único)
cat /etc/machine-id
```

> **VMs clonadas:** si este servidor proviene de una plantilla o clon, regenerar `machine-id`:
>
> ```bash
> sudo rm /etc/machine-id /var/lib/dbus/machine-id
> sudo systemd-machine-id-setup
> sudo dbus-uuidgen --ensure
> sudo reboot
> ```
>
> Omitir este paso causa problemas sutiles con logs, DHCP leases duplicadas y systemd.

##### 2.5.2 Revisión del arranque

```bash
su -

# Log de arranque
journalctl -b --no-pager | head -100

# Errores y advertencias del arranque actual
journalctl -b -p err --no-pager

# Servicios que fallaron
systemctl --failed
exit
```

El objetivo es que `systemctl --failed` devuelva *"0 loaded units listed"* antes de continuar. Cualquier fallo a este nivel se resuelve aquí, no más adelante.

##### 2.5.3 Estado del disco y particiones

```bash
# Esquema de bloques
lsblk -f

# Espacio en cada punto de montaje
df -hT

# Verificar que LVM y LUKS (si aplica) están activos
sudo pvs && sudo vgs && sudo lvs
sudo cryptsetup status $(ls /dev/mapper/ | grep -v control | head -1)  # si hay LUKS
```

##### 2.5.4 Red funcionando

```bash
# Interfaces e IPs
ip -brief address

# Ruta por defecto
ip route

# Resolución DNS
getent hosts deb.debian.org
```

Los tres comandos deben funcionar. Si `getent` falla pero `ip route` muestra gateway, revisar `/etc/resolv.conf` y la configuración de `systemd-resolved` o `resolv.conf` estático.

##### 2.5.5 Acceso SSH funcional (desde otra máquina)

Desde el equipo del administrador:

```bash
ssh usuario@ip.del.servidor
```

Este acceso en este momento es con la configuración por defecto del instalador (puerto 22, password auth posiblemente habilitado). El hardening de SSH se hace en la sección 8, pero la verificación de que el canal funciona se hace aquí.

#### 2.6 Checklist de cierre de la Etapa I

Antes de pasar a la Etapa II, confirmar:

- [ ] Sistema arranca sin errores (`systemctl --failed` vacío)
- [ ] `/etc/os-release` indica Debian 13 (Trixie)
- [ ] Particionado y sistema de archivos coinciden con lo planificado
- [ ] LUKS y LVM activos si fueron configurados
- [ ] Unlock remoto probado con un reinicio (si aplica)
- [ ] `machine-id` único (especialmente si es clon de VM)
- [ ] Red y DNS operativos
- [ ] SSH accesible desde el equipo del administrador
- [ ] Reloj del sistema razonablemente correcto (`date`)

Solo con todos estos puntos verificados tiene sentido avanzar al hardening, porque el hardening asume un sistema funcional.

#### Notas sobre el enfoque generalista

**Lo que esta sección deliberadamente no incluye:**

- Configuración de IP estática definitiva: se hace en sección 4 (identidad) o varía demasiado según proveedor de nube.
- Instalación de QEMU guest agent / open-vm-tools / cloud-init: pertenece a un anexo específico por plataforma.
- Firmware no-free adicional (microcódigo Intel/AMD): se gestiona con los repositorios en sección 5, no durante la instalación base.

**Lo que los manuales específicos (mail, web, Odoo, ISPConfig) deben añadir sobre esta base:**

- Ajustes de particionado si el rol lo requiere (ej. `/var/lib/mysql` en LV separado para DB server).
- Recursos mínimos específicos del rol.
- Consideraciones de red adicionales (múltiples IPs, interfaces dedicadas, etc.).

---

## Etapa II — Configuración base del sistema

Esta parte lleva al sistema recién instalado desde un estado "arranca y es accesible" a un estado "utilizable como servidor administrable de forma estándar". Cubre la instalación de herramientas administrativas básicas, el acceso remoto, la identidad del sistema, los repositorios y usuarios.

El hardening propiamente dicho (SSH endurecido, firewall, fail2ban, sysctl, etc.) corresponde a la Etapa III. Aquí solo se prepara el terreno.

---

### 3. Preparación mínima para administración

Debian instalado desde ISO `netinst` con contraseña de root definida no incluye `sudo` por defecto. Es una decisión deliberada de Debian: si hay cuenta de root activa, asume administración con `su -`. Sin embargo, todo el resto del manual asume `sudo` disponible como herramienta estándar, por lo que esta sección lo instala y habilita antes de cualquier otra tarea.

En VPS de nube y plantillas corporativas, lo normal es que `sudo` ya venga instalado y con el usuario administrativo añadido al grupo correspondiente. En ese caso, esta sección se verifica en pocos segundos.

#### 3.1 Diagnóstico de instalación de `sudo`

Desde la sesión del usuario administrativo:

```bash
sudo -V 2>/dev/null | head -1
```

Tres posibles resultados:

| Resultado                                           | Estado                 | Acción              |
|-----------------------------------------------------|------------------------|---------------------|
| `Sudo version 1.9.x ...`                            | Instalado y funcional  | Saltar a 3.3        |
| `-bash: sudo: orden no encontrada`                  | No instalado           | Continuar con 3.2   |
| Instalado pero `usuario is not in the sudoers file` | Instalado sin permisos | Ir a 3.2.2          |

#### 3.2 Instalación de `sudo`

##### 3.2.1 Elevar privilegios temporalmente

Como `sudo` no existe, se usa `su -` para obtener una shell de root:

```bash
su -
```

Introducir la contraseña de root. El prompt cambiará a `root@<hostname>:~#`.

> Nota: `su -` con guion carga el entorno completo de root (`PATH`, `HOME`, etc.). `su` sin guion conserva parte del entorno del usuario invocador y puede causar comportamientos inesperados. Para tareas administrativas siempre se usa con guion.

##### 3.2.2 Instalar el paquete y añadir el usuario al grupo `sudo`

```bash
export MIUSUARIO=usuario
apt update
apt install -y sudo
usermod -aG sudo $MIUSUARIO
```

En Debian, pertenecer al grupo `sudo` es suficiente para obtener permisos completos, porque `/etc/sudoers` incluye por defecto:

```
%sudo   ALL=(ALL:ALL) ALL
```

No se edita `/etc/sudoers` en este momento. El endurecimiento de la configuración de `sudo` (políticas, `pwfeedback`, timeouts, archivos en `/etc/sudoers.d/`) se cubre en la sección 7.

##### 3.2.3 Salir de la shell de root

```bash
exit
```

#### 3.3 Activación de la pertenencia al grupo

La pertenencia a un nuevo grupo no se activa en sesiones ya abiertas. Hay que cerrar y reabrir la sesión completa:

```bash
exit
# Reconectar por SSH o consola
```

Tras reconectar, verificar:

```bash
id
# La salida debe incluir ... grupos=...,27(sudo),...

sudo -v
# Pide la contraseña del usuario (no la de root). Si no devuelve error, sudo está operativo.
```

#### 3.4 Checklist de cierre

- [ ] `sudo -V` devuelve la versión instalada
- [ ] El usuario administrativo pertenece al grupo `sudo`
- [ ] `sudo -v` funciona tras reconectar la sesión
- [ ] La contraseña de root sigue conocida y funcional (aún necesaria temporalmente)

---

### 4. Acceso remoto inicial

Esta sección asegura que el servidor es accesible por SSH con llave pública desde el equipo del administrador, usando todavía la configuración por defecto del instalador (puerto 22, password auth habilitado). El endurecimiento de SSH se hace en la Etapa III.

En VPS y VMs donde ya estás conectado por SSH (como seguramente es el caso), esta sección se reduce a formalizar el acceso por llave pública antes de desactivar el password auth en hardening.

#### 4.1 Verificación del servidor SSH

```bash
sudo systemctl status ssh --no-pager
```

El servicio debe aparecer como `active (running)`. Si no está instalado (instalación manual sin marcar la tarea "SSH server"):

```bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

#### 4.2 Copia de la llave pública del administrador

Se requiere contar con una llave SSH Ed25519, en caso de no tenerla se puede acceder al manual [aquí](https://github.com/noggalito/manuales/blob/main/ssh-ed25519.md). La variable `$MINOMBREDELLAVE` se define en el manual citado en la línea anterior; en caso de no seguir ese manual, se la debe definir ahora antes de seguir. Desde el equipo del administrador (no desde el servidor), copiar la llave pública al servidor:

```bash
ssh-copy-id -i ~/.ssh/${MINOMBREDELLAVE}.pub usuario@servidor
```

En caso que no haya `ssh-copy-id`, aunque es menos recomendado se puede usar:

```bash
cat ~/.ssh/${MINOMBREDELLAVE}.pub | ssh usuario@servidor '
  umask 077 &&
  mkdir -p ~/.ssh &&
  cat >> ~/.ssh/authorized_keys &&
  chmod 700 ~/.ssh &&
  chmod 600 ~/.ssh/authorized_keys
'
```

O manualmente, añadiendo el contenido de `${MINOMBREDELLAVE}.pub` a `~/.ssh/authorized_keys` del servidor.

Ed25519 es la opción recomendada para llaves nuevas: es más corta, más rápida y criptográficamente sólida. RSA de 4096 bits sigue siendo aceptable pero sin ventajas prácticas.

#### 4.3 Verificación del acceso por llave

Desde el equipo del administrador:

```bash
ssh <usuario>@<ip.del.servidor>
```

Debe entrar sin pedir contraseña. Si pide contraseña, la llave no se copió correctamente o los permisos de `~/.ssh` en el servidor están mal.

Comprobación de permisos en el servidor:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Permisos esperados:

```
drwx------  .ssh
-rw-------  authorized_keys
```

#### 4.4 Mantener el canal de rescate abierto

Durante el resto del manual, no cerrar la sesión SSH que funciona. Abrir una segunda sesión en paralelo para probar cambios. Si algo rompe la configuración SSH, la sesión original sigue conectada y permite reparar.

Una opción es usar SSH con sesión persistente a través de `tmux`. En caso de un cierre no deseado de sesión SSH, basta con reconectar a través de `tmux` para continuar desde el punto de la desconexión

```bash
# Instalación de tmux en el servidor
sudo apt update
sudo apt install -y tmux

# Iniciar la sesión de tmux
tmux new -A -s admin
```

En caso de cierre de sesión de SSH, se debe volver a reconectar SSH y luego ejecutar:

```bash
tmux attach -t admin
```

Finalmente, para cerrar la sesión `tmux` y regresar a la sesión SSH original: `Ctrl+b` luego `d`

Este hábito se consolida en la Etapa III cuando se cambia el puerto SSH, se deshabilita password auth, y se activa el firewall.

#### 4.5 Checklist de cierre

- [ ] `ssh <usuario>@<ip>` entra sin pedir contraseña
- [ ] Permisos de `~/.ssh` y `authorized_keys` correctos
- [ ] Sesión de rescate documentada en una pestaña aparte

### 5. Configuración regional y de identidad

Esta sección fija la identidad del servidor: hostname, zona horaria, locales, y red básica.

#### 5.1 Hostname

El hostname puede haberse definido durante la instalación. Verificar:

```bash
sudo hostnamectl
```

Para cambiarlo (o fijarlo si no se definió):

```bash
sudo hostnamectl set-hostname <hostname>
```

`hostnamectl` actualiza `/etc/hostname` y lo aplica en caliente. No requiere reinicio.

#### 5.2 Archivo `/etc/hosts`

El archivo `/etc/hosts` debe reflejar el hostname para que resoluciones locales no pasen por DNS. Edición recomendada:

```bash
# Respaldar el archivo original
sudo cp /etc/hosts /etc/hosts.ori

# Definir la variable $MIHOST
export MIHOST=villonaco

# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-hosts.txt" -O /tmp/plantilla-etc-hosts.txt

# Verificar integridad (reemplazar <SHA256> con el hash obtenido para este SHA_COMMIT)
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-hosts.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

# Reemplazar variables y generar el archivo en destino
sed "s/\$MIHOST/$MIHOST/g" /tmp/plantilla-etc-hosts.txt | sudo tee /etc/hosts > /dev/null
```

Es probable que la terminal arroje un error como: `sudo: unable to resolve host villonaco: Nombre o servicio desconocido`, sin embargo se debe omitir ese mensaje y avanzar.

Si el servidor no pertenece a ningún dominio, usar `<hostname>.lan` o simplemente omitir la parte FQDN y dejar `127.0.1.1 <hostname>`.

Verificación:

```bash
hostname -f     # FQDN completo
hostname -s     # nombre corto
getent hosts <hostname>
```

#### 5.3 Zona horaria

```bash
timedatectl                              # estado actual
timedatectl list-timezones | grep <Region>   # buscar zona, por ejemplo America
sudo timedatectl set-timezone <Region/Ciudad>  # por ejemplo America/Lima
```

Ejemplo para operaciones en Perú:

```bash
sudo timedatectl set-timezone America/Lima
```

Verificación:

```bash
date
```

La sincronización NTP se configura en la Etapa IV (mantenimiento). En este punto basta con la zona horaria correcta.

#### 5.4 Locales

Los locales definen idioma, formato de fecha, separador decimal, orden alfabético, etc. La mayoría de servidores se benefician de mantener `C.UTF-8` o `en_US.UTF-8` como locale por defecto (logs en inglés, menos sorpresas al buscar errores en documentación), y añadir el locale regional como opción.

```bash
sudo apt install -y locales
sudo dpkg-reconfigure locales
```

En el diálogo:

1. Marcar los locales a generar. Recomendación: `en_US.UTF-8 UTF-8` y el locale regional, por ejemplo `es_PE.UTF-8 UTF-8`.
2. Seleccionar el locale por defecto del sistema.

Para un servidor, una recomendación práctica:

- Locale por defecto: `en_US.UTF-8` o `C.UTF-8` (mantiene los logs en inglés).
- Locales adicionales generados: los que el operador necesite para tareas puntuales.

Alternativamente, para dejar todo en idioma regional:

```bash
sudo update-locale LANG=es_PE.UTF-8 LC_ALL=es_PE.UTF-8
```

Verificación:

```bash
locale
locale -a | grep -E 'es_PE|en_US'
```

El cambio de locale requiere cerrar y reabrir la sesión para que las variables se apliquen.

#### 5.5 Red

La instalación netinst de Debian 13 Trixie configura la red durante el instalador y la deja gestionada por `ifupdown` en `/etc/network/interfaces`, sin instalar `dhcpcd` ni `NetworkManager` ni `systemd-networkd`. Este es el punto de partida tanto en bare metal como en VMs locales. En VPS la situación varía: la mayoría de proveedores entrega el sistema con `ifupdown` y configuración estática inyectada en `/etc/network/interfaces`, pero algunos (especialmente los que usan imágenes cloud-init) llegan con `systemd-networkd` o `netplan` ya activos. El primer paso del manual es verificar qué backend está realmente en uso antes de decidir el camino.

Sea cual sea el backend, para un servidor el objetivo es el mismo: una IP estable, un gateway por defecto, DNS funcional, y configuración persistente entre reinicios. Esta sección documenta dos caminos: mantener `ifupdown` (la opción más conservadora y la que coincide con la configuración por defecto de la netinst), o migrar a `systemd-networkd` (recomendado solo si el servidor va a tener configuraciones de red complejas o se prefiere coherencia con el resto del ecosistema systemd).

##### 5.5.1 Estado actual de la red

Antes de modificar nada, verificar qué backend gestiona la red y qué configuración tiene ahora mismo:

```bash
# Estado de IPs y rutas
ip -brief address
ip route

# Resolución DNS
cat /etc/resolv.conf
getent hosts deb.debian.org

# Qué paquetes de red están instalados
dpkg -l | grep -E '^ii  (ifupdown|systemd-networkd|netplan|network-manager|dhcpcd)' | awk '{print $2}'

# Qué servicios de red están activos
systemctl is-active networking systemd-networkd NetworkManager 2>/dev/null
```

Interpretación de la salida:

- `networking` activo y `ifupdown` instalado: backend es `ifupdown` (configuración en `/etc/network/interfaces`). Es el caso por defecto en netinst.
- `systemd-networkd` activo: backend es `systemd-networkd` (configuración en `/etc/systemd/network/*.network`). Típico en algunas imágenes cloud y VPS modernos.
- `NetworkManager` activo: backend es `NetworkManager` (configuración con `nmcli` o en `/etc/NetworkManager/system-connections/`). Típico en instalaciones con escritorio, raro en server.

Si la interfaz aparece con IP del rango esperado, `ip route` muestra gateway por defecto, y `getent hosts` resuelve, la red está funcional y solo queda decidir si hacer la IP estática o mantener DHCP.

##### 5.5.2 Identificar el nombre de la interfaz

Debian 13 usa nombres predecibles de interfaz (`enp1s0`, `eno1`, `ens3`, etc.) en lugar de los antiguos `eth0`. Para identificar la interfaz que tiene la IP del servidor:

```bash
ip -brief address | grep -v '^lo'
```

Salida típica:

```
enp1s0           UP             192.168.1.50/24 fe80::5054:ff:fe12:3456/64
```

Aquí la interfaz se llama `enp1s0`. En el resto de la sección, donde aparece `<interfaz>`, sustituir por el nombre real.

Para mantener el nombre de la interfaz en una variable de entorno y reutilizarla en los comandos siguientes:

```bash
export MIIFAZ=$(ip -o -4 route show default | awk '{print $5}')
echo "Interfaz por defecto: $MIIFAZ"
```

Esto extrae la interfaz por la que sale el tráfico hacia internet, que en un servidor con una sola tarjeta es siempre la correcta.

##### 5.5.3 Opción A — DHCP con reserva por MAC (lo más simple)

Si el servidor está detrás de un router donde se puede reservar la IP por MAC, este es el camino más simple y robusto: no se modifica nada en el servidor, y la IP permanece estable porque el router siempre asigna la misma.

Pasos:

1. Anotar la MAC de la interfaz: `ip link show $MIIFAZ` (línea `link/ether ...`).
2. En la interfaz del router, crear una reserva DHCP para esa MAC con la IP deseada.
3. Renovar el lease desde el servidor:

   ```bash
   sudo systemctl restart networking
   ```

4. Verificar que la IP asignada coincide con la reservada:

   ```bash
   ip -brief address show $MIIFAZ
   ```

Ventajas: cero configuración en el servidor; fácil de migrar entre redes.
Desventajas: depende del router; no aplica si el servidor está en una red sin control del DHCP.

##### 5.5.4 Opción B — IP estática con `ifupdown` (configuración por defecto)

Esta es la opción recomendada cuando se necesita IP estática y se quiere mantener el backend que viene por defecto en la netinst. La configuración vive en `/etc/network/interfaces`.

Respaldar el archivo original antes de tocarlo:

```bash
sudo cp /etc/network/interfaces /etc/network/interfaces.ori
```

Inspeccionar el contenido actual:

```bash
cat /etc/network/interfaces
```

Salida típica tras una netinst con DHCP:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp1s0
iface enp1s0 inet dhcp
```

Para convertir esa interfaz a IP estática, reemplazar el bloque `iface ... inet dhcp` por uno con configuración fija:

```bash
sudoedit /etc/network/interfaces
```

Sustituir el bloque correspondiente por algo como:

```
# The primary network interface
allow-hotplug enp1s0
iface enp1s0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    dns-nameservers 192.168.1.1 1.1.1.1
```

Notas sobre la sintaxis:

- La indentación de las líneas dentro del bloque `iface` no es estrictamente obligatoria, pero mejora la legibilidad y es la convención.
- `address` admite el formato CIDR (`192.168.1.50/24`). El formato antiguo `address 192.168.1.50` con `netmask 255.255.255.0` por separado también funciona, pero el CIDR es más conciso.
- `dns-nameservers` requiere el paquete `resolvconf` para que los servidores se propaguen a `/etc/resolv.conf` automáticamente. En netinst este paquete suele venir instalado; verificar con `dpkg -l resolvconf`. Si no está instalado, instalarlo (`sudo apt install -y resolvconf`) o gestionar `/etc/resolv.conf` manualmente (ver subsección 5.5.6).
- Si el servidor necesita IPv6 estática, añadir un bloque `iface enp1s0 inet6 static` adicional con `address`, `gateway` y `dns-nameservers`.

Aplicar los cambios. La opción más segura es reiniciar el servidor (`sudo reboot`), porque garantiza que la configuración persiste tras un reinicio y no quedan estados intermedios. Si se prefiere aplicar en caliente sin reiniciar:

```bash
sudo ifdown $MIIFAZ && sudo ifup $MIIFAZ
```

> **Atención:** ejecutar `ifdown` por SSH desde la propia interfaz que se está reconfigurando deja al servidor inaccesible durante unos segundos. Si la nueva configuración tiene un error, la sesión SSH se pierde y la única forma de recuperar el servidor es por consola física, hypervisor o panel del VPS. Para minimizar el riesgo, usar `&&` (como arriba) en lugar de comandos separados, o ejecutar el ciclo dentro de `tmux` o `screen` para que sobreviva a la desconexión.

Verificar el resultado:

```bash
ip -brief address show $MIIFAZ
ip route
cat /etc/resolv.conf
```

##### 5.5.5 Opción C — Migrar a `systemd-networkd` (solo si se justifica)

`systemd-networkd` es el backend recomendado cuando el servidor va a tener configuraciones de red complejas (bonding, VLAN, bridges, múltiples interfaces con políticas de routing distintas), o cuando se prefiere coherencia con el resto del ecosistema systemd. Para un servidor con una sola interfaz y configuración simple, no aporta ventajas respecto a `ifupdown` y añade un punto de cambio innecesario.

Si se decide migrar, los pasos son:

**1. Crear el archivo de configuración antes de desactivar `ifupdown`.** Esto evita quedarse sin red durante la transición:

```bash
sudo tee /etc/systemd/network/10-${MIIFAZ}.network > /dev/null <<EOF
[Match]
Name=${MIIFAZ}

[Network]
Address=192.168.1.50/24
Gateway=192.168.1.1
DNS=192.168.1.1
DNS=1.1.1.1
EOF
```

Para configuración por DHCP en lugar de IP estática, reemplazar el bloque `[Network]` por:

```
[Network]
DHCP=yes
```

**2. Habilitar `systemd-networkd` y `systemd-resolved`.** El primero gestiona interfaces; el segundo proporciona el resolver DNS local que `systemd-networkd` espera:

```bash
sudo apt install -y systemd-resolved
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
```

> **Nota:** en Trixie, `systemd-resolved` ya no viene preinstalado por defecto (se separó del paquete `systemd` principal). Hay que instalarlo explícitamente. La instalación reemplaza `/etc/resolv.conf` por un symlink al stub de resolved.

**3. Desactivar `ifupdown` para la interfaz que se va a migrar.** La forma más limpia es comentar el bloque correspondiente en `/etc/network/interfaces` (no borrarlo, para tener referencia si hay que revertir):

```bash
sudoedit /etc/network/interfaces
```

Comentar las líneas `allow-hotplug` e `iface` de la interfaz que pasará a `systemd-networkd`. Mantener intacto el bloque `lo` (loopback).

**4. Reiniciar para aplicar el cambio.** Aplicar en caliente es posible pero arriesgado por la posibilidad de que las dos pilas peleen por la misma interfaz; un reinicio limpio es más predecible:

```bash
sudo reboot
```

**5. Verificar tras el reinicio:**

```bash
networkctl status
networkctl status $MIIFAZ
resolvectl status
ip -brief address
ip route
```

`networkctl status <interfaz>` debe mostrar la interfaz como `configured` y con la IP correcta. `resolvectl status` debe mostrar los DNS configurados.

##### 5.5.6 DNS sin gestor automático (caso especial)

Si por algún motivo `/etc/resolv.conf` no se pobla automáticamente (servidor sin `resolvconf` ni `systemd-resolved`, configuración heredada inconsistente, etc.), se puede gestionar manualmente:

```bash
sudo tee /etc/resolv.conf > /dev/null <<'EOF'
nameserver 1.1.1.1
nameserver 9.9.9.9
EOF
```

> **Atención:** un `/etc/resolv.conf` editado a mano será sobrescrito por `resolvconf`, `systemd-resolved` o el cliente DHCP la próxima vez que reciban una nueva configuración. Si se quiere fijarlo de forma realmente permanente sin instalar un gestor, se puede aplicar el atributo de inmutabilidad:
>
> ```bash
> sudo chattr +i /etc/resolv.conf
> ```
>
> Esto impide cualquier modificación, incluso por root, hasta que se quite el atributo con `sudo chattr -i /etc/resolv.conf`. Es una solución de fuerza bruta y conviene documentarla en el inventario del servidor (sección 19.1) para que no sorprenda a quien intente modificar la red en el futuro.

##### 5.5.7 Verificación final

Independientemente del backend elegido y de si la IP es estática o por DHCP:

```bash
ip -brief address           # IP correcta en la interfaz
ip route                    # gateway por defecto presente
getent hosts deb.debian.org # DNS funcional (resuelve nombre)
ping -c 3 1.1.1.1           # conectividad IP a internet
ping -c 3 deb.debian.org    # conectividad + DNS
```

Los cinco comandos deben tener éxito. Si `ping 1.1.1.1` funciona pero `ping deb.debian.org` falla, el problema es DNS. Si `ip route` no muestra una ruta `default`, el problema es el gateway. Si `ip -brief address` no muestra IP en la interfaz esperada, el problema está en la configuración del backend (revisar `journalctl -u networking` para `ifupdown`, `journalctl -u systemd-networkd` para `systemd-networkd`).

##### 5.5.8 Persistencia entre reinicios

Antes de dar por cerrada la sección, conviene reiniciar el servidor una vez y volver a ejecutar las verificaciones del paso 5.5.7. Una configuración que funciona en caliente pero no sobrevive al reinicio es uno de los errores más frustrantes y comunes; detectarlo ahora, mientras la sesión de rescate (sección 4.4) sigue abierta y el servidor está en su entorno de despliegue inicial, es mucho más barato que descubrirlo tras el primer reinicio en producción.

```bash
sudo reboot
```

Tras el reinicio, reconectar y repetir las verificaciones de 5.5.7. Solo cuando la red está funcional tras un reinicio limpio se puede pasar a la siguiente sección.

#### 5.6 `machine-id`

El `machine-id` identifica unívocamente la instalación. En VMs clonadas desde plantilla, este identificador se duplica y causa problemas sutiles (leases DHCP duplicadas, logs mezclados, identificación errónea en systemd).

Verificar:

```bash
cat /etc/machine-id
```

Si esta instalación proviene de un clon de plantilla, regenerar:

```bash
sudo rm /etc/machine-id /var/lib/dbus/machine-id
sudo systemd-machine-id-setup
sudo dbus-uuidgen --ensure
sudo reboot
```

Para instalaciones limpias (no clonadas), no hacer nada: el ID ya es único.

#### 5.7 Checklist de cierre

- [ ] `hostnamectl` muestra el hostname correcto
- [ ] `/etc/hosts` coherente con el hostname
- [ ] `date` muestra la zona horaria correcta
- [ ] `locale` muestra el locale correcto
- [ ] Red funcional con el backend elegido
- [ ] DNS resuelve correctamente
- [ ] `machine-id` único (confirmado o regenerado)

### 6. Repositorios y primera actualización completa

La ISO `netinst` instala un sistema mínimo con el CD-ROM como repositorio inicial. Esta sección ajusta `/etc/apt/sources.list` para usar los mirrors oficiales y deja el sistema completamente actualizado.

#### 6.1 Respaldo del `sources.list` original

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.ori
```

Conservar el original es útil para comparar en caso de problemas futuros.

#### 6.2 Edición del `sources.list`

```bash
sudoedit /etc/apt/sources.list
```

Contenido recomendado para Debian 13 Trixie:

```
# Repositorio principal
deb https://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware

# Actualizaciones entre point releases
deb https://deb.debian.org/debian/ trixie-updates main contrib non-free non-free-firmware

# Parches de seguridad
deb https://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

Notas sobre los componentes:

- `main` — software 100% libre, soportado por el proyecto Debian.
- `contrib` — software libre pero con dependencias no libres.
- `non-free` — software no libre (licencias propietarias). Se incluye en servidores solo si es necesario.
- `non-free-firmware` — firmware no libre separado en Trixie por cambio de política. Recomendable mantenerlo activo para compatibilidad con hardware moderno (tarjetas de red, controladores, etc.).

Las líneas `deb-src` (fuentes) se omiten por defecto en un servidor. Añadirlas solo si se van a compilar paquetes desde fuente.

Comentar o eliminar la línea que referencia al CD-ROM:

```
# deb cdrom:[...]  (línea comentada o eliminada)
```

#### 6.3 Actualización completa

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove --purge -y
sudo apt clean
```

Diferencia entre `upgrade` y `full-upgrade`:

- `upgrade` — actualiza paquetes sin instalar nuevos ni eliminar existentes. Si un paquete requiere cambios de dependencias, se queda sin actualizar.
- `full-upgrade` — actualiza todo, incluyendo paquetes que requieren instalar o eliminar otros. Es el equivalente a `dist-upgrade` en versiones antiguas.

En un sistema de producción `full-upgrade` se revisa antes de aplicarse (`apt full-upgrade --dry-run`). En una instalación recién desplegada, aplicarlo directamente es correcto.

#### 6.4 Reinicio si fue actualizado el kernel

```bash
ls /boot/vmlinuz-*
uname -r
```

Si la versión listada en `/boot` es superior a la activa (`uname -r`), reiniciar para usar el kernel nuevo:

```bash
sudo reboot
```

Alternativamente, Debian informa cuándo se necesita reinicio mediante el paquete `needrestart` (opcional):

```bash
sudo apt install -y needrestart
sudo needrestart
```

#### 6.5 Checklist de cierre

- [ ] `/etc/apt/sources.list` configurado con mirrors oficiales
- [ ] `sudo apt update` sin errores ni warnings de repositorios
- [ ] Sistema completamente actualizado (`apt list --upgradable` vacío)
- [ ] Kernel en uso coincide con el instalado
- [ ] Sistema reiniciado si hubo actualización de kernel

### 7. Usuarios y privilegios

Esta sección formaliza la política de usuarios del servidor: confirma que el usuario administrativo tiene los grupos correctos, define política de contraseñas, y bloquea la cuenta de root.

La configuración detallada de `sudo` (políticas en `/etc/sudoers.d/`, insults, pwfeedback, etc.) se ha dejado deliberadamente para la Etapa III (hardening), porque son decisiones de endurecimiento, no de configuración base.

#### 7.1 Verificación del usuario administrativo

```bash
export MIUSUARIO=usuario
id $MIUSUARIO
```

Salida esperada:

```
uid=1000(<usuario>) gid=1000(<usuario>) grupos=1000(<usuario>),27(sudo),...
```

El grupo `sudo` (GID 27 en Debian) debe estar presente. Si no, añadirlo:

```bash
sudo usermod -aG sudo $MIUSUARIO
```

#### 7.2 Grupos recomendados para el usuario administrativo

Añadir el usuario al grupo `adm` permite leer logs del sistema sin `sudo`. Es una comodidad razonable en un servidor con un solo administrador:

```bash
sudo usermod -aG adm $MIUSUARIO
```

Otros grupos que pueden ser útiles según el rol futuro del servidor (no se añaden ahora, solo se documentan):

- `systemd-journal` — lectura del journal de systemd (redundante si ya está en `adm`).
- `wireshark` — captura de tráfico sin privilegios (solo si se instala wireshark).
- `docker` — administración de Docker sin `sudo` (nota: equivale a root; solo en servidores donde esto se acepte).
- `kvm`, `libvirt` — gestión de VMs sin `sudo`.

La pertenencia a grupos privilegiados debe tratarse como concesión de permisos, no como conveniencia. Revisar en cada caso si el acceso directo al grupo se justifica o conviene mantener `sudo` como barrera.

#### 7.3 Hardening de sudo

`sudo` tiene su propia superficie de seguridad: tickets de larga duración permiten que una sesión desatendida pueda ser aprovechada; sin logging, las acciones ejecutadas con privilegios de root no quedan registradas; sin entorno limpio, variables heredadas pueden manipular el comportamiento de los programas invocados como root.

La configuración se hace en un archivo separado dentro de `/etc/sudoers.d/` para no tocar el archivo canónico gestionado por el paquete.

##### 7.3.1 Archivo de hardening de sudoers

```bash
sudo tee /etc/sudoers.d/99-hardening > /dev/null <<'EOF'
# Limpiar el entorno heredado antes de ejecutar como root
Defaults env_reset

# Ruta segura de binarios (no heredar $PATH del usuario invocador)
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# El ticket de sudo expira a los 5 minutos de inactividad
Defaults timestamp_timeout=5

# Requiere TTY real; evita que scripts sin supervisión usen sudo
# Comentar esta línea si el servidor es gestionado con Ansible u otra herramienta sin TTY
Defaults requiretty

# Registrar la sesión completa (output) de cada ejecución sudo en /var/log/sudo-io/
Defaults log_output

# Notificar al administrador cuando se introduce una contraseña incorrecta en sudo
Defaults mail_badpass
EOF

# Verificar la sintaxis antes de que tome efecto (un error aquí puede inhabilitar sudo)
sudo visudo -c -f /etc/sudoers.d/99-hardening
```

Si `visudo -c` no devuelve error, aplicar permisos correctos:

```bash
sudo chmod 440 /etc/sudoers.d/99-hardening
sudo chown root:root /etc/sudoers.d/99-hardening
```

> **Sobre `requiretty`:** esta directiva puede romper herramientas de automatización sin TTY (Ansible, algunos scripts en systemd units). Si el servidor va a ser gestionado con una de estas herramientas, comentar esa línea o añadir una excepción por usuario:
>
> ```
> Defaults:miusuario_ansible !requiretty
> ```

##### 7.3.2 Verificación

```bash
sudo visudo -c      # verifica la sintaxis de todo el árbol sudoers
sudo -l             # lista los privilegios del usuario actual con la nueva configuración
```

##### 7.3.3 pam_faillock — lockout PAM para escalada local

fail2ban (sección 10) protege contra fuerza bruta sobre SSH. No protege contra intentos de escalada local: un atacante con acceso a la consola o con una sesión no privilegiada puede intentar `su -` o contraseñas de `sudo` repetidamente sin que fail2ban intervenga. `pam_faillock` (incluido en `libpam-modules`, sin instalación adicional) añade lockout a nivel PAM cubriendo cualquier servicio que pase por `pam_unix`: login de consola, `su`, `sudo`, y SSH cuando usa PAM.

**Configuración del comportamiento:**

```bash
sudo tee /etc/security/faillock.conf > /dev/null <<'EOF'
# Número de fallos antes de bloquear la cuenta
deny = 5

# Ventana de tiempo en segundos durante la que se cuentan los fallos (15 minutos)
fail_interval = 900

# Tiempo de bloqueo en segundos tras alcanzar el límite (10 minutos)
unlock_time = 600

# Registrar eventos en el subsistema de auditoría (auditd)
audit

# No revelar si la cuenta está bloqueada (mismo mensaje que contraseña incorrecta)
silent
EOF
```

**Activación en PAM:**

`pam_faillock` se integra en `/etc/pam.d/common-auth`. Editar este archivo con extremo cuidado: un error puede dejar el sistema sin autenticación funcional.

```bash
# Respaldar siempre antes de modificar archivos PAM
sudo cp /etc/pam.d/common-auth /etc/pam.d/common-auth.ori
sudoedit /etc/pam.d/common-auth
```

Añadir la línea de `preauth` **antes** del módulo `pam_unix.so`, y la de `authfail` inmediatamente **después**. La estructura resultante debe quedar así (las líneas existentes se conservan; solo se añaden las marcadas):

```
# [AÑADIR] pam_faillock: contar fallos antes de autenticar
auth    required                        pam_faillock.so preauth
# [EXISTENTE] autenticación principal de contraseñas
auth    [success=1 default=ignore]      pam_unix.so nullok
# [AÑADIR] pam_faillock: registrar el fallo si pam_unix no autenticó
auth    [default=die]                   pam_faillock.so authfail
# [AÑADIR] pam_faillock: limpiar el contador si la autenticación tuvo éxito
auth    sufficient                      pam_faillock.so authsucc
```

> **Atención:** la forma exacta de la línea `pam_unix.so` varía según las opciones instaladas en el sistema. Verificar el aspecto actual del archivo antes de editar. Si tras la modificación el login falla completamente, desde la sesión paralela (sección 4.4) ejecutar:
>
> ```bash
> sudo cp /etc/pam.d/common-auth.ori /etc/pam.d/common-auth
> ```

**Verificación:**

```bash
# Autenticarse con contraseña incorrecta varias veces y luego verificar el bloqueo
sudo faillock --user $MIUSUARIO

# Desbloquear manualmente (útil si se bloqueó uno mismo durante pruebas)
sudo faillock --user $MIUSUARIO --reset
```

**Comandos de operación:**

```bash
# Ver el estado de bloqueo de cualquier usuario
sudo faillock --user <usuario>

# Desbloquear manualmente
sudo faillock --user <usuario> --reset

# Ver los eventos de fallo en auditd (requiere auditd activo, sección 12)
sudo ausearch -m USER_LOGIN -sv no | tail -20
```

#### 7.4 Política de contraseñas

Debian trae valores por defecto razonables. Reforzarlos editando `/etc/login.defs`:

```bash
sudoedit /etc/login.defs
```

Valores recomendados:

```
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_MIN_LEN    12
PASS_WARN_AGE   14
```

> **Nota:** `PASS_MIN_LEN` en `login.defs` es **ignorado** cuando `pam_pwquality` está activo (que es el caso tras instalar `libpam-pwquality` en el siguiente paso). El mínimo efectivo lo determina el parámetro `minlen` de `pam_pwquality`, no esta directiva. `PASS_MIN_LEN` solo aplica si PAM no usa pwquality, lo que no es el caso en este servidor. Se mantiene el valor por coherencia y como fallback, pero el control real está en `pam_pwquality`.

Estos valores afectan a usuarios creados después del cambio. Para aplicar la política al usuario actual:

```bash
sudo chage -M 90 -m 1 -W 14 $MIUSUARIO
sudo chage -l $MIUSUARIO
```

Instalar `libpam-pwquality` para exigir contraseñas robustas:

```bash
sudo apt install -y libpam-pwquality
```

Editar `/etc/pam.d/common-password`. Buscar la línea que contiene `pam_pwquality.so` y ajustarla (o añadir si no existe) para requerir complejidad razonable:

```
password requisite pam_pwquality.so retry=3 minlen=12 difok=3 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 reject_username enforce_for_root
```

Significado de los parámetros:

- `minlen=12` — mínimo 12 caracteres (este es el valor efectivo, no el de `login.defs`).
- `difok=3` — al menos 3 caracteres distintos respecto a la contraseña anterior.
- `ucredit=-1` `lcredit=-1` `dcredit=-1` `ocredit=-1` — al menos una mayúscula, una minúscula, un dígito y un símbolo.
- `reject_username` — rechaza contraseñas que contengan el nombre de usuario.
- `enforce_for_root` — aplica la política también a root.

#### 7.5 Creación de usuarios adicionales (si aplica)

Para un servidor administrado por una sola persona, normalmente basta con el usuario creado durante la instalación. Si hay más administradores:

```bash
sudo adduser <nuevo_usuario>
sudo usermod -aG sudo,adm <nuevo_usuario>
```

`adduser` (frontend de `useradd`) gestiona automáticamente el home, el shell, el grupo primario y solicita contraseña.

#### 7.6 Bloqueo de la cuenta de root

Llegados a este punto, el usuario administrativo tiene `sudo` funcional, acceso SSH por llave verificado, y la sesión activa confirmada. Se puede bloquear root con seguridad.

Bloquear la contraseña de root (deshabilita login por contraseña pero mantiene la cuenta operativa para procesos internos):

```bash
sudo passwd -l root
```

Verificar:

```bash
sudo passwd -S root
# Debe mostrar:  root L <fecha> ...   (L = locked)
```

Consecuencias del bloqueo:

- `su -` ya no funciona pidiendo contraseña de root. Sigue funcionando como `sudo su -` o `sudo -i`.
- No se puede iniciar sesión como root por consola ni por SSH (SSH además debe tener `PermitRootLogin no`, que se aplica en la Etapa III).
- Procesos del sistema que necesitan ejecutarse como root siguen funcionando sin cambios (cron, systemd, etc.).

Si se quiere recuperar el acceso a root más adelante:

```bash
sudo passwd root
```

Esto establece una contraseña nueva y desbloquea la cuenta. No requiere conocer la anterior.

#### 7.7 Checklist de cierre de la Etapa II

- [ ] Usuario administrativo con `sudo` y `adm` verificados
- [ ] `/etc/sudoers.d/99-hardening` creado y validado con `visudo -c`
- [ ] `pam_faillock` configurado en `/etc/security/faillock.conf` e integrado en `common-auth`
- [ ] Política de contraseñas configurada (`login.defs`, `pwquality`)
- [ ] Usuarios adicionales creados si aplica
- [ ] Cuenta de root bloqueada (`passwd -l root`)
- [ ] Acceso del usuario administrativo por SSH con llave confirmado funcional
- [ ] Sesión de rescate abierta en paralelo (hábito para la Etapa III)

Con la Etapa II cerrada, el servidor es administrable con herramientas estándar, tiene identidad definida, repositorios correctos, está actualizado, y tiene política básica de usuarios. Está listo para la fase de endurecimiento (Etapa III).

---

## Etapa III — Hardening

Las partes I y II dejaron el servidor instalado, identificable, accesible por SSH con llave pública, actualizado y con un usuario administrativo definido. Esta parte lo endurece: reduce la superficie de ataque, dificulta los abusos cuando un atacante ya tiene un punto de apoyo, y deja huellas de auditoría útiles para un análisis forense posterior.

El hardening que se cubre aquí es generalista — el mínimo razonable para cualquier servidor Debian 13 expuesto a una red. Roles específicos (servidor web, base de datos, mail, etc.) añaden capas adicionales que se documentan en sus respectivos manuales.

Tres principios atraviesan toda esta parte:

1. Mantener una sesión de rescate abierta en paralelo. Cualquiera de los cambios de esta parte puede dejar el servidor inaccesible si se aplica mal. La sesión de rescate (sección 4.4) y el `tmux` persistente son la red de seguridad. Antes de cada modificación que toca SSH, firewall o red, confirmar que la sesión paralela sigue viva.

2. Cambio modular, no edición de archivos canónicos. Donde sea posible, los cambios van en archivos `*.conf` dentro de directorios `*.d/` (`/etc/ssh/sshd_config.d/`, `/etc/sysctl.d/`, `/etc/sudoers.d/`). Esto preserva los archivos originales del paquete, facilita los upgrades futuros, y permite revertir un cambio borrando un solo archivo.

3. Probar antes de aplicar. Cada herramienta clave tiene un modo de validación (`sshd -t`, `ufw --dry-run`, `unattended-upgrades --dry-run`). Usarlo siempre antes de recargar o activar.

### 8. Hardening de SSH

SSH es la puerta principal del servidor. Su endurecimiento es la primera prioridad porque cualquier otra medida (firewall, fail2ban, sysctl) cae si SSH está mal configurado y un atacante consigue una sesión.

El objetivo de esta sección es:

- Cambiar el puerto por defecto.
- Deshabilitar autenticación por contraseña.
- Restringir qué usuarios pueden entrar.
- Acotar tiempos y sesiones.
- Limitar los algoritmos criptográficos a opciones modernas.
- Validar y recargar sin perder la sesión activa.

#### 8.1 Archivo modular `/etc/ssh/sshd_config.d/99-hardening.conf`

Desde OpenSSH 8.2 (Debian 11 en adelante), `/etc/ssh/sshd_config` incluye al inicio una directiva `Include /etc/ssh/sshd_config.d/*.conf`. Las directivas declaradas en archivos de ese directorio se aplican antes que las del archivo principal, y las primeras ocurrencias ganan. Esto significa que un archivo en `sshd_config.d/` con prefijo numérico bajo (`99-`) toma precedencia sobre el `sshd_config` original sin modificarlo.

Ventaja práctica: en una actualización mayor, el `sshd_config` lo reemplaza el paquete sin conflicto. El hardening propio queda intacto en su archivo separado.

```bash
# Definir las variables
export MIPUERTO=17177
export MIUSUARIO=usuario # Ya se definió en 3.2.2

# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt" \
    -O /tmp/plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt

# Verificar integridad (reemplazar <SHA256> con el hash obtenido para este SHA_COMMIT)
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

# Reemplazar variables y generar el archivo en destino
sed -e "s|\$MIPUERTO|${MIPUERTO}|g" \
    -e "s|\$MIUSUARIO|${MIUSUARIO}|g" \
    /tmp/plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt \
    | sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null

# Permisos correctos (recomendado para sshd_config.d)
sudo chmod 600 /etc/ssh/sshd_config.d/99-hardening.conf
sudo chown root:root /etc/ssh/sshd_config.d/99-hardening.conf
```

#### 8.2 Cambio de puerto, password auth y solo pubkey

**Puerto.** Cambiar el puerto por defecto (22) a uno alto (17177) reduce drásticamente el ruido de los escáneres automáticos. No es seguridad real (un escaneo dirigido encuentra el puerto en segundos), pero corta el 99% de los intentos de fuerza bruta automatizados, lo cual reduce el volumen de logs y libera CPU. Es un cambio gratuito con un efecto medible.

Si el servidor está detrás de un firewall corporativo o un proveedor de nube con grupos de seguridad, es mandatorio abrir el nuevo puerto en ambos lados antes de reiniciar SSH.

Antes de aplicar:

- Abrir puerto en:
  - Firewall del proveedor
  - Security groups
  - Router/NAT

Validar:

```bash
nc -zv <ip> <nuevo_puerto>
```

**Password auth deshabilitado.** Solo se acepta autenticación por llave pública. La sesión paralela del paso 4.4 ya está abierta con llave funcional, así que apagar `PasswordAuthentication` no rompe nada. Si la llave aún no se ha probado, **no aplicar este cambio todavía**: volver al paso 4.3.

**`KbdInteractiveAuthentication no` y `ChallengeResponseAuthentication no`.** Sin estas dos, OpenSSH puede caer de vuelta a un prompt interactivo aunque `PasswordAuthentication` esté en `no` (vía PAM). Cerrar las dos garantiza que solo la llave funciona.

**`UsePAM yes` se mantiene.** PAM se sigue usando para gestión de sesión (límites, cuotas, mensajes de login), no para autenticación.

#### 8.3 Restricción de usuarios (`AllowUsers`)

`AllowUsers` es una lista blanca: solo los usuarios listados pueden iniciar sesión por SSH, sin importar que la llave esté instalada en otra cuenta. Esto previene escenarios donde un usuario del sistema sin shell (creado por algún paquete) tenga llave SSH configurada por error.

```
AllowUsers $MIUSUARIO
```

Para varios administradores:

```
AllowUsers <usuario1> <usuario2>
```

`AllowUsers` también acepta restricción por origen: `AllowUsers usuario2@192.168.1.0/24` permite a `usuario2` solo desde esa red. Útil cuando se sabe desde dónde se administra el servidor.

`AllowGroups` es una alternativa más mantenible si hay muchos usuarios: se permite por pertenencia a un grupo, por ejemplo `ssh-admins`. Para servidores con uno o dos administradores, `AllowUsers` con nombres explícitos es más claro.

#### 8.4 Límites de sesión y timeouts

| Directiva                 | Valor | Efecto                                                                       |
|---------------------------|-------|------------------------------------------------------------------------------|
| `MaxAuthTries 3`          | 3     | El cliente se desconecta tras 3 intentos fallidos en una misma conexión.     |
| `MaxSessions 5`           | 5     | Máximo de sesiones (canales) por conexión TCP. Suficiente para un humano.    |
| `LoginGraceTime 30`       | 30 s  | Tiempo máximo para completar el login. Corta sesiones que solo abren TCP.    |
| `ClientAliveInterval 300` | 5 min | Cada 5 min envía un keepalive cifrado al cliente.                            |
| `ClientAliveCountMax 2`   | 2     | Tras 2 keepalives sin respuesta (10 min), cierra la sesión.                  |

`ClientAliveInterval` + `ClientAliveCountMax` cumplen dos propósitos: mantener la sesión viva contra firewalls que cortan conexiones inactivas, y desconectar sesiones zombies (atacante que abre conexión y la deja colgada).

`X11Forwarding`, `AllowAgentForwarding`, `AllowTcpForwarding`, `StreamLocalForwarding` y `PermitTunnel` se desactivan porque en un servidor administrado por SSH plano no se usan, y cada uno permite vectores de uso indebido (port forwarding hacia servicios internos, túneles SOCKS, reenvío de sockets Unix).

En OpenSSH 8.9 y posteriores (disponible en Debian 13 Trixie), existe la directiva `DisableForwarding yes` que desactiva en una sola línea todos los tipos de forwarding (TCP, Unix socket, X11, agent, tunnel). Es equivalente a desactivar cada opción individualmente, pero más concisa y más robusta ante forwardings futuros que pudieran añadirse al protocolo. Si la plantilla no la incluye, puede añadirse manualmente al archivo `99-hardening.conf`:

```
DisableForwarding yes
```

> Si en el futuro se necesita forwarding puntual (`ssh -L`), habilitar `AllowTcpForwarding` únicamente para el usuario que lo requiere con un bloque `Match User`, sin modificar la política base:
>
> ```
> Match User operador-tuneles
>     AllowTcpForwarding yes
> ```

#### 8.5 Algoritmos criptográficos modernos

OpenSSH soporta algoritmos legacy por compatibilidad (3DES, SHA1, RSA-SHA1, grupos DH débiles). En un servidor moderno donde los clientes son administrados, se puede recortar la lista a algoritmos de calidad actual. Beneficio: cierra ataques de downgrade y reduce superficie criptográfica.

**Key exchange (`KexAlgorithms`).**

```
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,sntrup761x25519-sha512@openssh.com
```

- `curve25519-sha256` — el estándar moderno, presente en todos los OpenSSH recientes.
- `sntrup761x25519-sha512@openssh.com` — híbrido post-cuántico, default desde OpenSSH 9.0. Protege contra "harvest now, decrypt later".

**Cifradores (`Ciphers`).**

```
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
```

Solo AEAD modernos. Se excluyen explícitamente CBC y CTR.

**MACs.**

```
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

Solo Encrypt-then-MAC con SHA-2. SHA-1 queda fuera.

**Algoritmos de host key (`HostKeyAlgorithms`).**

```
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

Excluye `ssh-rsa` (firmas RSA con SHA-1). En clientes muy antiguos puede causar incompatibilidad. Si el operador llega desde una estación reciente, no hay problema.

> Para validar que la combinación es compatible con el cliente desde el que se administra, antes de recargar SSH ejecutar desde el cliente: `ssh -Q kex`, `ssh -Q cipher`, `ssh -Q mac`. Las listas devueltas deben incluir al menos uno de cada categoría configurada en el servidor.

#### 8.6 Validación y recarga segura

**Antes de tocar nada en producción:**

1. La sesión paralela del paso 4.4 sigue conectada.
2. La llave del administrador está probada y en `~/.ssh/authorized_keys`.
3. Se conoce la IP del servidor y el puerto nuevo.

**Validar la sintaxis del archivo:**

```bash
sudo sshd -t
```

Si no hay salida, la configuración es sintácticamente correcta. Si hay errores, no recargar — corregir primero.

**Recarga sin cortar sesiones existentes:**

```bash
sudo systemctl reload ssh
```

`reload` (a diferencia de `restart`) no termina las conexiones SSH activas. La sesión actual sigue viva con la configuración antigua, y las nuevas conexiones usan la configuración nueva.

**Probar el acceso desde otra ventana antes de cerrar la sesión actual:**

```bash
ssh -p $MIPUERTO $MIUSUARIO@<ip.del.servidor>
```

Solo cuando la conexión nueva funciona, se puede cerrar la sesión antigua con tranquilidad.

> Si la conexión nueva falla, **no cerrar la sesión antigua**. Diagnosticar desde ahí: `journalctl -u ssh -n 50`, `sudo sshd -T | grep -E 'port|allowusers|passwordauthentication'` (muestra la configuración efectiva), `sudo ss -tlnp | grep ssh` (¿está escuchando en el puerto correcto?).

**Actualizar `~/.ssh/config` del administrador** para no escribir el puerto cada vez:

```
# En el cliente, en ~/.ssh/config
Host miservidor
    HostName <ip.o.dns.del.servidor>
    Port $MIPUERTO
    User $MIUSUARIO
    IdentityFile ~/.ssh/<llave>
```

A partir de ese momento, `ssh miservidor` reemplaza a `ssh -p $MIPUERTO $MIUSUARIO@<ip.del.servidor>`.

#### 8.7 Checklist de cierre

- [ ] `/etc/ssh/sshd_config.d/99-hardening.conf` creado con el contenido endurecido
- [ ] `sudo sshd -t` sin errores
- [ ] `sudo systemctl reload ssh` aplicado
- [ ] Conexión por el puerto nuevo verificada **desde otra ventana antes de cerrar la actual**
- [ ] `~/.ssh/config` del cliente actualizado
- [ ] `sudo ss -tlnp | grep -E ":(22|${MIPUERTO}) "` muestra solo el puerto nuevo

### 9. Firewall (UFW)

UFW (Uncomplicated Firewall) es un frontend de `nftables` (en Debian 13; antes era `iptables`) pensado para configuraciones de servidor sencillas. No reemplaza a `nftables` directamente para escenarios complejos (NAT, balanceo, reglas por interfaz), pero para un servidor base con unos pocos puertos abiertos es la opción más manejable.

La filosofía es:

- Denegar por defecto el tráfico entrante. Solo se abre lo estrictamente necesario.
- Permitir todo el tráfico saliente. Restringirlo es posible, pero rompe demasiadas cosas (DNS, NTP, repositorios) y se gana poco. Mantenerlo abierto es la práctica estándar en servidores que no son endpoints de red corporativa.
- IPv6 activo si la red lo soporta. Apagar IPv6 en UFW no apaga IPv6 en el sistema, solo lo deja sin firewall.

El firewall se configura antes de activarlo. Activarlo con reglas incorrectas puede cortar la sesión SSH y el servidor queda inaccesible. Por eso esta sección termina con `ufw enable` solo cuando todas las reglas están listas y verificadas.

#### 9.1 Instalación y políticas por defecto

```bash
sudo apt install -y ufw
```

UFW viene desactivado tras instalarse. Hay tiempo para configurarlo sin riesgo.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

`deny routed` solo importa si el servidor actúa como router (forwarding habilitado). En un servidor estándar es buena práctica dejarlo en `deny`: si en el futuro se habilita IP forwarding por algún motivo, este default impide que se convierta en router abierto por accidente.

#### 9.2 IPv6 (`/etc/default/ufw`)

Verificar que UFW gestiona IPv6:

```bash
sudoedit /etc/default/ufw
```

La línea relevante debe estar en:

```
IPV6=yes
```

Es el valor por defecto en Debian 13, pero conviene confirmarlo. Si se deja en `no` y la interfaz tiene IPv6 activo, el tráfico IPv6 entrante no es filtrado por UFW (queda gobernado por las cadenas raw de nftables).

#### 9.3 Apertura selectiva de puertos según rol

**SSH con rate limiting.** El puerto SSH (17177 según la sección 8) se abre con `limit` en lugar de `allow`. Esto activa una regla integrada en UFW que rechaza conexiones desde una misma IP si superan 6 intentos en 30 segundos:

```bash
export MIPUERTO=17177 # Definido en 8.1
sudo ufw limit ${MIPUERTO}/tcp comment 'SSH endurecido con rate limit'
```

`ufw limit` no reemplaza a fail2ban (sección 10): el rate limit de UFW es de grano fino y temporal; fail2ban banea por minutos u horas con base en patrones de log. Se complementan.

**Servicios web (cuando aplique).** Si el servidor expone web público:

```bash
sudo ufw allow 80/tcp  comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
```

**Reglas más restrictivas con origen específico.** Cuando se conoce desde dónde se administra:

```bash
# SSH solo desde una red administrativa
sudo ufw limit from 192.0.2.0/24 to any port ${MIPUERTO} proto tcp comment 'SSH desde red admin'

# Base de datos accesible solo desde el servidor de aplicación
sudo ufw allow from 10.0.0.5 to any port 5432 proto tcp comment 'Postgres desde app'
```

**No abrir lo que no se va a usar.** En el manual base solo se abre SSH. Los puertos de servicios concretos los abre el manual del rol cuando corresponda.

**Filtrado de egreso (cuando el rol es conocido).** La política base `allow outgoing` es correcta para un servidor generalista. Cuando el rol del servidor está definido, restringir el tráfico saliente a lo estrictamente necesario reduce el impacto de un compromiso: un servidor web comprometido que no puede establecer conexiones salientes arbitrarias dificulta la exfiltración de datos y la comunicación con infraestructura de C2. Por ejemplo, un servidor que solo necesita acceder a repositorios Debian, servidores NTP y un relay SMTP puede restringirse a:

```bash
# Cambiar la política de egreso a deny y abrir solo lo necesario
sudo ufw default deny outgoing
sudo ufw allow out 53/udp comment 'DNS'
sudo ufw allow out 80/tcp comment 'HTTP (apt, mirrors)'
sudo ufw allow out 443/tcp comment 'HTTPS (apt, mirrors)'
sudo ufw allow out 123/udp comment 'NTP'
sudo ufw allow out 587/tcp comment 'SMTP relay'
```

Este nivel de control es opcional en el manual base y se documenta en los manuales de rol cuando aplica.

#### 9.4 Inspección antes de activar

Antes de `enable`, listar las reglas pendientes:

```bash
sudo ufw show added
```

Esto muestra las reglas que están registradas pero todavía no activas. Revisar que aparece la de SSH y que el puerto es el correcto. Si hay una regla mal escrita, eliminarla:

```bash
sudo ufw delete <numero o regla literal>
```

#### 9.5 Activación final y verificación

```bash
sudo ufw enable
```

UFW advierte: *"Command may disrupt existing ssh connections. Proceed with operation (y|n)?"*. Aquí es cuando importa que la regla de SSH esté correctamente configurada. Confirmar `y`.

Inmediatamente, **desde otra ventana**, probar la conexión SSH al puerto nuevo. Si funciona, todo está bien. Si no funciona, la sesión paralela aún conectada permite recuperar:

```bash
sudo ufw disable
# Diagnosticar y volver a aplicar
```

**Verificación del estado:**

```bash
sudo ufw status verbose
sudo ufw status numbered
```

`numbered` es útil cuando hay que borrar una regla específica: `sudo ufw delete <n>`.

**Ver las reglas reales en nftables (para depuración avanzada):**

```bash
sudo nft list ruleset | less
```

UFW genera reglas nftables a partir de su configuración de alto nivel. Esto permite ver exactamente qué está aplicando el kernel.

#### 9.6 Checklist de cierre

- [ ] `IPV6=yes` confirmado en `/etc/default/ufw`
- [ ] Política por defecto `deny incoming`, `allow outgoing`, `deny routed`
- [ ] Regla SSH (`limit`) con el puerto correcto
- [ ] Solo se abren puertos justificados
- [ ] `sudo ufw enable` ejecutado y conexión nueva verificada
- [ ] `sudo ufw status verbose` revisado

### 10. Protección contra fuerza bruta (fail2ban)

UFW con `limit` corta ráfagas cortas en SSH. fail2ban añade una capa complementaria: lee los logs (en Trixie, vía journald), detecta patrones de abuso (autenticación fallida, escaneo, bots), y banea las IPs ofensoras durante un período configurable.

En un servidor con SSH solo por llave (sección 8), un atacante automatizado nunca pasa de la fase de autenticación, pero igual genera ruido en logs, consume CPU, y deja huella en `journalctl` que dificulta encontrar eventos reales. fail2ban silencia ese ruido.

> **Relación con `pam_faillock` (§7.3.3):** fail2ban actúa sobre intentos de acceso remoto por red (SSH, servicios web, etc.). `pam_faillock` actúa sobre autenticación local a nivel PAM, cubriendo `su`, `sudo`, login de consola y otros servicios que no generan entradas que fail2ban pueda parsear. Las dos capas son complementarias y no se solapan.

#### 10.1 Instalación

```bash
sudo apt install -y fail2ban
```

En Debian 13, fail2ban se integra con `nftables` y `journald` por defecto. Esto cambia respecto a versiones anteriores que usaban `iptables` y archivos de log planos. La consecuencia práctica es que no hay que tocar la configuración del backend — los defaults funcionan con el sistema actual.

#### 10.2 Configuración local (`jail.local`)

fail2ban distribuye su configuración en `/etc/fail2ban/jail.conf`. Ese archivo se sobrescribe en upgrades. La forma correcta de personalizar es crear `/etc/fail2ban/jail.local`:

```bash
export MIPUERTO=17177 # Definido en 8.1

# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-fail2ban-jail.local.txt" \
    -O /tmp/plantilla-etc-fail2ban-jail.local.txt

# Verificar integridad (reemplazar <SHA256> con el hash obtenido para este SHA_COMMIT)
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-fail2ban-jail.local.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

# Reemplazar variables y generar el archivo en destino
sed -e "s|\$MIPUERTO|${MIPUERTO}|g" \
    /tmp/plantilla-etc-fail2ban-jail.local.txt \
    | sudo tee /etc/fail2ban/jail.local > /dev/null

# Permisos correctos 
sudo chmod 640 /etc/fail2ban/jail.local
```

**Significado de los parámetros clave:**

- `bantime` — duración del baneo. 1 hora es agresivo pero razonable.
- `findtime` — ventana en la que se cuentan los `maxretry` fallos.
- `maxretry` — fallos permitidos en `findtime` antes de banear.
- `bantime.increment` — cada vez que la misma IP reincide, el ban se multiplica (1h, 2h, 4h, 8h... hasta `maxtime`).
- `mode = aggressive` en el jail `sshd` — incluye no solo "Failed password" sino también desconexiones por preauth, autenticación incorrecta, intentos de protocolos viejos.

`port = 17177` debe coincidir con el puerto SSH endurecido. Si en algún momento se cambia el puerto, recordar actualizar este valor.

> **Atención al `ignoreip`.** Añadir aquí las redes desde las que se administra el servidor evita un autoban accidental tras varios fallos legítimos (llave equivocada, agente sin cargar). Pero no añadir redes amplias o no controladas — eso anula la protección.

```ini
ignoreip = 127.0.0.1/8 ::1 192.0.2.0/24
```

#### 10.3 Activación

```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban --no-pager
```

Si la configuración tiene un error de sintaxis, `status` muestra `failed`. Inspeccionar:

```bash
sudo journalctl -u fail2ban -n 50 --no-pager
```

#### 10.4 Monitoreo de bans

Estado general:

```bash
sudo fail2ban-client status
```

Detalle del jail SSH:

```bash
sudo fail2ban-client status sshd
```

Salida típica:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     147
|  `- File list:        /var/log/auth.log (via systemd)
`- Actions
   |- Currently banned: 5
   |- Total banned:     38
   `- Banned IP list:   45.x.x.x 91.x.x.x ...
```

**Comandos útiles de operación:**

```bash
# Desbanear una IP manualmente (por ejemplo, si fue uno mismo)
sudo fail2ban-client set sshd unbanip 1.2.3.4

# Banear manualmente una IP
sudo fail2ban-client set sshd banip 1.2.3.4

# Ver IPs que han sido baneadas alguna vez en este jail
sudo fail2ban-client get sshd banip --with-time
```

#### 10.5 Checklist de cierre

- [ ] `fail2ban` instalado
- [ ] `/etc/fail2ban/jail.local` con jail `sshd` activo y puerto correcto
- [ ] `systemctl status fail2ban` muestra `active (running)`
- [ ] `fail2ban-client status sshd` muestra el jail operativo
- [ ] `ignoreip` incluye solo redes de administración legítimas

### 11. Hardening del kernel y del sistema

Esta sección aplica hardening a nivel de kernel mediante `sysctl`, ajusta límites de recursos, deshabilita módulos innecesarios, y protege el bootloader cuando aplica.

Los parámetros `sysctl` modifican comportamiento del kernel en caliente. Las modificaciones persistentes se hacen en archivos `*.conf` dentro de `/etc/sysctl.d/`. Como en SSH y UFW, se evita editar archivos canónicos (`/etc/sysctl.conf`) y se usa un archivo propio de hardening.

#### 11.1 Parámetros sysctl de red y de kernel

```bash
# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-sysctl.d-99-hardening.conf.txt" \
    -O /tmp/plantilla-etc-sysctl.d-99-hardening.conf.txt

# Verificar integridad antes de aplicar como root
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-sysctl.d-99-hardening.conf.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

sudo cp /tmp/plantilla-etc-sysctl.d-99-hardening.conf.txt /etc/sysctl.d/99-hardening.conf
```

**Justificación de los más relevantes:**

- `rp_filter = 1` — Reverse Path Filtering en modo estricto. Si un paquete llega desde una IP origen para la que la ruta de retorno usaría otra interfaz, se descarta. Bloquea spoofing trivial.
- `tcp_syncookies = 1` — Cuando la cola de SYN se llena (típico en un SYN flood), el kernel responde con cookies en lugar de mantener estado. Permite distinguir conexiones reales bajo ataque.
- `accept_redirects = 0` y `send_redirects = 0` — Los ICMP redirects son una vía clásica de man-in-the-middle local. Solo tienen sentido en routers, y este servidor no lo es.
- `log_martians = 1` — Paquetes con direcciones de origen imposibles se loguean en `dmesg`. Útil para detectar configuraciones de red rotas o intentos de ataque.

**Aplicar los parámetros:**

```bash
sudo sysctl --system
```

`--system` recarga `/etc/sysctl.conf`, todos los archivos en `/etc/sysctl.d/`, y los del paquete (`/usr/lib/sysctl.d/`). La salida lista cada parámetro aplicado.

**Verificar un parámetro concreto:**

```bash
sudo sysctl kernel.kptr_restrict
sudo sysctl net.ipv4.tcp_syncookies
```

En Debian 13, varios de estos parámetros ya tienen valores razonables por defecto. El archivo `99-hardening.conf` los fija explícitamente para que el endurecimiento no dependa de defaults futuros.

#### 11.2 Límites de recursos (`/etc/security/limits.conf`)

`limits.conf` define límites de procesos, memoria, archivos abiertos, etc. para usuarios y sesiones. En un servidor base, los defaults son aceptables. Conviene ajustar dos cosas:

**Deshabilitar core dumps para usuarios normales:**

```bash
sudoedit /etc/security/limits.d/99-hardening.conf
```

Agregar:

```
# Sin core dumps por defecto (los core pueden contener datos sensibles en RAM)
*               hard    core            0
*               soft    core            0
```

Los core dumps son útiles para depurar, pero en producción su valor es bajo y su riesgo (filtración de credenciales, claves) es real. Si en algún momento se necesita depurar un proceso concreto, se levanta el límite solo para esa sesión.

#### 11.3 Deshabilitar módulos de kernel innecesarios

Cada módulo cargado es código ejecutándose en kernel space. Los módulos no usados son superficie de ataque gratuita. En un servidor, varios buses y filesystems no se necesitan nunca.

```bash
# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-modprobe.d-99-hardening.conf.txt" \
    -O /tmp/plantilla-etc-modprobe.d-99-hardening.conf.txt

# Verificar integridad antes de aplicar como root
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-modprobe.d-99-hardening.conf.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

sudo cp /tmp/plantilla-etc-modprobe.d-99-hardening.conf.txt /etc/modprobe.d/99-hardening.conf
```

`install <modulo> /bin/true` hace que cualquier intento de cargar el módulo ejecute `/bin/true` (que termina con éxito sin hacer nada). Más limpio que `blacklist`, que solo evita la carga automática pero permite la carga explícita.

> **Especial cuidado con USB storage en servidores físicos donde el operador usa pendrives administrativos.** No deshabilitar `usb-storage` si se necesita conectar discos externos para respaldos. En servidores de colocation o data center, donde nadie debería conectar nada físicamente, sí tiene sentido bloquearlo:
>
> ```
> install usb-storage /bin/true
> ```
>
> En VMs y VPS no aplica — no hay USB físico.

**Aplicar:**

Los `install` directives toman efecto al siguiente intento de carga del módulo. Para descargar un módulo ya cargado:

```bash
sudo lsmod | grep firewire
sudo modprobe -r firewire-core   # si está cargado y no se usa
```

#### 11.4 Protección de GRUB (solo bare metal)

En servidores físicos donde el atacante puede acceder a la consola, GRUB permite editar la línea de kernel y arrancar con `init=/bin/bash`, lo que da shell de root sin contraseña. La protección se hace exigiendo contraseña para **editar** entradas, no para arrancar (eso convertiría el servidor en no-rebooteable sin intervención humana).

**No aplicar esto en VMs ni VPS.** En VMs, la consola la controla el hypervisor; en VPS, el proveedor maneja el booteo. Aplicarlo solo si el servidor es físico y la consola es accesible.

```bash
# Generar hash de contraseña para GRUB
grub-mkpasswd-pbkdf2
# Pide contraseña dos veces, devuelve un hash que comienza con 'grub.pbkdf2.sha512.10000...'
```

Crear `/etc/grub.d/40_password`:

```bash
sudoedit /etc/grub.d/40_password
```

```
#!/bin/sh
exec tail -n +3 $0
# Este archivo añade un usuario admin al menú de GRUB.
# El usuario admin puede editar entradas; los usuarios normales solo arrancan.
set superusers="admin"
password_pbkdf2 admin <hash-pegado-aqui>
```

```bash
sudo chmod +x /etc/grub.d/40_password
```

Configurar las entradas para que sean arrancables sin contraseña pero no editables. En `/etc/grub.d/10_linux`, la directiva por defecto añade `--unrestricted` automáticamente desde Debian 11 si se detecta `superusers`, pero conviene verificar tras regenerar:

```bash
sudo update-grub
sudo grep -E 'menuentry|unrestricted' /boot/grub/grub.cfg | head -20
```

Cada `menuentry` debe tener `--unrestricted` para que arranque sin contraseña.

#### 11.5 Checklist de cierre

- [ ] `/etc/sysctl.d/99-hardening.conf` creado y aplicado con `sudo sysctl --system`
- [ ] Parámetros clave verificados (`tcp_syncookies`, `kptr_restrict`, `randomize_va_space`)
- [ ] `/etc/security/limits.d/99-hardening.conf` con core dumps deshabilitados
- [ ] `/etc/modprobe.d/99-hardening.conf` con módulos innecesarios bloqueados
- [ ] (Solo bare metal) GRUB protegido con contraseña para edición

### 12. Auditoría y detección

Las secciones anteriores previenen: cierran puertos, deniegan accesos, endurecen el kernel. Esta sección detecta: deja constancia de lo que ocurre en el sistema para que un compromiso futuro pueda investigarse. La detección sin prevención es ciega; la prevención sin detección es sorda. Las dos se complementan.

El alcance aquí es deliberadamente modesto: reglas mínimas de `auditd` que cubren los archivos críticos de identidad y privilegio, escaneo periódico contra rootkits conocidos, y verificación de integridad de paquetes y archivos del sistema. Configuraciones más exhaustivas (PCI-DSS, CIS Level 2, etc.) las cubren manuales de compliance específicos.

#### 12.1 `auditd` — reglas mínimas

`auditd` es el subsistema de auditoría del kernel Linux. Registra eventos del kernel (llamadas al sistema, accesos a archivos, ejecución de binarios) en `/var/log/audit/audit.log`. Es invasivo y verboso si se usa sin criterio, así que el enfoque es vigilar pocos archivos pero los importantes.

**Instalación:**

```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

**Reglas básicas:**

```bash
# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-audit-rules.d-99-hardening.rules.txt" \
    -O /tmp/plantilla-etc-audit-rules.d-99-hardening.rules.txt

# Verificar integridad antes de aplicar como root
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-audit-rules.d-99-hardening.rules.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

sudo cp /tmp/plantilla-etc-audit-rules.d-99-hardening.rules.txt /etc/audit/rules.d/99-hardening.rules
```

**Recargar reglas:**

```bash
sudo augenrules --load
sudo systemctl restart auditd
sudo auditctl -l   # listar reglas activas
```

**Ejemplos de consulta del log:**

```bash
# Eventos recientes con la clave 'identidad'
sudo ausearch -k identidad -ts recent

# Resumen por usuario
sudo aureport -u --summary

# Resumen de ejecuciones como root por usuarios normales
sudo ausearch -k root_exec | sudo aureport -x
```

El log de auditoría crece. `auditd` rota automáticamente, pero conviene revisar `/etc/audit/auditd.conf` para ajustar `max_log_file` y `num_logs` según el espacio disponible en `/var/log` (que en el esquema de particionado de la sección 2.2 es un LV separado precisamente por esto).

#### 12.2 `rkhunter` — escaneo de rootkits

`rkhunter` busca firmas conocidas de rootkits, archivos sospechosos en directorios del sistema, y modificaciones en binarios críticos. No es infalible (rootkits modernos lo evaden), pero detecta el malware automatizado de uso amplio.

```bash
sudo apt install -y rkhunter

# Crear/agregar overrides locales
sudo tee -a /etc/rkhunter.conf.local > /dev/null <<'EOF'
# Override: PermitRootLogin está en sshd_config.d/99-hardening.conf
ALLOW_SSH_ROOT_USER=no
ALLOW_SSH_PROT_V1=0

# Whitelist de archivos ocultos legítimos del sistema
ALLOWHIDDENFILE=/etc/.updated
EOF

# Verificar la configuración antes de escanear
sudo rkhunter -C
```

**Inicialización:** establecer la línea base (hashes "buenos" de los binarios del sistema):

```bash
sudo rkhunter --propupd
```

Esto debe ejecutarse inmediatamente después de la instalación inicial o de cualquier `apt upgrade` de lo contrario, los upgrades aparecen como modificaciones sospechosas.

**Escaneo manual:**

```bash
sudo rkhunter --check --skip-keypress
```

**Escaneo automático.** El paquete instala un job en `/etc/cron.daily/rkhunter`. Verificar que está activo y que los reportes llegan a algún sitio (correo del root local, syslog):

```bash
sudoedit /etc/default/rkhunter
```

```
CRON_DAILY_RUN="true"
CRON_DB_UPDATE="true"
DB_UPDATE_EMAIL="true"
REPORT_EMAIL="root"
APT_AUTOGEN="true"
```

`APT_AUTOGEN="true"` regenera la base de hashes después de cada `apt upgrade`, evitando falsos positivos por actualizaciones legítimas.

#### 12.3 `debsums` — integridad de paquetes

`debsums` compara los hashes de los archivos instalados contra los registrados en los paquetes Debian. Detecta modificaciones de archivos que pertenecen a paquetes (binarios, configs por defecto).

```bash
sudo apt install -y debsums
```

**Escaneo manual:**

```bash
sudo debsums -ac
```

- `-a` — incluye archivos de configuración.
- `-c` — solo muestra los que difieren del original.

**Escaneo automático:**

```bash
sudoedit /etc/default/debsums
```

```
CRON_CHECK=daily
```

Esto deja un job en `/etc/cron.daily/debsums` que ejecuta el chequeo y manda el reporte por correo a root.

> `debsums` complementa a `rkhunter`: el primero detecta modificaciones en archivos de paquetes Debian (lo que un atacante podría parchear); el segundo busca binarios sospechosos que **no** pertenecen a ningún paquete.

#### 12.4 AIDE — integridad de archivos del sistema

AIDE (*Advanced Intrusion Detection Environment*) crea una "fotografía" criptográfica del sistema (hashes, permisos, propietarios, tamaños) y permite detectar cualquier modificación posterior. Es la línea de defensa que avisa si un atacante modificó un binario, agregó un backdoor o alteró un archivo de configuración después de un compromiso.

##### Instalación

```bash
sudo apt install -y aide aide-common
```

El paquete `aide` provee el binario; `aide-common` provee el archivo de configuración base (`/etc/aide/aide.conf`), el script de inicialización (`aideinit`) y el job diario (`/usr/share/aide/bin/dailyaidecheck`).

##### Inicialización de la base de datos

El comando `aideinit` genera la base de datos inicial recorriendo todo el sistema y calculando hashes. Tarda entre **2 y 10 minutos** dependiendo del tamaño del sistema:

```bash
sudo aideinit
```

Salida esperada:

```
Running aide --init...
```

La base recién creada queda en `/var/lib/aide/aide.db.new`. Para que AIDE la use como referencia, hay que renombrarla a `aide.db`:

```bash
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

> **Nota sobre la compresión**: la configuración por defecto de Debian 13 incluye `gzip_dbout=yes`, lo que significa que la base de datos se genera **comprimida con gzip aunque el archivo no tenga la extensión `.gz`**. Esto es normal y AIDE la lee correctamente. Se puede verificar con `sudo file /var/lib/aide/aide.db`, que reportará `gzip compressed data`.

##### Verificación de la base de datos

Antes de ejecutar el primer check, confirmar que todo está en orden:

```bash
sudo ls -la /var/lib/aide/
```

Salida esperada, los permisos restrictivos son parte del hardening:

```
drwx------  2 _aide root      4096 Apr 26 23:12 .
drwxr-xr-x 28 root  root      4096 Apr 26 23:11 ..
-rw-------  1 _aide _aide 14111274 Apr 26 23:12 aide.db
```

El directorio y el archivo pertenecen al usuario de sistema `_aide` (sin shell de login) con permisos `700` y `600` respectivamente. Solo `_aide` y `root` pueden acceder.

##### Ejecución del primer check

```bash
sudo aide --config /etc/aide/aide.conf --check
```

El check tarda entre 2 y 10 minutos. La terminal parecerá colgada mientras AIDE hashea cada archivo monitoreado — no interrumpir el comando con Ctrl+C.

##### Salida esperada

Hay dos resultados posibles, ambos aceptables en la primera corrida:

**A) Sin diferencias (ideal):**

```
AIDE 0.19.1 found NO differences between database and filesystem. Looks okay!!
```

**B) Con diferencias menores (común en la primera corrida):**

```
Summary:
  Total number of entries:    XXXXX
  Added entries:              X
  Removed entries:            0
  Changed entries:            Y
```

Si aparecen diferencias, típicamente serán en:

- `/var/log/*` — logs que se escribieron entre `aideinit` y el `--check`.
- `/var/lib/aide/aide.db` — la propia base de datos (porque fue movida con `mv` después de crearla).
- `/var/cache/*`, `/var/lib/dpkg/*` — caché del sistema que cambia constantemente.
- `/etc/.updated` — archivo legítimo del sistema gestionado por `base-files`.

Estas diferencias son normales y esperables. Indican que el sistema está vivo (escribiendo logs, actualizando timestamps), no que haya un problema de seguridad.

##### Regenerar el baseline tras el primer check

Si el primer check reportó diferencias menores legítimas, regenerar la base como nuevo punto de referencia limpio:

```bash
# Regenerar la base de datos
sudo aide --config /etc/aide/aide.conf --init

# Reemplazar la base actual con la nueva
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Verificar que ahora no hay diferencias
sudo aide --config /etc/aide/aide.conf --check
```

A partir de este momento, cualquier diferencia futura entre la base y el sistema indicará un cambio real que merece atención.

##### Programación del check diario automático

El paquete `aide-common` provee `/usr/share/aide/bin/dailyaidecheck` y un cronjob en `/etc/cron.daily/`. Verifica que esté activado:

```bash
ls -la /etc/cron.daily/ | grep -i aide
sudo cat /etc/default/aide
```

Para activar el check diario automático, asegurarse de tener en `/etc/default/aide`:

```bash
sudo sed -i 's|^CRON_DAILY_RUN=.*|CRON_DAILY_RUN=yes|' /etc/default/aide
```

El cron diario ejecuta el check, registra el resultado en `/var/log/aide/aide.log` y envía un correo a root con el reporte si hay diferencias (configurable en `/etc/default/aide`).

##### Mantenimiento tras cambios legítimos

Cada vez que se actualicen paquetes (`apt upgrade`), se modifique la configuración o se instale software, AIDE reportará diferencias en el próximo check. Esto es esperado. Después de validar que los cambios son legítimos, regenera la base:

```bash
sudo aide --config /etc/aide/aide.conf --init
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

Guarda una copia de la base de datos en un medio externo o de solo lectura. Si un atacante compromete el sistema, podría modificar tanto los binarios como la base de datos local de AIDE para ocultar el rastro. Una copia offline es la única referencia confiable.

#### 12.5 Revisión periódica de logs y registros

Aunque las herramientas anteriores generan reportes automáticos, conviene tener el hábito de revisar manualmente:

```bash
# Últimos logins exitosos
last -a | head -20

# Logins exitosos
sudo wtmpdb last --fulltimes --service -n 20

# Intentos fallidos
sudo journalctl SYSLOG_FACILITY=10 -p warning --since '7 days ago' | tail -30

# Eventos SSH específicos de los últimos 7 días
sudo journalctl -u ssh --since '7 days ago' | grep -iE 'fail|invalid|preauth' | tail -20

# Warnings y errores del sistema del día actual
sudo journalctl --since today -p warning..emerg --no-pager
```

#### 12.6 `lynis` — auditoría periódica del baseline de seguridad

Las herramientas anteriores detectan cambios en archivos (AIDE), rootkits conocidos (rkhunter) e integridad de paquetes (debsums). `lynis` cubre un ángulo distinto: audita la *configuración* del sistema contra un conjunto de mejores prácticas de seguridad y genera un reporte con puntuación y sugerencias concretas. Es útil como auditoría periódica para detectar derivas en la configuración del servidor.

```bash
sudo apt install -y lynis
```

**Primera auditoría:**

```bash
sudo lynis audit system
```

La salida incluye una sección `Hardening index` con la puntuación (0–100), y secciones `Suggestions` y `Warnings` con las recomendaciones. En un servidor recién desplegado siguiendo este manual, la puntuación típica oscila entre 70 y 80 puntos; los puntos restantes corresponden a configuraciones avanzadas de compliance (AppArmor profiles personalizados, auditd con reglas exhaustivas, etc.) que van más allá del baseline generalista.

**Auditoría automática mensual:**

```bash
sudo tee /etc/cron.monthly/lynis-audit > /dev/null <<'EOF'
#!/bin/bash
/usr/sbin/lynis audit system --quiet --cronjob 2>&1 | mail -s "Lynis audit $(hostname) $(date +%Y-%m)" root
EOF
sudo chmod 700 /etc/cron.monthly/lynis-audit
sudo chown root:root /etc/cron.monthly/lynis-audit
```

La opción `--cronjob` suprime la salida interactiva; `--quiet` elimina el progreso visual. El reporte llega por correo a root (y por lo tanto al destino externo configurado en el Anexo C).

**Actualización de la base de datos de lynis:**

```bash
sudo lynis update info   # ver versión actual y disponible
sudo apt upgrade lynis   # actualizar desde los repositorios Debian
```

> lynis no modifica el sistema: solo audita y reporta. Es seguro ejecutarlo en cualquier momento sin riesgo de alterar la configuración.

#### 12.7 Checklist de cierre

- [ ] `auditd` activo con reglas mínimas en `/etc/audit/rules.d/99-hardening.rules`
- [ ] `rkhunter` con base inicializada y cron diario configurado
- [ ] `debsums` con chequeo diario activo
- [ ] (Opcional, recomendable) `aide` con base inicial creada
- [ ] `lynis` instalado y primera auditoría ejecutada
- [ ] Reportes diarios llegan al correo de root — requiere MTA configurado (ver Anexo C)

### 13. AppArmor

AppArmor es el sistema de Mandatory Access Control (MAC) que Debian usa por defecto desde Buster. Confina procesos: cada perfil declara qué archivos puede leer, escribir y ejecutar un programa, qué capabilities puede usar, qué red puede tocar. Si un atacante explota un servicio confinado, el perfil limita el daño.

A diferencia de SELinux (Red Hat), AppArmor identifica programas por ruta del binario, lo que lo hace más simple de entender y mantener.

En Debian 13 Trixie, AppArmor viene activo por defecto y con perfiles cargados para servicios comunes (Avahi, CUPS, ntpd, parts of systemd, etc.). El trabajo de hardening aquí es: confirmar que está activo, entender el estado, y considerar añadir perfiles adicionales.

#### 13.1 Verificación del estado

```bash
sudo aa-status
```

Salida típica:

```
apparmor module is loaded.
N profiles are loaded.
M profiles are in enforce mode.
K profiles are in complain mode.
P processes have profiles defined.
...
```

**Interpretación:**

- `enforce mode` — el perfil bloquea cualquier acción que no permita explícitamente. Es el modo "real".
- `complain mode` — el perfil registra violaciones en el log pero no bloquea. Útil para depurar perfiles nuevos.
- `unconfined` — el proceso ejecuta sin perfil aplicado (la mayoría de los procesos del sistema, salvo los servicios con perfil empaquetado).

Si AppArmor no está cargado:

```bash
sudo systemctl status apparmor --no-pager
sudo systemctl enable --now apparmor
```

Verificar también el parámetro de boot del kernel:

```bash
cat /sys/kernel/security/lsm
```

La salida debe incluir `apparmor` en la lista (típicamente junto con `lockdown`, `capability`, `landlock`, `yama` y otros). Si aparece, AppArmor está cargado y operativo.

#### 13.2 Perfiles en `enforce` vs `complain`

Inspeccionar perfiles individualmente:

```bash
sudo aa-status | grep -A100 'profiles are in'
```

**Mover un perfil a complain (para depurar problemas):**

```bash
sudo aa-complain /etc/apparmor.d/<perfil>
```

**Volver a enforce:**

```bash
sudo aa-enforce /etc/apparmor.d/<perfil>
```

**Recargar todos los perfiles:**

```bash
sudo systemctl reload apparmor
```

#### 13.3 Añadir perfiles adicionales

El paquete base de AppArmor en Debian incluye perfiles esenciales. El paquete `apparmor-profiles` añade perfiles para más programas (Firefox, Apache, MariaDB, PostgreSQL, etc.), y `apparmor-profiles-extra` aporta perfiles experimentales adicionales.

```bash
sudo apt install -y apparmor-profiles apparmor-profiles-extra apparmor-utils
```

Tras instalar:

```bash
sudo systemctl reload apparmor
sudo aa-status
```

El número de perfiles cargados aumenta. Algunos perfiles vienen en `complain` mode por defecto (para que no rompan nada al instalarlos sin probar). Tras un período de operación normal, revisar el log:

```bash
sudo journalctl -k | grep -i apparmor | grep DENIED | head -50
```

Si un perfil no genera denegaciones legítimas durante días de uso normal, se puede pasar a enforce con `aa-enforce`.

Crear perfiles propios para aplicaciones custom (un script, un binario propio, un servicio interno) requiere `aa-genprof` y comprensión del modelo de AppArmor. Es un nivel de hardening avanzado, pertinente cuando el servidor expone una aplicación específica de alto valor.

#### 13.4 Checklist de cierre

- [ ] `aa-status` muestra perfiles cargados en `enforce mode`
- [ ] AppArmor activo en boot (`cat /sys/kernel/security/lsm` lo confirma)
- [ ] `apparmor-profiles` instalado si se desean perfiles extra
- [ ] Log de denegaciones revisado tras operación normal

### Cierre de la Etapa III

Al terminar esta parte, el servidor está endurecido en sus capas críticas:

- **Acceso (SSH):** puerto cambiado, solo llave pública, usuarios restringidos, criptografía moderna.
- **Red (UFW):** denegación por defecto, solo SSH abierto, rate limiting integrado.
- **Fuerza bruta (fail2ban):** baneo automático por patrones de log, escalado para reincidentes.
- **Kernel (sysctl, modprobe):** parámetros anti-spoofing, ASLR, módulos innecesarios bloqueados.
- **Auditoría (auditd, rkhunter, debsums, AIDE):** registro y verificación de integridad.
- **MAC (AppArmor):** confinamiento de procesos críticos.

El servidor sigue siendo administrable y funcional. Las capas añadidas son transparentes para el operador legítimo y costosas para un atacante.

La Etapa IV (mantenimiento) se ocupa de mantener este estado en el tiempo: actualizaciones automáticas, sincronización de tiempo, rotación de logs, monitoreo, respaldos.

---

## Etapa IV — Mantenimiento y actualizaciones

Las partes I a III dejaron el servidor instalado, configurado y endurecido. Esta parte se ocupa de mantenerlo así en el tiempo. Un servidor endurecido en el día 1 deja de estarlo a los pocos meses si no se actualiza, si su reloj se desincroniza, si los logs desbordan el disco, si nadie repara en una caída de un servicio crítico, o si una falla de hardware sorprende sin un respaldo reciente.

El alcance es deliberadamente operativo: actualizaciones automáticas razonables, sincronización de tiempo confiable, manejo de logs, monitoreo mínimo, y una estrategia de respaldos que pueda restaurar el servidor desde cero. No se cubren herramientas avanzadas de observabilidad (Prometheus, Grafana, ELK) ni soluciones de respaldo empresariales — pertenecen a manuales específicos cuando el rol del servidor lo justifique.

Tres principios atraviesan esta parte:

1. La automatización debe ser auditable. Una tarea que corre sola y nadie revisa equivale a una tarea que no corre. Cada automatización de esta parte (apt-daily, chronyd, logrotate, borgbackup) deja huella verificable en logs o correos al administrador.

2. El reinicio automático es un compromiso, no un dogma. Reiniciar para aplicar parches del kernel es necesario; reiniciar a mitad de jornada es destructivo. Esta parte propone ventanas horarias y banderas explícitas para que el comportamiento sea predecible.

3. Probar la restauración antes de necesitarla. Un respaldo que nunca se restauró no es un respaldo, es una esperanza. La sección 17 incluye prueba periódica de restauración como parte del flujo, no como recomendación opcional.

### 14. Actualizaciones automáticas

Las actualizaciones manuales `apt update && apt full-upgrade` son apropiadas durante el despliegue inicial (sección 6) y para cambios mayores, pero no escalan en operación cotidiana. Un servidor administrado por una persona que olvida actualizar durante semanas es un servidor con CVEs sin parchear durante semanas.

`unattended-upgrades` es el mecanismo estándar de Debian para aplicar actualizaciones automáticas con criterio: solo paquetes de orígenes específicos, con posibilidad de listas negras, con notificación por correo, y con reinicio automático opcional dentro de ventanas horarias.

#### 14.1 Instalación

```bash
sudo apt install -y unattended-upgrades apt-listchanges
```

`apt-listchanges` muestra los changelogs de los paquetes que se van a actualizar. En modo automático, los envía por correo a root antes de aplicar las actualizaciones. Es útil para detectar cambios disruptivos antes de que ocurran.

#### 14.2 Configuración de orígenes

La configuración principal está en `/etc/apt/apt.conf.d/50unattended-upgrades`. El paquete viene con valores conservadores por defecto (solo `Debian-Security`). Esta sección lo amplía a security y updates, según la decisión documentada al inicio de la Etapa IV.

Editar el archivo:

```bash
sudoedit /etc/apt/apt.conf.d/50unattended-upgrades
```

En la sección `Unattended-Upgrade::Origins-Pattern` o `Unattended-Upgrade::Allowed-Origins`, dejar habilitadas las siguientes líneas (descomentándolas si vienen comentadas):

```
Unattended-Upgrade::Origins-Pattern {
    "origin=Debian,codename=${distro_codename},label=Debian";
    "origin=Debian,codename=${distro_codename},label=Debian-Security";
    "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
    "origin=Debian,codename=${distro_codename}-updates";
};
```

Significado:

- La primera línea cubre el repositorio principal `trixie` (correcciones de bugs entre point releases que llegan vía `main`).
- Las dos siguientes cubren `trixie-security` (parches de seguridad). Aparecen duplicadas porque la metadata del repositorio puede usar dos formatos según el momento, y conviene cubrir ambos.
- La cuarta cubre `trixie-updates` (actualizaciones que el equipo de release publica entre point releases).

Aplicar `updates` además de `security` mantiene el servidor más al día (correcciones funcionales, no solo de seguridad), a cambio de mayor superficie de cambio entre reinicios. Es una decisión defendible para servidores administrados activamente. Para servidores donde la estabilidad importa más que la frescura, dejar solo `security` es válido — basta con comentar las líneas que no aplican.

##### 14.2.1 Lista de paquetes excluidos

`unattended-upgrades` permite excluir paquetes específicos del flujo automático. Esto es útil para paquetes que requieren intervención manual al actualizarse (por ejemplo, bases de datos en producción, donde el reinicio del servicio debe coordinarse).

En el mismo archivo:

```
Unattended-Upgrade::Package-Blacklist {
//  "vim";
//  "libc6";
//  "libc6-dev";
//  "libc6-i686";
};
```

En el manual base no se excluye ningún paquete. Los manuales específicos (mail, base de datos, web) pueden añadir exclusiones según el rol.

#### 14.3 Notificación por correo

Las actualizaciones automáticas sin notificación equivalen a actualizaciones invisibles. El administrador debe enterarse de qué se actualizó, qué falló, y qué requiere reinicio.

En el mismo archivo `/etc/apt/apt.conf.d/50unattended-upgrades`, configurar:

```
Unattended-Upgrade::Mail "root";
Unattended-Upgrade::MailReport "on-change";
```

Significado:

- `Mail "root"` — envía el reporte al usuario `root` local. Si el servidor tiene un MTA configurado para reenviar el correo de root a una dirección externa, llega ahí.
- `MailReport "on-change"` — solo envía correo cuando hubo actualizaciones aplicadas o errores. Las opciones son `always` (siempre, ruidoso), `only-on-error` (solo errores, demasiado silencioso) y `on-change` (equilibrio razonable).

Para que el correo a root sea útil, hay que asegurar que se reenvía a una dirección real. Esto se cubre en la sección 17 (monitoreo) con la configuración de un MTA mínimo. Mientras tanto, el correo queda en `/var/mail/root` y se puede leer con `mail` desde el servidor.

#### 14.4 Reinicio automático controlado

Algunos paquetes (kernel, libc, OpenSSL, systemd) requieren reinicio para que la actualización surta efecto. Sin reinicio, el sistema sigue ejecutando código vulnerable aunque el paquete nuevo esté instalado.

`unattended-upgrades` puede reiniciar automáticamente, pero hay que controlarlo: un reinicio a las 14:00 de un martes en pleno horario laboral es destructivo, mientras que uno a las 04:00 del domingo suele ser tolerable.

En `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

Significado:

- `Automatic-Reboot "true"` — habilita el reinicio automático cuando es necesario.
- `Automatic-Reboot-WithUsers "false"` — no reinicia si hay usuarios con sesión activa. Es una salvaguarda razonable: si alguien está trabajando en el servidor, el reinicio se pospone al siguiente ciclo.
- `Automatic-Reboot-Time "04:00"` — hora a la que se ejecuta el reinicio (en la zona horaria del servidor configurada en la sección 5.3).

Si el servidor tiene servicios donde el reinicio nocturno no es aceptable (por ejemplo, mail server con tráfico 24/7 que necesita coordinación con balanceador), la opción correcta es dejar `Automatic-Reboot "false"` y hacer los reinicios manualmente tras revisar la cola y notificar a usuarios. Esto se documenta en el manual del rol específico.

Para detectar manualmente si hay reinicio pendiente:

```bash
ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required.pkgs
```

Si el archivo existe, hay reinicio pendiente. El segundo comando muestra qué paquetes lo requieren.

#### 14.5 Habilitación del flujo periódico

`unattended-upgrades` se ejecuta a través de timers de systemd, no de cron. Hay dos timers relevantes:

- `apt-daily.timer` — corre dos veces al día. Hace `apt update` y descarga paquetes nuevos al caché local sin instalarlos.
- `apt-daily-upgrade.timer` — corre una vez al día. Aplica las actualizaciones que `unattended-upgrades` autorice.

Para que el flujo automático funcione, hay que asegurar que ambos timers están habilitados y que el archivo `20auto-upgrades` instruye a usar `unattended-upgrades`:

```bash
sudoedit /etc/apt/apt.conf.d/20auto-upgrades
```

Contenido:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

Significado:

- `Update-Package-Lists "1"` — `apt update` diario.
- `Download-Upgradeable-Packages "1"` — descarga paquetes actualizables al caché.
- `AutocleanInterval "7"` — cada 7 días limpia paquetes obsoletos del caché.
- `Unattended-Upgrade "1"` — aplica las actualizaciones diariamente.

Una alternativa equivalente es ejecutar `sudo dpkg-reconfigure -plow unattended-upgrades` y responder "Sí" al diálogo, que crea este mismo archivo.

#### 14.6 Verificación de los timers

```bash
sudo systemctl status apt-daily.timer --no-pager
sudo systemctl status apt-daily-upgrade.timer --no-pager
```

Ambos deben aparecer como `active (waiting)` con próxima ejecución (`Trigger:`) listada.

Listado de todos los timers activos para contexto:

```bash
sudo systemctl list-timers --all
```

#### 14.7 Simulación antes de la primera ejecución real

Antes de esperar al primer ciclo automático, simular el comportamiento:

```bash
sudo unattended-upgrades --dry-run --debug 2>&1 | less
```

La salida muestra:

- Qué paquetes serían actualizados.
- Qué paquetes serían omitidos y por qué (origen no permitido, blacklist, etc.).
- Si habría reinicio pendiente.

Esta simulación es la mejor manera de detectar errores de configuración antes de que el flujo automático aplique cambios reales.

#### 14.8 Revisión de logs tras la primera ejecución

Tras la primera ejecución automática (que ocurrirá en menos de 24 horas):

```bash
sudo less /var/log/unattended-upgrades/unattended-upgrades.log
sudo less /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
```

El primero registra la lógica de decisión (qué se actualizó, qué se saltó). El segundo registra la salida cruda de `dpkg` durante las actualizaciones.

Si el correo a root está bien configurado, el reporte llega solo cuando hubo cambios o errores. La revisión manual del log es útil al principio para confirmar que el flujo funciona como se espera.

#### 14.9 Checklist de cierre

- [ ] `unattended-upgrades` y `apt-listchanges` instalados
- [ ] `/etc/apt/apt.conf.d/50unattended-upgrades` con orígenes security + updates
- [ ] Notificación por correo configurada (`Mail "root"`, `MailReport "on-change"`) — requiere MTA configurado (ver Anexo C)
- [ ] Reinicio automático configurado en ventana horaria
- [ ] `/etc/apt/apt.conf.d/20auto-upgrades` con los cuatro parámetros activos
- [ ] `apt-daily.timer` y `apt-daily-upgrade.timer` activos
- [ ] Simulación con `--dry-run --debug` revisada sin errores

### 15. Sincronización de tiempo

Un reloj desincronizado en un servidor causa problemas que parecen no tener relación con la hora: certificados que se rechazan, autenticaciones Kerberos que fallan, logs imposibles de correlacionar entre máquinas, replicación de bases de datos rota, fail2ban baneando IPs por eventos en el "futuro". La sincronización de tiempo no es opcional.

Debian 13 viene con `systemd-timesyncd` activo por defecto. Es un cliente NTP simple y suficiente para escritorios y servidores casuales. Para un servidor administrado seriamente, `chrony` es la opción recomendada: implementa NTP completo (no solo SNTP), maneja mejor los ajustes en redes inestables, soporta marcado de pasos cuando el reloj se desvía mucho, y expone métricas detalladas vía `chronyc`.

#### 15.1 Migración de `systemd-timesyncd` a `chrony`

Verificar el estado actual:

```bash
timedatectl
```

Salida típica con timesyncd activo:

```
               Local time: ...
           Universal time: ...
                 RTC time: ...
                Time zone: America/Lima (-05, -0500)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

Detener y deshabilitar `timesyncd`:

```bash
sudo systemctl disable --now systemd-timesyncd
```

Instalar `chrony`:

```bash
sudo apt install -y chrony
```

Es posible que `apt install...` muestre `dpkg-statoverride: warning: --update given but /var/log/chrony does not exist`. Es un warning cosmético: el paquete declara permisos para `/var/log/chrony/` pero no crea el directorio porque, por defecto, chrony envía sus logs a journald y no a archivos propios. El servicio funciona correctamente sin ese directorio. Para silenciar el warning de cara a futuras reinstalaciones, crearlo con los permisos correctos:

```bash
sudo install -d -o _chrony -g _chrony -m 0755 /var/log/chrony
```

El paquete habilita y arranca el servicio automáticamente. Verificar:

```bash
sudo systemctl status chrony --no-pager
```

Y confirmar el reemplazo:

```bash
timedatectl
```

Ahora `NTP service: active` se corresponde con chrony, no con timesyncd. `timedatectl` detecta automáticamente cuál cliente NTP gestiona la sincronización.

#### 15.2 Configuración de servidores NTP

El archivo principal es `/etc/chrony/chrony.conf`. Debian configura por defecto los pools de Debian (`pool.ntp.org` con sub-pools regionales). Para la mayoría de servidores esto es suficiente. Conviene revisar y ajustar.

Inspeccionar la configuración por defecto:

```bash
grep -E '^(pool|server)' /etc/chrony/chrony.conf
```

Salida típica en Debian 13:

```
pool 2.debian.pool.ntp.org iburst
```

Para un servidor ubicado en una región específica, conviene usar el pool por país (más preciso y con menor latencia que el pool continental) y el pool regional como respaldo. Por ejemplo, para servidores en Perú:

```bash
sudoedit /etc/chrony/chrony.conf
```

Reemplazar la línea `pool` por:

```
# Pool por país (menor latencia, más precisión; ajustar según el país del servidor)
# Lista de pools por país: https://www.pool.ntp.org/zone/@
pool 0.pe.pool.ntp.org iburst maxsources 3
pool 1.pe.pool.ntp.org iburst maxsources 2

# Pool regional de Sudamérica como respaldo
pool 2.south-america.pool.ntp.org iburst maxsources 2

# Pool global como último respaldo
pool 2.debian.pool.ntp.org iburst maxsources 1
```

Significado:

- `iburst` — al iniciar, envía 4-8 paquetes rápidos para sincronizar más rápido.
- `maxsources N` — limita el número de servidores que el cliente toma del pool.

Para servidores en otras regiones, sustituir el código de país según <https://www.pool.ntp.org/zone/@>, y el pool regional por el correspondiente: `north-america`, `europe`, `asia`, `oceania`.

> Si el servidor está en una red corporativa con servidor NTP propio, usar ese servidor en lugar de los pools públicos:
>
> ```
> server ntp.empresa.local iburst
> ```

**NTS (Network Time Security) — para entornos de alta exigencia:**

La configuración estándar con pools públicos envía los paquetes NTP sin autenticación, lo que en principio permite ataques de manipulación del reloj en la red local. NTS (RFC 8915) añade autenticación criptográfica sobre TLS al protocolo NTP. chrony lo soporta desde la versión 4.0 (disponible en Debian 12+).

Para activarlo, cambiar las directivas `pool`/`server` a directivas `pool`/`server` con la opción `nts`:

```
# Servidores con soporte NTS (verificar disponibilidad en el momento de la instalación)
server time.cloudflare.com iburst nts
server ntppool1.time.nl iburst nts

# Pool estándar como respaldo sin NTS
pool 2.debian.pool.ntp.org iburst maxsources 2
```

> NTS requiere que el servidor tenga la hora ya aproximadamente correcta para negociar el TLS. Si el reloj inicia muy desviado, `makestep` (ya configurado por defecto en Debian) lo corregirá en las primeras iteraciones sin NTS antes de que NTS entre en acción. Verificar el estado con `chronyc authdata`.

Reiniciar chrony tras los cambios:

```bash
sudo systemctl restart chrony
```

#### 15.3 Verificación

```bash
chronyc tracking
```

Salida típica de un servidor sincronizado correctamente:

```
Reference ID    : C0A80101 (ntp.example.com)
Stratum         : 3
Ref time (UTC)  : Mon Apr 27 09:00:00 2026
System time     : 0.000123456 seconds slow of NTP time
Last offset     : -0.000045678 seconds
RMS offset      : 0.000234567 seconds
Frequency       : 12.345 ppm slow
Residual freq   : +0.012 ppm
Skew            : 0.456 ppm
Root delay      : 0.012345678 seconds
Root dispersion : 0.001234567 seconds
Update interval : 64.2 seconds
Leap status     : Normal
```

Indicadores de buen estado:

- `Stratum` 2, 3 o 4. Stratum 0 es la fuente de tiempo (reloj atómico, GPS); cada salto añade 1. Un servidor doméstico típicamente queda en stratum 3.
- `System time` con offset de microsegundos a milisegundos. Si está en segundos, hay problema.
- `Leap status: Normal`. Otros valores (`Insert second`, `Delete second`) indican un leap second pendiente, lo que es normal cerca del 30 de junio o 31 de diciembre.

Listar las fuentes que chrony está usando:

```bash
chronyc sources -v
```

La columna `MS` (Mode/State) indica el estado: `^*` es la fuente seleccionada como sincronización primaria, `^+` son fuentes válidas adicionales, `^-` son fuentes excluidas, `^?` son fuentes no contactadas todavía.

#### 15.4 Comportamiento ante desviaciones grandes

Por defecto, chrony ajusta el reloj de forma gradual (slewing) para no causar saltos. Esto es lo correcto en la mayoría de los casos: un salto hacia atrás puede romper aplicaciones que asumen monotonicidad.

Sin embargo, si el reloj inicia con desviación muy grande (más de unos segundos, típico tras un apagado prolongado o en VMs que estuvieron suspendidas), el ajuste gradual puede tomar horas. Para permitir un paso (step) en arranque cuando la desviación es grande, la configuración por defecto en Debian incluye:

```
makestep 1.0 3
```

Significado: si en alguna de las primeras 3 mediciones la desviación supera 1 segundo, ajustar el reloj de un salto. Después de las primeras 3 mediciones, solo slewing.

Esto es buen comportamiento para un servidor. No se modifica.

#### 15.5 Verificación cruzada con `timedatectl`

```bash
timedatectl
```

Confirmar:

- `System clock synchronized: yes`.
- `NTP service: active`.
- Zona horaria correcta (configurada en sección 5.3).

#### 15.6 Checklist de cierre

- [ ] `systemd-timesyncd` deshabilitado
- [ ] `chrony` instalado y activo
- [ ] Pool por país (y regional como respaldo) configurado en `/etc/chrony/chrony.conf`
- [ ] `chronyc tracking` muestra stratum bajo y offset pequeño
- [ ] `chronyc sources -v` muestra al menos una fuente con `^*`
- [ ] `timedatectl` confirma sincronización y zona horaria
- [ ] (Opcional) NTS configurado y verificado con `chronyc authdata`

### 16. Logs y rotación

Los logs son la memoria operativa del servidor. Son la primera fuente de diagnóstico cuando algo falla, la única evidencia tras un incidente de seguridad, y la base de cualquier análisis de tendencia. Pero también son grandes consumidores de disco, y un descontrol en el manejo de logs puede dejar el servidor sin espacio en `/var/log` (con todas las consecuencias que eso conlleva).

Esta sección configura tres frentes:

- `journald` — el log estructurado de systemd. Maneja todo lo que registran los servicios systemd (la mayoría del sistema en Debian 13).
- `rsyslog` o `logrotate` — para logs tradicionales en archivos planos (algunas aplicaciones, especialmente legacy, siguen escribiendo a `/var/log/*.log` directamente).
- Reenvío opcional a un servidor central — solo se menciona, no se configura aquí.

#### 16.1 Configuración de `journald`

`journald` viene activo por defecto y guarda los logs en `/var/log/journal/` si ese directorio existe (persistente), o en `/run/log/journal/` si no existe (volátil, se pierde al reiniciar).

Verificar el modo actual:

```bash
sudo journalctl --disk-usage
ls -ld /var/log/journal/ 2>/dev/null && echo "Logs persistentes" || echo "Logs volátiles"
```

`--disk-usage` muestra el espacio total ocupado, sin distinguir ubicación. La verificación del directorio sí es concluyente:

- Si `/var/log/journal/` existe, los logs son persistentes y journald los escribe ahí.
- Si no existe, los logs están en `/run/log/journal/` (tmpfs) y se pierden al reiniciar.

Para un servidor, los logs deben ser persistentes. Si no lo son:

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

Tras esto, journald empieza a escribir en disco. Los logs anteriores que estaban en `/run/log/journal/` se pierden, pero no es problema, es la primera configuración del servidor.

##### 16.1.1 Límites de tamaño y retención

Los valores por defecto de journald son razonables, pero conviene fijarlos explícitamente para que no dependan de defaults futuros y para que el comportamiento sea predecible.

Editar `/etc/systemd/journald.conf`:

```bash
sudoedit /etc/systemd/journald.conf
```

Descomentar y ajustar:

```
[Journal]
Storage=persistent
Compress=yes
SystemMaxUse=1G
SystemKeepFree=512M
SystemMaxFileSize=128M
MaxRetentionSec=3month
ForwardToSyslog=no
```

Significado:

- `Storage=persistent` — guardar en disco aunque `/var/log/journal/` no exista (lo crea si hace falta).
- `Compress=yes` — comprimir logs viejos. Reduce el espacio en disco a costa de un poco de CPU.
- `SystemMaxUse=1G` — el journal no usa más de 1 GB en total. Este valor es adecuado para el LV `/var/log` mínimo de 2 GiB recomendado en la sección 2.2, dejando margen para los logs de aplicaciones. Si el LV `/var/log` es de 5 GiB o más, se puede subir a `2G` o `3G`.
- `SystemKeepFree=512M` — siempre dejar al menos 512 MB libres en el filesystem. Si el disco se llena, journald empieza a borrar logs viejos antes de seguir escribiendo.
- `SystemMaxFileSize=128M` — cada archivo de journal individual no supera 128 MB. Cuando se llena, se rota a uno nuevo.
- `MaxRetentionSec=3month` — borrar logs más viejos que 3 meses. Ajustar según los requerimientos de retención de la organización.
- `ForwardToSyslog=no` — no duplicar logs a `/var/log/syslog` vía rsyslog. Reduce I/O y espacio. Si se necesita compatibilidad con herramientas que esperan `/var/log/syslog`, dejar `yes`.

Aplicar:

```bash
sudo systemctl restart systemd-journald
```

##### 16.1.2 Comandos básicos de consulta

Esta sección no es referencia exhaustiva de `journalctl` (la documentación es excelente: `man journalctl`), pero sí los patrones más comunes en operación:

```bash
# Logs del arranque actual
sudo journalctl -b

# Logs del arranque anterior
sudo journalctl -b -1

# Logs de un servicio específico
sudo journalctl -u ssh -n 100

# Logs en tiempo real
sudo journalctl -f

# Logs desde una fecha
sudo journalctl --since "2026-05-20" --until "2026-05-21"

# Solo errores y peor
sudo journalctl -p err

# Verificar integridad del journal (importante para auditoría)
sudo journalctl --verify
```

#### 16.2 `logrotate` para logs de aplicaciones

Algunas aplicaciones (especialmente las que se ejecutan fuera de systemd o que escriben logs propios además del journal) usan archivos planos en `/var/log/`. Para ellas, `logrotate` es la herramienta estándar. Viene preinstalado en Debian.

Verificar configuración global:

```bash
cat /etc/logrotate.conf
```

Las configuraciones específicas de cada aplicación están en `/etc/logrotate.d/`. Cada paquete que instale logs propios suele dejar ahí su archivo.

Plantilla recomendada para una aplicación nueva (ejemplo, `/etc/logrotate.d/miapp`):

```
/var/log/miapp/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 0640 miapp adm
    sharedscripts
    postrotate
        systemctl reload miapp >/dev/null 2>&1 || true
    endscript
}
```

Significado:

- `weekly` — rotar semanalmente.
- `rotate 8` — guardar 8 archivos rotados (8 semanas de historia).
- `compress` — comprimir los archivos rotados (gzip por defecto).
- `delaycompress` — no comprimir el archivo más reciente (el de la semana pasada). Útil porque algunas aplicaciones siguen escribiendo en él durante un tiempo.
- `missingok` — no fallar si el archivo no existe (típico al instalar la app por primera vez).
- `notifempty` — no rotar archivos vacíos.
- `create 0640 miapp adm` — al rotar, crear un archivo nuevo con esos permisos y propietarios.
- `sharedscripts` — ejecutar los scripts pre/postrotate una sola vez aunque haya múltiples archivos.
- `postrotate` — recargar el servicio para que reabra los handles de log.

Para probar la configuración sin esperar a la próxima rotación:

```bash
sudo logrotate -d /etc/logrotate.d/miapp     # debug, no aplica nada
sudo logrotate -f /etc/logrotate.d/miapp     # fuerza la rotación ahora
```

`logrotate` se ejecuta diariamente vía systemd (`logrotate.timer`). Verificar:

```bash
sudo systemctl status logrotate.timer --no-pager
```

#### 16.3 Reenvío a un servidor central (opcional)

Para infraestructuras con múltiples servidores, conviene enviar los logs a un servidor central de log management. Las dos opciones más comunes son:

- `rsyslog` con reenvío UDP/TCP a un servidor remoto. Tradicional, ampliamente compatible.
- `journald` con `systemd-journal-upload` y `systemd-journal-remote`. Más moderno, integrado con systemd.

Esta configuración no se cubre en el manual base porque depende de la infraestructura disponible (servidor central, formato esperado, autenticación). Se documentará en un manual de infraestructura específico. Lo único que conviene dejar registrado en este manual base: cuando llegue el momento de reenviar logs, recordar abrir el puerto correspondiente en el firewall (UFW) y configurar autenticación TLS para que el tráfico no vaya en claro.

#### 16.4 Verificación de uso de disco

```bash
df -hT /var/log
sudo du -sh /var/log/* | sort -h
sudo journalctl --disk-usage
```

El primer comando muestra el espacio del filesystem `/var/log`. El segundo muestra qué directorios consumen más espacio. El tercero muestra cuánto usa el journal específicamente.

Si `/var/log` está cerca de llenarse, las opciones son:

- Reducir `SystemMaxUse` y `MaxRetentionSec` en journald.
- Reducir el número de archivos rotados en logrotate.
- Ampliar el LV `/var/log` con `lvresize` (si se configuró LVM en la sección 2.2).

#### 16.5 Checklist de cierre

- [ ] `/var/log/journal/` existe y journald escribe ahí (persistente)
- [ ] `/etc/systemd/journald.conf` con límites explícitos
- [ ] `journalctl --verify` sin errores
- [ ] `logrotate.timer` activo
- [ ] Configuraciones de logrotate revisadas para aplicaciones existentes
- [ ] Espacio en `/var/log` con margen razonable

### 17. Monitoreo básico

Un servidor sin monitoreo puede caer y nadie se entera hasta que un usuario reporta el incidente. El monitoreo no tiene que ser sofisticado para ser útil: un par de chequeos básicos ejecutados periódicamente y enviando correo cuando algo va mal cubre el 80% de los problemas operativos.

Esta sección cubre lo mínimo razonable: herramientas de consulta puntual, alertas básicas con scripts y cron/timers, y verificación del estado de servicios críticos. Soluciones completas (Prometheus + Grafana, Zabbix, Netdata) pertenecen a manuales específicos.

#### 17.1 Herramientas de consulta puntual

Conjunto mínimo que conviene tener instalado en cualquier servidor para diagnóstico ad-hoc:

```bash
sudo apt install -y htop iotop iftop sysstat lsof wtmpdb libpam-wtmpdb
```

Resumen de uso:

- `htop` — vista interactiva de procesos, CPU, memoria. Reemplazo moderno de `top`.
- `iotop` — qué procesos están haciendo I/O. Crítico cuando el sistema está lento sin razón aparente y `htop` no muestra carga de CPU.
- `iftop` — qué conexiones consumen más ancho de banda.
- `sysstat` — incluye `sar`, `iostat`, `mpstat`, `pidstat`. Para análisis histórico de rendimiento.
- `lsof` — qué procesos tienen abiertos qué archivos/sockets. Útil para encontrar qué está bloqueando un puerto, qué está reteniendo un filesystem para desmontar, etc.

Comandos rápidos de referencia:

```bash
# Top 10 procesos por uso de memoria
ps aux --sort=-%mem | head -11

# Top 10 procesos por uso de CPU
ps aux --sort=-%cpu | head -11

# Espacio por filesystem
df -hT

# Inodos por filesystem (a veces el problema no es espacio sino inodos)
df -i

# Tamaño de directorios bajo /
sudo du -sh /* 2>/dev/null | sort -h

# Conexiones de red activas con proceso
sudo ss -tunap

# Puertos en escucha
sudo ss -tlnp

# Carga del sistema
uptime
w

# Memoria
free -h
```

`sysstat` requiere un paso adicional para habilitar la recolección histórica:

```bash
sudo sed -i 's/^ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
sudo systemctl restart sysstat
```

A partir de ahí, `sar` muestra estadísticas históricas (CPU, memoria, disco, red) hasta varios días atrás.

#### 17.2 Alertas mínimas vía cron

Para alertas básicas sin instalar herramientas complejas, un script en `/etc/cron.hourly/` que verifica condiciones críticas y envía correo si algo está mal es suficiente.

```bash
# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-cron.hourly-server-check.txt" \
    -O /tmp/plantilla-etc-cron.hourly-server-check.txt

# Verificar integridad antes de instalar un script que se ejecutará como root cada hora
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-cron.hourly-server-check.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO instalar este script"; exit 1; }

sudo cp /tmp/plantilla-etc-cron.hourly-server-check.txt /etc/cron.hourly/server-check
```

Permisos:

```bash
sudo chmod +x /etc/cron.hourly/server-check
sudo chmod 700 /etc/cron.hourly/server-check
sudo chown root:root /etc/cron.hourly/server-check
```

Adaptar la lista de servicios críticos según el rol del servidor. En un servidor de mail, añadir `postfix` y `dovecot`. En un web server, `nginx` o `apache2`.

Para que el correo a root llegue a una dirección externa, el servidor necesita un MTA local configurado para reenvío. La opción más simple es `msmtp` o `bsd-mailx` con un relay SMTP autenticado. La configuración detallada del MTA pertenece al manual de mail server cuando aplique; en este manual base hay una referencia más detallada en el Anexo C.

#### 17.3 Estado de servicios críticos

Comando único para una visión general:

```bash
systemctl --failed
systemctl list-units --state=running --type=service | head -30
```

`systemctl --failed` debe estar vacío en condiciones normales. Cualquier servicio que aparezca ahí merece investigación inmediata.

Verificación específica de servicios introducidos en el manual hasta ahora:

```bash
for svc in ssh chrony fail2ban ufw apparmor unattended-upgrades; do
    printf "%-25s " "$svc"
    systemctl is-active "$svc" 2>/dev/null || echo "no activo"
done
```

#### 17.4 Checklist de cierre

- [ ] Herramientas de diagnóstico puntual instaladas
- [ ] `sysstat` habilitado para histórico de rendimiento
- [ ] Script de alertas básicas en `/etc/cron.hourly/`
- [ ] Correo a root configurado para llegar a humano (MTA o lectura periódica)
- [ ] `systemctl --failed` vacío
- [ ] Servicios críticos del rol identificados y monitoreados

### 18. Respaldos

Sin respaldos, un servidor es un punto único de falla. Y un respaldo que nunca se restauró es una hipótesis, no un respaldo. Esta sección establece una estrategia mínima razonable para un servidor base: qué respaldar, dónde guardarlo, cómo automatizarlo, y cómo probar la restauración.

El alcance es generalista. Servidores con datos pesados (bases de datos en producción, mail con miles de buzones, sistemas de archivos compartidos) requieren estrategias específicas que se documentan en sus manuales correspondientes.

#### 18.1 Estrategia 3-2-1

La regla 3-2-1 sigue siendo el estándar mínimo razonable:

- 3 copias de los datos: el original más dos respaldos.
- 2 medios distintos: por ejemplo, disco local y almacenamiento remoto.
- 1 copia fuera del sitio físico: protege contra incendio, robo, o fallo de la infraestructura completa.

Aplicado al servidor base:

- Original: el propio sistema de archivos del servidor.
- Copia local: respaldos en `/var/backups/borg-local/` o un disco secundario montado.
- Copia remota: repositorio Borg en un servidor de respaldos, almacenamiento en la nube (Backblaze B2, S3, Hetzner Storage Box) o un NAS en otra ubicación.

#### 18.2 Qué respaldar

Para un servidor base, el conjunto mínimo es:

- `/etc` — toda la configuración del sistema y de los servicios.
- `/root` — el home de root (scripts administrativos, notas).
- `/home` — homes de usuarios administrativos.
- `/var/log` (opcional) — útil para análisis forense, no para restaurar.
- `/var/spool/cron` — tareas cron del sistema.
- Lista de paquetes instalados — para reconstruir el sistema desde cero.
- Datos específicos del rol — bases de datos, buzones de correo, sitios web. Se añaden en los manuales de rol.

Lo que típicamente no se respalda en el servidor base:

- `/usr`, `/bin`, `/sbin`, `/lib` — vienen de los paquetes; reinstalando los paquetes se recuperan.
- `/var/cache`, `/var/tmp`, `/tmp` — datos transitorios.
- Bases de datos directamente desde el filesystem — se respaldan con su propia herramienta (`pg_dump`, `mariabackup`, `mongodump`) y luego ese dump se incluye en el respaldo.

##### 18.2.1 Lista de paquetes instalados

```bash
sudo apt-mark showmanual | sudo tee /root/paquetes-manual.txt > /dev/null
sudo dpkg --get-selections | sudo tee /root/paquetes-completo.txt > /dev/null
```

Estos archivos quedan en `/root/`, así que entran en el respaldo automáticamente. Para restaurar:

```bash
sudo dpkg --set-selections < paquetes-completo.txt
sudo apt-get dselect-upgrade
```

#### 18.3 BorgBackup

`borgbackup` (binario `borg`) es la herramienta recomendada para servidores Linux modernos. Combina deduplicación a nivel de bloque (los respaldos sucesivos solo guardan lo que cambió), compresión, cifrado de extremo a extremo, y verificación de integridad. Los repositorios pueden estar en disco local, NFS, SSH remoto, o servicios compatibles (BorgBase, rsync.net).

##### 18.3.1 Instalación

```bash
sudo apt install -y borgbackup
```

##### 18.3.2 Inicialización del repositorio local

```bash
sudo mkdir -p /var/backups/borg-local
sudo borg init --encryption=repokey-blake2 /var/backups/borg-local
```

`repokey-blake2` cifra el repositorio con una clave que se guarda dentro del propio repositorio, protegida por una passphrase. Es la opción más simple para un solo administrador. Para escenarios multi-administrador o donde la passphrase no debe estar accesible desde el cliente, existen alternativas (`keyfile-blake2`).

> Importante: borg pide passphrase durante el `init`. Esta passphrase es la única forma de descifrar el repositorio. Guardarla en un gestor de contraseñas independiente del servidor. Si se pierde, los respaldos son irrecuperables.

##### 18.3.3 Inicialización del repositorio remoto

Para el respaldo fuera del sitio, asumiendo un servidor de respaldos accesible por SSH:

```bash
sudo borg init --encryption=repokey-blake2 \
    backup@servidor-backup.example.com:/srv/borg/$(hostname)
```

Requisitos previos:

- Llave SSH del usuario `root` del servidor de respaldos copiada al servidor backup.
- Cuenta `backup` en el servidor backup con `borg` instalado y permisos de escritura sobre `/srv/borg/`.
- Conexión SSH funcional desde el servidor cliente al servidor backup en el puerto correspondiente.

Para reducir la superficie de ataque, conviene restringir la llave SSH a ejecutar solo `borg serve` en el servidor backup, mediante `command="borg serve --restrict-to-path /srv/borg/<hostname>"` en el `authorized_keys` del lado servidor. Esto evita que una llave robada pueda usarse para sesión interactiva u otros comandos.

##### 18.3.4 Script de respaldo

Crear `/usr/local/sbin/borg-backup.sh`:

```bash
sudoedit /usr/local/sbin/borg-backup.sh
```

Contenido:

```bash
#!/bin/bash
# Respaldo Borg del servidor.

set -eu

# Repositorios destino
REPO_LOCAL="/var/backups/borg-local"
REPO_REMOTO="backup@servidor-backup.example.com:/srv/borg/$(hostname)"

# Passphrase desde archivo con permisos 600 propiedad de root
export BORG_PASSCOMMAND="cat /root/.borg-passphrase"

# Etiqueta del archivo (snapshot)
ARCHIVE_NAME="$(hostname)-$(date +%Y%m%dT%H%M%S)"

# Lista de paquetes (se actualiza cada vez)
apt-mark showmanual > /root/paquetes-manual.txt
dpkg --get-selections > /root/paquetes-completo.txt

# Función de respaldo
do_backup() {
    local repo="$1"
    borg create \
        --verbose \
        --stats \
        --compression zstd,3 \
        --exclude-caches \
        --exclude '/var/cache/*' \
        --exclude '/var/tmp/*' \
        --exclude '/var/lib/apt/lists/*' \
        --exclude '/var/log/journal/*' \
        "${repo}::${ARCHIVE_NAME}" \
        /etc /root /home /var/spool/cron /var/log

    borg prune \
        --verbose \
        --list \
        --keep-daily=7 \
        --keep-weekly=4 \
        --keep-monthly=6 \
        "${repo}"

    borg compact "${repo}"
}

# Ejecutar para cada repositorio
do_backup "$REPO_LOCAL"
do_backup "$REPO_REMOTO"
```

Permisos y passphrase:

```bash
sudo chmod 700 /usr/local/sbin/borg-backup.sh

# Crear el archivo de passphrase sin que el valor aparezca en el historial del shell
# read -s no hace eco de lo que se escribe; printf no añade newline final
sudo bash -c 'read -rsp "Passphrase para Borg (no se mostrará): " PHRASE && printf "%s" "$PHRASE" > /root/.borg-passphrase'
sudo chmod 600 /root/.borg-passphrase
```

> **`/var/log/journal/*` excluido:** el journal de systemd ya tiene su propia retención configurada en la sección 16.1.1 y puede ser de cientos de MB. Incluirlo en el respaldo diario lo haría crecer rápidamente sin valor proporcional de recuperación (el journal es diagnóstico, no datos de negocio). Los demás subdirectorios de `/var/log` (logs de aplicaciones rotados) sí se incluyen porque pueden tener valor forense o de auditoría.

Política de retención:

- 7 respaldos diarios (la última semana).
- 4 respaldos semanales (el último mes).
- 6 respaldos mensuales (los últimos seis meses).

Total: en cualquier momento hay aproximadamente 17 archivos retenidos, cubriendo desde un día hasta seis meses atrás. Ajustar según necesidades de retención y capacidad del repositorio.

##### 18.3.5 Automatización con systemd timer

systemd timers son preferibles a cron para tareas que requieren coordinación con el resto del sistema (esperar a que la red esté disponible, no correr si el sistema está suspendido, etc.).

Unidad de servicio `/etc/systemd/system/borg-backup.service`:

```bash
sudoedit /etc/systemd/system/borg-backup.service
```

```
[Unit]
Description=Respaldo Borg del servidor
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/borg-backup.sh
Nice=19
IOSchedulingClass=idle
```

`Nice=19` y `IOSchedulingClass=idle` aseguran que el respaldo no compita con servicios de producción por CPU o I/O.

Timer `/etc/systemd/system/borg-backup.timer`:

```bash
sudoedit /etc/systemd/system/borg-backup.timer
```

```
[Unit]
Description=Respaldo Borg diario

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=30m
Persistent=true

[Install]
WantedBy=timers.target
```

`RandomizedDelaySec=30m` distribuye la ejecución entre 03:00 y 03:30 — útil si hay múltiples servidores respaldando al mismo destino, evita que todos lleguen a la vez.

`Persistent=true` ejecuta el respaldo al arrancar el sistema si el timer no se disparó porque el servidor estaba apagado en ese momento.

Activar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now borg-backup.timer
sudo systemctl list-timers borg-backup.timer
```

#### 18.4 Verificación del repositorio

Borg incluye herramientas para verificar la integridad de los respaldos sin tener que restaurarlos completos:

```bash
# Listar archivos en el repositorio
sudo borg list /var/backups/borg-local

# Información detallada de un archivo
sudo borg info /var/backups/borg-local::nombre-del-archivo

# Verificar integridad del repositorio
sudo borg check --verify-data /var/backups/borg-local
```

`borg check --verify-data` lee y verifica todos los datos. Es lento (puede tomar horas en repositorios grandes), pero detecta corrupción silenciosa en disco. Se recomienda ejecutarlo mensualmente.

#### 18.5 Prueba de restauración

Esto no es opcional. Un respaldo no probado puede fallar silenciosamente. Establecer una rutina mensual de prueba.

Restauración de un archivo concreto:

```bash
sudo mkdir -p /tmp/restore-test
cd /tmp/restore-test
sudo borg extract /var/backups/borg-local::nombre-del-archivo etc/ssh/sshd_config
ls etc/ssh/sshd_config
```

Restauración de un directorio completo:

```bash
sudo borg extract /var/backups/borg-local::nombre-del-archivo etc/
diff -r /etc /tmp/restore-test/etc | head -20
```

Las diferencias son normales (logs nuevos, archivos de runtime). Lo importante es que la extracción funcione y que los archivos críticos estén ahí.

Tras la prueba, limpiar:

```bash
sudo rm -rf /tmp/restore-test
```

Documentar el resultado de cada prueba en un registro operativo.

#### 18.6 Restauración del servidor desde cero

Procedimiento de alto nivel para reconstruir el servidor si se pierde completamente:

1. Instalar Debian 13 base siguiendo las partes I y II del manual, hasta tener un sistema arrancable con SSH.
2. Instalar borgbackup.
3. Recuperar la passphrase del repositorio desde el gestor de contraseñas.
4. Apuntar al repositorio remoto: `borg list backup@servidor-backup.example.com:/srv/borg/<hostname>`.
5. Restaurar `/etc`, `/root`, `/home`: `sudo borg extract <repo>::<archivo>`.
6. Reinstalar paquetes: `sudo dpkg --set-selections < /root/paquetes-completo.txt && sudo apt-get dselect-upgrade`.
7. Restaurar datos específicos del rol (bases de datos, sitios, etc.).
8. Verificar servicios y reanudar operaciones.

Este procedimiento debe estar documentado en un runbook accesible incluso si el servidor está caído (en otro sistema, en papel, en el gestor de contraseñas del equipo).

#### 18.7 Checklist de cierre

- [ ] borgbackup instalado
- [ ] Repositorio local inicializado y cifrado
- [ ] Repositorio remoto inicializado y accesible
- [ ] Passphrase guardada en gestor de contraseñas externo
- [ ] Script de respaldo en `/usr/local/sbin/`
- [ ] systemd service y timer activos
- [ ] Primera ejecución manual exitosa (`systemctl start borg-backup.service`)
- [ ] Prueba de restauración documentada
- [ ] Procedimiento de restauración total escrito y guardado fuera del servidor

### 19. Documentación operativa

Las secciones anteriores configuraron el servidor; esta deja por escrito qué se configuró, por qué, y cómo operarlo. Sin documentación, un servidor administrable se convierte en un servidor "que solo entiende quien lo desplegó" — y el día que esa persona no esté disponible (vacaciones, cambio de rol, accidente), la organización se queda sin capacidad real de operación.

El alcance de esta sección es deliberadamente modesto. No se documenta cada decisión de hardening (eso lo hace el propio manual que se está siguiendo), ni se reproducen los checklists individuales de cada sección (esos ya quedaron donde corresponden). Lo que se documenta aquí es lo que sobrevive más allá del manual: el inventario propio de este servidor concreto, su diagrama lógico, y un runbook básico de operaciones rutinarias.

Tres principios atraviesan esta sección:

1. La documentación vive fuera del servidor que documenta. Si el servidor cae, la documentación debe seguir accesible. El sitio adecuado es un repositorio de la organización (Git interno, Wiki corporativa, gestor de contraseñas con notas largas), no archivos en el propio servidor que se está documentando.

2. El manual base documenta lo genérico; los manuales de rol documentan lo específico. Un servidor de mail tiene un runbook de mail; un servidor web tiene el suyo; este manual base solo documenta lo que aplica a cualquier servidor Debian endurecido. Las acciones específicas del rol viven en sus manuales correspondientes.

3. La documentación se actualiza con el sistema. Si hoy se cambia una IP, mañana se añade un servicio o el mes siguiente se modifica una regla de firewall, esos cambios deben reflejarse en la documentación en el mismo momento. Documentación desactualizada es peor que no tener documentación, porque se confía en ella.

#### 19.1 Inventario del servidor

El inventario es la "ficha técnica" del servidor: qué es, qué tiene, dónde está. Sirve para que cualquier administrador autorizado pueda situarse rápidamente sin tener que descubrir todo desde cero.

El inventario mínimo para un servidor base incluye:

| Campo                        | Ejemplo                                                             | Origen                                     |
|------------------------------|---------------------------------------------------------------------|--------------------------------------------|
| Hostname                     | `villonaco`                                                         | `hostnamectl`                              |
| FQDN                         | `villonaco.lan`                                                     | `hostname -f`                              |
| Rol                          | "Servidor base — sin rol específico aún"                            | manual / decisión del operador             |
| Ubicación física / proveedor | "VM local en estación de trabajo del admin" / "VPS en Hetzner FSN1" | conocida por el operador                   |
| Sistema operativo            | "Debian 13.5 Trixie"                                                | `cat /etc/os-release`                      |
| IP de gestión                | `192.168.1.10/24`                                                   | `ip -brief address`                        |
| Puerto SSH                   | `17177`                                                             | `/etc/ssh/sshd_config.d/99-hardening.conf` |
| Usuario administrativo       | `usuario`                                                           | manual                                     |
| Fecha de despliegue          | `2026-05-27`                                                        | conocida por el operador                   |
| Versión del manual seguido   | `v0.2`                                                              | encabezado del manual                      |

Comandos útiles para extraer el inventario actualizado del servidor:

```bash
hostnamectl
cat /etc/os-release | grep -E '^(NAME|VERSION)='
ip -brief address
sudo grep -E '^(Port|AllowUsers)' /etc/ssh/sshd_config.d/99-hardening.conf
sudo apt-mark showmanual | wc -l    # número de paquetes instalados manualmente
uptime -s                            # fecha del último arranque
```

El inventario se mantiene en una nota o documento del repositorio operativo (Wiki, Git interno, gestor de contraseñas con notas largas), no en el propio servidor. La razón es práctica: si el servidor cae, el inventario debe seguir accesible.

#### 19.2 Diagrama lógico

Un diagrama lógico responde rápidamente a preguntas como "¿qué puertos están expuestos?", "¿desde dónde se accede?", "¿qué servicios corren?". En un servidor base sin rol específico, el diagrama es muy simple — apenas el servidor mismo, su interfaz de red, y los puertos abiertos. Cuando se añadan roles (web, mail, base de datos), el diagrama se complica y conviene mantenerlo actualizado.

Un diagrama básico de un servidor recién desplegado siguiendo este manual luce así (en notación de texto plano):

```
            Internet / LAN
                  │
                  ▼
        ┌─────────────────────┐
        │ UFW (deny incoming) │
        │ Solo: SSH/17177     │
        └─────────┬───────────┘
                  │
        ┌─────────▼───────────┐
        │  villonaco          │
        │  Debian 13 Trixie   │
        │  192.168.1.10       │
        ├─────────────────────┤
        │ Servicios activos:  │
        │  - sshd (17177)     │
        │  - chrony           │
        │  - fail2ban         │
        │  - apparmor         │
        │  - msmtp (saliente) │
        └─────────────────────┘
                  │
                  ▼ (correo saliente)
            smtp.gmail.com:587
```

Para diagramas más visuales, herramientas como [Excalidraw](https://excalidraw.com/), [draw.io](https://app.diagrams.net/) o [Mermaid](https://mermaid.live/) generan SVG/PNG mantenibles. La elección importa menos que mantener la coherencia: usar siempre la misma herramienta facilita la lectura y la edición.

Un buen diagrama responde a estas preguntas sin que haya que abrirlo dos veces:

- ¿Qué hostname y qué IP tiene el servidor?
- ¿Qué puertos están expuestos hacia el exterior y hacia qué redes?
- ¿Qué servicios corren en él?
- ¿Con qué otros sistemas se comunica (relay SMTP, servidor de respaldos, monitoreo central, etc.)?
- ¿Por dónde entra la administración (SSH, VPN)?

Lo que un diagrama no necesita en este nivel: detalles internos del kernel, listas exhaustivas de paquetes, configuraciones específicas. Eso vive en el manual o en el inventario de paquetes (sección 18.2.1).

#### 19.3 Runbook básico

Un runbook documenta cómo se hacen las operaciones cotidianas del servidor. Para un servidor base, el runbook es corto porque no hay rol que operar; los runbooks largos vienen con los manuales específicos. Aun así, hay tres operaciones que aplican a cualquier servidor y conviene tener documentadas:

##### 19.3.1 Reinicio controlado

Reiniciar el servidor no es operación rutinaria, pero ocurre. Los pasos para hacerlo de forma ordenada:

```bash
# 1. Verificar que no hay actualizaciones a medio aplicar
sudo apt list --upgradable
ls /var/run/reboot-required 2>/dev/null && echo "Reinicio pendiente" || echo "Sin reinicio pendiente"

# 2. Notificar a usuarios conectados (si hubiese)
who
sudo wall "Reinicio en 1 minuto" 2>/dev/null || true

# 3. Verificar servicios críticos antes
sudo systemctl --failed
sudo systemctl is-active ssh chrony fail2ban

# 4. Ejecutar el reinicio
sudo reboot

# 5. Tras reconectar, verificar que todo arrancó bien
sudo systemctl --failed
sudo journalctl -b -p err --no-pager | head -20
```

Si el servidor usa LUKS con unlock remoto (sección 2.4), recordar que tras el reinicio hay que hacer el unlock desde otra máquina antes de que SSH normal esté disponible.

##### 19.3.2 Restauración tras desastre

El procedimiento completo está en la sección 18.6, pero conviene tener el resumen accesible en el runbook para no buscarlo en el manual completo en mitad de un incidente:

1. Instalar Debian 13 base siguiendo Partes I y II del manual.
2. Instalar borgbackup y recuperar la passphrase del repositorio del gestor de contraseñas.
3. Listar los respaldos disponibles: `borg list <repositorio>`.
4. Restaurar `/etc`, `/root`, `/home`, `/var/spool/cron`: `borg extract <repo>::<archivo>`.
5. Reinstalar paquetes del archivo de selecciones:

   ```bash
   sudo cat /root/paquetes-completo.txt | sudo dpkg --set-selections
   sudo apt-get dselect-upgrade
   ```

6. Restaurar datos específicos del rol (si los hubiese).
7. Verificar servicios y reanudar operaciones.

El runbook debe identificar claramente dónde está la passphrase del repositorio Borg y dónde están los datos de acceso al servidor de respaldos remoto. Sin esos dos elementos, la restauración no es posible.

##### 19.3.3 Acceso de emergencia

Si SSH falla, fail2ban banea por error la IP del administrador, o un cambio en UFW corta el acceso, conviene tener documentado:

- Cómo acceder a la consola física o a la consola del hypervisor / VPS.
- Cómo reiniciar el servidor en modo single-user o rescue mode si fuera necesario.
- Las credenciales de root (donde sea que estén guardadas, típicamente en el gestor de contraseñas de la organización; en este manual la cuenta de root está bloqueada al final de la Etapa II, así que el acceso de emergencia es vía sudo desde el usuario administrativo, lo cual cambia el procedimiento).
- En caso de IP baneada por fail2ban: desde la consola física, ejecutar `sudo fail2ban-client unban <ip>` o `sudo fail2ban-client set sshd unbanip <ip>`.

#### 19.4 Mantenimiento de la documentación

Documentación desactualizada es peor que no tenerla. Hay que cuidar el ciclo de actualización:

- Cada cambio de configuración significativo (IP, puerto, servicio nuevo, política de firewall) actualiza el inventario y el diagrama en el mismo momento.
- Las pruebas periódicas de restauración (sección 18.5) son una buena oportunidad para confirmar que el runbook sigue siendo correcto: si los pasos documentados ya no funcionan, el runbook no refleja la realidad.
- El propio manual base que se está siguiendo cambia con el tiempo (versiones nuevas, correcciones, actualizaciones del propio Debian). Anotar en el inventario qué versión del manual se usó al desplegar este servidor permite saber, años más tarde, qué decisiones se tomaron y cuáles podrían necesitar revisión.

#### 19.5 Checklist de cierre

- [ ] Inventario del servidor escrito en el repositorio operativo de la organización
- [ ] Diagrama lógico básico creado y guardado fuera del servidor
- [ ] Runbook de reinicio, restauración y acceso de emergencia documentado
- [ ] Versión del manual base usada anotada en el inventario
- [ ] Procedimiento de actualización de la documentación acordado con el equipo

### Cierre de la Etapa IV

Al terminar esta parte, el servidor opera de forma sostenible y queda documentado:

- Actualizaciones aplicadas automáticamente con notificación y reinicio controlado.
- Reloj sincronizado por `chrony` con pool regional.
- Logs persistentes, acotados en tamaño y retenidos por tres meses.
- Monitoreo básico que alerta por correo cuando algo va mal.
- Respaldos diarios cifrados, locales y remotos, con prueba periódica de restauración.
- Inventario, diagrama lógico y runbook básico registrados fuera del servidor.

Con esta parte cerrada, el manual base llega a su fin. El servidor está desplegado, endurecido, mantenido y documentado en su capa genérica.

A partir de aquí, cada manual que extiende esta base añade sus propias capas: servicios específicos del rol (mail, web, base de datos, ISPConfig, Odoo), hardening complementario, runbooks operativos del rol, diagramas enriquecidos, y esquemas de respaldo extendidos. Lo que el manual base estableció — usuarios, SSH endurecido, firewall, fail2ban, sysctl, AppArmor, actualizaciones automáticas, sincronización de tiempo, logs, monitoreo básico, respaldos y MTA mínimo — sirve de cimiento para todos esos manuales y no se vuelve a documentar en cada uno: se hereda y se referencia.

---

## Anexos

### Anexo A — Índice consolidado de plantillas

Este manual usa plantillas alojadas en el repositorio público <https://github.com/noggalito/manuales/> que se descargan con `wget` y se sustituyen variables con `sed` durante la lectura. Esa estructura modular evita repetir bloques largos de configuración dentro del propio manual y facilita mantenerlas: si una plantilla cambia, basta con actualizarla en el repositorio sin tocar el documento.

Este anexo concentra en un solo lugar la lista completa de plantillas referenciadas, su ruta de destino en el sistema, la sección del manual donde se aplican, y las variables que cada una requiere. Sirve como mapa rápido cuando se necesita recordar qué plantilla corresponde a qué archivo, especialmente útil al revisar un servidor desplegado tiempo atrás.

#### A.1 Convenciones del sistema de plantillas

Las plantillas están en el repositorio `https://github.com/noggalito/manuales/`, bajo el directorio `assets/`. El nombre del archivo de plantilla refleja la ruta de destino con guiones en lugar de barras. Por ejemplo, `plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt` se despliega en `/etc/ssh/sshd_config.d/99-hardening.conf`.

Las variables dentro de las plantillas usan el prefijo `$` y se sustituyen con `sed` antes de copiar al destino:

```bash
# Patrón general
export MIVARIABLE=valor
wget "${ASSETS_BASE}/<plantilla>.txt" -O /tmp/<plantilla>.txt
echo "<SHA256-ESPERADO>  /tmp/<plantilla>.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }
sed -e "s|\$MIVARIABLE|${MIVARIABLE}|g" /tmp/<plantilla>.txt | sudo tee /<ruta-destino> > /dev/null
```

Todas las plantillas del repositorio se mantienen con saltos de línea LF (Unix) gracias al archivo `.gitattributes` en la raíz del repositorio. Esto evita que un script descargado falle por shebang con CRLF si el repositorio fuera editado desde un sistema Windows.

##### A.1.1 Modelo de seguridad: anclaje a commit SHA y verificación de integridad

Las URLs que usan `refs/heads/main` apuntan a la punta de la rama, que es mutable: si el repositorio fuera comprometido, las instalaciones futuras recibirían contenido diferente sin saberlo. Para mitigar esto, el manual usa un commit SHA fijo (`SHA_COMMIT`) como referencia inmutable.

**Configuración inicial (ejecutar una sola vez antes de la primera plantilla):**

```bash
# Obtener el SHA del commit que se desea anclar.
# Opción A — si se tiene git instalado localmente:
#   git ls-remote https://github.com/noggalito/manuales.git HEAD | awk '{print $1}'
# Opción B — consultar la página de commits en GitHub y copiar el SHA completo (40 caracteres)
#   https://github.com/noggalito/manuales/commits/main

export SHA_COMMIT="<REEMPLAZAR-POR-SHA-DE-40-CARACTERES>"
export ASSETS_BASE="https://raw.githubusercontent.com/noggalito/manuales/${SHA_COMMIT}/assets"

# Verificar que la variable quedó definida
echo "SHA_COMMIT = ${SHA_COMMIT}"
echo "ASSETS_BASE = ${ASSETS_BASE}"
```

> **Nota sobre los placeholders `<SHA256-ESPERADO>`:** la primera vez que se descarga una plantilla para un `SHA_COMMIT` dado, el hash aún no se conoce. El flujo es: descargar → calcular hash con `sha256sum /tmp/<plantilla>.txt` → anotar el resultado → usar ese valor en ejecuciones futuras para verificar coherencia. Para un despliegue único no es estrictamente necesario conocer el hash de antemano, pero sí es importante anclar el commit SHA para garantizar reproducibilidad.
>
> Si los valores de las variables contienen caracteres especiales (barras, pipes, ampersands), los comandos `sed` de sustitución pueden romperse. Los valores problemáticos para el delimitador `|` son los que contienen `|` literales; en la práctica los valores de hostname, usuario y puerto raramente los contienen, pero se debe verificar antes de ejecutar.

#### A.2 Inventario de plantillas

| Plantilla                                               | Sección | Ruta destino                               | Variables                                                                                       |
|---------------------------------------------------------|---------|--------------------------------------------|-------------------------------------------------------------------------------------------------|
| `plantilla-etc-hosts.txt`                               | 5.2     | `/etc/hosts`                               | `$MIHOST`                                                                                       |
| `plantilla-etc-ssh-sshd_config.d-99-hardening.conf.txt` | 8.1     | `/etc/ssh/sshd_config.d/99-hardening.conf` | `$MIPUERTO`, `$MIUSUARIO`                                                                       |
| `plantilla-etc-fail2ban-jail.local.txt`                 | 10.2    | `/etc/fail2ban/jail.local`                 | `$MIPUERTO`                                                                                     |
| `plantilla-etc-sysctl.d-99-hardening.conf.txt`          | 11.1    | `/etc/sysctl.d/99-hardening.conf`          | (sin variables)                                                                                 |
| `plantilla-etc-modprobe.d-99-hardening.conf.txt`        | 11.3    | `/etc/modprobe.d/99-hardening.conf`        | (sin variables)                                                                                 |
| `plantilla-etc-audit-rules.d-99-hardening.rules.txt`    | 12.1    | `/etc/audit/rules.d/99-hardening.rules`    | (sin variables)                                                                                 |
| `plantilla-etc-cron.hourly-server-check.txt`            | 17.2    | `/etc/cron.hourly/server-check`            | (sin variables)                                                                                 |
| `plantilla-etc-msmtprc.txt`                             | C.4     | `/etc/msmtprc`                             | `$MIRELAY`, `$MIPUERTO_SMTP` -> `$MIPUERTO` en plantilla, `$MIUSUARIO_SMTP`, `$MIFROM`          |
| `plantilla-etc-msmtp-aliases.txt`                       | C.6     | `/etc/msmtp/aliases`                       | `$MIDESTINO`                                                                                    |

#### A.3 Resumen de variables del manual

A lo largo del manual se usan variables de entorno para personalizar las plantillas. Las más comunes son:

| Variable          | Significado                                                       | Ejemplo                                       |
|-------------------|-------------------------------------------------------------------|-----------------------------------------------|
| `$MIHOST`         | Hostname corto del servidor                                       | `villonaco`                                   |
| `$MIUSUARIO`      | Usuario administrativo                                            | `usuario`                                     |
| `$MIPUERTO`       | Puerto SSH personalizado (también usado por fail2ban)             | `17177`                                       |
| `$MIRELAY`        | Hostname del relay SMTP saliente                                  | `smtp.gmail.com`                              |
| `$MIPUERTO_SMTP`  | Puerto SMTP del relay (Anexo C — distinto de `$MIPUERTO`)         | `587`                                         |
| `$MIUSUARIO_SMTP` | Usuario para autenticarse contra el relay                         | `cuenta@dominio.com`                          |
| `$MIFROM`         | Dirección "From" autorizada en el relay                           | `cuenta@dominio.com`                          |
| `$MIDESTINO`      | Dirección externa donde llega el correo del sistema               | `admin@dominio.com`                           |
| `$SHA_COMMIT`     | SHA del commit del repositorio de plantillas anclado              | `a1b2c3d4...` (40 chars)                      |
| `$ASSETS_BASE`    | URL base para descarga de plantillas (derivada de `SHA_COMMIT`)   | `https://raw.githubusercontent.com/.../assets`|

Conviene anotar los valores efectivos usados en este servidor concreto en el inventario del repositorio operativo (sección 19.1). Esto facilita reconstruir o auditar el servidor más adelante sin tener que adivinar qué se sustituyó en qué momento.

#### A.4 Modificación local de plantillas

Si se necesita un cambio que no admite ninguna de las variables existentes (por ejemplo, una directiva adicional de `sshd`), hay dos opciones razonables:

Hacer un fork del repositorio y mantener una versión propia de la plantilla. Apropiado cuando los cambios son sostenidos en el tiempo y aplican a varios servidores de la organización. La URL de descarga apuntaría al fork en lugar del repositorio original.

Editar el archivo destino tras desplegarlo. Apropiado cuando el cambio es específico de un servidor concreto. Anotar el cambio en el runbook (sección 19.3) para que sobreviva a redespliegues.

No editar las plantillas en el repositorio original sin coordinación. Hacerlo afecta a todos los manuales y servidores futuros que dependan del mismo archivo.

### Anexo B — Diagnóstico rápido post-incidente

Cuando algo va mal — un servicio que no responde, una alerta que llega, una caída inesperada — el tiempo invertido en diagnosticar correctamente determina si el incidente se resuelve en minutos o en horas. Este anexo concentra los comandos más útiles agrupados por síntoma. No reemplaza un análisis profundo, pero sí permite levantar rápidamente una imagen del estado del sistema y descartar las causas más comunes.

El anexo asume que el operador llegó al servidor por SSH (o consola) y tiene acceso `sudo`. Si SSH falla, el primer recurso es la consola física o del hypervisor / VPS, documentada en el runbook de la sección 19.3.3.

#### B.1 Imagen general del sistema

Cuando se recibe una alerta sin contexto claro o se sospecha que algo no anda bien, conviene empezar por una vista panorámica:

```bash
# Estado general en una pantalla
uptime
sudo systemctl --failed
free -h
df -hT
sudo ss -tlnp | head -20

# Kernel y arranque actual
uname -a
who -b                          # cuándo arrancó el sistema
last -a | head -10              # últimos logins

# Eventos recientes con prioridad warning o peor
sudo journalctl --since "1 hour ago" -p warning --no-pager | tail -50
```

Si algo aquí salta a la vista (un servicio caído, disco al 100%, carga muy alta), el diagnóstico continúa por esa pista.

#### B.2 Síntoma: el servidor está lento o no responde

```bash
# Carga y uso de CPU
uptime                          # load average
top -bn1 | head -20
ps aux --sort=-%cpu | head -10

# Memoria y swap
free -h
ps aux --sort=-%mem | head -10

# I/O — frecuente causa real cuando "todo está lento" sin CPU alta
sudo iotop -bn1 -o | head -20   # paquete iotop
vmstat 2 5                      # 5 muestras a 2 segundos
sudo dmesg | tail -50           # mensajes del kernel recientes (OOM, errores de disco)
```

`iotop` muestra qué proceso está haciendo I/O en tiempo real. Si el servidor está esperando al disco, es la herramienta clave. La columna `IO>` indica el porcentaje del tiempo que ese proceso está bloqueado por I/O.

#### B.3 Síntoma: un servicio no funciona

```bash
# Reemplazar <servicio> por el nombre real (ssh, chrony, fail2ban, etc.)
sudo systemctl status <servicio> --no-pager
sudo journalctl -u <servicio> -n 100 --no-pager
sudo journalctl -u <servicio> --since "30 minutes ago" -p err

# Verificar dependencias del servicio
sudo systemctl list-dependencies <servicio>

# Verificar configuración antes de recargar (cuando aplique)
sudo sshd -t                    # SSH
sudo nginx -t                   # nginx (si aplica)
sudo postfix check              # postfix (si aplica)
```

Si el servicio falló al arrancar, `journalctl -u <servicio>` muestra el motivo. Errores comunes: archivo de configuración con sintaxis inválida tras una edición reciente, puerto ocupado por otro proceso, falta de permisos en algún archivo del que depende.

#### B.4 Síntoma: el disco está lleno

```bash
# Uso por filesystem
df -hT

# Inodos (a veces el problema no es espacio sino inodos agotados)
df -i

# Qué directorios consumen el espacio
sudo du -sh /* 2>/dev/null | sort -h
sudo du -sh /var/* | sort -h
sudo du -sh /var/log/* | sort -h

# Archivos grandes en /var/log o /tmp
sudo find /var/log -type f -size +100M -exec ls -lh {} \;
sudo find /tmp -type f -size +100M -exec ls -lh {} \;

# Espacio que journald está usando
sudo journalctl --disk-usage
```

Causas frecuentes: logs sin rotar (logrotate falló), un proceso que escribe sin parar a un archivo (revisar `/var/log` y `/tmp`), un dump de base de datos olvidado, un respaldo a medio hacer. Si journald está consumiendo demasiado, ajustar `SystemMaxUse` en `/etc/systemd/journald.conf` (sección 16.1.1) y ejecutar `sudo journalctl --vacuum-time=7d` para forzar la limpieza inmediata.

#### B.5 Síntoma: no puedo conectar por SSH

Esto se diagnostica desde otra máquina si SSH no funciona, o desde la consola física / hypervisor si nada responde.

```bash
# Desde otra máquina, contra el servidor
ping <ip-del-servidor>
nc -vz <ip-del-servidor> <puerto-ssh>

# Resolución DNS si se usa hostname
dig +short <hostname>

# Prueba SSH con verbose
ssh -vvv -p <puerto> <usuario>@<ip>
```

Una vez en consola, en el servidor:

```bash
# ¿Está SSH escuchando?
sudo systemctl status ssh --no-pager
sudo ss -tlnp | grep -E ":(22|17177)"

# ¿UFW está bloqueando?
sudo ufw status numbered
sudo ufw status verbose

# ¿fail2ban baneó la IP?
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip <ip-del-administrador>

# Logs de SSH
sudo journalctl -u ssh -n 50
sudo journalctl -u ssh --since "30 minutes ago" | grep -iE 'fail|invalid|preauth'
```

Causas frecuentes: cambio reciente en `sshd_config.d/99-hardening.conf` que rompe la sintaxis (verificar siempre con `sudo sshd -t` antes de recargar), regla de UFW eliminada por error, IP del administrador baneada por fail2ban tras varios fallos legítimos, puerto cambiado y olvidado actualizar el `~/.ssh/config` del cliente.

#### B.6 Síntoma: red sin conectividad

```bash
# Interfaces e IPs
ip -brief address
ip route
cat /etc/resolv.conf

# Conectividad por capas
ping 127.0.0.1                  # capa loopback
ping <ip-del-gateway>           # capa enlace
ping 1.1.1.1                    # capa IP externa
ping deb.debian.org             # capa IP + DNS

# Estado del backend de red activo
sudo systemctl status systemd-networkd --no-pager   # si se usa networkd
sudo systemctl status NetworkManager --no-pager     # si se usa NetworkManager
sudo systemctl status dhcpcd --no-pager             # si se usa dhcpcd

# Conexiones activas
sudo ss -tunap | head -30
```

Si los pings a IPs externas funcionan pero los pings a hostnames no, el problema es DNS. Si el ping al gateway falla, es problema de capa enlace o de configuración de la interfaz.

#### B.7 Síntoma: alerta de fail2ban con muchas IPs baneadas

```bash
# Estado general
sudo fail2ban-client status

# Detalle del jail SSH
sudo fail2ban-client status sshd

# IPs actualmente baneadas
sudo fail2ban-client banned

# Últimos eventos en fail2ban
sudo journalctl -u fail2ban -n 100 --no-pager
sudo journalctl -u fail2ban --since "1 hour ago" | grep -i ban
```

Una cantidad alta de bans no es necesariamente alarmante: SSH expuesto al internet recibe escaneos automatizados constantemente. Pero si el patrón cambia (de pronto cientos de bans nuevos por hora) puede indicar un ataque dirigido. Revisar también los logs del kernel por si hay flood SYN: `sudo dmesg | grep -i 'syn flood'`.

#### B.8 Síntoma: el sistema no arrancó del todo

Aplica cuando el servidor reinició y algunos servicios no levantaron:

```bash
# Servicios que fallaron en este arranque
sudo systemctl --failed

# Detalle de cada uno
for svc in $(sudo systemctl --failed --no-legend | awk '{print $2}'); do
    echo "=== $svc ==="
    sudo systemctl status "$svc" --no-pager | head -20
    echo
done

# Errores del arranque actual
sudo journalctl -b -p err --no-pager

# Comparar con el arranque anterior (si fue distinto)
sudo journalctl -b -1 -p err --no-pager
```

Si AppArmor, AIDE o auditd reportan diferencias o denegaciones tras el arranque, revisarlos con sus comandos específicos:

```bash
sudo aa-status
sudo aide --config /etc/aide/aide.conf --check
sudo journalctl -k | grep -iE 'apparmor.*denied' | tail -20
```

#### B.9 Recolección de evidencia para análisis posterior

Cuando el incidente requiere análisis profundo (forense, soporte externo, post-mortem), conviene capturar el estado actual antes de tocar nada:

```bash
# Crear directorio temporal con marca de tiempo
INCIDENT_DIR="/var/tmp/incident-$(date +%Y%m%dT%H%M%S)"
sudo mkdir -p "$INCIDENT_DIR"
cd "$INCIDENT_DIR"

# Capturar estado del sistema
sudo systemctl --failed > systemctl-failed.txt
sudo journalctl -b > journal-current-boot.txt
sudo journalctl -b -1 > journal-previous-boot.txt 2>/dev/null
sudo dmesg > dmesg.txt
sudo ss -tunap > ss-connections.txt
sudo ps auxf > processes.txt
sudo lsof > lsof.txt 2>/dev/null
ip -brief address > ip-address.txt
ip route > ip-route.txt
sudo ufw status verbose > ufw-status.txt
sudo fail2ban-client status > fail2ban-status.txt 2>/dev/null
sudo aureport --summary > aureport-summary.txt 2>/dev/null
df -hT > df.txt
free -h > free.txt
uptime > uptime.txt

# Comprimir todo y mover a /var/backups o copiar fuera del servidor
cd /var/tmp
sudo tar czf "incident-$(date +%Y%m%dT%H%M%S).tar.gz" "$(basename $INCIDENT_DIR)"
ls -la *.tar.gz
```

Este paquete comprimido se copia fuera del servidor (a la estación del administrador, a un servidor de evidencia, a un sistema de tickets) antes de seguir investigando o aplicar correcciones. Una vez que se modifica el estado del sistema, la evidencia inicial se pierde.

Para casos donde se sospecha compromiso, la regla de oro es no apagar el servidor todavía (se pierden procesos en RAM, conexiones activas, archivos en `/tmp` que algunos rootkits limpian al reiniciar). Aislar de la red sí — desconectando o cambiando la regla de UFW por `sudo ufw default deny outgoing` temporalmente — pero sin reiniciar hasta haber capturado la evidencia volátil.

#### B.10 Cuándo escalar

Si tras estas verificaciones no se identifica la causa, o el incidente excede la capacidad del operador único, los pasos típicos son: levantar un ticket en el sistema de la organización, contactar al equipo de seguridad si se sospecha compromiso, y dejar el servidor en un estado controlado (servicios críticos detenidos si causan más daño que beneficio, red aislada si se sospecha compromiso) mientras llega ayuda.

El runbook (sección 19.3) debe documentar a quién escalar y por qué canal, antes de que el incidente ocurra. Un incidente real no es momento de descubrir que nadie tiene el teléfono del responsable de seguridad.

### Anexo C — Reenvío de correo del sistema con `msmtp`

Varias secciones del manual base envían correo al usuario `root` local: `unattended-upgrades` (sección 14.3), `rkhunter`, `debsums` y `aide` (sección 12), y el script de alertas básicas (sección 17.2). Sin un agente de transporte de correo (MTA) configurado, esos mensajes quedan en `/var/mail/root` y un humano debe leerlos manualmente desde el servidor — algo que en la práctica casi nunca ocurre.

Este anexo configura un canal mínimo de salida: el correo dirigido a `root` (o a cualquier otro usuario local del sistema) se reenvía a una dirección externa usando un relay SMTP autenticado. No instala un servidor de correo completo. El servidor solo envía; no recibe.

El cliente elegido es `msmtp`, que es solo un transmisor SMTP — sin demonio en escucha, sin cola persistente compleja, sin superficie de ataque adicional. Es la opción más liviana y razonable cuando lo único que se necesita es que el correo de notificaciones del sistema llegue a algún sitio.

#### C.1 Decisiones de diseño

Tres elecciones atraviesan este anexo y conviene entenderlas antes de aplicar los pasos:

Aliases gestionados por msmtp, no por `/etc/aliases`. El archivo `/etc/aliases` es leído por MTAs tradicionales (Postfix, Exim, sendmail-original) que aplican la traducción `root: dirección@externa` antes de enviar al relay. msmtp no es un MTA tradicional y no consulta ese archivo. Por eso el anexo configura los aliases en `/etc/msmtp/aliases`, que sí es leído por msmtp en cada envío. Un correo dirigido a `root` se traduce a la dirección externa antes de salir al relay, evitando errores como `recipient address root not accepted by the server` que rechazaría Gmail u Office 365.

Logs en journald, no en archivo dedicado. La plantilla deshabilita por defecto el `logfile` separado y registra todo vía syslog/journald. Esto evita un bug de permisos conocido (msmtp falla al abrir `/var/log/msmtp.log` cuando se invoca desde usuarios no-root, incluso con grupos correctos), aprovecha la persistencia de journald ya configurada en sección 16.1, y reduce mantenimiento (sin logrotate adicional, sin permisos especiales que vigilar).

Gmail con app password como ejemplo realista. El anexo asume un relay corporativo genérico, pero documenta Gmail como caso concreto en la sub-sección C.4.1 porque es la opción más accesible cuando no hay relay propio. Outlook/Hotmail ya no se incluye porque Microsoft deshabilitó la autenticación SMTP básica en 2024 para cuentas personales y la alternativa (XOAUTH2) es demasiado compleja para un servidor de notificaciones.

#### C.2 Requisitos previos

Para configurar este anexo se necesita:

- Un relay SMTP accesible desde el servidor por TCP. Puede ser un servidor corporativo, un servicio transaccional (Mailgun, Brevo, Amazon SES), Gmail con app password, o un relay propio. Lo único que importa es que acepte autenticación SMTP estándar.
- Credenciales del relay: usuario y contraseña.
- Una dirección "From" autorizada en el relay. Muchos relays rechazan correo cuyo `From` no esté en su lista permitida; Gmail reescribe el `From` automáticamente a la dirección autenticada.
- Una dirección de destino (a dónde llegan los correos del sistema).

#### C.3 Instalación

```bash
sudo apt install -y msmtp msmtp-mta mailutils
```

Significado de cada paquete:

- `msmtp` — el cliente SMTP propiamente dicho.
- `msmtp-mta` — instala un symlink de `/usr/sbin/sendmail` apuntando a `msmtp`. Esto hace que cualquier programa que use la interfaz tradicional `sendmail` (cron, mailx, scripts del sistema) envíe correo a través de `msmtp` sin saberlo.
- `mailutils` — proporciona el comando `mail` para enviar correo desde la línea de comandos. Hay alternativas (`bsd-mailx`, `s-nail`); `mailutils` es la más común en Debian.

#### C.4 Configuración global de msmtp

`msmtp` admite configuración global en `/etc/msmtprc` (para todo el sistema) y por usuario en `~/.msmtprc`. Para correo del sistema (que se envía como root desde cron, systemd, etc.), se usa la configuración global.

Definir variables del relay y descargar la plantilla:

```bash
export MIRELAY=smtp.gmail.com
export MIPUERTO_SMTP=587    # Puerto SMTP — usar este nombre para no colisionar con $MIPUERTO (puerto SSH)
export MIUSUARIO_SMTP=tu-cuenta@gmail.com
export MIFROM=tu-cuenta@gmail.com

# Descargar la plantilla (requiere SHA_COMMIT y ASSETS_BASE definidos según §A.1.1)
wget "${ASSETS_BASE}/plantilla-etc-msmtprc.txt" -O /tmp/plantilla-etc-msmtprc.txt

# Verificar integridad
echo "<SHA256-ESPERADO>  /tmp/plantilla-etc-msmtprc.txt" | sha256sum -c \
  || { echo "✗ ERROR: hash no coincide — NO aplicar este archivo"; exit 1; }

# Reemplazar variables y generar el archivo en destino
# Nota: la plantilla usa $MIPUERTO internamente; el sed lo sustituye por el valor de $MIPUERTO_SMTP
sed -e "s|\$MIRELAY|${MIRELAY}|g" \
    -e "s|\$MIPUERTO|${MIPUERTO_SMTP}|g" \
    -e "s|\$MIUSUARIO_SMTP|${MIUSUARIO_SMTP}|g" \
    -e "s|\$MIFROM|${MIFROM}|g" \
    /tmp/plantilla-etc-msmtprc.txt \
    | sudo tee /etc/msmtprc > /dev/null

# Permisos correctos (el archivo referencia credenciales)
sudo chmod 0640 /etc/msmtprc
sudo chown root:msmtp /etc/msmtprc
```

> Nota: el grupo `msmtp` lo crea el paquete durante la instalación. Verificar con `getent group msmtp`. Si no existe (instalación atípica), usar `root:root` con modo `0600`.

##### C.4.1 Caso particular: Gmail con app password

Si el relay es Gmail (cuenta personal o de Google Workspace), las variables a usar son:

```bash
export MIRELAY=smtp.gmail.com
export MIPUERTO_SMTP=587    # Puerto SMTP — distinto de $MIPUERTO (puerto SSH)
export MIUSUARIO_SMTP=tu-cuenta@gmail.com
export MIFROM=tu-cuenta@gmail.com
```

Pasos previos imprescindibles en la cuenta Google:

1. Activar la verificación en dos pasos en <https://myaccount.google.com/security>. Sin 2FA, las app passwords no están disponibles.
2. Generar una app password específica para el servidor en <https://myaccount.google.com/apppasswords>. Asignarle un nombre descriptivo (ej. `msmtp-servidorxyz`). Google muestra una clave de 16 caracteres una sola vez; copiarla inmediatamente.
3. Esa clave de 16 caracteres reemplaza la contraseña normal de Google en el archivo `/etc/msmtp/password` (paso C.5).

Notas adicionales sobre Gmail:

- Gmail reescribe el header `From` a la dirección autenticada, independientemente de lo que se configure en `from`. Esto es política antifishing de Google y no se puede cambiar.
- Cuentas Gmail gratuitas tienen un límite de 500 destinatarios por día. Para alertas de un servidor (volumen típico: pocos correos al día) es irrelevante.
- La app password sigue siendo válida indefinidamente hasta que se revoque manualmente o se cambie la contraseña principal de Google. Guardarla en un gestor de contraseñas con anotación clara, porque solo se muestra una vez.

#### C.5 Almacenamiento seguro de la contraseña

La contraseña del relay no debe estar en `/etc/msmtprc` directamente. La práctica correcta es guardarla en un archivo separado con permisos restrictivos:

```bash
sudo install -d -o root -g msmtp -m 0750 /etc/msmtp
printf '%s' 'la-contraseña-o-app-password' | sudo tee /etc/msmtp/password > /dev/null
sudo chmod 0640 /etc/msmtp/password
sudo chown root:msmtp /etc/msmtp/password
```

`printf '%s'` no añade newline final (a diferencia de `echo`), lo cual algunos relays estrictos requieren.

Verificar:

```bash
sudo ls -la /etc/msmtp/
```

Salida esperada:

```
drwxr-x--- 2 root msmtp 4096 ... .
drwxr-xr-x ... root root 4096 ... ..
-rw-r----- 1 root msmtp   XX ... password
```

Para infraestructuras donde las credenciales en archivos planos no son aceptables, msmtp soporta integración con `gpg` (cifrado simétrico) o con `secret-tool` (GNOME Keyring). Esas alternativas requieren GUI o sesión de usuario y no son prácticas en un servidor headless. La protección efectiva del archivo plano es: permisos `0640` con propietario `root:msmtp`, y mantener el servidor endurecido (sin acceso lateral de usuarios sin privilegios).

#### C.6 Aliases para destinatarios locales

Cuando un programa del sistema envía correo a `root`, `postmaster`, `daemon` o cualquier otro usuario local, msmtp recibe ese nombre como destinatario. Si lo envía tal cual al relay externo, el servidor SMTP rechaza el correo con error `5.1.3 not a valid RFC 5321 address` (porque `root` no tiene `@dominio`).

Para resolver esto, msmtp usa su propio mecanismo de aliases (separado de `/etc/aliases`, que solo es leído por MTAs tradicionales como Postfix). Se configura con un archivo de mapping y la directiva `aliases` en `/etc/msmtprc`.

Crear el archivo de aliases:

```bash
export MIDESTINO=admin@example.com

sudo tee /etc/msmtp/aliases > /dev/null <<EOF
# Aliases de msmtp: traducen destinatarios locales a direcciones externas.
# Formato: <local>: <externa>
# Una entrada por línea. Líneas que empiezan con # son comentarios.

root: ${MIDESTINO}
default: ${MIDESTINO}
EOF

sudo chmod 0644 /etc/msmtp/aliases
sudo chown root:root /etc/msmtp/aliases
```

La entrada `default:` es importante: cualquier destinatario sin `@` que no esté listado explícitamente se redirige a esa dirección. Sin ella, otros usuarios del sistema (`postmaster`, `daemon`, `mail`, cuentas de servicios) que reciban correo del sistema no se entregarán.

Verificar:

```bash
sudo cat /etc/msmtp/aliases
sudo grep '^aliases' /etc/msmtprc
```

La salida del segundo comando debe mostrar:

```
aliases         /etc/msmtp/aliases
```

Si esa línea no aparece, añadirla manualmente al bloque `defaults` de `/etc/msmtprc`:

```bash
sudo sed -i '/^timeout/a aliases         /etc/msmtp/aliases' /etc/msmtprc
```

> Nota sobre `/etc/aliases`: el archivo tradicional de Unix sigue siendo respetable y otras herramientas pueden consultarlo. Si se prefiere mantener coherencia, configurarlo también con la misma redirección (`root: dirección@externa`). Pero la fuente de verdad operativa para msmtp es `/etc/msmtp/aliases`. Si en algún momento cambia la dirección de destino, cambiarla en ambos archivos.

##### C.6.1 Override de AppArmor para leer el archivo de aliases

Debian 13 incluye un perfil AppArmor activo para `/usr/bin/msmtp` que limita qué archivos puede leer el binario, independientemente de los permisos POSIX. El perfil contempla `/etc/msmtprc`, `/etc/mailname`, `/etc/aliases` y archivos en el home del usuario, pero no contempla `/etc/msmtp/aliases`. La denegación de AppArmor puede no aparecer en `journalctl -k` si el perfil no fuerza auditoría de ese tipo de eventos, lo que dificulta diagnosticar la causa real.

La solución es agregar un override local sin desactivar el perfil, usando el mecanismo `local/` que el propio perfil ya incluye para extensiones del operador:

```bash
sudo tee /etc/apparmor.d/local/usr.bin.msmtp > /dev/null <<'EOF'
# Permitir lectura del archivo de aliases gestionado en /etc/msmtp/
/etc/msmtp/ r,
/etc/msmtp/aliases r,
EOF

sudo apparmor_parser -r /etc/apparmor.d/usr.bin.msmtp
```

Verificar que el perfil sigue cargado en modo enforce y que el override está activo:

```bash
sudo aa-status | grep -A1 msmtp
sudo cat /etc/apparmor.d/local/usr.bin.msmtp
```

Si más adelante se decide colocar el archivo de aliases en otra ruta (por ejemplo `/etc/mail/aliases.msmtp` para alinearse con otras herramientas), reemplazar las dos líneas del override en consecuencia y recargar el perfil con `apparmor_parser -r`.

> Diagnóstico cuando se sospecha de AppArmor: la pista decisiva es `sudo aa-status | grep msmtp`. Si aparece `msmtp` listado bajo "profiles are in enforce mode", el perfil está activo. La ausencia de mensajes `DENIED` en `journalctl -k` no descarta el bloqueo; algunos perfiles tienen `audit=off` para ciertas operaciones y deniegan silenciosamente.

#### C.7 Configuración de mailutils

`mailutils` por defecto usa su propia configuración mínima. Para que use `msmtp` para enviar y respete configuración estándar:

```bash
sudo tee /etc/mail.rc > /dev/null <<'EOF'
set sendmail="/usr/sbin/sendmail"
set sendwait
EOF
```

Significado:

- `set sendmail` indica el binario a usar; en Debian con `msmtp-mta` instalado, `/usr/sbin/sendmail` es un symlink a msmtp.
- `set sendwait` hace que `mail` espere a que el envío termine y reporte errores en pantalla, en lugar de encolar silenciosamente. Útil para que las pruebas y los scripts detecten fallos.

#### C.8 Acceso del usuario administrativo al grupo `msmtp`

`/etc/msmtprc` tiene permisos `0640 root:msmtp`. Para que el usuario administrativo pueda enviar correo manualmente sin sudo (útil en pruebas y diagnóstico), añadirlo al grupo:

```bash
sudo usermod -aG msmtp $MIUSUARIO
```

Cerrar y reabrir sesión SSH para que el grupo tome efecto. Verificar:

```bash
exit
# Reconectar SSH
groups | grep msmtp
```

`groups` debe incluir `msmtp` en la lista. Alternativamente:

```bash
id -G | tr ' ' '\n' | grep -q '^105$' && echo "ok" || echo "falta reconectar sesión"
```

> Si no se quiere expandir el grupo al usuario administrativo, todas las pruebas de envío manual deben hacerse con `sudo`. Los servicios del sistema (cron, unattended-upgrades, scripts en `/etc/cron.hourly/`) ya corren como root, así que no necesitan esta concesión. Es decisión del operador según la política de la organización.

#### C.9 Prueba de envío directo

Probar primero con un comando directo, antes de depender de la traducción de aliases:

```bash
echo "Mensaje de prueba directo desde $(hostname) a $(date)." | mail -s "Prueba de msmtp desde $(hostname)" admin@example.com
```

Reemplazar `admin@example.com` por la dirección real de destino. Tras unos segundos, el mensaje debe llegar a esa dirección.

Si el envío falla, las dos primeras fuentes a revisar son:

```bash
sudo journalctl -t msmtp -n 20
sudo journalctl -u systemd-journald --since "5 minutes ago"
```

Errores comunes y su causa:

| Mensaje en el log                                          | Causa probable                                          | Solución                                                                              |
|------------------------------------------------------------|---------------------------------------------------------|---------------------------------------------------------------------------------------|
| `authentication failed (method LOGIN)`                     | Usuario o contraseña incorrectos                        | Verificar `/etc/msmtp/password` y el `user`. En Gmail usar app password, no la real.  |
| `basic authentication is disabled`                         | El relay rechaza auth básica (típico Microsoft 365)     | Cambiar de relay o implementar XOAUTH2. Hotmail/Outlook personales requieren OAuth2.  |
| `connection refused`                                       | Puerto o host incorrecto, o firewall bloqueando         | Verificar `host` y `port` en `/etc/msmtprc`, probar `nc -vz <host> <puerto>`          |
| `sender address not allowed`                               | El `from` no está autorizado en el relay                | Pedir al admin del relay que añada la dirección, o usar una autorizada                |
| `TLS handshake failed`                                     | Certificado del relay no confiable                      | Verificar `tls_trust_file` (típicamente `/etc/ssl/certs/ca-certificates.crt`)         |
| `recipient address ... not a valid RFC 5321`               | Destinatario sin `@dominio` y aliases mal configurados  | Verificar `/etc/msmtp/aliases` y la directiva `aliases` en `/etc/msmtprc`             |
| `cannot log to /var/log/msmtp.log`                         | Logfile dedicado activado pero sin permisos             | Comentar la línea `logfile` en `/etc/msmtprc` (la plantilla v1.1 ya viene así)        |
| `Permiso denegado` sobre `/etc/msmtp/aliases` (POSIX bien) | Perfil AppArmor bloquea la lectura                      | Aplicar override de C.6.1                                                             |

#### C.10 Prueba de redirección de aliases

Una vez verificado el envío directo, probar que la traducción de `root` a la dirección externa funciona:

```bash
echo "Prueba de redirección de root a $(date)" | mail -s "Test redirección root" root
```

Verificar en journald que se haya enviado a la dirección correcta:

```bash
sudo journalctl -t msmtp -n 3
```

La línea correspondiente debe contener `recipients=admin@example.com` (la dirección externa), no `recipients=root`. Si aparece `recipients=root` y el correo fue rechazado, la directiva `aliases` en `/etc/msmtprc` o el archivo `/etc/msmtp/aliases` no están bien configurados; volver a C.6.

#### C.11 Prueba con el flujo completo del sistema

Para verificar que el script de alertas básicas (sección 17.2) llega al correo externo, forzar una alerta cambiando temporalmente un umbral:

```bash
df -h /
```

Si el `/` está al 4%, bajar el umbral a `1` para forzar alerta:

```bash
sudo sed -i 's/-ge 85/-ge 1/' /etc/cron.hourly/server-check
sudo /etc/cron.hourly/server-check
```

El correo con la alerta debe llegar a la dirección externa en pocos segundos. Restaurar el umbral:

```bash
sudo sed -i 's/-ge 1/-ge 85/' /etc/cron.hourly/server-check
sudo /etc/cron.hourly/server-check
```

El segundo `server-check` debe terminar sin output y exit code `0` (ninguna alerta activa).

#### C.12 Consideraciones de seguridad

Algunas observaciones útiles para mantener este canal seguro a lo largo del tiempo:

La contraseña (o app password) del relay en `/etc/msmtp/password` es la pieza más sensible. Cualquier compromiso de root expone esta credencial. Mitigar usando una credencial dedicada al servidor (no la cuenta personal del administrador) y rotándola cuando se reinstala el servidor o se sospecha exposición.

Si el relay corporativo soporta autenticación por OAuth2 o por certificado de cliente, son alternativas más seguras que usuario/contraseña. msmtp soporta XOAUTH2 con un script auxiliar; la configuración detallada queda fuera del alcance de este anexo y depende del proveedor.

El correo enviado puede contener información sensible (nombres de paquetes con CVEs sin parchear, listados de archivos modificados, IPs baneadas por fail2ban). Confirmar que el destinatario es una dirección controlada y que el relay no la registra en logs accesibles a terceros.

El firewall (UFW) no necesita reglas adicionales para correo saliente porque la política por defecto de la sección 9 es `allow outgoing`. Si en algún momento se restringe el tráfico saliente, recordar abrir el puerto del relay (típicamente 587 o 465) hacia el host del relay.

Las app passwords de Gmail tienen el alcance completo de la cuenta (no se pueden restringir a SMTP únicamente). Si la cuenta tiene acceso a documentos sensibles en Drive, Calendar, etc., considerar crear una cuenta de Gmail dedicada solo para envío de notificaciones del servidor, sin acceso a otros datos.

#### C.13 Logs

Con la configuración recomendada (logfile dedicado deshabilitado, syslog activo), todos los logs de msmtp van a journald y son consultables con:

```bash
# Últimos 50 envíos
sudo journalctl -t msmtp -n 50

# Solo errores
sudo journalctl -t msmtp -p err

# Envíos de una ventana de tiempo
sudo journalctl -t msmtp --since "1 hour ago"

# Verificación de integridad del journal
sudo journalctl --verify
```

journald aplica automáticamente la rotación y retención configurada en sección 16.1 (`MaxRetentionSec=3month`, `SystemMaxUse=1G` para el LV mínimo). No se requiere logrotate adicional.

Si por algún motivo se prefiere un logfile dedicado para msmtp (no es la recomendación del manual base, pero puede ser útil en debugging puntual o en escenarios con auditoría específica), descomentar la línea correspondiente en `/etc/msmtprc` y crear el archivo con permisos compatibles:

```bash
sudoedit /etc/msmtprc
# descomentar la línea: logfile         /var/log/msmtp.log

sudo install -o root -g msmtp -m 0660 /dev/null /var/log/msmtp.log
```

Y añadir una entrada en logrotate:

```bash
sudo tee /etc/logrotate.d/msmtp > /dev/null <<'EOF'
/var/log/msmtp.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    create 0660 root msmtp
}
EOF
```

Aunque los permisos de `/var/log/msmtp.log` sean correctos, msmtp puede mostrar el warning `cannot log to /var/log/msmtp.log: cannot open: Permission denied` al ser invocado vía el wrapper `/usr/sbin/sendmail` desde usuarios no-root, incluso con el grupo `msmtp` cargado. El envío funciona correctamente (el correo se entrega), pero el log local no se escribe. Si esto ocurre, la solución es volver a deshabilitar el `logfile` y usar solo journald.

#### C.14 Checklist de cierre

- [ ] `msmtp`, `msmtp-mta` y `mailutils` instalados
- [ ] `/etc/msmtprc` configurado con permisos `0640 root:msmtp` y la directiva `aliases` activa
- [ ] `/etc/msmtp/password` con permisos `0640 root:msmtp`, sin newline final
- [ ] `/etc/msmtp/aliases` con redirección de `root` y `default` a la dirección externa
- [ ] `/etc/mail.rc` configura `mailutils` para usar `sendmail`
- [ ] Usuario administrativo añadido al grupo `msmtp` (o políticas de envío con sudo definidas)
- [ ] Envío directo verificado (`mail` a una dirección externa funciona)
- [ ] Redirección de `root` verificada (`mail root` llega a la dirección externa, journald confirma `recipients=...`)
- [ ] Alerta del script `server-check` verificada llegando al correo externo
- [ ] Logs revisables vía `sudo journalctl -t msmtp`
- [ ] Credencial del relay registrada en gestor de contraseñas externo

---

[Despliegue de Debian 13 Trixie server](https://github.com/noggalito/manuales/blob/main/sistemas-operativos/debian/13-server.md) © 2026 by [Calú](https://github.com/calu777) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">
