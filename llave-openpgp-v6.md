# Generación y gestión de llaves OpenPGP v6 (RFC 9580) con Sequoia

Manual generalista para administrar una llave OpenPGP en perfil RFC 9580 (OpenPGP v6) usando Sequoia-PGP (`sq`) sobre tres distribuciones GNU/Linux concretas: Debian 13 "trixie", Fedora 40+ y Nix/NixOS. El modelo operativo asumido es llave primaria custodiada offline en Tails y subkeys de uso diario en el equipo de trabajo. Cubre instalación, generación airgapped, gestión del ciclo de vida, backup cifrado, publicación, e integración con consumidores GnuPG vía la familia chameleon (`gpg-sq` en Debian, `sequoia-chameleon-gnupg` en Fedora y Nix).

Este manual no reemplaza la documentación oficial de Sequoia (sq user guide, Sequoia Book). Lo que aporta es un procedimiento secuencial y opinionado para un escenario concreto: distribución GNU/Linux moderna con `sq 1.3.1`, perfil RFC 9580 desde el día uno, llave primaria offline en Tails y subkeys en producción. Las decisiones técnicas (v6 sobre v4, capacidades separadas por subkey, expiración corta de subkeys con primaria larga, subkey única de autenticación SSH) están justificadas en la sección 2.

El manual cubre explícitamente Debian 13 (y derivadas APT-compatibles tipo Ubuntu 24.04+), Fedora 40+ (y derivadas DNF-compatibles tipo RHEL 9 con EPEL), y Nix/NixOS. Otras distribuciones GNU/Linux (Arch, openSUSE, Gentoo, etc.) suelen empaquetar Sequoia bajo nombres similares; quien las use puede tomar la sección de Fedora como plantilla. macOS, FreeBSD, Windows e instalación desde fuente quedan fuera del alcance.

<p style="text-align: center"><img src="./assets/OpenPGP-v6.png" style="width: 50%;" alt="OpenPGP v6" /></p>

**Autor:** [Calú](https://github.com/calu777)
**Fecha de inicio:** 2026-05-20
**Última actualización:** 2026-05-20
**Versión:** 0.0.1
**Versión de referencia de sq:** 1.3.1 (estándar en Debian 13 trixie, Fedora 40+, Nix/NixOS)
**Variables principales:** `${USER_REAL}`, `${USER_EMAIL}`, `${USER_ID}`, `${KEY_FP}`, `${EXPIRATION}`, `${AIRGAP_MEDIA}`, `${BACKUP_DIR}`, `${WORK_HOST}`, `${OFFLINE_HOST}` (lista completa en Anexo A.1).

## a. Historial de cambios

- [0.0.1] – 2026-05-20
  - Estructura inicial

## b. To-do

- Reevaluar la sección 16 cuando Debian publique GnuPG con soporte v6 completo (probablemente con Debian 14 o desde backports); en ese momento, gpg-sq debería poder firmar v6 nativamente y el wrapper `git-sq-sign` pasará a ser opcional
- Documentar el estado de soporte v6 en GitHub, GitLab y Codeberg al momento de publicar la pública del titular
- Actualmente sq no incorpora un equivalente a `--quick-set-primary-uid` (sección 9.5). Optar por re-emitir la self-signature manualmente con la API de Sequoia (no la CLI). Implica un programa Rust que parsea el cert, modifica el flag `Primary User ID` de la self-signature del UserID elegido, firma de nuevo.
- Validar el procedimiento completo en Fedora 40+ y Nix/NixOS (hasta ahora solo validado parcialmente en Debian 13)
- Verificar la sintaxis exacta de `sq key subkey export-ssh` en sq 1.3.1 instalado (incertidumbre reconocida inline en §17.2)

## c. Requerimientos previos

- Tails 7.7.3 (2026-05-12) en `${OFFLINE_HOST}` airgapped, y una de las distros soportadas (Debian 13 trixie, Fedora 40+, Nix/NixOS) en `${WORK_HOST}` de uso diario
- Medio extraíble cifrado dedicado a la llave primaria (`${AIRGAP_MEDIA}`, por ejemplo USB con LUKS)
- Usuario administrativo con privilegios de root (`sudo` en Debian/Fedora, `sudo` o configuración equivalente en NixOS) en ambas instancias (Tails y la distro de `${WORK_HOST}`)
- Espacio cifrado independiente para el backup (distinto del medio que carga la llave primaria)
- Conocimiento básico de OpenPGP (qué es una llave, una subkey, un UserID, un certificado de revocación)

---

## Índice

- [a. Historial de cambios](#a-historial-de-cambios)
- [b. To-do](#b-to-do)
- [c. Requerimientos previos](#c-requerimientos-previos)
- Etapa I — Preparación
  - [1. Instalación de Sequoia en `${WORK_HOST}` y `${OFFLINE_HOST}`](#1-instalación-de-sequoia-en-work_host-y-offline_host)
    - [1.1 Paquetes en las tres distros soportadas](#11-paquetes-en-las-tres-distros-soportadas)
      - [1.1.1 Debian 13 / Ubuntu 24.04+](#111-debian-13--ubuntu-2404)
      - [1.1.2 Fedora 40+ (y RHEL 9 con EPEL)](#112-fedora-40-y-rhel-9-con-epel)
      - [1.1.3 Nix / NixOS](#113-nix--nixos)
    - [1.2 Sequoia en Tails 7](#12-sequoia-en-tails-7)
    - [1.3 Verificación de versión y backend](#13-verificación-de-versión-y-backend)
    - [1.4 Rutas y artefactos de Sequoia en disco](#14-rutas-y-artefactos-de-sequoia-en-disco)
  - [2. Decisiones de diseño y política](#2-decisiones-de-diseño-y-política)
    - [2.1 RFC 9580 vs RFC 4880](#21-rfc-9580-vs-rfc-4880)
    - [2.2 Cipher-suite y algoritmos](#22-cipher-suite-y-algoritmos)
    - [2.3 Política de expiración](#23-política-de-expiración)
    - [2.4 Política de passphrase](#24-política-de-passphrase)
    - [2.5 Modelo de capacidades por subkey](#25-modelo-de-capacidades-por-subkey)
    - [2.6 Subkey SSH: subkey única vs una subkey por servidor](#26-subkey-ssh-subkey-única-vs-una-subkey-por-servidor)
    - [2.7 Checklist de cierre de Etapa I](#27-checklist-de-cierre-de-etapa-i)
- Etapa II — Generación airgapped de la llave primaria
  - [3. Inicialización del entorno offline](#3-inicialización-del-entorno-offline)
    - [3.1 Preparación de la sesión Tails](#31-preparación-de-la-sesión-tails)
    - [3.2 Montaje cifrado de `${AIRGAP_MEDIA}`](#32-montaje-cifrado-de-airgap_media)
    - [3.3 Verificación del aislamiento antes de generar](#33-verificación-del-aislamiento-antes-de-generar)
  - [4. Generación de la llave primaria RFC 9580](#4-generación-de-la-llave-primaria-rfc-9580)
    - [4.1 Definición de las variables del despliegue](#41-definición-de-las-variables-del-despliegue)
    - [4.2 Comando de generación](#42-comando-de-generación)
    - [4.3 Salida esperada y registro del fingerprint](#43-salida-esperada-y-registro-del-fingerprint)
  - [5. Certificado de revocación y custodia inicial](#5-certificado-de-revocación-y-custodia-inicial)
    - [5.1 Verificación del certificado de revocación autogenerado](#51-verificación-del-certificado-de-revocación-autogenerado)
    - [5.2 Custodia separada del revocation cert](#52-custodia-separada-del-revocation-cert)
    - [5.3 Export inicial del certificado público](#53-export-inicial-del-certificado-público)
    - [5.4 Checklist de cierre de Etapa II](#54-checklist-de-cierre-de-etapa-ii)
- Etapa III — Subkeys y traslado al equipo de trabajo
  - [6. Creación de subkeys (Sign / Authenticate / Transport+Storage)](#6-creación-de-subkeys-sign--authenticate--transportstorage)
    - [6.1 Importar la primaria al keystore local de Tails](#61-importar-la-primaria-al-keystore-local-de-tails)
    - [6.2 Subkey de firma (S)](#62-subkey-de-firma-s)
    - [6.3 Subkey de autenticación (A)](#63-subkey-de-autenticación-a)
    - [6.4 Subkey de cifrado (T+R)](#64-subkey-de-cifrado-tr)
    - [6.5 Verificación de la estructura final](#65-verificación-de-la-estructura-final)
  - [7. Exportación de subkeys e importación en `${WORK_HOST}` sin material primario](#7-exportación-de-subkeys-e-importación-en-work_host-sin-material-primario)
    - [7.1 Export selectivo: solo subkeys secretas, no la primaria](#71-export-selectivo-solo-subkeys-secretas-no-la-primaria)
    - [7.2 Transporte físico a `${WORK_HOST}`](#72-transporte-físico-a-work_host)
    - [7.3 Import en `${WORK_HOST}`](#73-import-en-work_host)
    - [7.4 Borrado seguro del USB de transporte](#74-borrado-seguro-del-usb-de-transporte)
  - [8. Verificación del estado "primary key stub" en el equipo de trabajo](#8-verificación-del-estado-primary-key-stub-en-el-equipo-de-trabajo)
    - [8.1 Listado e interpretación](#81-listado-e-interpretación)
    - [8.2 Prueba de firma con la subkey S](#82-prueba-de-firma-con-la-subkey-s)
    - [8.3 Prueba de cifrado y descifrado con la subkey T+R](#83-prueba-de-cifrado-y-descifrado-con-la-subkey-tr)
    - [8.4 Prueba de intento de certificación (debe fallar)](#84-prueba-de-intento-de-certificación-debe-fallar)
    - [8.5 Checklist de cierre de Etapa III](#85-checklist-de-cierre-de-etapa-iii)
    - [8.6 Preparación de una sesión offline para operaciones de mantenimiento](#86-preparación-de-una-sesión-offline-para-operaciones-de-mantenimiento)
- Etapa IV — Gestión del ciclo de vida
  - [9. UserIDs: adición, revocación, primario](#9-userids-adición-revocación-primario)
    - [9.1 Listado e inspección de UserIDs](#91-listado-e-inspección-de-userids)
    - [9.2 Añadir un UserID nuevo (requiere primaria)](#92-añadir-un-userid-nuevo-requiere-primaria)
    - [9.3 Propagar el certificado actualizado a `${WORK_HOST}`](#93-propagar-el-certificado-actualizado-a-work_host)
    - [9.4 Revocar un UserID obsoleto](#94-revocar-un-userid-obsoleto)
    - [9.5 Cuál UserID se muestra como primario](#95-cuál-userid-se-muestra-como-primario)
  - [10. Renovación de expiración (primaria y subkeys) desde el entorno offline](#10-renovación-de-expiración-primaria-y-subkeys-desde-el-entorno-offline)
    - [10.1 Diagnóstico de proximidad de vencimiento](#101-diagnóstico-de-proximidad-de-vencimiento)
    - [10.2 Renovar la expiración de la primaria](#102-renovar-la-expiración-de-la-primaria)
    - [10.3 Renovar la expiración de una subkey](#103-renovar-la-expiración-de-una-subkey)
    - [10.4 Propagación post-renovación](#104-propagación-post-renovación)
  - [11. Revocación: subkey comprometida, UserID obsoleto, llave completa](#11-revocación-subkey-comprometida-userid-obsoleto-llave-completa)
    - [11.1 Revocar una subkey comprometida](#111-revocar-una-subkey-comprometida)
    - [11.2 Revocar la llave completa](#112-revocar-la-llave-completa)
    - [11.3 Reemisión de subkeys tras un compromiso parcial](#113-reemisión-de-subkeys-tras-un-compromiso-parcial)
    - [11.4 Comunicación a contactos](#114-comunicación-a-contactos)
  - [12. Rotación periódica de subkey de firma](#12-rotación-periódica-de-subkey-de-firma)
    - [12.1 Cuándo rotar](#121-cuándo-rotar)
    - [12.2 Procedimiento de rotación](#122-procedimiento-de-rotación)
    - [12.3 Solapamiento controlado](#123-solapamiento-controlado)
    - [12.4 Particularidad de la subkey de cifrado](#124-particularidad-de-la-subkey-de-cifrado)
  - [13. Checklist de cierre de Etapa IV](#13-checklist-de-cierre-de-etapa-iv)
- Etapa V — Operación
  - [14. Backup cifrado y procedimiento de restauración](#14-backup-cifrado-y-procedimiento-de-restauración)
    - [14.1 Qué se respalda, dónde vive cada cosa](#141-qué-se-respalda-dónde-vive-cada-cosa)
    - [14.2 Crear el backup en frío](#142-crear-el-backup-en-frío)
    - [14.3 Restauración de prueba en entorno aislado](#143-restauración-de-prueba-en-entorno-aislado)
    - [14.4 Cuándo refrescar el backup](#144-cuándo-refrescar-el-backup)
  - [15. Publicación y descubrimiento (keyservers verifying, WKD)](#15-publicación-y-descubrimiento-keyservers-verifying-wkd)
    - [15.1 Keyservers verifying](#151-keyservers-verifying)
    - [15.2 Web Key Directory (WKD) propio](#152-web-key-directory-wkd-propio)
    - [15.3 Descubrimiento de certificados de contactos](#153-descubrimiento-de-certificados-de-contactos)
    - [15.4 Re-publicar tras cambios](#154-re-publicar-tras-cambios)
  - [16. Integración con consumidores GnuPG (`gpg-sq` / chameleon)](#16-integración-con-consumidores-gnupg-gpg-sq--chameleon)
    - [16.1 Limitación en Debian 13 con llaves v6 y el wrapper `git-sq-sign`](#161-limitación-en-debian-13-con-llaves-v6-y-el-wrapper-git-sq-sign)
    - [16.2 Activación del chameleon](#162-activación-del-chameleon)
      - [16.2.1 Debian: invocar `gpg-sq` directamente o vía alternatives](#1621-debian-invocar-gpg-sq-directamente-o-vía-alternatives)
      - [16.2.2 Fedora: script de activación con shims](#1622-fedora-script-de-activación-con-shims)
      - [16.2.3 Nix / NixOS: shims vía `environment.systemPackages` o Home Manager](#1623-nix--nixos-shims-vía-environmentsystempackages-o-home-manager)
    - [16.3 Configuración para firma de commits con git](#163-configuración-para-firma-de-commits-con-git)
    - [16.4 Firma de paquetes Debian con `debsign`](#164-firma-de-paquetes-debian-con-debsign)
    - [16.5 Limitaciones conocidas del chameleon](#165-limitaciones-conocidas-del-chameleon)
  - [17. SSH con la subkey de autenticación](#17-ssh-con-la-subkey-de-autenticación)
    - [17.1 Elección del agente](#171-elección-del-agente)
    - [17.2 Export de la subkey A en formato OpenSSH](#172-export-de-la-subkey-a-en-formato-openssh)
    - [17.3 Configurar `ssh-agent` para usar la subkey A](#173-configurar-ssh-agent-para-usar-la-subkey-a)
    - [17.4 Instalar la pública en los servidores remotos](#174-instalar-la-pública-en-los-servidores-remotos)
    - [17.5 Probar la conexión](#175-probar-la-conexión)
    - [17.6 Rotación de la subkey A y propagación a servidores](#176-rotación-de-la-subkey-a-y-propagación-a-servidores)
  - [18. Autenticación de certificados de terceros: shadow CAs y trust root](#18-autenticación-de-certificados-de-terceros-shadow-cas-y-trust-root)
    - [18.1 El Local Trust Root](#181-el-local-trust-root)
    - [18.2 Shadow CAs](#182-shadow-cas)
    - [18.3 Autenticar manualmente el certificado de un contacto](#183-autenticar-manualmente-el-certificado-de-un-contacto)
    - [18.4 Trust depth: cuando un contacto firma a otros contactos](#184-trust-depth-cuando-un-contacto-firma-a-otros-contactos)
    - [18.5 Elevar la confianza en una shadow CA entera](#185-elevar-la-confianza-en-una-shadow-ca-entera)
    - [18.6 Listar y retractar autenticaciones](#186-listar-y-retractar-autenticaciones)
    - [18.7 Expiración de los links](#187-expiración-de-los-links)
    - [18.8 Relación con `sq pki vouch`](#188-relación-con-sq-pki-vouch)
    - [18.9 Checklist de cierre](#189-checklist-de-cierre)
  - [19. Checklist de cierre operativo](#19-checklist-de-cierre-operativo)
- [Anexos](#anexos)
  - [Anexo A — Variables, rutas y artefactos generados](#anexo-a--variables-rutas-y-artefactos-generados)
    - [A.1 Variables del manual](#a1-variables-del-manual)
    - [A.2 Rutas de Sequoia en GNU/Linux](#a2-rutas-de-sequoia-en-gnulinux)
    - [A.3 Convenciones de nombre de archivo](#a3-convenciones-de-nombre-de-archivo)
    - [A.4 Mapa de "qué vive dónde"](#a4-mapa-de-qué-vive-dónde)
  - [Anexo B — Diagnóstico por síntoma](#anexo-b--diagnóstico-por-síntoma)
    - [B.1 Síntoma: `sq key list` en `${WORK_HOST}` muestra la primaria como secreta](#b1-síntoma-sq-key-list-en-work_host-muestra-la-primaria-como-secreta)
    - [B.2 Síntoma: una firma RFC 9580 no verifica en un cliente GnuPG ajeno](#b2-síntoma-una-firma-rfc-9580-no-verifica-en-un-cliente-gnupg-ajeno)
    - [B.3 Síntoma: la passphrase es rechazada tras restaurar el backup](#b3-síntoma-la-passphrase-es-rechazada-tras-restaurar-el-backup)
    - [B.4 Síntoma: el keyserver no acepta el certificado v6](#b4-síntoma-el-keyserver-no-acepta-el-certificado-v6)
    - [B.5 Síntoma: `git commit -S` con el chameleon falla con "BAD signature" o "no public key"](#b5-síntoma-git-commit--s-con-el-chameleon-falla-con-bad-signature-o-no-public-key)
    - [B.6 Síntoma: una subkey expiró antes de rotarla](#b6-síntoma-una-subkey-expiró-antes-de-rotarla)
    - [B.7 Síntoma: una operación falla con "no secret key for primary" desde `${WORK_HOST}`](#b7-síntoma-una-operación-falla-con-no-secret-key-for-primary-desde-work_host)
    - [B.8 Síntoma: `gpg-agent` no ofrece la subkey A a `ssh`](#b8-síntoma-gpg-agent-no-ofrece-la-subkey-a-a-ssh)
    - [B.9 Síntoma: tras importar el cert actualizado en `${WORK_HOST}`, los cambios no aparecen](#b9-síntoma-tras-importar-el-cert-actualizado-en-work_host-los-cambios-no-aparecen)
    - [B.10 Síntoma: al cifrar contra un contacto traído del keyserver, sq advierte "not authenticated"](#b10-síntoma-al-cifrar-contra-un-contacto-traído-del-keyserver-sq-advierte-not-authenticated)
    - [B.11 Síntoma: `gpg-sq` no lista la llave v6 ni siquiera con el fingerprint exacto](#b11-síntoma-gpg-sq-no-lista-la-llave-v6-ni-siquiera-con-el-fingerprint-exacto)
    - [B.12 Síntoma: `git commit -S` con llave v6 reporta "Unusable secret key"](#b12-síntoma-git-commit--s-con-llave-v6-reporta-unusable-secret-key)
  - [Anexo C — Migración desde una llave GnuPG existente](#anexo-c--migración-desde-una-llave-gnupg-existente)
    - [C.1 Decisión previa: importar o regenerar](#c1-decisión-previa-importar-o-regenerar)
    - [C.2 Camino 1 — Import directo de la llave GnuPG](#c2-camino-1--import-directo-de-la-llave-gnupg)
    - [C.3 Camino 2 — Llave v6 nueva con firma cruzada](#c3-camino-2--llave-v6-nueva-con-firma-cruzada)
    - [C.4 Material descifrable y backups históricos](#c4-material-descifrable-y-backups-históricos)

---

## Etapa I — Preparación

### 1. Instalación de Sequoia en `${WORK_HOST}` y `${OFFLINE_HOST}`

El primer paso es disponer de `sq` en las dos máquinas que participan en el ciclo de vida de la llave: la estación de trabajo (`${WORK_HOST}`), donde vivirán las subkeys y se realizan las operaciones diarias, y la sesión Tails (`${OFFLINE_HOST}`), donde se genera y custodia la llave primaria sin contacto con la red.

#### 1.1 Paquetes en las tres distros soportadas

Las tres distribuciones que este manual cubre empaquetan Sequoia como ciudadano de primera clase, con versión `sq 1.3.1` o superior. Los nombres de paquete varían y, sobre todo, el **chameleon** (la reimplementación de la CLI de GnuPG sobre Sequoia) tiene nombres y mecanismos de activación distintos en cada distro. Esto importa porque la sección 16 depende de ello.

##### 1.1.1 Debian 13 / Ubuntu 24.04+

Debian 13 "trixie" empaqueta Sequoia en su repositorio principal, incluyendo el chameleon bajo el nombre Debian-específico `gpg-from-sq`, que provee binarios `gpg-sq` y `gpgv-sq` directamente en `/usr/bin/`. Como punto de partida, se recomienda seguir el manual de despliegue de Debian disponible [en este enlace](https://github.com/noggalito/manuales/blob/main/sistemas-operativos/debian/13-server.md), y luego seguir con:

```bash
sudo apt update
sudo apt install -y sq sqop sqv gpg-from-sq
```

Qué hace cada paquete:

- `sq` — frontend principal de línea de comandos.
- `sqop` — implementación de la interfaz Stateless OpenPGP. Útil para integraciones; opcional.
- `sqv` — verificador de firmas separadas. Lo usa `apt` internamente cuando corresponde.
- `gpg-from-sq` — instala `gpg-sq` y `gpgv-sq`, drop-in para reemplazar `gpg`/`gpgv` cuando una herramienta los invoca. Se cubre en detalle en la sección 16.

##### 1.1.2 Fedora 40+ (y RHEL 9 con EPEL)

Fedora empaqueta los componentes Sequoia con prefijo `sequoia-`. El chameleon en Fedora se llama `sequoia-chameleon-gnupg` (es el upstream real del proyecto; en Debian está rebautizado como `gpg-from-sq`).

```bash
sudo dnf install -y sequoia-sq sequoia-sqv sequoia-sop sequoia-chameleon-gnupg
```

Diferencia operativa importante respecto a Debian: en Fedora `sequoia-chameleon-gnupg` **no** instala binarios directamente como `gpg-sq`. En su lugar instala los shims en `/usr/share/sequoia-chameleon-gnupg/shims/` (que se llaman `gpg` y `gpgv`), y un script de activación que pone esos shims al principio del `$PATH`. La activación se documenta en 16.2.2.

En RHEL 9 los paquetes están disponibles en EPEL:

```bash
sudo dnf install -y epel-release
sudo dnf install -y sequoia-sq sequoia-sqv sequoia-sop sequoia-chameleon-gnupg
```

##### 1.1.3 Nix / NixOS

En Nix y NixOS los paquetes se llaman igual que en Fedora (prefijo `sequoia-`). Hay dos formas de instalarlos según el modelo de gestión de paquetes del usuario.

**Instalación imperativa (Nix profile, válido en cualquier distro con Nix instalado):**

```bash
nix-env -iA nixpkgs.sequoia-sq nixpkgs.sequoia-sqv nixpkgs.sequoia-sqop \
            nixpkgs.sequoia-chameleon-gnupg
```

**Instalación declarativa en NixOS (`configuration.nix`):**

```nix
environment.systemPackages = with pkgs; [
  sequoia-sq
  sequoia-sqv
  sequoia-sqop
  sequoia-chameleon-gnupg
];
```

Tras aplicar con `sudo nixos-rebuild switch`, los binarios `sq`, `sqv`, `sqop` quedan disponibles en `/run/current-system/sw/bin/`. El chameleon en Nix sigue el mismo modelo de shims que Fedora; la activación se documenta en 16.2.2.

**Instalación declarativa en Home Manager (per-user):**

```nix
home.packages = with pkgs; [
  sequoia-sq
  sequoia-sqv
  sequoia-sqop
  sequoia-chameleon-gnupg
];
```

> Si el sistema ya tenía GnuPG instalado encualquier distro, no se desinstala. `gpg` y la familia chameleon coexisten; en la sección 16 se documenta cómo decidir cuál responde por defecto cuando una herramienta invoca el binario `gpg`.

#### 1.2 Sequoia en Tails 7

No es mandatorio disponer de otra estación de trabajo para `${OFFLINE_HOST}`, basta con asegurarse de arrancar Tails con el USB correcto en la misma estación `${WORK_HOST}`, ejecutar el procedimiento, luego apagar Tails, retirar el USB y arrancar con la distro habitual de `${OFFLINE_HOST}`.

Tails está basado en Debian trixie por construcción, independientemente de qué distro corra `${WORK_HOST}`. La diferencia operativa es que Tails es un live system: por defecto cualquier paquete instalado desaparece al reiniciar. Hay dos opciones:

1. Instalar `sq` cada vez que se inicia la sesión. Es el modelo más limpio desde el punto de vista de aislamiento -la sesión nace siempre desde cero-, pero requiere conexión a red al menos una vez para descargar el paquete, contradiciendo el principio airgapped.
2. Usar Additional Software de Tails para que `sq` se reinstale automáticamente al iniciar desde el Persistent Storage. Requiere haber configurado Persistent Storage previamente y haber descargado `sq` al menos una vez con red.

Para este manual se asume la opción 2, configurada antes de cortar la red. El procedimiento de habilitar Persistent Storage y agregar `sq` como Additional Software se hace una sola vez, con red activa, en una sesión preparatoria:

```bash
sudo apt update
sudo apt install -y sq
```

Tails ofrecerá un diálogo "Add to your Additional Software"; aceptar con "Install Every Time" para que persista entre reinicios.

<p style="text-align: center"><img src="./assets/captura01.png" style="width: 50%;" alt="Captura 01" /></p>

A partir de ese momento, cada sesión Tails iniciada con Persistent Storage desbloqueado tendrá `sq` disponible sin requerir red. Toda la generación de llave en la Etapa II se hace en sesiones posteriores, ya con la red cortada.

> En Tails no se instala el chameleon, ni `gpg-from-sq`, ni `sequoia-chameleon-gnupg`. La sesión airgapped no necesita interoperar con clientes GnuPG; su único trabajo es generar y custodiar la llave primaria.

#### 1.3 Verificación de versión y backend

Antes de cualquier operación con llaves, confirmar que `sq` está bien instalado y reporta la versión esperada:

```bash
sq version
```

Salida esperada:

```
sq 1.3.1
using sequoia-openpgp 2.0.0
with cryptographic backend Nettle 3.10 (Cv448: true, OCB: true)
```

Lo importante: que la primera línea diga `1.3.1` o superior, y que la línea del backend mencione Nettle. Las tres distros soportadas (Debian, Fedora, Nix) construyen los paquetes contra Nettle por defecto. Si la salida menciona otro backend (OpenSSL, Botan), probablemente se instaló `sq` por una vía no convencional (cargo, build local) y conviene revisarlo antes de seguir — algunos comandos pueden comportarse diferente con otros backends.

#### 1.4 Rutas y artefactos de Sequoia en disco

Sequoia organiza su estado en varios directorios bajo `$HOME`. Conocer estas rutas evita sorpresas más adelante, sobre todo al hacer backup, restaurar o auditar qué material vive en cada máquina.

```bash
ls -la ${HOME}/.local/share/pgp.cert.d/     # certificate store (claves públicas)
ls -la ${HOME}/.local/share/sequoia/        # keystore y revocation certs
ls -la ${HOME}/.config/sequoia/             # configuración (config.toml si existe)
```

Función de cada uno:

- `~/.local/share/pgp.cert.d/` — almacén estándar de certificados (claves públicas, tuyas y de terceros). Sigue el formato OpenPGP Certificate Directory.
- `~/.local/share/sequoia/keystore/` — donde el key-store server de sq guarda el material secreto que administra. En `${WORK_HOST}` vivirán aquí las subkeys; en `${OFFLINE_HOST}` vivirá la primaria + subkeys.
- `~/.local/share/sequoia/revocation-certificates/` — destino por defecto de los certificados de revocación autogenerados por `sq key generate`. Conviene **no** dejar el revocation cert aquí en producción: se mueve a custodia separada (sección 5).
- `~/.config/sequoia/sq/config.toml` — archivo opcional de configuración. No se usa en este manual.

> La variable de entorno `SEQUOIA_HOME` si está definida, redirige todo lo anterior a otra raíz. Útil para entornos de prueba aislados (por ejemplo, restaurar un backup en `/tmp/sq-test/` sin tocar el keystore real). Se usa en la sección 14.3.

### 2. Decisiones de diseño y política

Antes de generar nada, conviene fijar las decisiones de diseño que se mantendrán durante toda la vida de la llave. Cambiarlas después es costoso: significa rotar, revocar, o regenerar.

#### 2.1 RFC 9580 vs RFC 4880

OpenPGP tiene dos versiones de formato vigentes. [RFC 4880](https://datatracker.ietf.org/doc/rfc4880/) de 2007 es la versión 4, soportada por todo el ecosistema, incluyendo GnuPG y Gpg4win. [RFC 9580](https://datatracker.ietf.org/doc/rfc9580/) de julio 2024 es la versión 6, soportada por Sequoia, OpenPGP.js de Proton, y otras implementaciones modernas, pero rechazada por GnuPG que mantiene un estándar paralelo llamado [LibrePGP](https://librepgp.org/) también basado en v4.

Este manual usa **RFC 9580** desde el día uno. Las razones:

1. La versión 6 corrige problemas de seguridad y diseño documentados en la versión 4, en particular alrededor del padding de cifrado y la estructura de signatures.
2. Las tres distros que cubre el manual empaquetan Sequoia como ciudadano de primera clase. Debian va el paso más adelantado, la verificación de archivos de `apt` ya usa `sqv`, y `gpg-from-sq` está disponible como reemplazo drop-in; Fedora empaqueta toda la familia incluyendo `sequoia-chameleon-gnupg` aunque sin reemplazar la verificación de `rpm`/`dnf`; Nix la trata como cualquier otro paquete pero con todos los componentes disponibles. El ecosistema en el que opera el manual está alineado con v6 en las tres.
3. La interoperabilidad con consumidores GnuPG (git sign, debsign, mutt) no requiere que la llave sea v4 indefinidamente. El chameleon (`gpg-sq` / `sequoia-chameleon-gnupg`) está pensado para leer y firmar con llaves v6, y los verificadores GnuPG remotos solo necesitan la firma, no la llave completa. Existen casos límite -por ejemplo un destinatario que solo corre GnuPG clásico y quiere cifrar contra tu certificado- y se cubren en la sección 16.5.

> **Limitación práctica detectada en Debian 13 trixie (mayo 2026).** `chameleon` depende internamente de `gpg-agent` de GnuPG clásico para custodiar material secreto, y GnuPG clásico **no acepta material secreto en formato v6** (responde `Invalid operation: Keygrip not defined for this kind of public key` al intentar importarlo). El resultado: `gpg-sq` ve la llave v6 en el cert-store pero al firmar reporta `Unusable secret key`. La sección 16 documenta un wrapper basado en `sq sign` directo (`git-sq-sign`) que sortea esta limitación hasta que GnuPG con soporte v6 completo llegue al ecosistema (probablemente Debian 14 o backports). Quien priorice la integración drop-in con git y `debsign` sin wrapper, o quien necesite máxima compatibilidad con destinatarios GnuPG legacy, puede preferir generar la llave en v4 con `--profile rfc4880`. El resto del manual aplica igual, cambiando solo ese flag.

#### 2.2 Cipher-suite y algoritmos

Sequoia ofrece varias cipher-suites para `sq key generate`. La opción por defecto en `sq` 1.3.1 es `cv25519` (Curve25519 + Ed25519), basada en curva elíptica. Es la opción correcta para casi todos los escenarios actuales: claves pequeñas (256 bits), operaciones rápidas, sin patentes, sin problemas conocidos de aleatoriedad de parámetros.

Las alternativas que ofrece Sequoia son `rsa3k`, `rsa4k` (RSA tradicional, claves grandes, más lento, compatibilidad universal) y `nistp256`/`nistp384`/`nistp521` (curvas NIST, menos preferidas por la sospecha histórica sobre los parámetros). En este manual se usa `cv25519` explícitamente, aunque sea el default, para que quede registrado en la línea de comando.

#### 2.3 Política de expiración

Política recomendada:

- Llave primaria (capacidad Certify): expiración larga, `${EXPIRATION}` = `3y` (tres años). Al estar offline, su renovación implica desbloquear el medio airgapped y firmar; conviene que no sea frecuente.
- Subkeys de uso diario (Sign, Authenticate, Transport, Storage): expiración corta, `1y` (un año). La rotación anual de subkeys es una práctica defensiva: si una subkey se ve comprometida y no se detecta, el daño tiene fecha de caducidad.

La rotación de subkeys (sección 12) y la extensión de expiración de la primaria (sección 10) son operaciones distintas, ambas documentadas.

#### 2.4 Política de passphrase

Toda llave secreta generada en este manual va protegida con passphrase. No se documenta `--without-password`. La passphrase de la primaria y la de las subkeys pueden ser distintas (sq lo permite); este manual lo recomienda y asume que sí lo son: una passphrase fuerte para la primaria (memorizada y/o custodiada por separado), y passphrase distinta para las subkeys de uso diario. También se recomienda el uso de un gestor de contraseñas para almacenar de forma segura todas las passphrase.

#### 2.5 Modelo de capacidades por subkey

OpenPGP distingue cinco capacidades, identificadas por una letra cuando `sq inspect` lista una llave:

- C — Certify: firmar otras llaves y UserIDs. Es la capacidad nuclear; vive solo en la primaria.
- S — Sign: firmar datos (commits de git, archivos, correos).
- A — Authenticate: usar la llave como identidad de autenticación, SSH es el caso principal.
- T — Transport (Encryption Communications): cifrar mensajes en tránsito.
- R — Storage (Encryption Storage): cifrar datos en reposo.

Sequoia agrupa T y R bajo la categoría "Encryption" y, por defecto, una subkey de cifrado con `--can-encrypt universal` vale para ambas. Por eso en la tabla siguiente, las dos capacidades aparecen como una sola fila ("T+R universal"), es una subkey con dos flags simultáneos, no dos subkeys distintas.

El modelo de subkeys de este manual:

| Subkey | Capacidad | Expiración | Vive en |
| --- | --- | --- | --- |
| Primaria | C | 3 años | `${OFFLINE_HOST}` (Tails) |
| Subkey de firma | S | 1 año | `${WORK_HOST}` (Debian/Fedora/NixOS) |
| Subkey de autenticación | A | 1 año | `${WORK_HOST}` (Debian/Fedora/NixOS) |
| Subkey de cifrado | T+R (universal) | 1 año | `${WORK_HOST}` (Debian/Fedora/NixOS) |

La primaria nunca firma datos directamente, solo certifica subkeys, UserIDs y otras llaves. Las tres operaciones cotidianas (firmar, autenticar, cifrar) se delegan en subkeys. El total: una llave primaria con capacidad C, y tres subkeys que entre ellas cubren S, A, T y R.

#### 2.6 Subkey SSH: subkey única vs una subkey por servidor

Para autenticación SSH con OpenPGP existen dos modelos posibles: una sola subkey de autenticación que sirve para todos los servidores, o una subkey distinta por cada host de destino. Este manual usa el primer modelo: una sola subkey A.

Razones:

1. Es la práctica predominante en la comunidad OpenPGP cuando se elige usar OpenPGP para SSH -mantenedores Debian con smartcard, equipos de Tor, usuarios de Nitrokey/YubiKey en modo PGP-. La identidad criptográfica representa a la persona, no la conexión.
2. La revocación granular por servidor se resuelve mejor a nivel de `~/.ssh/authorized_keys` del host afectado (borrar la línea correspondiente) que a nivel criptográfico. Mantener N subkeys de autenticación complica la gestión sin aportar valor proporcional al esfuerzo.
3. El flujo `gpg-agent` / `ssh-agent` clásico, que `gpg-sq` reproduce, asume una sola identidad de autenticación. Múltiples subkeys A obligan a configurar `IdentitiesOnly yes` y `IdentityFile` por host en `~/.ssh/config`, agregando superficie de error.

Casos donde una subkey A por ambiemte sí tiene sentido: cuando se quiere segmentar acceso por entorno -producción / staging / desarrollo-, no por servidor individual. En ese caso se generarían 2 o 3 subkeys A, no decenas. Este manual no lo cubre, sin embargo quien lo necesite puede replicar el patrón de la sección 6 con `--can-authenticate` varias veces.

#### 2.7 Checklist de cierre de Etapa I

- [ ] `sq version` en `${WORK_HOST}` reporta `1.3.1` o superior, backend Nettle
- [ ] `sq version` en `${OFFLINE_HOST}`: Tails con Persistent Storage reporta versión equivalente
- [ ] Chameleon instalado en `${WORK_HOST}`: `gpg-sq --version` en Debian; `source /usr/share/sequoia-chameleon-gnupg/activate && gpg --version` en Fedora; equivalente vía PATH en Nix
- [ ] Decisiones de diseño documentadas: perfil v6, cipher-suite cv25519, expiraciones 3y/1y, una sola subkey A, registradas en bitácora o en la cabecera del archivo de notas del despliegue

---

## Etapa II — Generación airgapped de la llave primaria

La llave primaria es la pieza más sensible del despliegue: certifica subkeys, UserIDs y otras llaves, y su compromiso obliga a regenerar todo el material y comunicar el incidente a los contactos. Por eso se genera y vive en un entorno airgapped (Tails), nunca toca el equipo de trabajo, y solo se "trae" cuando hay que firmar algo nuevo (subkey adicional, UserID nuevo, extensión de expiración).

### 3. Inicialización del entorno offline

#### 3.1 Preparación de la sesión Tails

El procedimiento asume que Tails 7.7.3 ya se instaló en un USB dedicado, con Persistent Storage habilitado y `sq` agregado como Additional Software (sección 1.2). Cada sesión de generación o mantenimiento de la primaria empieza así:

1. Reiniciar o arrancar `${OFFLINE_HOST}` desde el USB de Tails.
2. En la pantalla de bienvenida, desbloquear Persistent Storage con la passphrase del volumen.
3. **No** habilitar redes, ni Wi-Fi, ni Ethernet. Si la máquina tiene tarjeta Wi-Fi, conviene retirarla físicamente o cubrir el puerto Ethernet con cinta para evitar conexiones accidentales.

Además de la inhabilitación por hardware, es recomendable también hacerlo por software:

```bash
sudo rfkill block all

# Verificar
sudo rfkill list

# Listar interfaces
ip link show

# Apagar cada interfaz que no sea loopback
sudo ip link set <interfaz> down
# Ejemplo: sudo ip link set eth0 down
# Ejemplo: sudo ip link set enp3s0 down

# Desconectar NetworkManager por completo:
sudo systemctl stop NetworkManager
sudo systemctl stop NetworkManager-wait-online

# Verificar
ip addr show
```

Solo debe aparecer `lo` (loopback) con `127.0.0.1`. No debe haber otras interfaces con IP, o si aparecen deben estar state DOWN y sin IP asignada.

```bash
ip route show
```

No debe devolver nada (sin `default via ...`). Si devuelve algo, hay ruta de red activa.

```bash
ss -tan
```

No debe mostrar conexiones en estado `ESTAB`. Solo `LISTEN` locales (procesos escuchando en `127.0.0.1`) son aceptables.

**Confirmar la hora del sistema:**

```bash
date
timedatectl status     # confirma que no hay sincronización NTP activa
```

Tails sin red no sincroniza por NTP, así que la hora la fija el reloj del hardware. Si el reloj está desfasado más de unos minutos, ajustar manualmente antes de generar: la fecha de creación de la llave queda embebida en las firmas y un desfase grande causa confusiones en los verificadores.

```bash
sudo date -s "2026-07-07 17:00:00"   # ajustar solo si el reloj está mal
```

#### 3.2 Montaje cifrado de `${AIRGAP_MEDIA}`

`${AIRGAP_MEDIA}` es un USB separado del de Tails, dedicado exclusivamente a transportar y custodiar material de la llave. Debe estar formateado con LUKS. Si aún no lo está, esta es la sesión preparatoria para hacerlo:

```bash
lsblk -o NAME,SIZE,MODEL,VENDOR,TRAN,MOUNTPOINTS # Para identificar el dispositivo
export MIUSB=/dev/sdb                           # Reemplazar `/dev/sdb` por el dispositivo real, vverificar con `lsblk` antes; un error aquí destruye el dispositivo equivocado.
sudo cryptsetup luksFormat $MIUSB               # destruye lo que haya, confirmar con MAYÚSCULAS
sudo cryptsetup open $MIUSB airgap-media
sudo mkfs.ext4 -L airgap /dev/mapper/airgap-media
sudo mkdir -p /mnt/airgap
sudo mount /dev/mapper/airgap-media /mnt/airgap
sudo chown amnesia:amnesia /mnt/airgap            # amnesia es el usuario por defecto en Tails
```

En sesiones posteriores, solo se monta:

```bash
sudo cryptsetup open $MIUSB airgap-media
sudo mount /dev/mapper/airgap-media /mnt/airgap
```

Estructura de directorios en el medio airgapped:

```bash
mkdir -p /mnt/airgap/{keys,revocation,backup,exports}
```

- `keys/` — material secreto completo (primaria + subkeys), nunca sale de aquí.
- `revocation/` — copia local del cert de revocación. La copia maestra se custodia en otro medio (sección 5.2).
- `backup/` — copia adicional del material secreto, también nunca sale del medio (sección 13).
- `exports/` — exports temporales (cert público, subkeys secretas) que sí salen hacia `${WORK_HOST}` vía un segundo USB de transporte.

#### 3.3 Verificación del aislamiento antes de generar

Última comprobación antes de tocar `sq key generate`:

```bash
ip addr show           # ninguna interfaz debe tener IP de red real
ip route show          # no debe haber default route
ss -tan                # no debe haber conexiones establecidas
```

Si cualquiera de las tres muestra actividad de red, abortar y diagnosticar antes de seguir. La integridad airgapped del procedimiento depende de este paso.

### 4. Generación de la llave primaria RFC 9580

#### 4.1 Definición de las variables del despliegue

Antes del comando, fijar las variables en el shell de la sesión. Esto vale para esta sesión solamente; Tails las olvida al reiniciar.

```bash
export USER_REAL="Nombre Apellido"
export USER_EMAIL="usuario@ejemplo.org"
export USER_ID="villonaco"                   # pseudónimo / handle del usuario
export EXPIRATION="3y"                        # expiración de la primaria
export KEY_OUT="/mnt/airgap/keys/primary.key"
export REV_OUT="/mnt/airgap/revocation/primary.rev"
```

Sobre `${USER_ID}`: además de los UserIDs estructurados que generan `--name` y `--email`, sq permite añadir UserIDs arbitrarios con `--userid`. Esto sirve para asociar la llave a un pseudónimo, un handle de GitHub, un nombre de proyecto o cualquier identificador no convencional. Ejemplos válidos para `${USER_ID}`: `villonaco`, `webserver`, `nextcloudserver`, `noggalito`. La llave terminará con tres UserIDs: el nombre real, el correo y el pseudónimo, cada uno certificable de forma independiente.

#### 4.2 Comando de generación

Una característica de `sq key generate` que conviene conocer antes de ejecutarlo: por defecto, el comando produce no solo la llave primaria sino también tres subkeys: Sign, Authenticate, y Encrypt universal, todas con la misma expiración heredada de la primaria. Es el modelo "primaria + subkeys de un tirón" que la mayoría de tutoriales asume.

Este manual no usa ese modelo. Por la política decidida en 2.3 (primaria con expiración 3y, subkeys con expiración 1y) y por la pedagogía del manual (cada subkey con su flag explícito en la Etapa III), aquí generamos solo la primaria en esta sesión, y las subkeys aparte en la sección 6. Para suprimir la generación automática de subkeys, pasamos los flags negativos `--cannot-sign`, `--cannot-authenticate` y `--cannot-encrypt`:

```bash
sq key generate \
  --profile rfc9580 \
  --own-key \
  --cipher-suite cv25519 \
  --name "${USER_REAL}" \
  --email "${USER_EMAIL}" \
  --userid "${USER_ID}" \
  --expiration "${EXPIRATION}" \
  --cannot-sign \
  --cannot-authenticate \
  --cannot-encrypt \
  --output "${KEY_OUT}" \
  --rev-cert "${REV_OUT}"
```

Significado de cada flag:

- `--profile rfc9580` — fuerza el formato OpenPGP v6 (decisión 2.1). El default de sq 1.3.1 sigue siendo `rfc4880`; sin este flag se generaría una llave v4.
- `--own-key` — marca la llave como "unconstrained trust introducer" en el cert store local. Significa que sq tratará automáticamente como autenticadas las certificaciones que esta llave emita sobre otras. Es el comportamiento correcto para *tu propia* llave. El modelo completo (Local Trust Root, shadow CAs, cadenas de certificación) se explica en la sección 18; baste por ahora saber que sin `--own-key` la propia llave aparecería como "no autenticada" en su propio cert store, lo cual es absurdo y `--own-key` resuelve.
- `--cipher-suite cv25519` — Curve25519 + Ed25519 (decisión 2.2). Explícito aunque sea el default, para que quede registrado.
- `--name` y `--email` — generan dos UserIDs estructurados separados, siguiendo la recomendación del proyecto Sequoia de no concatenarlos en uno solo. Eso permite que un tercero certifique tu correo sin necesariamente certificar tu nombre legal (o viceversa).
- `--userid "${USER_ID}"` — añade un tercer UserID arbitrario con el pseudónimo. Útil para asociar la llave a una identidad no civil.
- `--expiration "${EXPIRATION}"` — duración de validez de la primaria. Acepta sufijos `y`, `m`, `w`, `d`, `s`. La palabra `never` desactiva expiración; este manual no la usa.
- `--cannot-sign` — suprime la subkey de firma automática. La crearemos en 6.2 con expiración 1y.
- `--cannot-authenticate` — suprime la subkey de autenticación automática. La crearemos en 6.3 con expiración 1y.
- `--cannot-encrypt` — suprime la subkey de cifrado automática. La crearemos en 6.4 con expiración 1y.
- `--output` — archivo de salida con el material secreto. Va al medio cifrado `${AIRGAP_MEDIA}`, nunca al disco persistente de Tails. Sin `--output`, sq guardaría la llave directamente en el keystore local; lo evitamos para tener un archivo manejable.
- `--rev-cert` — archivo donde sq escribe el certificado de revocación autogenerado. Si se omite y `--output` también, sq lo guarda en `~/.local/share/sequoia/revocation-certificates/`. Como aquí sí usamos `--output`, conviene también nombrar `--rev-cert` explícitamente para controlar dónde queda.

Al ejecutar, sq pide la passphrase de la llave de forma interactiva (dos veces, para confirmar). Esta es la passphrase de la llave primaria; conviene que sea distinta de la que se usará para las subkeys (decisión 2.4).

#### 4.3 Salida esperada y registro del fingerprint

Tras la generación, sq imprime un resumen parecido a:

```
/mnt/airgap/keys/primary.key: Transferable Secret Key.

      Fingerprint: F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8
  Public-key algo: Ed25519
  Public-key size: 256 bits
       Secret key: Encrypted
    Creation time: 2026-05-19 00:11:25 UTC
  Expiration time: 2029-05-18 17:37:46 UTC (creation time + 2years 11months 30days 9h 16m 45s)
        Key flags: certification

           UserID: <usuario@ejemplo.org>

           UserID: Nombre Apellido

           UserID: villonaco
```

Es evidente que los datos personalizados propios de la llave (fingerprint, fecha de creación y expiración, UserID, etc.) se reflejarán de forma distinta al resumen listado en este manual. Verificar en la salida:

- `Public-key algo: Ed25519` y `Public-key size: 256 bits` para la primaria.
- `Secret key: Encrypted`, es una confirmación de que está protegida con passphrase.
- Los tres UserIDs presentes.
- `Key flags: certification` en la primaria — solo certifica, nada más.
- Importante: no aparecen subkeys. Si la salida muestra subkeys con flags `signing`, `authentication` o `encryption`, alguno de los `--cannot-*` no se aplicó; revisar el comando antes de seguir. Las subkeys de uso diario se crean explícitamente en la Etapa III.

Registrar el fingerprint en bitácora:

```bash
export KEY_FP="F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8"   # el de la salida
echo "${KEY_FP}" > /mnt/airgap/keys/FINGERPRINT.txt
```

El fingerprint es la identidad pública de la llave. Se compartirá en tarjetas, firmas de correo, perfiles. Es seguro publicarlo. Para una mejor comprensión, también se estila compartir los últimos 16 caracteres del fingerprint, sin  embargo en los entornos netamente técnicos esto podría causar colisiones

### 5. Certificado de revocación y custodia inicial

#### 5.1 Verificación del certificado de revocación autogenerado

El comando del paso 4.2 ya creó el certificado de revocación en `${REV_OUT}`. Inspeccionar:

```bash
sq inspect "${REV_OUT}"
```

Salida esperada:

```
/mnt/airgap/revocation/primary.rev: Revocation Certificate.

      Fingerprint: F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8
                   Revoked:
                    - No reason specified
                      On: 2026-05-19 00:11:25 UTC
                      Message: Unspecified
                   Invalid: No binding signature at time 2026-05-19T00:14:45Z
  Public-key algo: Ed25519
  Public-key size: 256 bits
    Creation time: 2026-05-19 00:11:25 UTC
```

A esta altura el archivo en `${KEY_OUT}` contiene exclusivamente la llave primaria con capacidad C, sin subkeys. Las subkeys de uso diario (S, A, T+R) se crean en la Etapa III, sección 6, cada una con expiración 1y según la política 2.3.

El certificado de revocación es, en términos prácticos, una declaración firmada por la propia llave que dice "esta llave ya no debe usarse". No se aplica todavía: solo se distribuye e importa cuando se quiere revocar la llave (por pérdida, compromiso, o retiro). Hasta entonces se custodia.

#### 5.2 Custodia separada del revocation cert

El revocation cert nunca debe vivir solo en el mismo medio donde está la llave primaria. La lógica: si pierdes el medio `${AIRGAP_MEDIA}`, pierdes a la vez la llave y la capacidad de revocarla — el peor escenario posible. La regla operativa:

- Copia A del revocation cert: en `${AIRGAP_MEDIA}/revocation/` donde sq lo escribió originalmente.
- Copia B del revocation cert: en un segundo medio cifrado físicamente distinto y guardado en otra ubicación geográfica (caja fuerte, casa de un familiar de confianza, depósito).
- Pasphrase del segundo medio: distinta de la del primero, custodiada por separado.

Procedimiento para preparar la copia B, asumiendo un segundo USB LUKS ya formateado y montado en `/mnt/safe-copy`:

```bash
sudo mkdir -p /mnt/safe-copy/${KEY_FP}/
sudo cp "${REV_OUT}" /mnt/safe-copy/${KEY_FP}/primary.rev
sudo cp /mnt/airgap/keys/FINGERPRINT.txt /mnt/safe-copy/${KEY_FP}/
sudo umount /mnt/safe-copy
sudo cryptsetup close safe-copy
```

El segundo USB se guarda fuera del entorno habitual. En un incidente -laptop perdida, casa robada- aún se cuenta con la capacidad de revocar.

#### 5.3 Export inicial del certificado público

El certificado público -sin material secreto- es lo que se distribuye a contactos y keyservers. Sequoia lo extrae con `sq cert export`:

```bash
sq --keyring "${KEY_OUT}" cert export --cert "${KEY_FP}" \
   --output /mnt/airgap/exports/"$(date +'%Y-%m-%d_%H-%M')"-${USER_EMAIL/@/_at_}-${KEY_FP: -16}.pub.asc
```

En sq 1.3.1, `sq cert export` opera sobre el cert store del usuario. Cuando se trabaja con un archivo de llave aislado en lugar del store -el caso de Tails airgapped- conviene pasar `--keyring` explícito o importar primero la llave al store local de Tails con `sq key import`. Si la llave ya está en el store de Tails, basta con `sq cert export --cert "${KEY_FP}" --output ...`.

El archivo resultante (`2026-05-17-usuario_at_ejemplo.org-EF567890ABCD1234.pub.asc`) sí puede salir de `${AIRGAP_MEDIA}`, no contiene material secreto, solo claves públicas y firmas. Se moverá a `${WORK_HOST}` en la Etapa III.

#### 5.4 Checklist de cierre de Etapa II

- [ ] `sq inspect` sobre `${KEY_OUT}` muestra perfil v6, EdDSA, secret key Encrypted, tres UserIDs
- [ ] `${KEY_FP}` registrado en bitácora y en `/mnt/airgap/keys/FINGERPRINT.txt`
- [ ] Certificado de revocación existe en `${REV_OUT}` y `sq inspect` lo reconoce
- [ ] Copia B del revocation cert custodiada en medio físico separado
- [ ] Certificado público exportado a `/mnt/airgap/exports/${KEY_FP}.pub.asc`

---

## Etapa III — Subkeys y traslado al equipo de trabajo

Con la llave primaria generada y aún en el medio airgapped, esta etapa crea las subkeys de uso diario (S, A, T+R), las exporta selectivamente, y las importa en `${WORK_HOST}` sin que el material secreto de la primaria viaje fuera de Tails.

### 6. Creación de subkeys (Sign / Authenticate / Transport+Storage)

#### 6.1 Importar la primaria al keystore local de Tails

Las operaciones de `sq key subkey add` necesitan que la llave primaria esté en el keystore local de sq, no en un archivo suelto. En `${OFFLINE_HOST}`, dentro de la sesión Tails con `${AIRGAP_MEDIA}` montado:

```bash
sq key import "${KEY_OUT}"
```

Verificar que entró:

```bash
sq key list
```

Salida esperada:

```
 - Device gpg-agent/default has no keys.

 - F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8
   - user IDs:
     - <usuario@ejemplo.org> (UNAUTHENTICATED)
     - Nombre Apellido (UNAUTHENTICATED)
     - villonaco (UNAUTHENTICATED)
   - created 2026-05-19 00:11:25 UTC
   - will expire 2029-05-18T17:37:46Z
   - usable for signing
   - @softkeys/F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8: available, locked
```

#### 6.2 Subkey de firma (S)

```bash
sq key subkey add \
  --cert "${KEY_FP}" \
  --can-sign \
  --expiration 1y
```

sq pide dos passphrases: la de la primaria (para firmar la nueva subkey binding) y la nueva passphrase para la subkey secreta. Conviene usar una passphrase distinta para las subkeys de uso diario, manejable, que se tipea varias veces al día.

#### 6.3 Subkey de autenticación (A)

```bash
sq key subkey add \
  --cert "${KEY_FP}" \
  --can-authenticate \
  --expiration 1y
```

Esta es la subkey que se usará para SSH (decisión 2.6: una sola subkey A para todos los servidores). El flujo completo de uso con `ssh-agent` y `authorized_keys` se documenta en la sección 16.

> Si por algún caso particular se necesita una segunda subkey A para segmentación por entorno (no por servidor, ver 2.6), basta con repetir el comando. La sección 16 se concentra en el caso de una sola.

#### 6.4 Subkey de cifrado (T+R)

```bash
sq key subkey add \
  --cert "${KEY_FP}" \
  --can-encrypt universal \
  --expiration 1y
```

El valor `universal` para `--can-encrypt` significa que la subkey vale tanto para cifrado en tránsito (Transport, T) como en reposo (Storage, R). Es el default en sq y suele ser lo que se quiere: separar T y R en dos subkeys distintas tiene sentido en escenarios muy específicos (cifrado de backups con una llave que nunca se carga en cliente de correo), que este manual no cubre.

Alternativas si se necesitaran:

- `--can-encrypt transport` — solo cifrado en tránsito.
- `--can-encrypt storage` — solo cifrado en reposo.

#### 6.5 Verificación de la estructura final

```bash
sq inspect --cert "${KEY_FP}"
```

Salida esperada:

```
OpenPGP Certificate.

      Fingerprint: F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8
  Public-key algo: Ed25519
  Public-key size: 256 bits
    Creation time: 2026-05-19 00:11:25 UTC
  Expiration time: 2029-05-18 17:37:46 UTC (creation time + 2years 11months 30days 9h 16m 45s)
        Key flags: certification

           Subkey: 0EDCC3CD931C718819BA0DA7ECCB99F1E6B2B8FA1BEDF9B06D1282AAC71444F5
  Public-key algo: X25519
  Public-key size: 256 bits
    Creation time: 2026-05-19 00:18:29 UTC
  Expiration time: 2027-05-19 06:07:16 UTC (creation time + 11months 30days 9h 39m 11s)
        Key flags: transport encryption, data-at-rest encryption

           Subkey: A62601755871313C79082742025A14621C7A592AFDDCF4444A0FA6B7068842E5
  Public-key algo: Ed25519
  Public-key size: 256 bits
    Creation time: 2026-05-19 00:17:36 UTC
  Expiration time: 2027-05-19 06:06:23 UTC (creation time + 11months 30days 9h 39m 11s)
        Key flags: signing

           Subkey: B6C7078ADBC3ADB84C4EAB1B511490D00D8ECE2B3D28F75FCC3D7DAEDFFC3DD5
  Public-key algo: Ed25519
  Public-key size: 256 bits
    Creation time: 2026-05-19 00:17:59 UTC
  Expiration time: 2027-05-19 06:06:46 UTC (creation time + 11months 30days 9h 39m 11s)
        Key flags: authentication

           UserID: <usuario@ejemplo.org>

           UserID: Nombre Apellido

           UserID: villonaco
```

Se recalca que los datos personalizados (fingerprint, fechas de creación y expiración, UserID, etc.) van a diferir de lo mostrado en este manual. Todas las subkeys están como `secret` en el keystore local de Tails. Eso es correcto en `${OFFLINE_HOST}`. En `${WORK_HOST}`, después de la importación selectiva de 7.3, solo las subkeys deben estar como `secret`; la primaria debe aparecer como `public` (stub).

Registrar las huellas de cada subkey:

```bash
sq inspect --cert "${KEY_FP}" | awk '/Subkey:/ {print $2}' > /mnt/airgap/keys/SUBKEY_FINGERPRINTS.txt
```

### 7. Exportación de subkeys e importación en `${WORK_HOST}` sin material primario

#### 7.1 Export selectivo: solo subkeys secretas, no la primaria

Para identificar cuál fingerprint corresponde a qué capacidad, usar `sq inspect` con grep filtrando subkeys y sus flags. Cada subkey aparece seguida en la línea siguiente por su `Key flags`:

```bash
sq inspect --cert "${KEY_FP}" | grep -E "Subkey:|Key flags:"
```

Salida esperada:

```
        Key flags: certification
           Subkey: 0EDCC3CD931C718819BA0DA7ECCB99F1E6B2B8FA1BEDF9B06D1282AAC71444F5
        Key flags: transport encryption, data-at-rest encryption
           Subkey: A62601755871313C79082742025A14621C7A592AFDDCF4444A0FA6B7068842E5
        Key flags: signing
           Subkey: B6C7078ADBC3ADB84C4EAB1B511490D00D8ECE2B3D28F75FCC3D7DAEDFFC3DD5
        Key flags: authentication
```

> **Importante**: `sq key list` (la vista resumida que ya se usó arriba) etiqueta tanto las subkeys S como las A como `usable for signing`, porque la autenticación es criptográficamente "firmar un desafío". Para distinguirlas con certeza, **usar siempre `sq inspect`**, que muestra los flags reales del paquete OpenPGP. Esta salvedad aplica a toda verificación de capacidades en el resto del manual.

Que no confunda la primera Key flag listada que corresponde a la llave primaria; luego viene el fingerprint de la subkey (0EDCC3CD931C718819BA0DA7ECCB99F1E6B2B8FA1BEDF9B06D1282AAC71444F5), luego la Key flag correspondiente (transport encryption, data-at-rest encryption), y así mismo con las dos últimas subkeys.

`sq key subkey export` extrae el material secreto de una subkey específica sin tocar la primaria. Hay que hacerlo para cada subkey, identificándolas por su huella:

```bash
# Huellas tomadas del paso 6.5
export SUB_S_FP="A62601755871313C79082742025A14621C7A592AFDDCF4444A0FA6B7068842E5"   # huella de la subkey S (signing)
export SUB_A_FP="B6C7078ADBC3ADB84C4EAB1B511490D00D8ECE2B3D28F75FCC3D7DAEDFFC3DD5"   # huella de la subkey A (authentication)
export SUB_E_FP="0EDCC3CD931C718819BA0DA7ECCB99F1E6B2B8FA1BEDF9B06D1282AAC71444F5"   # huella de la subkey de cifrado T+R (transport encryption, data-at-rest encryption)

sq key subkey export --cert "${KEY_FP}" --key "${SUB_S_FP}" \
   --output /mnt/airgap/exports/${KEY_FP}.subkey-S.sec.asc

sq key subkey export --cert "${KEY_FP}" --key "${SUB_A_FP}" \
   --output /mnt/airgap/exports/${KEY_FP}.subkey-A.sec.asc

sq key subkey export --cert "${KEY_FP}" --key "${SUB_E_FP}" \
   --output /mnt/airgap/exports/${KEY_FP}.subkey-E.sec.asc
```

Cada archivo contiene solo el material secreto de su subkey y el certificado público con los stubs de las otras claves. La primaria secreta no aparece en ninguno.

Confirmar antes de seguir:

```bash
sq inspect /mnt/airgap/exports/${KEY_FP}.subkey-S.sec.asc | grep -i "secret"
```

Debe mostrar `Secret key: Unencrypted` o `Secret key: Encrypted` **solo para la subkey S**. Las otras (primaria y subkeys A, E) deben aparecer sin material secreto.

**Persistir la llave completa actualizada en `${AIRGAP_MEDIA}`**

El archivo `${KEY_OUT}` generado en 4.2 contiene solo la primaria con capacidad C, fue creado antes de que existieran las subkeys. Las subkeys S, A y T+R creadas en 6.2-6.4 viven actualmente en el keystore de la sesión Tails (`~/.local/share/sequoia/keystore/`), que es volátil y se borra al apagar.

Antes de cerrar la sesión, hay que exportar la llave completa actualizada al medio airgapped para que sobreviva. De lo contrario, la próxima sesión offline traerá solo la primaria desnuda y cualquier operación que requiera las subkeys (rotación, revocación granular, renovación de subkeys, re-export selectivo) fallará.

```bash
sq key export --cert "${KEY_FP}" \
   --output /mnt/airgap/keys/primary-and-subkeys.key
```

Este comando pide la passphrase de cada llave secreta que exporta: la de la primaria y la de las subkeys. El archivo de salida contiene los cuatro materiales secretos cifrados con sus respectivas passphrases.

Verificar el contenido del archivo recién creado:

```bash
sq inspect /mnt/airgap/keys/primary-and-subkeys.key | grep -E "Fingerprint:|Key flags:|Secret key:"
```

Debe mostrar el fingerprint de la primaria con `Key flags: certification` y las tres subkeys con sus flags respectivos (`signing`, `authentication`, `transport encryption, data-at-rest encryption`), todas con `Secret key: Encrypted`.

Opcional pero recomendado: conservar también el archivo original `${KEY_OUT}` (`primary.key`) como referencia histórica de "primaria desnuda recién generada". Ocupa pocos KB y permite reconstruir el estado del despliegue por etapas si en algún momento hace falta auditarlo.

A partir de este punto, el archivo canónico de la llave en `${AIRGAP_MEDIA}` es `primary-and-subkeys.key`, no `primary.key`. Toda referencia futura a `${KEY_OUT}` en operaciones de mantenimiento (Etapa IV) apunta al primero.

Actualizar la variable para el resto del despliegue:

```bash
export KEY_OUT="/mnt/airgap/keys/primary-and-subkeys.key"
```

#### 7.2 Transporte físico a `${WORK_HOST}`

Los tres archivos `.sec.asc` y el certificado público `${KEY_FP}.pub.asc` que estan dentro de la carpeta `exports` se copian a un segundo USB de transporte, no en `${AIRGAP_MEDIA}` — ese no sale de la sesión Tails. El USB de transporte puede o no estar cifrado; se va a borrar después de importar.

```bash
# Asumiendo el USB de transporte montado en /mnt/transport
cp /mnt/airgap/exports/${KEY_FP}.pub.asc           /mnt/transport/
cp /mnt/airgap/exports/${KEY_FP}.subkey-S.sec.asc  /mnt/transport/
cp /mnt/airgap/exports/${KEY_FP}.subkey-A.sec.asc  /mnt/transport/
cp /mnt/airgap/exports/${KEY_FP}.subkey-E.sec.asc  /mnt/transport/
sync
sudo umount /mnt/transport
```

Antes de cerrar la sesión Tails: desmontar `${AIRGAP_MEDIA}` y guardarlo. La llave primaria queda offline.

```bash
sudo umount /mnt/airgap
sudo cryptsetup close airgap-media
```

Apagar Tails (`shutdown now`). En este punto, la llave primaria existe únicamente en `${AIRGAP_MEDIA}`, físicamente guardado.

#### 7.3 Import en `${WORK_HOST}`

Arrancar `${WORK_HOST}` y verificar si está instalado `cryptsetup`:

```bash
sudo which cryptsetup
sudo cryptsetup --version

# En caso que no esté instalado, instalarlo con:
sudo apt update
sudo apt install -y cryptsetup
```

Montar el USB de transporte:

```bash
lsblk -o NAME,SIZE,MODEL,VENDOR,TRAN,MOUNTPOINTS # Para identificar el dispositivo
export MIUSB=/dev/sdb    # o /dev/sda según corresponda
sudo cryptsetup open $MIUSB transport
sudo mkdir -p /mnt/transport
sudo mount /dev/mapper/transport /mnt/transport
```

E importar primero el certificado público y después cada subkey secreta:

```bash
export KEY_FP="F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8"

sq cert import /mnt/transport/*.pub.asc
sq key import  /mnt/transport/${KEY_FP}.subkey-S.sec.asc
sq key import  /mnt/transport/${KEY_FP}.subkey-A.sec.asc
sq key import  /mnt/transport/${KEY_FP}.subkey-E.sec.asc
```

Finalmente, se puede verificar si las llaves fueron correctamente importadas:

```bash
sq key list
```

**Marcar la llave como propia en el cert store de `${WORK_HOST}`**

`sq key generate --own-key` aplicado en `${OFFLINE_HOST}` (sección 4.2) marcó la llave como propia solo en el cert store de Tails, que es amnésico. En `${WORK_HOST}` la llave llegó vía `sq cert import` (paso 7.3), que importa el certificado sin marcarlo como propio. Esto hace que `sq key list` muestre los UserIDs como `UNAUTHENTICATED`.

Funcionalmente la llave opera sin problemas (firma, cifra, descifra), pero las operaciones que dependen de autenticación (cifrar para el propio correo con `--for-email`, resolución por correo, verificación con `--signer-email`) emitirán advertencias. Para resolverlo, replicar el efecto de `--own-key` con `sq pki link authorize`:

```bash
sq pki link authorize --cert "${KEY_FP}" --all --unconstrained --depth 255
```

Significado de los flags:

- `--all` — autentica todos los UserIDs self-signed de la llave.
- `--unconstrained` — sin restricción de dominio: la llave puede certificar cualquier UserID. Necesario porque `sq pki link authorize` exige siempre un constraint de alcance (`--unconstrained`, `--domain`, o `--regex`); para una llave propia, `--unconstrained` es el equivalente al "sin restricción" que `--own-key` aplica.
- `--depth 255` — meta-introductor ilimitado: las certificaciones que tu llave emita se propagan a través de toda la cadena posible. Es lo que hace `--own-key` por defecto.

sq emitirá una línea `certification created` por cada UserID. Verificar:

```bash
sq key list --cert "${KEY_FP}"
```

Los UserIDs deben aparecer ahora con `(authenticated)` en lugar de `(UNAUTHENTICATED)`.

Nota sobre `sq pki link add --ca`: documentación antigua de Sequoia menciona un flag `--ca` para `sq pki link add` que combina "autenticar + tratar como CA" en un solo comando. Ese flag no existe en sq 1.3.1: la funcionalidad CA se expresa exclusivamente con `sq pki link authorize`. Si encuentras tutoriales o blogs que usan `sq pki link add --cert ... --ca '*'`, son de una versión anterior, el equivalente actual es el comando `link authorize` documentado arriba.

Este paso es específico de `${WORK_HOST}`. En `${OFFLINE_HOST}`, cada vez que se monta una sesión Tails nueva e importa la primaria desde `${AIRGAP_MEDIA}` con `sq key import` (sección 8.6), sq aplica equivalente a `--own-key` automáticamente al detectar que el archivo importado contiene material secreto completo. Es solo el caso `sq cert import` en `${WORK_HOST}` (que solo trae el cert público + stubs) el que requiere este `sq pki link authorize` adicional manual.

#### 7.4 Borrado seguro del USB de transporte

Una vez confirmado el import en 7.3, el USB de transporte ha cumplido su función:

```bash
sudo umount /mnt/transport
sudo cryptsetup close transport

# Borrado superficial: rellenar el dispositivo con ceros antes de reutilizarlo
sudo dd if=/dev/zero of=${MIUSB} bs=1M status=progress
```

El comando es destructivo: borra todo el contenido del USB. Después puede reformatearse para uso normal o conservarse exclusivamente para futuros transportes de subkeys.

### 8. Verificación del estado "primary key stub" en el equipo de trabajo

Este es el paso más importante de la Etapa III: confirmar que `${WORK_HOST}` tiene las subkeys secretas pero **no** tiene la primaria secreta. Si la primaria secreta apareciera en `${WORK_HOST}`, todo el modelo offline pierde sentido.

#### 8.1 Listado e interpretación

Dos comandos complementarios. El primero da la vista resumida:

```bash
sq key list --cert "${KEY_FP}"
```

Salida esperadaa en `${WORK_HOST}`, los fingerprints serán los del despliegue real:

```
 - F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8
   - user IDs:
     - <usuario@ejemplo.org> (UNAUTHENTICATED)
     - Nombre Apellido (UNAUTHENTICATED)
     - villonaco (UNAUTHENTICATED)
   - created 2026-05-19 00:11:25 UTC
   - will expire 2029-05-18T17:37:46Z

   - 0EDCC3CD931C718819BA0DA7ECCB99F1E6B2B8FA1BEDF9B06D1282AAC71444F5
     - created 2026-05-19 00:18:29 UTC
     - will expire 2027-05-19T06:07:16Z
     - usable for decryption
     - @softkeys/F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8: available, locked
   - A62601755871313C79082742025A14621C7A592AFDDCF4444A0FA6B7068842E5
     - created 2026-05-19 00:17:36 UTC
     - will expire 2027-05-19T06:06:23Z
     - usable for signing
     - @softkeys/F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8: available, locked
   - B6C7078ADBC3ADB84C4EAB1B511490D00D8ECE2B3D28F75FCC3D7DAEDFFC3DD5
     - created 2026-05-19 00:17:59 UTC
     - will expire 2027-05-19T06:06:46Z
     - usable for signing
     - @softkeys/F35E7B20D89BD6DE8DB7DCF4CE1354D98EFAB0FCD005F979739E98620BE941D8: available, locked
```

Diferencia crítica respecto al listado en `${OFFLINE_HOST}` (sección 6.5): las primeras líneas de información de la llave primaria **no** muestran `usable for signing`. Esto significa que `${WORK_HOST}` solo tiene el certificado de la primaria (la parte pública), no el material secreto. Las subkeys, en cambio, sí muestran `usable for decryption` o `usable for signing`. Si las primeras líneas de la primaria muestran `usable for signing`, algo se hizo mal: probablemente se importó accidentalmente la llave completa en lugar de hacer export selectivo de subkeys. En ese caso, ver Anexo B.1.

> **Advertencia importante sobre `sq key list`.** Esta vista resumida etiqueta tanto las subkeys con capacidad S (Signing) como las que tienen capacidad A (Authentication) como `usable for signing`. La razón es criptográfica: la autenticación SSH consiste en firmar un desafío, así que ambas capacidades son "usables for signing" desde el punto de vista de sq. **Esto no permite distinguir cuál es S y cuál es A**. Para verificación de capacidades reales, usar `sq inspect` como se indica abajo.

Para verificar la distribución correcta de capacidades (una subkey S, una A, una E), usar:

```bash
sq inspect --cert "${KEY_FP}" | grep -E "Subkey:|Key flags:"
```

Salida esperada:

```
        Key flags: certification                                  # ← primaria, OK
           Subkey: 0EDC...
        Key flags: transport encryption, data-at-rest encryption  # ← subkey T+R
           Subkey: A626...
        Key flags: signing                                        # ← subkey S
           Subkey: B6C7...
        Key flags: authentication                                 # ← subkey A
```

Confirmar:

- Una primaria con flag `certification`.
- Una subkey con `transport encryption, data-at-rest encryption` (la T+R, capacidad universal).
- Una subkey con `signing` (la S).
- Una subkey con `authentication` (la A).

Si falta alguna o aparece duplicada, revisar la sección 6: probablemente se ejecutó dos veces el comando para una capacidad y nunca para otra. La remediación es volver a `${OFFLINE_HOST}` para añadir la que falta (sección 9.2 con `--can-authenticate`, etc.) o revocar la duplicada (sección 11.1).

#### 8.2 Prueba de firma con la subkey S

```bash
echo "prueba de firma desde ${HOSTNAME}" | sq sign --cleartext --signer "${KEY_FP}" -
```

Se pide la passphrase de la subkey de firma (S), no la de la primaria, que ni siquiera está disponible. Si la firma se genera, la subkey S funciona.

#### 8.3 Prueba de cifrado y descifrado con la subkey T+R

```bash
echo "mensaje en claro" | sq encrypt --for "${KEY_FP}" --signer "${KEY_FP}" - > /tmp/test.pgp
# Pedirá la passphrase de la subkey S

sq decrypt /tmp/test.pgp
# Pedirá la passphrase de la subkey T+R
```

Devuelve "mensaje en claro". Si funciona, la subkey T+R está bien.

#### 8.4 Prueba de intento de certificación (debe fallar)

Este es el test negativo: confirma que la primaria no está disponible en `${WORK_HOST}`. Intentar añadir una nueva subkey debe fallar:

```bash
sq key subkey add --cert "${KEY_FP}" --can-sign --expiration 1y
```

Salida esperada:

```
  Error: F35E... was not considered because
         it is: missing the secret key
because: Found no suitable key on F35E...
```

El mensaje exacto puede variar según la versión, pero el sentido es el mismo: sq no puede firmar el binding de una nueva subkey porque la primaria está como stub público.

Este error es el comportamiento deseado. Significa que cualquier operación que requiera la primaria -añadir subkeys, añadir UserIDs, extender expiraciones, revocar- exige volver a `${OFFLINE_HOST}`, lo cual obliga al procedimiento controlado de las Etapas II y IV.

#### 8.5 Checklist de cierre de Etapa III

- [ ] `sq key list` en `${OFFLINE_HOST}` muestra primaria + 3 subkeys, todas como `secret`
- [ ] `sq key list` en `${WORK_HOST}` muestra primaria como `public` y subkeys S, A, T+R como `secret`
- [ ] Prueba de firma con subkey S en `${WORK_HOST}` exitosa
- [ ] Prueba de cifrado/descifrado con subkey S y T+R en `${WORK_HOST}` exitosa
- [ ] Intento de `sq key subkey add` en `${WORK_HOST}` falla con "missing the secret key"
- [ ] USB de transporte borrado con `dd if=/dev/zero`
- [ ] `${AIRGAP_MEDIA}` desmontado y guardado físicamente

#### 8.6 Preparación de una sesión offline para operaciones de mantenimiento

Cada operación de la Etapa IV que toca la primaria -añadir UserIDs (9.2, 9.4, 9.5), renovar expiraciones (10.2, 10.3), revocar componentes (11.1, 11.2, 11.3), rotar subkeys (12.2)- se ejecuta en `${OFFLINE_HOST}`. Cada sesión Tails arranca con el keystore de Sequoia vacío (Persistent Storage conserva los paquetes instalados, no el estado de `~/.local/share/sequoia/`), por lo que la primaria debe re-importarse desde `${AIRGAP_MEDIA}` antes de cualquier operación.

Procedimiento de apertura de sesión offline:

1. Arrancar Tails desde el USB, desbloquear Persistent Storage, no habilitar redes (sección 3.1).
2. Aplicar las medidas de aislamiento de red por software (`rfkill block all`, apagar interfaces, detener NetworkManager — sección 3.1).
3. Confirmar la hora del sistema. Si está desfasada más de unos minutos, ajustar con `sudo date -s` antes de operar.
4. Montar `${AIRGAP_MEDIA}`:

   ```bash
   lsblk -o NAME,SIZE,MODEL,VENDOR,TRAN,MOUNTPOINTS # Para identificar el dispositivo
   export MIUSB=/dev/sdb                            # Reemplazar `/dev/sdb` por el dispositivo real
   sudo mkdir -p /mnt/airgap
   sudo cryptsetup open $MIUSB airgap-media
   sudo mount /dev/mapper/airgap-media /mnt/airgap
   ```

5. Re-exportar las variables del despliegue:

   ```bash
   export KEY_FP="$(cat /mnt/airgap/keys/FINGERPRINT.txt)"
   export KEY_OUT="/mnt/airgap/keys/primary-and-subkeys.key"
   ```

6. Importar la primaria al keystore local de la sesión:

   ```bash
   sq key import "${KEY_OUT}"
   ```

7. Verificar:

   ```bash
   sq key list
   sq inspect --cert "${KEY_FP}" | grep -E "Subkey:|Key flags:"
   ```

   Debe aparecer el fingerprint de la primaria con sus tres UserIDs y las tres subkeys, todas marcadas como disponibles en `@softkeys/`.

A partir de este punto la sesión está lista para ejecutar cualquier operación de la Etapa IV. Al terminar, el procedimiento de cierre es:

1. Si la operación produjo cambios en el certificado, exportar el certificado actualizado a `/mnt/airgap/exports/"$(date +'%Y-%m-%d_%H-%M')"-${USER_EMAIL/@/_at_}-${KEY_FP: -16}.pub.asc` y al USB de transporte (sección 9.3).
2. Desmontar `${AIRGAP_MEDIA}`:

   ```bash
   sudo umount /mnt/airgap
   sudo cryptsetup close airgap-media
   ```

3. Apagar Tails (`shutdown now`).

El material secreto que quedó en el keystore de Tails se borra al apagar; la sesión es amnésica por construcción; solo Persistent Storage sobrevive, y el keystore de Sequoia no está incluido en Persistent Storage en este modelo.

---

## Etapa IV — Gestión del ciclo de vida

Con la llave primaria custodiada offline y las subkeys operando en `${WORK_HOST}`, esta etapa cubre las operaciones que mantienen el material vigente a lo largo del tiempo: añadir o quitar UserIDs cuando cambia la identidad publicada, extender expiraciones antes de que las llaves caduquen, revocar componentes cuando algo se compromete, y rotar subkeys de forma planificada como práctica defensiva.

Todas las operaciones de esta etapa que tocan la llave primaria se hacen en `${OFFLINE_HOST}` (sesión Tails con `${AIRGAP_MEDIA}` montado). El patrón se repite: traer la primaria, operar, exportar el certificado actualizado, llevarlo a `${WORK_HOST}`, publicar.

### 9. UserIDs: adición, revocación, primario

Los UserIDs identifican a la persona detrás de la llave. La primaria generada en la Etapa II tiene tres: nombre real, correo y pseudónimo. A lo largo del tiempo pueden añadirse otros (un correo nuevo, un proyecto), revocarse los obsoletos (un correo de trabajo al cambiar de empleador), o cambiar cuál se considera el "primario".

#### 9.1 Listado e inspección de UserIDs

Desde `${WORK_HOST}`, sin necesidad de traer la primaria, se puede inspeccionar la lista actual de UserIDs:

```bash
sq inspect --cert "${KEY_FP}"
```

En la salida aparece, por cada UserID, una línea como:

```
           UserID: <usuario@ejemplo.org>
           UserID: Nombre Apellido
           UserID: villonaco
```
<!--
La etiqueta `[primary]` aparece junto al UserID que está marcado como primario. Si ninguno la lleva, el cliente OpenPGP elige por defecto (típicamente el primero en orden de creación).
-->

#### 9.2 Añadir un UserID nuevo (requiere primaria)

En `${OFFLINE_HOST}`, tras la preparación de sesión offline (sección 8.6), se ejcuta la operación que firma con la primaria:

```bash
# Sesión Tails, ${AIRGAP_MEDIA} montado, primaria importada en el keystore local
sq key userid add --cert "${KEY_FP}" --email "nuevo@ejemplo.org"
```

Pide la passphrase de la primaria. El nuevo UserID queda firmado y vinculado a la llave.

Para añadir un UserID de tipo libre (no correo, no nombre estructurado):

```bash
sq key userid add --cert "${KEY_FP}" --userid "ahuaca"
```

Para añadir un UserID con la forma tradicional `Nombre <correo>`:

```bash
sq key userid add --cert "${KEY_FP}" --name "Nombre2 Apellido2" --email "nuevo2@ejemplo.org"
```

#### 9.3 Propagar el certificado actualizado a `${WORK_HOST}`

El cambio quedó firmado en `${OFFLINE_HOST}`. Para que `${WORK_HOST}` y los contactos vean el UserID nuevo, hay que exportar el certificado actualizado y reimportarlo:

```bash
# En ${OFFLINE_HOST}
export USER_EMAIL="usuario@ejemplo.org"
sq cert export --cert "${KEY_FP}" \
   --output /mnt/airgap/exports/"$(date +'%Y-%m-%d_%H-%M')"-${USER_EMAIL/@/_at_}-${KEY_FP: -16}.pub.asc
```

Borrar las lláves públicas previas existentes en el directorio `exports`de `${AIRGAP_MEDIA}` en caso que hayan cumplido su función. Llevar la llave pública (`2026-05-19-usuario_at_ejemplo.org-739E98620BE941D8.pub.asc`) por USB de transporte a `${WORK_HOST}` y allí:

```bash
# En ${WORK_HOST}
sq cert import ${TRANSPORT}/*.pub.asc
```

sq detecta que es una actualización del certificado ya existente y fusiona los componentes nuevos. Verificar:

```bash
sq inspect --cert "${KEY_FP}"
```

Debe mostrar el UserID añadido. La publicación a keyservers y WKD para que los contactos lo vean se cubre en la sección 14.

#### 9.4 Revocar un UserID obsoleto

Cuando un UserID ya no aplica (correo de trabajo abandonado, proyecto cerrado, pseudónimo retirado), se revoca. Operación que requiere la primaria:

```bash
# En ${OFFLINE_HOST}
sq key userid revoke \
  --cert "${KEY_FP}" \
  --email "nuevo2@ejemplo.org" \
  --reason retired \
  --message "Correo del empleador anterior, ya no en uso"
```

Valores válidos para `--reason`:

- `retired` — el UserID se retira por decisión del titular (cambio de trabajo, cambio de identidad).
- `unspecified` — sin razón documentada.

El `--message` es texto libre y queda registrado en la firma de revocación. Se ve cuando alguien hace `sq inspect` sobre el certificado.

Después de revocar, propagar el certificado actualizado a `${WORK_HOST}` con el mismo flujo de 9.3 (export → transporte → import). El UserID seguirá apareciendo en `sq inspect`, pero marcado como revocado, y los clientes OpenPGP modernos no lo ofrecerán para nuevas operaciones.

#### 9.5 Cuál UserID se muestra como primario

OpenPGP permite marcar un UserID concreto como "primario" mediante un flag en su self-signature (`Primary User ID: true`). Los clientes OpenPGP que respetan ese flag lo usan para decidir qué UserID mostrar por defecto cuando una llave tiene varios.

**Limitación de sq 1.3.1**: el subcomando `sq key userid add` no expone un flag para fijar este atributo, y `sq key userid` solo tiene los subcomandos `add` y `revoke`. La funcionalidad equivalente al `--quick-set-primary-uid` de GnuPG está documentada por el proyecto Sequoia como pendiente de implementar en la CLI. En la práctica esto deja tres opciones para influir en qué UserID se considera primario:

**Opción A — Confiar en el orden de creación y revocación.** La mayoría de clientes OpenPGP modernos (Thunderbird, KMail, mutt con Sequoia, OpenKeychain) muestran por defecto el UserID self-signed más reciente que no esté revocado. Para "promover" un correo nuevo a primario de facto:

1. Añadir el correo nuevo con `sq key userid add --email "..."` (sección 9.2).
2. Revocar los UserIDs viejos que ya no aplican con `sq key userid revoke --email "..."` (sección 9.4).

Tras la propagación a `${WORK_HOST}` y a keyservers, los contactos verán el correo nuevo como el visible por defecto.

**Opción B — Re-añadir el UserID para que su self-signature sea la más reciente.** Si el UserID que se quiere como primario ya existe y no se quieren revocar los demás, ejecutar `sq key userid add` con los mismos parámetros de creación emite una nueva self-signature con timestamp actual. La self-signature anterior queda, pero la nueva es la que la mayoría de clientes leerá:

```bash
sq key userid add --cert "${KEY_FP}" --email "nuevo@ejemplo.org"
```

Si sq advierte que el UserID ya existe, conviene revisar si la operación es realmente lo que se quería. El comportamiento exacto depende de la versión: algunas versiones de sq fallan, otras emiten silenciosamente una nueva self-signature.

**Opción C — Manipulación externa con GnuPG si se necesita el flag explícito.** Si por algún motivo (cliente OpenPGP exigente, integración con sistema externo) se necesita que la self-signature lleve `Primary User ID: true` de forma garantizada, el camino actual pasa por GnuPG:

1. En `${OFFLINE_HOST}`, exportar la llave secreta completa con `sq key export --cert "${KEY_FP}" --output /tmp/full.asc`.
2. Importarla a GnuPG: `gpg --import /tmp/full.asc`.
3. Editar con `gpg --edit-key "${KEY_FP}"`, seleccionar el UserID con `uid N`, marcar con `primary`, guardar con `save`.
4. Re-exportar de GnuPG: `gpg --export-secret-keys --armor "${KEY_FP}" > /tmp/full-updated.asc`.
5. Re-importar a sq tras borrar la versión anterior del keystore: `sq key delete --cert "${KEY_FP}" && sq key import /tmp/full-updated.asc`.
6. Limpiar GnuPG: `gpg --delete-secret-and-public-keys "${KEY_FP}"`.

Es un viaje de ida y vuelta incómodo. Solo vale la pena si la opción A o B no son suficientes para el caso de uso concreto.

Recomendación práctica para este manual: usar Opción A como modo normal. Mantener una identidad por correo activo, revocar los obsoletos. El concepto de "UserID primario" pierde relevancia cuando el cert tiene un único correo no revocado en cada momento.

Después de cualquiera de las opciones que produzca cambios, propagar el certificado actualizado a `${WORK_HOST}` y a los keyservers con el flujo de 9.3.

### 10. Renovación de expiración (primaria y subkeys) desde el entorno offline

Por la política decidida en 2.3, la primaria expira a los 3 años y las subkeys al año. Conviene renovar antes de la fecha real, no después: una llave expirada deja de validar firmas y descifrar (técnicamente las firmas viejas siguen siendo válidas, pero los clientes pueden marcarlas como sospechosas).

#### 10.1 Diagnóstico de proximidad de vencimiento

Desde `${WORK_HOST}`, sin traer la primaria, se ve la fecha de expiración:

```bash
sq inspect --cert "${KEY_FP}" | grep -i "expiration"
```

Salida típica:

```
  Expiration time: 2029-05-18 17:37:46 UTC (creation time + 2years 11months 30days 9h 16m 45s)
  Expiration time: 2027-05-19 06:07:16 UTC (creation time + 11months 30days 9h 39m 11s)
  Expiration time: 2027-05-19 06:06:23 UTC (creation time + 11months 30days 9h 39m 11s)
  Expiration time: 2027-05-19 06:06:46 UTC (creation time + 11months 30days 9h 39m 11s)
```

Hay una entrada por la primaria y una por cada subkey con expiración propia. Cuando una expiración esté a menos de 30 días, planificar una sesión en `${OFFLINE_HOST}`.

#### 10.2 Renovar la expiración de la primaria

En `${OFFLINE_HOST}`, tras la preparación de sesión offline (sección 8.6), con `${AIRGAP_MEDIA}` montado y la primaria en el keystore local:

```bash
sq key expire --cert "${KEY_FP}" --expiration 3y
```

Esto re-firma la primaria con una expiración nueva contando desde el momento actual (3 años a partir de hoy). Pide la passphrase de la primaria.

Si en su momento la primaria se generó con `--expiration never`, `sq key expire` también puede establecer una expiración por primera vez. Lo inverso (pasar de fechada a `never`) técnicamente se hace con `--expiration never`, pero rompe la política de este manual y no se recomienda.

#### 10.3 Renovar la expiración de una subkey

Por defecto las subkeys heredan la expiración de la primaria si no se especificó otra al crearlas. Como en este manual se especificó `--expiration 1y` para cada subkey en la Etapa III, cada una tiene su propia fecha y se renueva por separado:

```bash
sq key subkey expire \
  --cert "${KEY_FP}" \
  --key "${SUB_S_FP}" \
  --expiration 1y
```

Repetir para `${SUB_A_FP}` y `${SUB_E_FP}`. Cada operación pide la passphrase de la primaria (es la que firma el binding).

> Alternativa estratégica: en lugar de renovar las subkeys, generar subkeys nuevas y revocar las viejas. Eso se cubre como rotación en la sección 12. Renovar es más simple; rotar es más higiénico desde el punto de vista de exposición criptográfica. La práctica recomendada es **rotar** subkeys al menos cada 2-3 ciclos de renovación, no renovar indefinidamente.

#### 10.4 Propagación post-renovación

El export y el import son los mismos que en 9.3:

```bash
# En ${OFFLINE_HOST}
export USER_EMAIL="usuario@ejemplo.org"
sq cert export --cert "${KEY_FP}" \
   --output /mnt/airgap/exports/"$(date +'%Y-%m-%d_%H-%M')"-${USER_EMAIL/@/_at_}-${KEY_FP: -16}.pub.asc

# Borrar las llaves públicas anteriores en caso de que existan y de que hayan cumplido su función

# Llevar a ${WORK_HOST} por USB de transporte

# En ${WORK_HOST}
sq cert import ${TRANSPORT}/${KEY_FP}.pub.asc
```

Verificar con `sq inspect` que las nuevas fechas de expiración aparezcan tanto en la primaria como en las subkeys.

Después, re-publicar el certificado en keyservers y WKD (sección 15.4) para que los contactos reciban la actualización antes de que la versión vieja les caduque.

### 11. Revocación: subkey comprometida, UserID obsoleto, llave completa

Revocar es declarar firmadamente que un componente (UserID, subkey, o la llave entera) ya no debe usarse. Es irreversible: una vez publicada la revocación, no se "des-revoca". Por eso conviene tener claro el alcance antes de ejecutar.

#### 11.1 Revocar una subkey comprometida

Caso típico: una laptop con las subkeys es robada o se sospecha que la subkey de firma fue copiada. Tras la preparación de sesión offline (sección 8.6), la operación se hace en `${OFFLINE_HOST}` porque la firma de revocación viene de la primaria:

```bash
sq key subkey revoke \
  --cert "${KEY_FP}" \
  --key "${SUB_S_FP}" \
  --reason compromised \
  --message "Subkey de firma posiblemente comprometida tras robo de laptop"
```

Valores válidos para `--reason` en revocación de subkey:

- `compromised` — la subkey secreta puede estar en manos no autorizadas. Esto es lo que se usa tras un incidente.
- `superseded` — la subkey se reemplaza por otra (uso normal durante rotación, no implica compromiso).
- `retired` — la subkey deja de usarse sin reemplazo.
- `unspecified` — sin razón documentada.

La diferencia entre `compromised` y `superseded` es importante para terceros: una revocación `compromised` invalida también las firmas pasadas hechas con esa subkey (porque pueden haber sido falsificadas), mientras que `superseded` o `retired` mantienen las firmas viejas como válidas.

#### 11.2 Revocar la llave completa

Caso extremo: la primaria misma puede estar comprometida, o se decide retirar la identidad por completo. Hay dos formas, según cuánto material esté disponible.

**Caso A — la primaria está accesible** (en `${OFFLINE_HOST}`, con la passphrase):

```bash
sq key revoke \
  --cert "${KEY_FP}" \
  --reason compromised \
  --message "Compromiso confirmado tras incidente <fecha>"
```

Esto firma una revocación nueva con la primaria y la importa automáticamente en el keystore local. Después se exporta el certificado y se publica como cualquier actualización.

> Si se omite `--output`, sq aplica la revocación al cert store local inmediatamente. Si se pasa `--output FILE`, sq escribe el certificado de revocación en FILE sin aplicarlo, lo cual permite revisarlo antes de propagarlo. Para revocación de emergencia conviene omitir `--output`; para generar un cert de revocación de respaldo (uso "preventivo"), conviene escribir a archivo.

**Caso B — la primaria no está accesible** (medio perdido, passphrase olvidada, incendio): aquí entra en juego el certificado de revocación pre-generado y custodiado en la sección 5. Es el escenario para el que existe:

```bash
# En cualquier máquina con sq instalado
sq cert import /ruta/al/medio-de-revocacion/primary.rev
```

`sq cert import` reconoce que es un certificado de revocación y aplica la revocación al certificado que tenía en el store. Si no había certificado previo (porque se importa en una máquina que nunca vio la llave), igual lo importa marcado como revocado.

Tras revocar la llave completa, publicar urgentemente (sección 15.4). El daño que la revocación evita es proporcional a la velocidad con que llega a los contactos y keyservers.

#### 11.3 Reemisión de subkeys tras un compromiso parcial

Si solo se compromete una subkey (no la primaria), tras revocar esa subkey conviene generar una nueva con la misma capacidad. Es el mismo procedimiento que en la Etapa III (sección 6) para esa capacidad concreta:

```bash
# En ${OFFLINE_HOST}
sq key subkey add --cert "${KEY_FP}" --can-sign --expiration 1y
```

Exportar la nueva subkey secreta, llevarla a `${WORK_HOST}`, importarla. Las firmas que se hagan a partir de ese momento usan la subkey nueva. La subkey vieja, revocada, queda en el certificado pero ya no se usa.

#### 11.4 Comunicación a contactos

La revocación es solo efectiva si llega. Acciones recomendadas tras una revocación significativa (subkey comprometida o llave completa):

- Re-publicar el certificado actualizado en todos los keyservers donde estaba (sección 15.4).
- Actualizar el WKD propio si se mantiene uno.
- Avisar por canales secundarios (correo firmado con una subkey nueva, mensaje en redes, pin a un commit firmado en GitHub) que hubo un evento de revocación, con el fingerprint y la razón. Los contactos que verifiquen firmas viejas con la subkey revocada por `compromised` verán advertencias; entender por qué les ayuda a actuar.

### 12. Rotación periódica de subkey de firma

La rotación es distinta de la renovación. **Renovar** una subkey extiende su validez con la misma clave criptográfica. **Rotar** una subkey la reemplaza por otra completamente nueva, con material criptográfico nuevo. La rotación periódica es práctica defensiva: limita el período durante el cual una eventual fuga de clave secreta tiene impacto.

#### 12.1 Cuándo rotar

Política sugerida para este manual:

- **Subkey de firma (S)**: rotar cada año, alineado con su expiración. No renovar más de una vez la misma S; al segundo ciclo, rotar.
- **Subkey de autenticación (A)**: rotar cada 2 años. SSH es más sensible a cambios (hay que actualizar `authorized_keys` en todos los servidores), conviene espaciar.
- **Subkey de cifrado (E)**: rotar cada 2-3 años. Aquí la fricción es que el material cifrado con la subkey vieja sigue requiriendo esa subkey para descifrarse; no se borra inmediatamente.

Rotar antes (no después) de un evento de mayor exposición -viajes, sesiones de pentest contra el equipo, cambios de hardware- es buena práctica.

#### 12.2 Procedimiento de rotación

Es esencialmente: añadir subkey nueva, exportarla, revocar la vieja, propagar.

En `${OFFLINE_HOST}`:

```bash
# Añadir subkey nueva con la misma capacidad
sq key subkey add --cert "${KEY_FP}" --can-sign --expiration 1y

# Listar subkeys y registrar la huella de la nueva (será la más reciente)
sq inspect --cert "${KEY_FP}"
export SUB_S_NEW_FP="..."   # huella de la subkey S recién creada
export SUB_S_OLD_FP="..."   # huella de la subkey S anterior

# Exportar la nueva subkey secreta para ${WORK_HOST}
sq key subkey export --cert "${KEY_FP}" --key "${SUB_S_NEW_FP}" \
   --output /mnt/airgap/exports/${KEY_FP}.subkey-S-new.sec.asc
```

#### 12.3 Solapamiento controlado

Hay dos formas de hacer la transición:

**Modo solapado** (recomendado para la subkey S): la subkey vieja sigue válida hasta su expiración natural, y la nueva se empieza a usar inmediatamente. Durante el solapamiento, los contactos verán dos subkeys S firmando — ambas válidas. No revocar la vieja todavía.

**Modo abrupto** (recomendado para compromisos): revocar la subkey vieja al mismo tiempo que se añade la nueva. Solo se usa cuando hay sospecha de compromiso (caso 11.1, no rotación de rutina).

Para rotación de rutina, modo solapado:

```bash
# Llevar la subkey nueva a ${WORK_HOST} por USB de transporte e importarla

# En ${WORK_HOST}
sq key import ${TRANSPORT}/${KEY_FP}.subkey-S-new.sec.asc

# A partir de aquí, las firmas las hace por defecto sq con la subkey más reciente
# capaz de firmar; la vieja queda disponible pero secundaria.
```

Cuando la subkey vieja se acerca a su expiración natural, revocarla con razón `superseded`:

```bash
# En ${OFFLINE_HOST}, al final del período de solapamiento
sq key subkey revoke \
  --cert "${KEY_FP}" \
  --key "${SUB_S_OLD_FP}" \
  --reason superseded \
  --message "Reemplazada por rotación programada"
```

Propagar el certificado actualizado a `${WORK_HOST}` y republicar (sección 15.4).

#### 12.4 Particularidad de la subkey de cifrado

La subkey T+R no se puede simplemente reemplazar y revocar la vieja, porque cualquier archivo cifrado con la subkey vieja necesitará esa subkey para descifrarse en el futuro. Práctica recomendada:

- Añadir la nueva subkey T+R (los remitentes empezarán a cifrar contra la más reciente).
- Mantener la subkey vieja **sin revocar** hasta confirmar que no quedan archivos cifrados pendientes de descifrar contra ella. En la práctica esto suele significar nunca revocarla; solo dejarla expirar.

Si la subkey T+R vieja se compromete (no es rotación de rutina sino respuesta a incidente), se revoca con `compromised` y se asume la pérdida de poder descifrar material viejo.

### 13. Checklist de cierre de Etapa IV

- [ ] Procedimientos de UserID, expiración, revocación y rotación documentados en bitácora del despliegue
- [ ] Calendario de renovación/rotación establecido (recordatorio 30 días antes de cada vencimiento)
- [ ] Ubicación física del medio airgapped y del cert de revocación verificada
- [ ] Al menos una operación de ciclo de vida ejecutada y propagada con éxito (típicamente: añadir un UserID secundario como prueba) para validar el flujo offline→online completo
- [ ] Política clara sobre razones de revocación: `compromised` solo tras incidente, `superseded` para rotaciones, `retired` para retiro sin reemplazo

---

## Etapa V — Operación

Con la llave generada, las subkeys en producción y el ciclo de vida documentado, esta etapa cubre las operaciones cotidianas: respaldar de forma que un incidente no sea catastrófico, publicar el certificado para que los contactos puedan encontrarlo, integrar la llave con herramientas que esperan GnuPG, y usar la subkey de autenticación para SSH.

### 14. Backup cifrado y procedimiento de restauración

El backup en este modelo no es de la primaria que ya vive en `${AIRGAP_MEDIA}`, ese es el almacén canónico. El backup es la copia de respaldo del material completo -primaria y subkeys- en caso de que `${AIRGAP_MEDIA}` falle físicamente.

#### 14.1 Qué se respalda, dónde vive cada cosa

El modelo de custodia que sigue este manual usa tres ubicaciones físicas distintas, con propósitos no superpuestos:

- **`${AIRGAP_MEDIA}`** — almacén operativo de la primaria. Es el medio que se monta en `${OFFLINE_HOST}` cuando hay que firmar subkeys, añadir UserIDs, renovar o revocar. Vive cerca del usuario, accesible.
- **`${BACKUP_DIR}`** — backup en frío del material secreto completo (primaria + subkeys), en un segundo medio cifrado, guardado en una ubicación física distinta (caja fuerte, casa de familiar, depósito bancario). Se actualiza solo cuando cambia algo significativo (nuevas subkeys, renovación, rotación).
- **Medio de revocación** — el segundo USB con el cert de revocación pre-generado (sección 5.2). Tercera ubicación física, tercera passphrase. Su única función es permitir revocar si las otras dos se pierden.

Las passphrases de los tres medios son distintas y se almacenan en un gestor de contraseñaas. La de `${AIRGAP_MEDIA}` se memoriza (se usa con frecuencia). Las otras dos se custodian por separado (sobre sellado, password manager con custodia compartida, o fragmentación tipo Shamir).

#### 14.2 Crear el backup en frío

Se hace en `${OFFLINE_HOST}` cuando hay cambios significativos. Asumiendo el segundo USB LUKS preparado y montado en `/mnt/backup`, y tras la preparación de sesión offline (sección 8.6)

```bash
# En ${OFFLINE_HOST}, con ${AIRGAP_MEDIA} montado en /mnt/airgap

# Exportar la llave completa (primaria + todas las subkeys secretas)
sq key export --cert "${KEY_FP}" \
   --output /tmp/full-export.asc

# Copiar a ${BACKUP_DIR}
mkdir -p /mnt/backup/${KEY_FP}/
cp /tmp/full-export.asc /mnt/backup/${KEY_FP}/full-export-$(date +%Y%m%d).asc

# También copiar los archivos de referencia
cp /mnt/airgap/keys/FINGERPRINT.txt        /mnt/backup/${KEY_FP}/
cp /mnt/airgap/keys/SUBKEY_FINGERPRINTS.txt /mnt/backup/${KEY_FP}/

# Borrado del archivo intermedio en /tmp (Tails lo borra al apagar, pero es buena práctica)
shred -u /tmp/full-export.asc

# Desmontar
sudo umount /mnt/backup
sudo cryptsetup close backup-media
```

El nombre del archivo incluye la fecha (`full-export-YYYYMMDD.asc`) para tener historial cuando se actualiza. No se borran las versiones anteriores: si un cambio reciente resulta ser un error (UserID añadido por equivocación, rotación mal hecha), se puede restaurar la versión anterior. Limpieza periódica manual cuando el medio se llene.

#### 14.3 Restauración de prueba en entorno aislado

La regla operativa: un backup no probado es una superstición. Periódicamente -una vez al año, o tras cada actualización significativa- restaurar el backup en un entorno aislado y verificar que la llave funciona.

Procedimiento en `${OFFLINE_HOST}` (una sesión Tails de prueba, sin tocar `${AIRGAP_MEDIA}`):

```bash
# Crear un directorio temporal aislado y apuntar SEQUOIA_HOME ahí
export SEQUOIA_HOME=/tmp/sq-restore-test
mkdir -p ${SEQUOIA_HOME}

# Montar el backup
sudo cryptsetup open /dev/sdZ backup-test
sudo mount /dev/mapper/backup-test /mnt/backup-test

# Importar en el entorno aislado
sq key import /mnt/backup-test/${KEY_FP}/full-export-YYYYMMDD.asc

# Verificar
sq key list
sq inspect --cert "${KEY_FP}"
```

`sq key list` debe mostrar la primaria como `secret`, las tres subkeys como `secret`, los tres UserIDs presentes, las fechas de expiración correctas. La passphrase pedida durante el import debe ser la que se documentó al crear la primaria.

Después del test, limpiar:

```bash
# Salir de Tails (todo lo de /tmp se borra) o limpiar explícitamente
rm -rf ${SEQUOIA_HOME}
unset SEQUOIA_HOME

sudo umount /mnt/backup-test
sudo cryptsetup close backup-test
```

`SEQUOIA_HOME` redirigió todo el estado de sq a `/tmp`, así que el keystore real de la sesión Tails permanece intacto. El test no contamina el entorno habitual.

#### 14.4 Cuándo refrescar el backup

Eventos que disparan un refresh del backup en `${BACKUP_DIR}`:

- Tras añadir o revocar un UserID (sección 9).
- Tras renovar la expiración de la primaria o de cualquier subkey (sección 10).
- Tras una rotación de subkey (sección 12).
- Tras una revocación de subkey (sección 11.1).

La operación normal de día a día en `${WORK_HOST}` (firmar commits, descifrar correos, autenticar SSH) no requiere refresh, esas operaciones no modifican el material secreto en `${OFFLINE_HOST}`.

### 15. Publicación y descubrimiento (keyservers verifying, WKD)

Publicar el certificado público sirve para dos cosas: que los contactos puedan cifrar contra la llave sin recibirla por canal directo, y que las actualizaciones (UserIDs nuevos, revocaciones, renovaciones) lleguen a quien ya tiene una copia.

#### 15.1 Keyservers verifying

Los keyservers tradicionales tipo SKS aceptaban subidas sin verificación, lo cual derivó en ataques de "certificate flooding" que envenenaron muchas llaves. Los keyservers modernos son verifying: verifican que el correo del UserID pertenezca a quien sube la llave, mediante un challenge enviado a ese correo.

El más usado hoy es `keys.openpgp.org`. Para subir desde `${WORK_HOST}`:

```bash
sq network keyserver publish --cert "${KEY_FP}"
```

Por defecto sq usa `keys.openpgp.org`. El servidor recibirá el certificado público (solo subkeys públicas + UserIDs + firmas, nada secreto) y enviará un correo a cada UserID con dirección de correo, pidiendo confirmación. Hasta que se confirme, el correo no aparece asociado al certificado en búsquedas — solo el fingerprint.

Para publicar en un keyserver alternativo:

```bash
sq network keyserver publish --cert "${KEY_FP}" --server hkps://keys.example.org
```

> El UserID pseudónimo (`${USER_ID}` tipo `villonaco`) no recibe challenge: solo los UserIDs con correo se verifican. El pseudónimo se asocia al certificado, pero no es "buscable" por keys.openpgp.org. Eso es por diseño del servidor, no limitación de sq.

#### 15.2 Web Key Directory (WKD) propio

WKD es un mecanismo donde el dominio del correo (`ejemplo.org`) publica los certificados de sus usuarios en una URL bien conocida. Es más confiable que keyservers porque la fuente de autoridad es el propio dominio del correo, no un tercero.

Si se controla el dominio de `${USER_EMAIL}` (es decir, se administra el servidor web de `ejemplo.org`), publicar el certificado por WKD es:

```bash
# Generar la estructura WKD a partir del certificado
sq network wkd publish --cert "${KEY_FP}" \
   --domain ejemplo.org \
   --output /var/www/ejemplo.org/.well-known/openpgpkey/
```

> Esto genera los archivos en la estructura `.well-known/openpgpkey/hu/<hash>` que el cliente OpenPGP de un tercero buscará automáticamente cuando intente cifrar contra `usuario@ejemplo.org`. El servidor web tiene que servir esos archivos con CORS apropiado; la documentación del proyecto WKD detalla los headers requeridos. Verificar con un cliente externo: `sq network wkd fetch usuario@ejemplo.org`.

#### 15.3 Descubrimiento de certificados de contactos

Para encontrar el certificado de un tercero:

```bash
# Por dirección de correo (intenta WKD primero, después keyserver)
sq network search contacto@otro-dominio.org

# Por fingerprint conocido
sq network search ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890

# Solo por keyserver
sq network keyserver search contacto@otro-dominio.org
```

`sq network search` es el comando recomendado: prueba múltiples fuentes (WKD del dominio, keyserver por defecto, DANE si está configurado) y devuelve lo que encuentre. Lo recuperado se importa automáticamente al cert store local.

> Los certificados traídos por `sq network search` llegan firmados internamente por una shadow CA con confianza baja (típicamente 1/120), lo que significa que `sq` los conoce pero no considera autenticado el binding UserID↔llave. Al cifrar contra ellos aparecerán advertencias de "no autenticado". La sección 18 documenta cómo autenticar manualmente esos certificados verificando el fingerprint por canal alterno, y cómo elevar la confianza en una fuente entera si se decide que es fiable.

#### 15.4 Re-publicar tras cambios

Cualquier cambio en el certificado (sección 9-12) requiere re-publicar para que llegue a los contactos:

```bash
# Tras propagar el certificado actualizado desde ${OFFLINE_HOST} a ${WORK_HOST}
sq network keyserver publish --cert "${KEY_FP}"
sq network wkd publish --cert "${KEY_FP}" --domain ejemplo.org \
   --output /var/www/ejemplo.org/.well-known/openpgpkey/
```

Para revocaciones (sección 11), la urgencia es máxima: re-publicar inmediatamente, no esperar al próximo ciclo de mantenimiento.

### 16. Integración con consumidores GnuPG (`gpg-sq` / chameleon)

Muchas herramientas en el ecosistema Linux invocan `gpg` directamente: `git commit -S`, `debsign`, `mutt`, `pass`, varios scripts de empaquetado. Estas herramientas no hablan con sq nativamente. Para que sigan funcionando con la llave Sequoia, el proyecto Sequoia mantiene la **GnuPG Chameleon**, una reimplementación drop-in de `gpg` y `gpgv` construida sobre Sequoia.

La pieza es la misma upstream en las tres distros, pero el nombre del paquete y el mecanismo de activación cambian:

| Distro | Paquete | Cómo se invoca | Activación |
| --- | --- | --- | --- |
| Debian 13 / Ubuntu 24.04+ | `gpg-from-sq` | Binarios separados: `gpg-sq`, `gpgv-sq` en `/usr/bin/` | Directa: llamar `gpg-sq` por nombre |
| Fedora 40+ | `sequoia-chameleon-gnupg` | Shims `gpg` / `gpgv` en `/usr/share/sequoia-chameleon-gnupg/shims/` | Script: `source /usr/share/sequoia-chameleon-gnupg/activate` |
| Nix / NixOS | `sequoia-chameleon-gnupg` | Mismos shims, ruta bajo el store de Nix | Equivalente a Fedora; ver 16.2.3 |

Cada distro impone una mecánica distinta para que las herramientas reciban el chameleon en lugar de GnuPG. Las subsecciones 16.2.x cubren cada caso, y las 16.3-16.5 son operación común una vez activado.

#### 16.1 Limitación en Debian 13 con llaves v6 y el wrapper `git-sq-sign`

**El estado a mayo de 2026 en Debian 13 trixie es el siguiente**: el paquete `gpg-from-sq` (chameleon 0.13.1) instala correctamente `gpg-sq` y `gpgv-sq`, y para operaciones públicas (verificar, listar, cifrar contra cert ajeno) funciona como se espera. **Pero al firmar con una llave v6 emite `Unusable secret key` y falla.** La causa es que el chameleon delega la custodia de material secreto al `gpg-agent` de GnuPG clásico, y la versión de GnuPG en Debian 13 no entiende el formato de material secreto v6 (responde `Invalid operation: Keygrip not defined for this kind of public key` cuando se intenta importarlo con `gpg --import`).

Por tanto, en Debian 13 trixie **la integración de git con la llave Sequoia v6 vía `gpg-sq` no funciona out-of-the-box**. Hay tres formas de proceder:

**Camino A — Wrapper `git-sq-sign` basado en `sq sign` directo (recomendado para Debian 13 con v6).** sq tiene su propio keystore y puede firmar sin pasar por GnuPG. Se escribe un wrapper minimal que git invoca como si fuera `gpg`, y que traduce internamente al subcomando de sq. Es el camino que documenta esta sección 16.1.

**Camino B — Regenerar la llave en formato v4 con `--profile rfc4880`.** Pierde las mejoras criptográficas de v6, pero gana integración drop-in con `gpg-sq` y compatibilidad total con verificadores GnuPG legacy. Es decisión que se toma al inicio del despliegue (sección 2.1) y no se cambia a posteriori sin rotación completa.

**Camino C — Esperar a que GnuPG con soporte v6 completo llegue al ecosistema** (probablemente Debian 14 o backports). Mientras tanto, no hay integración nativa.

Este manual asume Camino A en Debian 13 con v6. Cuando se confirme soporte v6 nativo en `gpg-agent` (a verificar contra cada versión de Debian / GnuPG), las subsecciones 16.2-16.5 vuelven a aplicar tal cual están escritas y el wrapper deja de ser necesario.

> En Fedora 40+ y Nix/NixOS la situación puede diferir según la versión empaquetada de GnuPG. Verificar antes de descartar el wrapper: ejecutar `gpg --version` y mirar si la versión es ≥ 2.5.x o si el changelog menciona soporte de RFC 9580. Si no hay certeza, usar el Camino A es la opción segura.

##### Instalación del wrapper

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/git-sq-sign <<'WRAPPER_EOF'
#!/bin/sh
# Wrapper para que git firme commits con sq directamente, bypaseando gpg-sq.
# Necesario en Debian 13 trixie con llaves v6 mientras GnuPG clásico no las soporte.
#
# Git invoca el "gpg.program" con argumentos del estilo:
#   gpg --status-fd=2 -bsau <FP>
# Necesitamos leer stdin, devolver firma detached armored a stdout.

SIGNER=""
while [ $# -gt 0 ]; do
  case "$1" in
    -u|--local-user) SIGNER="$2"; shift 2 ;;
    --status-fd=*) shift ;;
    --status-fd) shift 2 ;;
    -b|--detach-sign|-s|--sign|-a|--armor) shift ;;
    --) shift ;;
    *) shift ;;
  esac
done

exec sq sign --signer "${SIGNER}" --signature-file - -
WRAPPER_EOF

chmod +x ~/.local/bin/git-sq-sign
```

Asegurar que `~/.local/bin` está en el PATH (la mayoría de distros lo añaden por defecto si existe el directorio al inicio de sesión, pero conviene verificar):

```bash
grep -q '.local/bin' ~/.bashrc || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
which git-sq-sign   # debe responder /home/${USER}/.local/bin/git-sq-sign
```

##### Configuración de git

```bash
git config --global gpg.program git-sq-sign
git config --global user.signingkey "${KEY_FP}"
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

##### Prueba de firma

```bash
cd /ruta/al/repositorio
git commit --allow-empty -m "Prueba de firma con git-sq-sign"
```

sq pide la passphrase de la subkey S **en terminal** (no vía pinentry gráfico de GnuPG, porque no estamos usando gpg-agent). Si la passphrase es correcta, el commit se crea. Verificar:

```bash
git log -1 --format=%H
git cat-file commit HEAD | head -5
```

Debe aparecer una línea `gpgsig -----BEGIN PGP SIGNATURE-----` cerca del inicio del objeto commit.

> `git log --show-signature -1` puede mostrar `gpg: signature made ... using EDDSA key ...` y a continuación `gpg: Can't check signature: No public key`. Esto es porque `git` para **verificar** sigue invocando `gpg-sq` (que tampoco entiende firmas v6 en este escenario), no nuestro wrapper. Es decir, este wrapper resuelve la **firma**, no la verificación local. La verificación queda pendiente hasta que el ecosistema GnuPG soporte v6, o hasta que git acepte un wrapper aparte para verificar (variable `gpg.ssh.program` para llaves SSH; no hay equivalente directo para llaves PGP). Operativamente, los receptores que tengan Sequoia (mantenedores Debian, usuarios de Codeberg con soporte v6, etc.) sí verán la firma como válida.

##### Cuando el wrapper deja de ser necesario

Cuando `gpg --version` reporte versión ≥ 2.5.x (con soporte v6 completo) o cuando `gpg-from-sq` haya migrado a un keystore independiente del agente clásico, la firma vía `gpg-sq` debería funcionar nativamente. En ese momento:

```bash
git config --global gpg.program gpg-sq
```

y el wrapper queda como herramienta histórica. Conservar el archivo `~/.local/bin/git-sq-sign` no estorba.

#### 16.2 Activación del chameleon

> **Nota**: las subsecciones 16.2.x a 16.5 documentan la integración estándar del chameleon como **debería** funcionar y como funciona cuando la llave es v4 o cuando GnuPG con soporte v6 esté disponible. En Debian 13 con v6, usar el Camino A documentado en 16.1; estas subsecciones quedan como referencia para Fedora, Nix, y futuros entornos.

##### 16.2.1 Debian: invocar `gpg-sq` directamente o vía alternatives

En Debian la convención es exponer dos binarios distintos: `gpg` (GnuPG clásico) y `gpg-sq` (chameleon). Por defecto, cuando una herramienta llama `gpg`, recibe el clásico. Hay tres caminos para que reciba el chameleon:

**Camino 1 — Usar `gpg-sq` explícitamente solo donde se necesite.** Configurar cada herramienta para llamar `gpg-sq` en lugar de `gpg`. Para git:

```bash
git config --global gpg.program gpg-sq
```

Para `debsign`:

```bash
debsign -k "${KEY_FP}" -p gpg-sq <archivo.changes>
```

Es el modo más conservador: no toca el binario `gpg` del sistema. Cada herramienta se configura una vez.

**Camino 2 — Hacer que `gpg` apunte a `gpg-sq` mediante alternatives.** Debian gestiona binarios alternativos con `update-alternatives`:

```bash
sudo update-alternatives --install /usr/bin/gpg gpg /usr/bin/gpg-sq 50
sudo update-alternatives --config gpg
```

Esto cambia el comportamiento de `gpg` a nivel sistema. Útil si todas las herramientas deben usar Sequoia, pero conviene tener claro que tras este cambio una llamada cruda a `gpg` ya no es GnuPG clásico.

**Camino 3 — Desinstalar GnuPG.** En sistemas nuevos donde no se necesita GnuPG en absoluto:

```bash
sudo apt remove gnupg gnupg2
```

`gpg-from-sq` ya provee suficiente API para que `apt`, `dpkg`, y herramientas de empaquetado funcionen. Este manual no recomienda esto en sistemas existentes hasta haber validado durante semanas que no falta nada — algunas herramientas usan flags que el chameleon aún no implementa.

##### 16.2.2 Fedora: script de activación con shims

Fedora no instala el chameleon como binario `gpg-sq` separado. En su lugar instala shims `gpg` y `gpgv` en `/usr/share/sequoia-chameleon-gnupg/shims/` y un script de activación que pone esos shims al principio del `$PATH`. Esto se debe a una limitación de RPM (no hay equivalente directo de `dpkg-divert` que permita reemplazar el binario `/usr/bin/gpg` del paquete `gnupg2` sin conflictos).

Hay dos formas de activarlo:

**Forma A — Por sesión (recomendado para evaluar):**

```bash
source /usr/share/sequoia-chameleon-gnupg/activate
```

A partir de ese momento, en esa sesión de shell, `gpg --version` reporta `gpg (GnuPG-compatible Sequoia Chameleon) 2.2.40` (o similar). Las herramientas lanzadas desde esa shell verán el chameleon. Cerrar la shell revierte el cambio.

**Forma B — Permanente para el usuario:**

```bash
mkdir -p ~/.config/environment.d/
cat > ~/.config/environment.d/sequoia-chameleon-gnupg.conf <<'EOF'
PATH=/usr/share/sequoia-chameleon-gnupg/shims:$PATH
EOF
```

Tras logout/login, el `$PATH` del usuario incluye los shims antes de `/usr/bin/`, y cualquier herramienta que invoque `gpg` (lanzada vía systemd-user, GNOME, KDE, terminal) recibe el chameleon. Revertir es borrar el archivo y volver a hacer login.

> A diferencia de Debian, en Fedora el chameleon **no** se puede instalar como reemplazo de `gpg` a nivel sistema sin conflicto con el paquete `gnupg2`. El cambio es siempre por usuario. Esto es por diseño de RPM, no limitación del chameleon.

Tras la activación, los siguientes `git config`/`debsign` ya no necesitan `gpg-sq` — basta con `gpg` porque el shim toma el control. En el resto del manual, cuando aparezca `gpg-sq` en un comando, en Fedora se traduce a `gpg` con el chameleon activado.

##### 16.2.3 Nix / NixOS: shims vía environment.systemPackages o Home Manager

En Nix los shims viven bajo el store: `/run/current-system/sw/share/sequoia-chameleon-gnupg/shims/` (o equivalente bajo `~/.nix-profile/` para instalación per-user).

**NixOS sistema-wide** — declarar la activación en `configuration.nix`:

```nix
environment.systemPackages = with pkgs; [
  sequoia-sq
  sequoia-sqv
  sequoia-sqop
  sequoia-chameleon-gnupg
];

environment.sessionVariables = {
  PATH = [ "/run/current-system/sw/share/sequoia-chameleon-gnupg/shims" ];
};
```

> En NixOS la sintaxis exacta de cómo prepender al PATH depende de la versión y de si se usa Home Manager. Verificar la ruta real del shim con `nix-store -q --references $(which sq) | grep chameleon` o equivalente. Las rutas bajo `/run/current-system/sw/` cambian con cada `nixos-rebuild switch`; el `environment.sessionVariables` que pone el manual es el patrón más estable.

**Home Manager** (per-user, sin tocar configuración del sistema):

```nix
home.packages = with pkgs; [
  sequoia-sq
  sequoia-sqv
  sequoia-sqop
  sequoia-chameleon-gnupg
];

home.sessionPath = [
  "${pkgs.sequoia-chameleon-gnupg}/share/sequoia-chameleon-gnupg/shims"
];
```

Tras `home-manager switch` y nueva sesión, `gpg --version` debe reportar el chameleon.

#### 16.3 Configuración para firma de commits con git

Una vez el chameleon está activo (por cualquiera de los caminos de 16.2), la configuración de git es la misma en las tres distros. En Debian con Camino 1 hay que usar `gpg-sq` explícito; en Debian con Camino 2/3, Fedora o Nix con activación, basta con `gpg`:

```bash
# Debian Camino 1
git config --global gpg.program gpg-sq

# Debian Camino 2/3, Fedora, Nix
git config --global gpg.program gpg

# Común a las tres distros y todos los caminos
git config --global user.signingkey "${KEY_FP}"
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

A partir de ahí, `git commit` firma automáticamente con la subkey S de Sequoia (el chameleon selecciona la subkey con capacidad de firma del certificado indicado en `user.signingkey`). Probar:

```bash
cd /ruta/al/repositorio
git commit --allow-empty -m "Prueba de firma con chameleon"
git log --show-signature -1
```

La última línea debe mostrar `gpg: Good signature from "Nombre Apellido <usuario@ejemplo.org>"` (o equivalente). Si dice "BAD signature" o "no public key", revisar Anexo B.5.

#### 16.4 Firma de paquetes Debian con `debsign`

Aplicable solo cuando se construyen paquetes `.deb`, típicamente en `${WORK_HOST}` Debian o Ubuntu. En Fedora y Nix esta sección no aplica salvo que se mantengan paquetes Debian por algún motivo.

```bash
# Debian Camino 1 (chameleon como gpg-sq explícito)
debsign -k "${KEY_FP}" -p gpg-sq mi-paquete_1.0_amd64.changes

# Debian Camino 2/3 (gpg ya es chameleon)
debsign -k "${KEY_FP}" mi-paquete_1.0_amd64.changes
```

`-p gpg-sq` instruye a `debsign` a usar el binario `gpg-sq` en lugar de `gpg`. Verificar la firma:

```bash
# Camino 1: usar binario explícito
gpgv-sq --keyring ~/.local/share/pgp.cert.d/<...>/cert mi-paquete_1.0_amd64.changes

# Camino 2/3 o Fedora/Nix con chameleon activado: gpgv ya es chameleon
gpgv --keyring ~/.local/share/pgp.cert.d/<...>/cert mi-paquete_1.0_amd64.changes
```

> El `<...>` es un placeholder: el archivo concreto está en `~/.local/share/pgp.cert.d/<2-hex-primeros>/<38-hex-restantes>/cert`, donde los hex se derivan del fingerprint. Para localizarlo: `find ~/.local/share/pgp.cert.d -name cert | head`.

#### 16.5 Limitaciones conocidas del chameleon

El chameleon implementa el subconjunto más usado de la CLI de GnuPG, no toda. Esto vale para las tres distros porque el código upstream es el mismo. Lo soportado:

- Firma y verificación (clearsign, detached, inline).
- Cifrado y descifrado.
- Listado de llaves (`--list-keys`, `--list-secret-keys`).
- Import y export básicos.
- Las opciones `--batch`, `--quiet`, `--with-colons`, `--with-fingerprint`.

Lo que **no** está completamente soportado:

- Algunos modelos de confianza (TOFU en particular).
- Operaciones interactivas de edición de llaves (`--edit-key`).
- Algunos flags específicos de scripts complejos. Cuando una herramienta falla con el chameleon y funciona con GnuPG clásico, suele ser por una opción de este grupo.

Si una herramienta concreta no funciona con el chameleon, el camino pragmático es: dejar esa herramienta usando `gpg` clásico (Camino 1 en Debian, no activar el shim para esa shell en Fedora/Nix) y reportar el caso al proyecto Sequoia. Recordar que el chameleon y GnuPG clásico **no comparten state**: si el chameleon importa un certificado a `~/.local/share/pgp.cert.d/`, GnuPG clásico no lo verá en su `~/.gnupg/`. Para mantener ambos sincronizados, exportar e importar manualmente cuando haga falta.

### 17. SSH con la subkey de autenticación

Esta sección documenta el flujo completo de uso de la subkey A para autenticación SSH: configurar el agente, exportar la clave pública en formato OpenSSH, instalarla en los servidores, y conectarse.

#### 17.1 Elección del agente

Hay dos formas de exponer la subkey A a `ssh`:

- **`gpg-agent` (de GnuPG real) con soporte SSH habilitado.** Es la forma clásica y la que la mayoría de la documentación describe. Requiere tener GnuPG instalado y configurado en paralelo a Sequoia, y que `gpg-agent` sepa de la subkey A.
- **`ssh-agent` directo, con la clave exportada en formato OpenSSH.** No requiere GnuPG. Es el modelo simple para un manual centrado en Sequoia.

Este manual usa el segundo modelo. Quien ya tenga un `gpg-agent` configurado y quiera mantenerlo, puede usar la primera vía consultando documentación específica de GnuPG; los pasos siguientes asumen `ssh-agent`.

#### 17.2 Export de la subkey A en formato OpenSSH

`sq` puede convertir una subkey de autenticación al formato que entiende `ssh`. En `${WORK_HOST}`:

```bash
# Export de la clave pública en formato OpenSSH (lo que va en authorized_keys)
sq key subkey export-ssh \
   --cert "${KEY_FP}" \
   --key "${SUB_A_FP}" \
   > ~/.ssh/id_openpgp.pub
```

> El subcomando exacto puede variar entre versiones menores de sq. En 1.3.1 el camino canónico es `sq key subkey export` con flag de formato, o un subcomando dedicado tipo `sq key subkey export-ssh`. Si el subcomando exacto difiere, `sq key --help` y `sq key subkey --help` listan las opciones disponibles. La pieza pública que se necesita es una línea `ssh-ed25519 AAAA... usuario@host` lista para `authorized_keys`.

Verificar el contenido:

```bash
cat ~/.ssh/id_openpgp.pub
```

Debe verse como una línea típica de OpenSSH:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... openpgp:0xABCD1234EF567890
```

#### 17.3 Configurar `ssh-agent` para usar la subkey A

El truco con `ssh-agent` y subkeys OpenPGP es que el agente espera leer la clave secreta de un archivo en disco (`~/.ssh/id_ed25519`, típicamente), no de un keystore de Sequoia. Hay dos soluciones:

**Opción A — `ssh-agent` con un wrapper que delegue en sq.** No es la práctica común; suele requerir herramientas adicionales.

**Opción B — usar `gpg-agent` (de `gnupg`) en modo SSH.** A pesar de lo dicho en 17.1, este es el camino con menos fricción si el sistema ya tiene GnuPG. Configurar `gpg-agent` para que actúe como `ssh-agent`:

Archivo `~/.gnupg/gpg-agent.conf`:

```
enable-ssh-support
default-cache-ttl 3600
max-cache-ttl 28800
```

Archivo `~/.gnupg/sshcontrol`:

```
# Huella de la subkey A (formato keygrip, no fingerprint OpenPGP)
<keygrip-de-la-subkey-A>
```

Para obtener el keygrip de la subkey A (comando depende de cómo esté activado el chameleon — ver 16.2):

```bash
# Debian Camino 1: chameleon como binario explícito
gpg-sq --list-keys --with-keygrip "${KEY_FP}"

# Debian Camino 2/3, Fedora con activate, Nix con shim en PATH
gpg --list-keys --with-keygrip "${KEY_FP}"
```

En la salida, buscar la línea `Keygrip = ...` debajo de la subkey con flag `[A]`. Ese valor va en `sshcontrol`.

Variables de entorno para que `ssh` use el `gpg-agent`:

```bash
# En ~/.bashrc o ~/.profile (Debian/Fedora) o equivalente en NixOS
export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
gpg-connect-agent updatestartuptty /bye > /dev/null
```

Tras logout/login (o source del archivo), `ssh-add -L` debe listar la subkey A.

> Sobre qué `gpg-connect-agent` se invoca: si se activó el chameleon como reemplazo de `gpg` (Debian Camino 2/3, Fedora con shim, Nix con shim), `gpg-connect-agent` ya viene del chameleon. Si se está en Debian Camino 1, hay que tener instalado `gnupg` además (provee `gpg-connect-agent` clásico). Verificar con `which gpg-connect-agent`. En cualquier caso, `gpg-agent` (el servicio que mantiene las llaves en memoria) es el mismo binario en todos los escenarios.

#### 17.4 Instalar la pública en los servidores remotos

En cada servidor donde se quiera autenticar:

```bash
# Desde ${WORK_HOST}
ssh-copy-id -i ~/.ssh/id_openpgp.pub usuario@servidor.example.org
```

O manualmente, copiando el contenido de `~/.ssh/id_openpgp.pub` al `~/.ssh/authorized_keys` del usuario remoto.

Convención de naming sugerida en `authorized_keys`: editar el comentario al final de la línea para identificar la subkey (en lugar del `openpgp:0x...` genérico):

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... openpgp-A-<fingerprint-corto>-<año>
```

Esto facilita auditorías y la operación de "borrar la línea X" cuando se rota la subkey A.

#### 17.5 Probar la conexión

```bash
ssh usuario@servidor.example.org
```

El `gpg-agent` (actuando como `ssh-agent`) pedirá la passphrase de la subkey A la primera vez en cada sesión. Tras eso, las conexiones siguientes durante el TTL configurado (3600 segundos en el ejemplo) no requieren reintroducirla.

Para confirmar qué identidad se está usando:

```bash
ssh -v usuario@servidor.example.org 2>&1 | grep -i "offering\|authentication"
```

Debe mostrar `Offering public key: ED25519 SHA256:... openpgp:0xABCD...` y, si el servidor la acepta, `Authentication succeeded (publickey)`.

#### 17.6 Rotación de la subkey A y propagación a servidores

Cuando llegue el momento de rotar la subkey A (sección 12, cadencia sugerida 2 años):

1. Generar la nueva subkey A en `${OFFLINE_HOST}` (sección 6.3).
2. Exportar e importar en `${WORK_HOST}` siguiendo el flujo de la sección 7.
3. Re-exportar en formato OpenSSH:

   ```bash
   sq key subkey export-ssh --cert "${KEY_FP}" --key "${SUB_A_NEW_FP}" > ~/.ssh/id_openpgp.pub
   ```

4. Actualizar `~/.gnupg/sshcontrol` con el keygrip nuevo (y eliminar el viejo si se quiere forzar el cambio inmediato).
5. Distribuir la nueva pública a todos los servidores y, después de confirmar que la nueva funciona, eliminar la entrada vieja de `authorized_keys` en cada servidor.

Es operativamente la parte más pesada del manual: si hay 30 servidores, hay 30 archivos `authorized_keys` que actualizar. Un script de despliegue (Ansible, fabric, o un loop simple sobre `ssh`) ahorra trabajo. Documentar la lista de servidores donde está instalada la pública facilita esta operación.

### 18. Autenticación de certificados de terceros: shadow CAs y trust root

Hasta aquí el manual ha tratado la llave propia: cómo generarla, custodiarla, rotarla, publicarla. Esta sección aborda la otra mitad de la operación cotidiana: cómo decidir si el certificado de otra persona es legítimo antes de cifrar contra él o de fiarse de sus firmas. Sequoia maneja este problema de forma distinta a GnuPG clásico, y entender el modelo evita advertencias confusas y decisiones a ciegas.

#### 18.1 El Local Trust Root

Cuando `sq` se invoca por primera vez en una máquina, crea automáticamente una llave llamada Local Trust Root. No es una llave OpenPGP de uso público -no se publica, no se firma con la primaria, no representa una identidad- sino una llave técnica que vive en el cert store local y firma todo lo demás. Su propósito es responder a la pregunta operativa: *"¿este certificado de un tercero está autenticado para mí?"*.

Un certificado se considera autenticado cuando existe una cadena de certificación desde el Local Trust Root hasta el UserID en cuestión, con un nivel de confianza acumulado del 100% (en la nomenclatura interna de Sequoia, 120 sobre 120). La cadena puede tener un solo eslabón (el Local Trust Root firma directamente el certificado del tercero) o varios (el Local Trust Root firma a una shadow CA que firma al tercero — ver 18.2).

La diferencia con GnuPG clásico es relevante. En GnuPG la confianza es una propiedad de cada llave (`ultimate`, `full`, `marginal`, `never`) que el usuario asigna manualmente con `gpg --edit-key trust`. En Sequoia la confianza emerge de cadenas de certificación medidas en porcentajes, todas ancladas en el Local Trust Root. Es más cercano al modelo de PKI X.509 (con una raíz local en lugar de una global) que al web of trust tradicional de PGP.

El comando `sq key generate --own-key` que se usó en la sección 4.2 inserta automáticamente la llave generada en este sistema: la marca como certificada por el Local Trust Root con profundidad ilimitada; también `--own-key` aplica trust depth 255 con `--unconstrained` implícito al Local Trust Root, el modelo "tu propia llave" no requiere los flags que sí necesita un introductor externo. Por eso desde el momento en que se importa la primaria en el cert store de Tails, sus UserIDs aparecen como autenticados sin pasos adicionales. La opción `--own-key` es exactamente "decirle al Local Trust Root que esta llave también es mía".

#### 18.2 Shadow CAs

Cuando `sq` trae un certificado del mundo exterior -por `sq network search`, `sq network keyserver fetch`, `sq network wkd fetch` o `sq cert import` desde un archivo descargado- necesita registrar de dónde vino. Para eso usa lo que el proyecto llama **shadow CAs**: autoridades certificadoras locales, internas, que representan fuentes de información.

En sq 1.3.1 la shadow CA principal para certificados traídos de directorios públicos se llama `Public Directories`. Aparece en el cert store al iniciar la primera operación de red y, por defecto, queda conectada al Local Trust Root con un link de tipo "partially trusted CA" con trust amount 40 (de los 120 que necesita un binding para considerarse plenamente autenticado).

Listar los links activos:

```bash
sq pki link list
```

Salida típica tras una sesión normal:

```
 - ┌ E3AE4A3459D652925DE70779D6D72DE94936366B
   └ Public Directories
     - created at 2002-02-20 20:02:00
     - linked as a partially trusted CA
     - trust amount: 40
```

El Local Trust Root le ha asignado a `Public Directories` 40 puntos de confianza. Cuando esa shadow CA firma a su vez un binding UserID↔llave traído de un keyserver o de WKD, el binding hereda esa confianza parcial. La consecuencia práctica: el binding existe en el cert store y `sq` lo reconoce, pero al cifrar contra él o al verificar una firma suya `sq` advertirá *"the User ID is not authenticated"* — porque 40 no llega a 120.

Esto es por diseño. Traer una llave de un keyserver no es prueba de identidad. Cualquier persona pudo subir esa llave con ese correo. Que el keyserver haya verificado que el correo recibe un challenge prueba control del buzón en un momento dado, no identidad de la persona. Los 40 puntos reflejan ese matiz: hay algún grado de evidencia, pero no suficiente para fiarse sin más.

`sq pki link list` solo muestra los links explícitos al Local Trust Root, no los bindings que las shadow CAs hayan firmado a su vez. Para ver el camino completo entre el Local Trust Root y un binding concreto (qué shadow CA participó, con qué profundidad y trust amount), usar `sq pki path` sobre el binding, o `sq cert list --show-paths <FINGERPRINT>` sobre un certificado de tercero.

Los nombres y trust amounts por defecto de las shadow CAs son detalles de implementación de sq que han cambiado entre versiones. Lo que sí es estable es el principio: certificados traídos del mundo exterior llegan con confianza parcial o nula y requieren `sq pki link add` explícito para autenticarse plenamente. La sección 18.3 documenta ese paso.

#### 18.3 Autenticar manualmente el certificado de un contacto

El procedimiento canónico, equivalente al `gpg --sign-key` o `gpg --lsign-key` tradicional, es `sq pki link add`. La operación firma localmente (con el Local Trust Root) el binding UserID↔certificado, declarando "para mi instalación local, esta llave efectivamente pertenece a este UserID".

Antes del comando, dos pasos imprescindibles:

1. **Obtener el fingerprint del contacto por un canal distinto al que se usó para traer la llave.** Si el certificado vino del keyserver, el fingerprint debe llegar por otro medio: una tarjeta de presentación, un mensaje firmado por otra llave ya autenticada, una llamada por teléfono donde la persona lo lee, un escaneo del QR del fingerprint en un encuentro físico. Si tanto la llave como el fingerprint vienen de la misma fuente, no hay verificación real — solo se confirma que la fuente no ha cambiado.
2. **Comparar el fingerprint** con el del certificado importado:

   ```bash
   sq inspect --cert "${CONTACT_FP}"
   ```

   La primera línea de la salida debe coincidir letra por letra con el fingerprint verificado por canal alterno.

Si coincide, autenticar:

```bash
export CONTACT_FP="..."   # fingerprint del contacto, verificado por canal alterno
sq pki link add --cert "${CONTACT_FP}" --all
```

`--all` autentica todos los UserIDs del certificado a la vez. Si solo se quiere autenticar un UserID específico (por ejemplo, solo el correo de trabajo, no el personal):

```bash
sq pki link add --cert "${CONTACT_FP}" --email "alice@example.com"
```

A partir de ese momento, ese binding está completamente autenticado para la instalación local. Las advertencias al cifrar contra ese certificado desaparecen y las firmas que ese certificado emita se consideran de fuente identificada.

#### 18.4 Trust depth: cuando un contacto firma a otros contactos

`sq pki link add` por defecto asigna profundidad de confianza cero: la autenticación se aplica al certificado autenticado y a nadie más. Si Alice (cuyo certificado acabo de autenticar) firma el certificado de Bob, ese hecho no convierte a Bob en autenticado para mí, solo significa que Alice se fía de Bob.

Para que las certificaciones que Alice emita "se propaguen" hacia mí, hay que autorizarla como introductora con `sq pki link authorize`. Este comando en sq 1.3.1 exige tres decisiones explícitas: **cuánta confianza** (`--amount`), **qué profundidad** (`--depth`) y **qué alcance** (`--domain`, `--regex` o `--unconstrained`). Las tres son opt-in deliberado: si se omite el alcance, sq aborta con error pidiéndolo explícitamente. Es protección contra otorgar poder de introductor sin pensar.

**Patrón típico — introductor restringido a su dominio.** Confías en Alice para identificar a gente de `example.com`:

```bash
sq pki link authorize \
  --cert "${ALICE_FP}" \
  --email "alice@example.com" \
  --depth 1 \
  --domain example.com
```

Significado: las certificaciones que Alice emita sobre bindings con correo `@example.com` se consideran autenticadas con `--depth 1` (las firmas de Alice valen; las firmas de alguien a quien Alice autorizó como sub-introductor, no).

**Patrón sin restricción de dominio:**

```bash
sq pki link authorize \
  --cert "${ALICE_FP}" \
  --email "alice@example.com" \
  --depth 1 \
  --unconstrained
```

Significado: Alice puede certificar cualquier UserID, sin restricción de dominio. Más permisivo. Reservar para contactos en quienes la confianza es muy alta.

**Trust depth con valores mayores.** Con `--depth 2`, también los certificados firmados por personas a quienes Alice autorizó como introductoras se consideran autenticados. `--depth 255` es cadena ilimitada (lo que `--own-key` aplica a la llave propia).

> Cuidado con el default de `--depth` en `sq pki link authorize`: si se omite el flag, sq usa **`--depth 255`** (meta-introductor ilimitado), no `--depth 0`. Esto es un comportamiento distinto al de `sq pki link add`, donde el default es profundidad 0. Cuando se ejecuta `link authorize`, conviene **siempre** pasar `--depth` explícitamente para no autorizar más poder del previsto.

**Trust amount.** El flag `--amount` controla cuánta confianza (de 120) aporta cada certificación del introductor. Por defecto es 120 (confianza plena: una certificación del introductor basta para autenticar). Para confianza parcial:

```bash
sq pki link authorize \
  --cert "${ALICE_FP}" \
  --email "alice@example.com" \
  --depth 1 \
  --domain example.com \
  --amount 60
```

Con `--amount 60`, harían falta dos certificaciones de introductores parciales independientes para alcanzar los 120 que sq necesita para considerar autenticado un binding. Es el modelo "two-out-of-N" clásico del Web of Trust.

En la práctica, `--depth 1` con `--domain X` y `--amount 120` (el default) es lo más común para un colega que conoce bien a su equipo. `--depth 2` o más es raro fuera de organizaciones grandes con jerarquía interna. `--unconstrained` y `--amount 120` juntos es el máximo: equivale a "confío en esta persona como en mí mismo para identificar a cualquiera".

#### 18.5 Elevar la confianza en una shadow CA entera

Existe la opción de declarar que toda una shadow CA es confiable. Por ejemplo, si se decide que keys.openpgp.org es suficientemente fiable para considerar autenticados todos los UserIDs que verifica (lo cual es una decisión política, no técnica — el servidor verifica que el correo recibe el challenge, pero no la identidad de la persona detrás):

```bash
# Identificar el fingerprint de la shadow CA con sq pki link list
sq pki link list

# Elevar la confianza
sq pki link authorize --cert-special public-directories --all --unconstrained --depth 0
```

`--ca '*'` declara que es una CA con dominio ilimitado. Tras esto, cualquier certificado descargado desde keys.openpgp.org queda automáticamente autenticado para sus UserIDs con correo.

**Esta decisión vale la pena pensarla.** Significa fiarse de la operación del servidor (que verifica direcciones de correo correctamente, que su base no se ha comprometido, que no firma certificados arbitrarios). Para uso cotidiano de bajo riesgo (correspondencia normal con contactos cuya identidad no necesita autenticación criptográfica fuerte) puede ser un buen compromiso. Para uso de alto riesgo (comunicación con periodistas, abogados, fuentes), la decisión correcta sigue siendo autenticar cada certificado individualmente con verificación de fingerprint por canal alterno (18.3).

#### 18.6 Listar y retractar autenticaciones

Ver qué bindings se han autenticado o autorizado localmente:

```bash
sq pki link list
```

Salida típica tras autenticar el certificado de Alice (`sq pki link add --cert "${ALICE_FP}" --all`):

```
 - ┌ 4D735E0D7DDE1B338E05F2B80AF4CE6DF697FB28
   └ Alice "<alice@example.com>"
     - created at 2026‑05‑20 04:07:20
```

Si además se autorizó al certificado como introductor con `sq pki link authorize`, la salida muestra también `trust depth`, `trust amount` y la restricción de alcance (`domain` o `regular expressions`).

Si en algún momento se descubre que la autenticación fue prematura -el fingerprint era falso, el contacto cambió de llave, el certificado se comprometió, etc.- se puede retractar. `sq pki link retract` exige siempre un selector explícito (al igual que `authorize` exige un alcance); no hay default.

Para retractar el link asociado a un UserID concreto del certificado:

```bash
sq pki link retract --cert "${CONTACT_FP}" --email "alice@example.com"
```

Esto solo funciona si `alice@example.com` es UserID self-signed del certificado `${CONTACT_FP}`. Si no lo es, sq responde `this binding was never certified, there is nothing to retract`, es la misma lógica que en `link add` y `link authorize`: el `--email` selecciona un UserID *dentro del certificado pasado*, no es un parámetro de alcance.

Para retractar el link asociado a un UserID concreto del certificado, el flag a usar depende de la forma del UserID en el certificado:

- Si el UserID es **solo el correo** entre corchetes (`<alice@example.com>`, formato Sequoia moderno), basta con `--email`:

  ```bash
  sq pki link retract --cert "${CONTACT_FP}" --email "alice@example.com"
  ```

- Si el UserID es **combinado** (`Alice <alice@example.com>`, formato GnuPG clásico — común en certificados antiguos importados de keyservers), `--email` falla porque sq lo interpreta literalmente como un UserID que contiene solo `<alice@example.com>`. Usar `--userid-by-email`, que busca el correo dentro de cualquier UserID self-signed:

  ```bash
  sq pki link retract --cert "${CONTACT_FP}" --userid-by-email "alice@example.com"
  ```

- Para casos donde se conoce la cadena exacta del UserID:

  ```bash
  sq pki link retract --cert "${CONTACT_FP}" --userid "Alice <alice@example.com>"
  ```

Si hay dudas en qué forma está el UserID, `sq inspect --cert "${CONTACT_FP}"` lo muestra literalmente. Como alternativa segura, `--all` se puede retractar todos los links asociados al certificado de un golpe.

La retracción no borra el certificado del cert store; emite una nueva certificación local que invalida la anterior. El certificado sigue accesible, pero al cifrar contra él volverán las advertencias de "no autenticado". Para confirmar el efecto, `sq pki link list --cert "${CONTACT_FP}"` tras la retracción debería mostrar el link con trust amount nulo o no listarlo.

Aplica también a links creados con `sq pki link authorize` (introductores): se retractan con el mismo comando. El selector `--email` / `--userid` apunta al UserID self-signed del introductor que se asoció al link en su momento, no al alcance regex/domain que se le puso. Para retractar tu autorización de `Public Directories` como introductor sobre el dominio `ejemplo.org`, lo correcto es `sq pki link retract --cert E3AE...366B --userid "Public Directories"` o `--all`, no `--email "contacto@ejemplo.org"`.

#### 18.7 Expiración de los links

Por defecto, `sq pki link add` crea autenticaciones sin expiración. Para contactos que se conocen bien y no se prevé que cambien de llave en años, es razonable. Para contactos puntuales -un periodista al que se contacta por un único caso, un proveedor con quien se intercambia un solo documento- conviene poner expiración explícita:

```bash
sq pki link add --cert "${CONTACT_FP}" --all --expiration 1y
```

Pasado el plazo, el binding deja de considerarse autenticado y vuelven las advertencias. Renovarlo es ejecutar `sq pki link add` otra vez. La idea es que la confianza, como todo lo demás en el modelo de seguridad de este manual, tenga ciclo de vida y no se acumule indefinidamente sin revisión.

#### 18.8 Relación con `sq pki vouch`

El comando `sq pki vouch add` que apareció en el Anexo C.3 (firma cruzada entre llave vieja y llave nueva) pertenece al mismo sistema, pero opera **hacia afuera** en lugar de hacia adentro: en lugar de declarar localmente que una llave de tercero es autenticada para mí, emite una firma OpenPGP que cualquiera puede ver. Es lo que tradicionalmente se llamó "key signing" en el ecosistema PGP.

La diferencia operativa:

- `sq pki link add` → firma con el Local Trust Root, queda en el cert store local, no se publica, vale solo para mi instalación.
- `sq pki vouch add` → firma con la llave primaria -requiere acceso a la primaria, por tanto requiere `${OFFLINE_HOST}`-, produce una firma que se incorpora al certificado del tercero y puede publicarse en keyservers.

Para el caso del Anexo C.3 -declarar criptográficamente la transición entre dos llaves propias- `sq pki vouch` es el comando correcto. Para autenticación cotidiana de contactos, `sq pki link` es el comando correcto y, además, no requiere bajar la primaria del medio offline.

#### 18.9 Checklist de cierre

- [ ] Existe en `${WORK_HOST}` al menos un certificado de tercero autenticado mediante `sq pki link add`, con su fingerprint verificado por canal alterno
- [ ] El procedimiento de verificación de fingerprint (canal alterno, comparación con `sq inspect`) está documentado en bitácora
- [ ] Política sobre `sq pki link add --ca` definida explícitamente: qué shadow CAs se elevan a confiables y cuáles no
- [ ] Política sobre expiración de links definida (sin expiración para contactos estables, expiración para contactos puntuales)

### 19. Checklist de cierre operativo

- [ ] `sq key list` en `${OFFLINE_HOST}` muestra primaria + 3 subkeys, todas como `secret`, perfil RFC 9580
- [ ] `sq key list` en `${WORK_HOST}` muestra primaria como `public` (stub) y subkeys S, A, E como `secret`
- [ ] Certificado de revocación pre-generado custodiado en ubicación física distinta a `${AIRGAP_MEDIA}` y a `${BACKUP_DIR}`
- [ ] Backup completo en `${BACKUP_DIR}` restaurado con éxito en entorno aislado al menos una vez
- [ ] Certificado público publicado en al menos un canal (keyserver verifying o WKD)
- [ ] Fechas de expiración (primaria y subkeys) registradas en bitácora, con recordatorio 30 días antes de cada vencimiento
- [ ] Firma de commit con `git` funcional en `${WORK_HOST}`: `git-sq-sign` (wrapper sq, sección 16.1) en Debian 13 con llave v6, o `gpg-sq` / `gpg` con chameleon activado cuando el ecosistema soporte v6 nativamente
- [ ] Autenticación SSH con la subkey A funcional contra al menos un servidor
- [ ] Inventario de servidores donde está instalada la pública de la subkey A documentado en bitácora
- [ ] Al menos un certificado de tercero autenticado con `sq pki link add` y fingerprint verificado por canal alterno
- [ ] Política sobre uso de `sq pki link add --ca` documentada (qué shadow CAs se elevan a confianza plena)

---

## Anexos

### Anexo A — Variables, rutas y artefactos generados

Esta sección consolida en un solo lugar las variables que el manual usa, las rutas donde Sequoia almacena su estado, los archivos que se generan a lo largo del procedimiento, y el mapa de qué vive en `${OFFLINE_HOST}` vs `${WORK_HOST}`.

#### A.1 Variables del manual

| Variable | Significado | Ejemplo de valor |
| --- | --- | --- |
| `${AIRGAP_MEDIA}` | Punto de montaje del medio airgapped | `/mnt/airgap` |
| `${BACKUP_DIR}` | Punto de montaje del backup en frío | `/mnt/backup` |
| `${CONTACT_FP}` | Fingerprint del certificado de un tercero a autenticar | (64 caracteres hex) |
| `${EXPIRATION}` | Expiración de la primaria | `3y` |
| `${KEY_FP}` | Fingerprint de la llave primaria | `ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890` |
| `${KEY_OUT}` | Archivo canónico de la llave en `${AIRGAP_MEDIA}` | `/mnt/airgap/keys/primary-and-subkeys.key` |
| `${MIUSB}` | Path del USB airgapped | `/dev/sda` |
| `${OFFLINE_HOST}` | Nombre lógico de la sesión Tails airgapped | `mango` |
| `${REV_OUT}` | Archivo del certificado de revocación pre-generado | `/mnt/airgap/revocation/primary.rev` |
| `${SEQUOIA_HOME}` | Home alterno de sq para entornos de prueba | `/tmp/sq-restore-test` |
| `${SUB_A_FP}` | Fingerprint de la subkey de autenticación | (64 caracteres hex) |
| `${SUB_E_FP}` | Fingerprint de la subkey de cifrado | (64 caracteres hex) |
| `${SUB_S_FP}` | Fingerprint de la subkey de firma | (64 caracteres hex) |
| `${TRANSPORT}` | Punto de montaje del USB de transporte | `/media/${USER}/transport` |
| `${USER_EMAIL}` | Correo principal asociado a la llave | `usuario@ejemplo.org` |
| `${USER_ID}` | Pseudónimo o handle adicional | `villonaco` |
| `${USER_REAL}` | Nombre real del titular de la llave | `Ada Lovelace` |
| `${WORK_HOST}` | Nombre lógico del equipo de trabajo (Debian/Fedora/NixOS) | `papaya` |

Las variables se definen al inicio de cada sesión con `export VAR=valor`. En sesiones Tails se redefinen siempre porque el shell no persiste.

#### A.2 Rutas de Sequoia en GNU/Linux

Rutas por defecto (sin `SEQUOIA_HOME` definida):

| Ruta | Contenido |
| --- | --- |
| `~/.local/share/pgp.cert.d/` | Cert store (claves públicas, tuyas y de terceros) |
| `~/.local/share/sequoia/keystore/` | Keystore (material secreto administrado por sq) |
| `~/.local/share/sequoia/revocation-certificates/` | Revocation certs autogenerados por `sq key generate` |
| `~/.config/sequoia/sq/config.toml` | Configuración opcional (no usada en este manual) |

Cuando `SEQUOIA_HOME` está definida (sección 14.3), todo lo anterior se relocaliza bajo ese directorio.

> En NixOS y en Home Manager el sistema de archivos del usuario sigue el estándar XDG igual que Debian o Fedora; las rutas anteriores son idénticas. Lo que cambia en Nix son las rutas de los binarios (`/run/current-system/sw/bin/` o `~/.nix-profile/bin/` en lugar de `/usr/bin/`), no las del estado del usuario.

#### A.3 Convenciones de nombre de archivo

Para mantener consistencia en `${AIRGAP_MEDIA}`, `${BACKUP_DIR}` y los USB de transporte:

| Nombre | Contenido |
| --- | --- |
| `$(date +'%Y-%m-%d_%H-%M')"-${USER_EMAIL/@/_at_}-${KEY_FP: -16}.pub.asc` | Certificado público (lo que se distribuye) |
| `${KEY_FP}.rev` | Certificado de revocación pre-generado |
| `${KEY_FP}.subkey-S.sec.asc` | Material secreto de la subkey de firma |
| `${KEY_FP}.subkey-A.sec.asc` | Material secreto de la subkey de autenticación |
| `${KEY_FP}.subkey-E.sec.asc` | Material secreto de la subkey de cifrado |
| `full-export-YYYYMMDD.asc` | Backup completo (primaria + subkeys) con fecha |
| `FINGERPRINT.txt` | Fingerprint de la primaria, en texto plano |
| `SUBKEY_FINGERPRINTS.txt` | Fingerprints de las tres subkeys |

#### A.4 Mapa de "qué vive dónde"

| Material | `${OFFLINE_HOST}` (Tails + `${AIRGAP_MEDIA}`) | `${WORK_HOST}` (Debian/Fedora/NixOS) | `${BACKUP_DIR}` (cold storage) | Medio de revocación |
| --- | --- | --- | --- | --- |
| Primaria secreta | sí | no | sí | no |
| Subkey S secreta | sí | sí | sí | no |
| Subkey A secreta | sí | sí | sí | no |
| Subkey T+R secreta | sí | sí | sí | no |
| Certificado público | sí | sí | sí | no |
| Revocation cert | sí (copia) | no | no | sí (copia maestra) |
| Passphrase de la primaria | gestor de contraseñas | no aplica | no aplica | custodia separada |
| Passphrase de las subkeys | gestor de contraseñas | gestor de contraseñas | no aplica | no aplica |

La regla operativa que se deriva del mapa: la primaria secreta solo aparece en dos lugares (operativo + backup), nunca en `${WORK_HOST}`. El revocation cert maestro nunca está en el mismo medio que la primaria. Las passphrases nunca están en el mismo medio que el material que protegen.

### Anexo B — Diagnóstico por síntoma

Los problemas que aparecen en la operación de una llave Sequoia con modelo offline tienden a repetirse. Este anexo lista los síntomas más comunes con su procedimiento de diagnóstico y la causa habitual.

#### B.1 Síntoma: `sq key list` en `${WORK_HOST}` muestra la primaria como secreta

Comandos de diagnóstico:

```bash
sq key list --cert "${KEY_FP}"
sq inspect --cert "${KEY_FP}" | grep -i "primary\|secret"
```

Salida problemática:

```
Primary key: secret, certify
```

Causa: durante la importación de subkeys en `${WORK_HOST}` (sección 7.3), se importó por error el archivo de export completo de la llave (`full-export.asc` o similar), no los exports selectivos por subkey. La primaria secreta está ahora en el keystore del equipo de trabajo, lo cual rompe la separación offline.

Remediación:

```bash
# Eliminar la primaria secreta del keystore local sin tocar las subkeys
sq key delete --cert "${KEY_FP}"

# Re-importar solo el certificado público
sq cert import ${TRANSPORT}/${KEY_FP}.pub.asc

# Re-importar las subkeys secretas selectivamente
sq key import ${TRANSPORT}/${KEY_FP}.subkey-S.sec.asc
sq key import ${TRANSPORT}/${KEY_FP}.subkey-A.sec.asc
sq key import ${TRANSPORT}/${KEY_FP}.subkey-E.sec.asc
```

> `sq key delete` borra material secreto del keystore. Verificar dos veces el fingerprint antes de ejecutar; el comando es destructivo. Si el procedimiento se hizo originalmente mal y la primaria viajó por un USB de transporte ya conectado a `${WORK_HOST}`, considerar también el riesgo de que el material haya quedado en swap o caché — el escenario seguro es regenerar la llave entera, no solo limpiar el keystore.

#### B.2 Síntoma: una firma RFC 9580 no verifica en un cliente GnuPG ajeno

Comando de diagnóstico desde el lado del receptor:

```bash
gpg --verify <archivo.sig>
```

Salida típica:

```
gpg: Signature made <fecha>
gpg: using ... key <huella>
gpg: Can't check signature: No public key
```

O bien:

```
gpg: Note: signatures using the SHA1 algorithm are rejected
gpg: bad signature
```

Causa: el receptor usa GnuPG clásico, que no implementa RFC 9580 (decisión 2.1). No es un error de la llave; es la incompatibilidad declarada del estándar.

Remediación: el receptor puede instalar `gpg-from-sq` (en Debian) o usar otra implementación que soporte v6 (Sequoia, OpenPGP.js de Proton). Si el receptor no controla esto, la opción es haber emitido la llave en v4 desde el principio (regenerar con `--profile rfc4880`) — decisión que se toma al inicio del manual y no se cambia a posteriori sin rotación completa.

#### B.3 Síntoma: la passphrase es rechazada tras restaurar el backup

Diagnóstico:

```bash
# En el entorno de prueba con SEQUOIA_HOME apuntando al directorio aislado
sq key import /ruta/al/backup/full-export-YYYYMMDD.asc
# pide passphrase, devuelve "Bad password" o equivalente
```

Causas posibles, en orden de probabilidad:

1. **Confusión de passphrase**: el backup se hizo en el ciclo en que se cambió la passphrase de la primaria (operación que sq permite con `sq key password`). Probar con la passphrase anterior.
2. **Backup corrupto**: el archivo se copió mal o el medio tiene errores. Verificar con `sq inspect` que el archivo parsea como llave OpenPGP. Si `sq inspect` falla con error de parsing, el archivo está corrupto y hay que usar el backup anterior o el medio de revocación.
3. **Encoding del archivo**: si el backup se transfirió por canales que reescribieron line endings (cliente de correo, copiar-pegar a través de un editor), el armor ASCII puede haberse roto. El archivo debe empezar con `-----BEGIN PGP PRIVATE KEY BLOCK-----` y terminar con `-----END PGP PRIVATE KEY BLOCK-----`, sin caracteres añadidos.

Remediación de los tres casos: probar primero con la versión anterior del backup (`full-export-YYYYMMDD.asc` con fecha más antigua). Si todas fallan, asumir que el material está perdido y proceder a revocar con el certificado pre-generado (sección 11.2 Caso B).

#### B.4 Síntoma: el keyserver no acepta el certificado v6

Diagnóstico:

```bash
sq network keyserver publish --cert "${KEY_FP}"
```

Salida problemática (ejemplo):

```
Error: server rejected the upload
HTTP 400: malformed certificate
```

Causa: el keyserver al que se intenta subir no soporta RFC 9580. Algunos keyservers viejos no han actualizado su parser y rechazan certificados v6.

Remediación: usar `keys.openpgp.org` (que sí soporta v6) explícitamente:

```bash
sq network keyserver publish --cert "${KEY_FP}" --server hkps://keys.openpgp.org
```

Si el destinatario es un keyserver privado de una organización, escalar internamente para actualizar la implementación. Como solución temporal, distribuir el certificado por WKD (sección 15.2) o por canal directo.

#### B.5 Síntoma: `git commit -S` con el chameleon falla con "BAD signature" o "no public key"

Diagnóstico (los comandos dependen de cómo está activado el chameleon en la distro; ver 16.2):

```bash
# Verificar configuración de git
git config --get gpg.program           # debe devolver "gpg-sq" o "gpg" según activación
git config --get user.signingkey       # debe devolver el fingerprint

# Debian Camino 1
gpg-sq --list-secret-keys "${KEY_FP}"

# Debian Camino 2/3, Fedora con activate, Nix con shim
gpg --list-secret-keys "${KEY_FP}"
```

En cualquiera de las dos formas, la salida debe listar la subkey S como secreta.

Causas posibles:

1. **`gpg.program` no está configurado**: git está usando `gpg` clásico, que no encuentra la llave Sequoia. En Debian Camino 1, configurar con `git config --global gpg.program gpg-sq`. En Fedora/Nix con el chameleon activado, basta con que `gpg --version` reporte chameleon — verificar también que el shim esté en PATH para el usuario que ejecuta git.
2. **`user.signingkey` apunta a la primaria, no a la subkey**: el chameleon selecciona automáticamente la subkey S del certificado indicado, así que el fingerprint de la primaria es correcto. Verificar que sea el primario, no el de una subkey individual.
3. **La verificación se hace con `gpg` clásico que no soporta v6**: ver Anexo B.2.

Remediación del caso 3: cuando el verificador es GitHub, GitLab o similar, hay que subir el certificado público al servicio (en su sección "GPG keys") para que la firma se vea como verificada. GitHub adoptó soporte de v6 con cierto retraso; verificar el estado en la documentación de la plataforma al momento de subir.

#### B.6 Síntoma: una subkey expiró antes de rotarla

Diagnóstico:

```bash
sq inspect --cert "${KEY_FP}" | grep -i "expir"
```

Salida problemática:

```
Subkey: signing [S]
   Expiration time: 2026-05-13 (expired 12 days ago)
```

Causa: el recordatorio de renovación falló o no se actuó a tiempo. Las firmas hechas durante el período post-expiración pueden ser rechazadas por verificadores estrictos; las firmas previas a la expiración siguen siendo válidas.

Remediación: extender la expiración retroactivamente desde `${OFFLINE_HOST}` con `sq key subkey expire`. La nueva firma de binding cubre el período hacia adelante; el agujero entre la expiración previa y la nueva renovación queda como una zona donde las firmas pueden o no aceptarse según el verificador.

Procedimiento limpio: tras la renovación, propagar el certificado actualizado a `${WORK_HOST}` y a los keyservers urgentemente (sección 10.4 / 15.4).

#### B.7 Síntoma: una operación falla con "no secret key for primary" desde `${WORK_HOST}`

Diagnóstico:

```bash
sq key subkey add --cert "${KEY_FP}" --can-sign --expiration 1y
# Error: No secret key found for ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890ABCD1234EF567890
```

Causa: este es el comportamiento **correcto** del modelo offline (sección 8.4). Cualquier operación que requiera la primaria — añadir subkeys, añadir UserIDs, extender expiraciones, revocar componentes — solo se puede hacer en `${OFFLINE_HOST}`.

Remediación: no hay nada que arreglar en `${WORK_HOST}`. Llevar la operación al entorno offline siguiendo el flujo de la Etapa II (preparar sesión Tails, montar `${AIRGAP_MEDIA}`, operar) y propagar el certificado actualizado de vuelta (sección 9.3).

Si la operación se necesita con urgencia y `${AIRGAP_MEDIA}` no está disponible, no hay atajo: la primaria está donde está. Esta es una característica del modelo, no un defecto.

#### B.8 Síntoma: `gpg-agent` no ofrece la subkey A a `ssh`

Diagnóstico:

```bash
ssh-add -L      # debería listar la subkey A
echo $SSH_AUTH_SOCK     # debería apuntar al socket de gpg-agent
ssh -v usuario@servidor 2>&1 | grep -i offering
```

Salida problemática: `ssh-add -L` devuelve "The agent has no identities" o se queda colgado.

Causas posibles:

1. **`enable-ssh-support` no está en `gpg-agent.conf`**: añadirlo y reiniciar el agente (`gpgconf --kill gpg-agent`).
2. **`sshcontrol` no tiene el keygrip correcto**: obtenerlo con `gpg-sq --list-keys --with-keygrip "${KEY_FP}"` (Debian Camino 1) o `gpg --list-keys --with-keygrip "${KEY_FP}"` (chameleon activado), y verificar que es el de la subkey con flag `[A]`, no el de la primaria u otra subkey.
3. **`SSH_AUTH_SOCK` apunta al socket equivocado**: si el sistema tiene además un `ssh-agent` clásico corriendo (lo lanzan algunos entornos de escritorio automáticamente), `SSH_AUTH_SOCK` puede apuntar ahí. Forzarlo con `export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)` en `.bashrc` y reiniciar la sesión.

#### B.9 Síntoma: tras importar el cert actualizado en `${WORK_HOST}`, los cambios no aparecen

Diagnóstico:

```bash
sq cert import ${TRANSPORT}/${KEY_FP}.pub.asc
sq inspect --cert "${KEY_FP}"
# Los UserIDs o expiraciones nuevas no aparecen
```

Causas posibles:

1. **El archivo importado es viejo**: verificar la fecha del archivo en el USB de transporte (`stat` o `ls -la`). Si es anterior al cambio que se intentaba propagar, el export en `${OFFLINE_HOST}` no incluyó los cambios. Volver a `${OFFLINE_HOST}` y reexportar tras confirmar que `sq inspect` ahí sí muestra los cambios.
2. **El cert store tiene el certificado pero `sq inspect` lo lee desde otra fuente**: improbable, pero puede ocurrir si hay un `--keyring` apuntado a un archivo externo. Limpiar variables `PGP_CERT_D` y `SEQUOIA_HOME` y reintentar.
3. **Caché de cliente**: algunos clientes OpenPGP (Thunderbird, mutt configurados) cachean certificados internamente. Reiniciar el cliente o forzar refresh.

#### B.10 Síntoma: al cifrar contra un contacto traído del keyserver, sq advierte "not authenticated"

Diagnóstico:

```bash
sq encrypt --for "${CONTACT_FP}" --output /tmp/test.pgp /tmp/test.txt
# Warning: certificate ABCD... is not authenticated for "alice@example.com"
```

Causa: el certificado se trajo por `sq network search` o `sq cert import` desde un keyserver. Quedó firmado internamente por una shadow CA con confianza baja (1/120). Sequoia no considera autenticado el binding UserID↔llave hasta que se autentique explícitamente.

Remediación:

1. Obtener el fingerprint del contacto por un canal distinto del que se usó para traer la llave (tarjeta de presentación, llamada telefónica, mensaje firmado por otra llave ya autenticada).
2. Comparar con `sq inspect --cert "${CONTACT_FP}"`.
3. Si coincide: `sq pki link add --cert "${CONTACT_FP}" --all`.

Procedimiento completo en sección 18.3.

Si el escenario es "no me importa autenticar caso a caso, confío en keys.openpgp.org como fuente": ver sección 18.5 sobre elevar confianza de una shadow CA entera. Es decisión política, no operacional.

#### B.11 Síntoma: `gpg-sq` no lista la llave v6 ni siquiera con el fingerprint exacto

Diagnóstico:

```bash
gpg-sq --list-secret-keys 7945DBBE52E82B621CDFE2DC8BE7B42D9A842396A863A804E0B496854B2A79C8
# (salida vacía)

sq key list   # sí lista la llave correctamente
```

`sq` ve la llave, `gpg-sq` no. La diferencia es estructural: `sq` consulta `~/.local/share/pgp.cert.d/` y su keystore propio en `~/.local/share/sequoia/keystore/softkeys/`; `gpg-sq` (versiones empaquetadas en Debian 13 trixie a mayo 2026) consulta el cert-store de sq pero delega el material secreto al `gpg-agent` de GnuPG clásico, que vive en `~/.gnupg/private-keys-v1.d/`.

Causa: las dos infraestructuras coexisten pero no comparten material secreto cuando la llave es v6. GnuPG clásico rechaza el material v6 con `Invalid operation: Keygrip not defined for this kind of public key`, así que aunque se intente `gpg --import` del export de sq, el material no entra en `~/.gnupg/`.

Remediación: este es el escenario que la sección 16.1 documenta. La integración nativa `gpg-sq` ↔ llave v6 no funciona en Debian 13 hasta que GnuPG con soporte v6 completo llegue al ecosistema. Mientras tanto, usar el wrapper `git-sq-sign` documentado en 16.1.

#### B.12 Síntoma: `git commit -S` con llave v6 reporta "Unusable secret key"

Diagnóstico:

```bash
git commit -m "..."
# error: gpg failed to sign the data:
# [GNUPG:] KEY_CONSIDERED 7945DBBE... 0
# [GNUPG:] INV_SGNR 9 7945DBBE...
# gpg: skipped "7945DBBE...": Unusable secret key
# [GNUPG:] FAILURE sign 54
# gpg: signing failed: Unusable secret key
# fatal: falló al escribir el objeto commit
```

Causa: misma raíz que B.11. `gpg-sq` ve el certificado público de la llave v6 en el cert-store, pero al pedirle el material secreto al `gpg-agent` clásico, este responde que no lo tiene (porque GnuPG no entiende v6 y nunca lo importó). El chameleon traduce esto a `Unusable secret key`.

Verificación previa para confirmar la causa:

```bash
echo "test" | sq sign --signer "${KEY_FP}" --message -
# Si sq sí firma (pide passphrase y devuelve un mensaje firmado),
# confirmado: sq tiene el material, el problema es el puente gpg-sq → gpg-agent.
```

Remediación: instalar y configurar el wrapper `git-sq-sign` descrito en sección 16.1. Una vez configurado, `git commit -m "..."` debe firmar pidiendo la passphrase **en terminal** (no vía pinentry gráfico).

### Anexo C — Migración desde una llave GnuPG existente

Este anexo es para quien ya tiene una llave OpenPGP creada con GnuPG (v4, formato RFC 4880) y quiere moverse al modelo de este manual: Sequoia, perfil v6, primaria offline, subkeys en producción. No es una migración trivial — implica decisiones sobre identidad criptográfica y plazos de coexistencia.

#### C.1 Decisión previa: importar o regenerar

Hay dos caminos posibles, con trade-offs distintos.

**Camino 1 — Importar la llave existente a Sequoia.** La identidad criptográfica (fingerprint) se preserva. Los contactos que ya tienen tu certificado siguen pudiendo cifrar contra ti sin acción de su parte. Las firmas pasadas que terceros emitieron sobre tu llave (web of trust acumulado) se conservan.

Desventajas: la llave sigue siendo v4 (RFC 4880). No se gana la mejora criptográfica de v6, y este manual no aplica del todo (Sequoia opera sobre v4 sin problemas, pero las decisiones de diseño de la sección 2 quedan a medias).

**Camino 2 — Generar una llave v6 nueva y firmarla cruzada con la vieja.** La llave nueva es independiente. Durante un período de coexistencia, ambas son válidas y se publican; mensajes nuevos se firman con la v6, mensajes a contactos que aún no actualizaron pueden firmarse con la vieja. Eventualmente la vieja se revoca.

Desventajas: los contactos tienen que aprender que tienes una llave nueva. El web of trust acumulado en la vieja no se transfiere automáticamente (terceros tendrían que re-firmar la nueva). Operacionalmente más pesado.

La recomendación de este manual, en línea con la postura adoptada para RFC 9580: **Camino 2** si la llave vieja se usa para algo serio (firma de releases, identidad de mantenedor Debian, comunicación con periodistas) y se quiere aprovechar v6. **Camino 1** si la llave vieja se usa de forma casual y no vale el costo de transición.

#### C.2 Camino 1 — Import directo de la llave GnuPG

Desde el equipo donde GnuPG tiene la llave (puede ser `${WORK_HOST}` si la llave nunca fue offline antes):

```bash
# Exportar la llave secreta desde GnuPG
gpg --export-secret-keys --armor <FP-llave-gpg> > /tmp/gpg-key.asc

# Importarla en sq
sq key import /tmp/gpg-key.asc

# Borrar el intermedio
shred -u /tmp/gpg-key.asc
```

`sq key import` pide la passphrase de la llave (la misma que tenía en GnuPG) y la almacena en el keystore de Sequoia. Verificar:

```bash
sq key list
sq inspect --cert "<FP-llave-gpg>"
```

A partir de aquí, la llave funciona con sq como cualquier otra. Si se quiere mover al modelo offline, hay que: (a) generar nuevas subkeys con sq, (b) exportar la primaria a un medio airgapped, (c) eliminarla del keystore activo, (d) seguir el procedimiento de la Etapa III para tener solo subkeys en `${WORK_HOST}`. Es un trabajo de migración considerable; conviene planearlo en una sesión dedicada y no en producción.

> Cuando la llave vieja tenía subkeys creadas por GnuPG, esas subkeys se importan también. Pueden coexistir con subkeys nuevas creadas con sq. Si las subkeys viejas tenían algoritmos antiguos (RSA-2048, DSA), conviene rotarlas a Curve25519 siguiendo el procedimiento de la sección 12.

#### C.3 Camino 2 — Llave v6 nueva con firma cruzada

Plan general:

1. **Generar la llave v6 nueva** siguiendo las Etapas II y III completas de este manual. Resultado: llave Sequoia v6 con primaria offline y subkeys en producción.

2. **Firmar la nueva llave con la vieja**, declarando criptográficamente que la misma persona controla ambas:

   ```bash
   # Exportar el cert público de la llave nueva v6
   sq cert export --cert "${KEY_FP_NEW}" --output /tmp/new-cert.pub.asc

   # Firmarla con la llave vieja v4 desde GnuPG
   gpg --default-key <FP-llave-vieja> --sign-key <FP-llave-nueva>
   # (este comando opera sobre la copia en el keyring de gpg, no sobre el archivo)
   ```

   Hay que importar primero `new-cert.pub.asc` al keyring de `gpg` para que el comando funcione. Alternativamente, hacerlo desde el lado opuesto:

   ```bash
   # Firmar la llave vieja con la nueva, desde sq
   sq pki vouch add --certifier "${KEY_FP_NEW}" --cert "<FP-llave-vieja>"
   ```

   El resultado idealmente es: ambas llaves tienen una firma cruzada que dice "yo soy la otra". Esto permite a contactos verificar la transición criptográficamente. `sq pki vouch` produce una firma OpenPGP exportable, distinta de `sq pki link add` (que solo afecta al cert store local); la diferencia entre ambos comandos se explica en la sección 18.8.

3. **Período de coexistencia**: ambas llaves se publican (keyservers, WKD, perfiles), ambas se pueden usar para firmar y descifrar. El correo se sigue firmando con la vieja para que contactos no actualizados puedan verificar; commits nuevos se firman con la nueva. Duración sugerida: 6-12 meses.

4. **Comunicación a contactos**: enviar un mensaje firmado por ambas llaves anunciando la transición, incluyendo los dos fingerprints. Publicarlo también en perfiles públicos (sitio web, GitHub, mastodon) firmado por la llave nueva.

5. **Revocación de la llave vieja**: pasado el período de coexistencia, revocar la vieja con razón `superseded` y referencia explícita a la nueva en el mensaje:

   ```bash
   gpg --gen-revoke <FP-llave-vieja>
   # (mensaje: "Superseded by <FP-llave-nueva>, RFC 9580")
   gpg --import <archivo-revocacion>
   gpg --send-keys <FP-llave-vieja>
   ```

   A partir de aquí, los contactos que tengan la vieja recibirán la actualización de revocación al refrescar contra keyservers, y verán el mensaje apuntando a la nueva.

#### C.4 Material descifrable y backups históricos

Una migración no debe perder acceso a material cifrado contra la llave vieja. Antes de revocar:

- **Inventariar** lo que está cifrado contra la subkey E vieja: backups, correos archivados, archivos en repositorios privados.
- **Re-cifrar lo crítico** contra la subkey E nueva, conservando los originales firmados.
- **Conservar la llave vieja en cold storage** indefinidamente (no eliminarla del backup), incluso después de revocarla. La revocación impide su uso para nuevas operaciones, pero la llave secreta sigue siendo necesaria para descifrar material histórico.

Este detalle suele confundir: revocar no es lo mismo que borrar. La llave vieja, revocada, sigue viva en cold storage hasta el día en que se confirme que ya no hay material cifrado pendiente de leer.

---
[Generación y gestión de llaves OpenPGP v6 (RFC 9580) con Sequoia](https://github.com/noggalito/manuales/blob/main/llave-openpgp-v6.md) © 2026 by [Calú](https://github.com/calu777) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="CC" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="BY" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="SA" style="max-width: 1em;max-height:1em;margin-left: .2em;">
