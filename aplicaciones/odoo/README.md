# Despliegue de Odoo 19 Enterprise

Este manual describe los pasos necesarios para desplegar Odoo Enterprise 19 en Debian 13 usando Docker + Docker Swarm, con PostgreSQL, volúmenes persistentes, *secrets* y soporte para módulos Enterprise montados. No existe una imagen oficial "Odoo Enterprise"; la forma soportada es usar la imagen oficial de Odoo y montar los addons Enterprise en `/mnt/extra-addons`. Antes de iniciar, debe estar instalado Debian 13.6 (Trixie) con todas las actualizaciones, y los puertos 80 y 443 abiertos en el firewall.

<p style="text-align: center"><img src="../../assets/odoo.svg" style="width: 25%;" alt="Odoo" /></p>

**Autor:** [Calú](https://github.com/calu777)  
**Fecha de inicio:** 2026-07-22  
**Última actualización:** 2026-07-22  
**Versión:** 0.1

## a. Historial de cambios

- [0.1] – 2026-04-28
  - Manual terminado para la versión 19 de Odoo

## b. To-do

- Bla vla

## c. Requerimientos previos

- Debian 13.6 (Trixie) instalado, se sugiere seguir las instrucciones de [este manual](https://github.com/noggalito/manuales/blob/main/sistemas-operativos/debian/13-server.md).
- Acceso al [repositorio](https://github.com/odoo/enterprise) en Github de Odoo Enterprise.

---

## Índice


---


## 1. Instalar Docker Engine en Debian 13

Seguir el método oficial (repositorio APT de Docker) para Debian 13:

```bash
sudo apt update
sudo apt install -y ca-certificates curl unzip

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

CODENAME="$(. /etc/os-release && echo "$VERSION_CODENAME")"
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: ${CODENAME}
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt -y update && sudo apt -y dist-upgrade
sudo apt -y upgrade && sudo apt -y autoremove --purge
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Opcional (para no usar `sudo` con docker):

```bash
sudo usermod -aG docker "$USER"
newgrp docker
```

Verificar:

```bash
docker --version
docker compose version
```

---

## 2. Preparar la estructura del alojamiento

Se recomienda usar `/opt/odoo19-docker`:

```bash
sudo mkdir -p /opt/odoo19-docker/{config,logs,addons/{enterprise,custom},secrets,volumes/{odoo,db}}
sudo chown -R $USER:$USER /opt/odoo19-docker
```

Definir parámetros para los logs:

```bash
sudo vi /etc/docker/daemon.json
```

y luego:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "20m",
    "max-file": "5"
  }
}
```

---

## 3. Obtener los addons de Odoo Enterprise 19

Odoo documenta el repositorio `odoo/enterprise` como **colección de addons**:

```bash
cd /opt/odoo19-docker
git clone --branch 19.0 https://github.com/odoo/enterprise.git addons/enterprise
```

Usar un Personal Access Token (PAT) provisto por GitHub. También bajar los archivos propios listos desde el repositorio propio:

```bash
mkdir -p /tmp/odoo
curl -L https://github.com/noggalito/manuales/archive/refs/heads/main.zip -o /tmp/odoo/manuales.zip
unzip -q /tmp/odoo/manuales.zip -d /tmp/odoo
cp -a /tmp/odoo/manuales-main/aplicaciones/odoo/assets/. /opt/odoo19-docker/
rm -rf /tmp/odoo/
```

Luego inicializar Swarm (single-node):

```bash
IP=$(ip -4 route get 1.1.1.1 | awk '{for(i=1;i<=NF;i++) if ($i=="src") {print $(i+1); exit}}')
echo $IP
sudo docker swarm init --advertise-addr "$IP"
```

Comprobar:

```bash
docker node ls
```

---

## 4. Crear el secret REAL para la contraseña de Postgres

Crear una contraseña fuerte directamente sin dejarla en un archivo:

```bash
openssl rand -base64 32 | tr -d '\n' | docker secret create postgresql_password -
```

Verificar:

```bash
docker secret ls
```

Esto usa el mecanismo de secrets de Swarm (encriptado en el Raft log y montado en memoria dentro del contenedor). Luego, verificar que el secret tiene contenido:

```bash
docker secret inspect postgresql_password
```

---

## 5. Crear imagen personalizada con las dependencias Python

La imagen `odoo:19.0` no tiene las librerías necesarias para ciertos módulos Enterprise (como Contabilidad, Documentos, Firmas, etc.). Al intentar instalarlos sin esto, Odoo lanzará errores de "Module not found" o fallará silenciosamente. El archivo `Dockerfile` ya fue descargado previamente, así que simplemente se debe construir la imagen localmente. Antes de desplegar el stack, se debe "buildear" esta imagen en el servidor.

```bash
# El punto final indica el directorio actual
docker build -t odoo-enterprise:19.0 .
```

## 6. Validar el archivo Swarm Stack: `stack.yml`

El archivo ya está bajado en una descarga previa. Tomar en cuenta que:

* Odoo soporta `HOST/USER/PASSWORD` y también `PASSWORD_FILE` según la imagen oficial
* Swarm monta los secrets en `/run/secrets/<nombre>` dentro del contenedor

Para validar el archivo:

```bash
docker compose -f stack.yml config
```

En caso de ausencia de errores, el comando devuelve el archivo completo.


## 7. Crear la configuración `config/odoo.conf`

La imagen oficial usa `/etc/odoo/odoo.conf` y permite sobrescribirlo montando volúmenes. El archivo ya está bajado en una descarga previa; es necesario hacer los cambios de la contraseña u otros parámetros:

```bash
vi config/odoo.conf
```

Luego, crear algunos directorios y actualizar permisos:

```bash
sudo mkdir -p /opt/odoo19-docker/logs
sudo chown -R 101:101 /opt/odoo19-docker/logs
sudo chmod -R u+rwX /opt/odoo19-docker/logs
sudo chmod 775 /opt/odoo19-docker/logs
sudo mkdir -p /opt/odoo19-docker/addons/custom
sudo chown -R $USER:$USER /opt/odoo19-docker/addons
sudo chmod 755 /opt/odoo19-docker/addons /opt/odoo19-docker/addons/custom
```

En la imagen oficial de Odoo 19, el usuario odoo tiene UID 100 y GID 101 (verifica con `docker exec <CID> id odoo`).. Con esto le das permiso de escritura sin abrirlo a todo el mundo.

---

## 8. Desplegar el stack

Desde `/opt/odoo19-docker`:

```bash
docker stack deploy -c stack.yml odoo19
```

Verifica estado:

```bash
docker stack services odoo19
docker stack ps odoo19
```

Logs:

```bash
docker service logs -f odoo19_odoo
docker service logs -f odoo19_db
```

---

## 9. Entrar a Odoo y habilitar Enterprise

Habilitar **temporalmente** el puerto 8069 sólo para la IP pública que accede remotamente. En la PC remota de instalación:

```bash
curl -s https://api.ipify.org
```

Y luego en el servidor:

```bash
export MI_IP=x.x.x.x # Ingresar la IP pública obtenida líneas arriba
sudo ufw allow from "$MI_IP" to any port 8069 proto tcp comment 'Odoo temporal'
sudo ufw show added
sudo ufw enable
```

También habilitar el pueerto 8069 en el firewall. Luego, abrir en navegador:

* `http://IP_DEL_SERVIDOR:8069`

Luego:

1. Crea la base de datos.
2. Ve a **Apps** → actualiza lista (si aplica).
3. Instala **`web_enterprise`** (debe aparecer si `addons_path` apunta bien al Enterprise).

---

## 10. Registrar tu base de datos con tu suscripción Enterprise

En Odoo on-premise, registra la DB ingresando el **subscription code** en el banner del dashboard (cuando aparezca).

Estos son los comandos típicos para ver logs:

### 10.1 Ver servicios del stack

```bash
sudo docker stack services odoo19
```

### 10.2 Logs del servicio (lo más común)

En vivo (follow):

```bash
sudo docker service logs -f odoo19_odoo
sudo docker service logs -f odoo19_db
```

Últimas N líneas:

```bash
sudo docker service logs --tail 200 odoo19_odoo
sudo docker service logs --tail 200 odoo19_db
```

Con timestamps:

```bash
sudo docker service logs -f --timestamps odoo19_odoo
```

### 10.3 Ver tareas (para saber si reinicia / falla)

```bash
sudo docker service ps odoo19_odoo
sudo docker service ps odoo19_db
```

### 10.4 Logs por contenedor (si `service logs` sale vacío)

Saca el container ID del task activo del servicio y míralo directo:

```bash
CID=$(sudo docker ps -q --filter label=com.docker.swarm.service.name=odoo19_odoo | head -n1)
echo "$CID"
sudo docker logs -f "$CID"
```

Para DB:

```bash
CID=$(sudo docker ps -q --filter label=com.docker.swarm.service.name=odoo19_db | head -n1)
echo "$CID"
sudo docker logs -f "$CID"
```

### 10.5 Si estás escribiendo logs a archivo (bind mount)

Si montaste `/var/log/odoo`:

```bash
sudo tail -n 200 /opt/odoo19-docker/logs/odoo.log
sudo tail -f /opt/odoo19-docker/logs/odoo.log
```

---

## 11. Operación diaria útil (Swarm)

Reiniciar Odoo (sin tumbar DB):

```bash
docker service update --force odoo19_odoo
```

Actualizar imagen (misma versión 19.0, "rolling update"):

```bash
docker pull odoo:19.0
docker service update --image odoo:19.0 odoo19_odoo
```

Bajar todo:

```bash
docker stack rm odoo19
```

Salir de Swarm (si algún día quieres revertir):

```bash
docker swarm leave --force
```

---

## Recomendaciones rápidas de "producción"

* Mantén **1 réplica** de Odoo (si escalas, necesitas filestore compartido y balanceo con sesiones).
* No uses usuario Postgres superuser; y considera endurecer `list_db=False` cuando sea público.
* Para exponer a internet: ocultar el puerto 8069, pon **reverse proxy + TLS** y habilita `proxy_mode=True`.
* Implementar healthchecks y respaldos.

---

## 12. Desplegar Caddy como *reverse proxy* seguro para exponer Odoo en Internet

En este paso vamos a colocar Caddy delante de Odoo como *reverse proxy* con HTTPS automático, de modo que:

* El puerto 8069 no quede expuesto directamente a Internet.
* Todo el tráfico pase por HTTPS con certificados gestionados automáticamente (Let's Encrypt u otra ACME CA).
* Se cuente con *logs* y cabeceras de seguridad razonables por defecto.

Este paso asume que se tiene el *stack* `odoo19` funcionando, escuchando en `http://127.0.0.1:8069` y que en `odoo.conf` está activado `proxy_mode = True` (ya está en el archivo que preparamos).

---

### 12.1. Prerrequisitos

1. **DNS apuntando al servidor**

   Crea un registro DNS tipo **A** (o **AAAA** si se usa IPv6) para el dominio, por ejemplo:

   *`dominio1.com` → IP pública del servidor Debian*

   Se debe esperar a que la propagación DNS funcione (un `ping dominio1.com` debe resolver a la IP del servidor).

2. **Verificar que Odoo responde localmente**

   En el servidor:

   ```bash
   curl -I http://127.0.0.1:8069
   ```

   Se espera un `HTTP/1.0 200 OK` o un `303 SEE OTHER` (redirección al login).
   Si no responde, primero corregir el despliegue del stack antes de seguir.

---

### 12.2. Instalar Caddy en Debian 13

Instalaremos Caddy desde el repositorio oficial para distribuciones derivadas de Debian, lo que te da actualizaciones y configuración vía `systemd`.

```bash
sudo apt update
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Importar la clave del repositorio de Caddy
curl -fsSL https://dl.cloudsmith.io/public/caddy/stable/gpg.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

# Añadir el repo estable de Caddy
curl -fsSL https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install -y caddy
```

Esto instala el binario `caddy` en el sistema y un servicio `systemd` (`caddy.service`) habilitado automáticamente para iniciar en cada arranque.

Comprueba que está activo:

```bash
systemctl status caddy
```

---

### 12.3. Preparar directorios de *logs*

Por orden y para futuras auditorías, crea un directorio de *logs* específico con los permisos correctos para el usuario `caddy`:

```bash
sudo mkdir -p /var/log/caddy
sudo chown -R caddy:caddy /var/log/caddy
sudo chmod 750 /var/log/caddy
```

> **Importante:** El directorio `/var/log/caddy/` debe ser propiedad del usuario `caddy`. Si Caddy no puede escribir en él, el `reload` fallará con `permission denied` y el servicio quedará colgado hasta que expire el timeout de systemd. Si el problema ya ocurrió, usa `sudo systemctl restart caddy` (no `reload`) para recuperarte.

---

### 12.4. Configurar `/etc/caddy/Caddyfile` para Odoo

Edita el archivo principal de configuración:

```bash
sudo mv /etc/caddy/Caddyfile /etc/caddy/Caddyfile.backup
sudo vi /etc/caddy/Caddyfile
```

Configuración de producción para un dominio con Odoo:

```caddyfile
{
    # Correo de contacto para ACME (Let's Encrypt / ZeroSSL)
    email admin@midominio.com
}

dominio1.com {
    encode gzip zstd

    log {
        output file /var/log/caddy/dominio1.com.access.log {
            roll_size 10MiB
            roll_keep 10
            roll_keep_for 720h
        }
    }

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "SAMEORIGIN"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    # Websocket hacia el puerto evented de Odoo (gevent)
   @websocket {
       path /websocket
   }
    reverse_proxy @websocket 127.0.0.1:8072

    # Proxy principal hacia Odoo
    reverse_proxy 127.0.0.1:8069
}
```

> **Notas importantes:**
>
> * Usa siempre **`127.0.0.1`** explícito, no `localhost`. En Debian, `localhost` puede resolver a `::1` (IPv6) y si Odoo no escucha en IPv6, la conexión se cuelga silenciosamente sin error visible — Caddy registra las peticiones en el access log con `status:0` pero nunca obtiene respuesta del backend.
>
> * El matcher `@websocket` debe ir **antes** del `reverse_proxy` general. Caddy evalúa los matchers en orden y el proxy general capturaría las conexiones websocket si va primero, enviándolas al puerto 8069 (werkzeug) en lugar del 8072 (gevent), lo que causa `KeyError: 'socket'` en Odoo y satura los workers.
>
> * El bloque `@websocket` usa matching por path (`path /websocket`), no por header. El cliente de Odoo hace un GET inicial plano a /websocket sin cabeceras de upgrade, por lo que el matcher por header no lo detecta y el path-based matching es el que funciona de forma confiable.
>
> * La directiva `reverse_proxy` envía las cabeceras `X-Forwarded-*` automáticamente, que Odoo utiliza cuando `proxy_mode = True` está activo.

---

### 12.5. Validar la configuración y aplicar

Antes de recargar el servicio, formatea y valida que el Caddyfile es correcto:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
# Debe mostrar: "Valid configuration"
```

Si la validación es OK, aplica la configuración. Usa siempre `restart` en lugar de `reload` para evitar timeouts de systemd durante la obtención del certificado TLS:

```bash
sudo systemctl restart caddy
```

Observa la emisión del certificado en tiempo real:

```bash
sudo journalctl -u caddy -f
# Esperar: "certificate obtained successfully" para dominio1.com
```

Verifica HTTPS:

```bash
# Desde una máquina externa (no el propio servidor)
curl -I https://dominio1.com
# Debe responder HTTP/2 200
```

> **Por qué no probar desde el propio servidor:** El servidor no puede conectarse a sí mismo por su IP pública (hairpin NAT), común en entornos cloud (Azure, AWS, GCP). Las pruebas de HTTPS siempre deben hacerse desde una máquina externa.

---

### 12.6. Ajustar el firewall para seguridad

Una vez que Caddy está en producción, sólo los puertos 80 y 443 deben ser accesibles desde Internet. Instala UFW y configura las reglas:

```bash
sudo apt update
sudo apt install -y ufw
sudo vi /etc/default/ufw
# Asegúrate de que IPV6=yes

sudo ufw allow 80,443,17176/tcp
sudo ufw deny 8069/tcp
sudo ufw reload
sudo ufw enable
sudo ufw status numbered
```

> En entornos cloud (Azure, AWS, GCP) también debes configurar las reglas del Network Security Group (NSG) o Security Groups en el portal del proveedor para permitir los puertos 80 y 443 desde cualquier origen.

Con esto, cualquier cliente externo sólo podrá llegar a Odoo a través de Caddy.

---

### 12.7. Consideraciones adicionales de fiabilidad

1. **Revisar que Caddy está habilitado al arranque:**

   ```bash
   sudo systemctl is-enabled caddy
   # debe responder: enabled
   ```

2. **Monitorizar logs de Caddy y Odoo:**

   ```bash
   # Accesos de Caddy
   sudo tail -f /var/log/caddy/dominio1.com.access.log

   # Logs del sistema de Caddy
   sudo journalctl -u caddy -f

   # Logs de Odoo
   docker service logs -f odoo19_odoo
   ```

3. **Certificados y renovaciones:** Caddy renueva los certificados automáticamente antes de su expiración. Si hubiera problemas de DNS o firewall (puerto 80 bloqueado), lo verás en los logs de Caddy.

4. **Pruebas finales:**
   * Accede desde un navegador externo a `https://dominio1.com`.
   * Verifica que no puedes llegar a `http://TU_IP_PUBLICA:8069`.
   * Verifica que el certificado es válido y emitido por Let's Encrypt.
   * Verifica que el login de Odoo funciona con la URL del dominio.

---

## 13. Agregar dominios adicionales (Odoo multiempresa)

Odoo soporta múltiples sitios web, uno por empresa, cada uno con su propio dominio. Caddy identifica qué empresa servir basándose en el header `Host` que llega — no requiere ninguna configuración especial del lado de Odoo más allá de configurar el dominio en la app Sitio web.

### 13.1. Prerrequisitos

1. Haber creado la segunda empresa en Odoo y habilitado la app **Sitio web** para ella.
2. En Odoo: **Sitio web → Configuración → Propiedades del sitio web** → campo **Dominio** con el valor `https://dominio2.com` (sin barra final).
3. DNS del nuevo dominio apuntando a la misma IP pública del servidor.

Verifica que el DNS resuelve antes de continuar:

```bash
curl -s https://api.ipify.org                    # IP pública del servidor
curl -s "https://dns.google/resolve?name=dominio2.com&type=A" | grep -o '"data":"[^"]*"'
# Ambas IPs deben coincidir
```

Antes de cualquier cambio, respalda el archivo de configuración original:

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.backup
```

### 13.2. Agregar el bloque en el Caddyfile

Edita `/etc/caddy/Caddyfile` y agrega un bloque nuevo con la misma estructura que el existente. El archivo completo debe quedar así:

```caddyfile
calu@maracuya:~$ sudo cat /etc/caddy/Caddyfile
{
	email admin@dominio.com
}

dominio1.com {
	encode gzip zstd
	log {
		output file /var/log/caddy/dominio1.com.access.log {
			roll_size 10MiB
			roll_keep 10
			roll_keep_for 720h
		}
		format json
	}
	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "strict-origin-when-cross-origin"
	}
   @websocket {
       path /websocket
   }
	reverse_proxy @websocket 127.0.0.1:8072
	reverse_proxy 127.0.0.1:8069
}

dominio2.com {
	encode gzip zstd
	log {
		output file /var/log/caddy/dominio2.com.access.log {
			roll_size 10MiB
			roll_keep 10
			roll_keep_for 720h
		}
		format json
	}
	header {
		Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "strict-origin-when-cross-origin"
	}
   @websocket {
       path /websocket
   }
	handle /media/* {
		uri strip_prefix /media
		root * /opt/archivos/noggalito.com
		file_server
		header X-Robots-Tag "noindex, nofollow"
	}
	reverse_proxy @websocket 127.0.0.1:8072
	reverse_proxy 127.0.0.1:8069
}
```

> Cada bloque es independiente y Caddy obtiene un certificado TLS distinto para cada dominio automáticamente vía Let's Encrypt. No hay ningún cambio necesario en `odoo.conf`, en el stack de Docker ni en la red — Caddy pasa el header `Host` original al backend, que es lo que Odoo usa para el routing multiempresa.

### 13.3. Validar y aplicar

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
# Debe mostrar: "Valid configuration"

# Camiar los permisos para el usuario y grupo correctos
sudo chown -R caddy:caddy /var/log/caddy
sudo chmod 750 /var/log/caddy

sudo systemctl restart caddy
```

Observa la emisión del certificado para el nuevo dominio:

```bash
sudo journalctl -u caddy -f
# Esperar: "certificate obtained successfully" para dominio2.com
```

Verifica desde una máquina externa:

```bash
curl -I https://dominio2.com
# Debe responder HTTP/2 200
```

Luego cierra el puerto 8069 del firewall, ya no debe ser accesible directamente desde internet, solo a través de Caddy:

```bash
sudo ufw delete allow 8069/tcp
sudo ufw status numbered
```

### 13.4. Redirección www (opcional)

Si también apuntaste `www.dominio2.com` a la misma IP, agrega un bloque de redirección:

```caddyfile
www.dominio2.com {
    redir https://dominio2.com{uri} permanent
}
```

### 13.5. Cómo funciona internamente

```
Navegador → https://dominio2.com
    → Caddy recibe la petición con Host: dominio2.com
    → Caddy hace proxy a 127.0.0.1:8069 (conservando el Host header)
    → Odoo lee el Host header y sirve la empresa cuyo dominio coincide
```

---

## 14. Mini servidor de archivos

La forma más simple es hacerlo directamente con Caddy, sin tocar Odoo, agregando un bloque `file_server` en el Caddyfile.

Por ejemplo, si quieres servir archivos desde `/opt/archivos/${MIDOMINIO}` en la URL `https://${MIDOMINIO}/media/`:

```bash
export MIDOMINIO=dominio.com
sudo vi /etc/caddy/Caddyfile
```
y luego, antes de `reverse_proxy`

```caddyfile
dominio1.com {
    # ... tu configuración existente ...

    handle /media/* {
        uri strip_prefix /media
        root * /opt/archivos/dominio
        file_server
        header X-Robots-Tag "noindex, nofollow"
    }
}
```



Creas el directorio y pones los archivos ahí:

```bash
sudo mkdir -p /opt/archivos/${MIDOMINIO}
sudo cp mi_archivo.pdf /opt/archivos/${MIDOMINIO}

# Luego de copiar los archivos, se debe camabiar permisos
sudo chown -R caddy:caddy /opt/archivos/${MIDOMINIO}
sudo chmod -R 755 /opt/archivos/${MIDOMINIO}
```

Recarga Caddy para que tome los cambios del Caddyfile:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Y el archivo quedaría accesible en `https://${MIDOMINIO}/media/archivo.pdf`.

---

## Apéndice: Problemas conocidos y soluciones

### A.1. El `reload` de Caddy se cuelga (timeout de systemd)

**Síntoma:** `sudo systemctl reload caddy` no regresa el prompt. Después de ~90 segundos aparece `Reload operation timed out. Killing reload process.`

**Causa más común:** El directorio `/var/log/caddy/` no es propiedad del usuario `caddy`, por lo que Caddy no puede crear el archivo de log del nuevo dominio y falla al cargar la configuración.

**Solución:**

```bash
sudo chown -R caddy:caddy /var/log/caddy/
sudo systemctl restart caddy   # usar restart, no reload
```

### A.2. El sitio carga con `status:0` en el access log (sin respuesta del backend)

**Síntoma:** Caddy registra las peticiones en el access log con `"status":0` y `"size":0`. El navegador y curl se cuelgan hasta timeout. Los logs de Odoo no muestran ningún hit.

**Causa:** `localhost` resuelve a `::1` (IPv6) y Odoo solo escucha en IPv4 (`0.0.0.0:8069`). La conexión de Caddy al backend se cuelga silenciosamente.

**Solución:** Usar siempre `127.0.0.1` explícito en lugar de `localhost` en el Caddyfile:

```caddyfile
reverse_proxy 127.0.0.1:8069   # correcto
# reverse_proxy localhost:8069  # puede fallar
```

### A.3. `KeyError: 'socket'` en logs de Odoo — workers saturados

**Síntoma:** Los logs de Odoo muestran repetidamente `KeyError: 'socket'` en `websocket.py`. El sitio carga lento o no carga. Con el tiempo los workers quedan bloqueados.

**Causa:** Las conexiones websocket del navegador llegan al puerto 8069 (werkzeug) en lugar del 8072 (gevent). Werkzeug no sabe manejar websockets y falla, pero el worker queda ocupado.

**Causa raíz en el Caddyfile:** El matcher `@websocket` usa `path /websocket` (insuficiente) en lugar de matching por header, o el bloque `reverse_proxy` general está antes del matcher `@websocket`.

**Solución:** Usar matcher por header y respetar el orden:

```caddyfile
# CORRECTO: matcher por header, antes del proxy general
@websocket {
    header Connection *Upgrade*
    header Upgrade websocket
}
reverse_proxy @websocket 127.0.0.1:8072
reverse_proxy 127.0.0.1:8069

# INCORRECTO: matcher por path (insuficiente)
# @websocket {
#     path /websocket
# }
```

### A.4. El challenge de Let's Encrypt falla con timeout

**Síntoma:** El log de Caddy muestra `Timeout during connect (likely firewall problem)`.

**Causa:** Los puertos 80 y/o 443 no son accesibles desde internet. Puede ser el firewall del sistema operativo (UFW) o las reglas del proveedor de nube (NSG en Azure, Security Groups en AWS).

**Solución:**

```bash
# Verificar UFW
sudo ufw status

# Verificar que los puertos son accesibles (desde una máquina externa)
curl -v --connect-timeout 5 http://TU_IP_PUBLICA

# Reiniciar Caddy para reintentar
sudo systemctl restart caddy
```

### A.5. Error `relation "ai_embedding" does not exist`

**Síntoma:** El log muestra errores sobre la tabla `ai_embedding` al ejecutar crons.

**Causa:** El módulo `ai` de Odoo Enterprise requiere la extensión `pgvector` en PostgreSQL. La imagen oficial `postgres:17` no la incluye.

**Solución:** Si no usas funciones de AI/embeddings, desactiva los crons relacionados en **Ajustes → Técnico → Automatización → Acciones planificadas** (deshabilitar "AI Embedding: Generate Embeddings" y similares). Si necesitas la funcionalidad, usa una imagen PostgreSQL que incluya `pgvector`.

### A.6. El dump pesa pocos bytes (dump corrupto)

**Síntoma:** El dump `.dump` pesa menos de 1 KB o exactamente 127 bytes.

**Causa:** Se usó redirección `>` desde fuera del contenedor, lo que captura la salida del shell en lugar de la salida del comando dentro del contenedor.

**Solución:** Usar siempre `-f /tmp/archivo` dentro del contenedor y luego `docker cp`.


### A.7. Para retirar la frase brandind de Odoo

Insertar este código HTNL en la app Sitio web

```html
<script>
(function () {
    function removeBranding() {
        document.querySelectorAll(
            '.o_brand_promotion, .o_survey_brand_message'
        ).forEach(el => el.remove());
    }

    new MutationObserver(removeBranding).observe(document.documentElement, {
        childList: true,
        subtree: true
    });

    removeBranding();
})();
</script>
```

---

## Apéndice: Comandos de referencia rápida

```bash
# Ver estado del stack
docker service ls
docker stack ps odoo19

# Reiniciar el servicio Odoo (aplica cambios de odoo.conf)
docker service update --force odoo19_odoo

# Ver logs en tiempo real
docker service logs -f odoo19_odoo
docker service logs -f odoo19_db

# Ver logs de Caddy (sistema)
sudo journalctl -u caddy -f

# Ver logs de acceso de Caddy
sudo tail -f /var/log/caddy/dominio1.com.access.log

# Verificar el certificado TLS
curl -vI https://dominio1.com 2>&1 | grep -E "subject|issuer|expire"

# Verificar que Odoo responde al backend con un Host header específico
curl -I -H "Host: dominio2.com" http://127.0.0.1:8069

# Verificar permisos del directorio de logs de Caddy
ls -la /var/log/caddy/

# Verificar en qué interfaz escucha Odoo
sudo ss -tlnp | grep 8069
```

---

# Manual de Migración de Odoo entre Versiones con Docker

**Versión:** 1.0  
**Alcance:** Migración de Odoo N a Odoo N+1 en entornos Docker/Docker Swarm con reverse proxy Caddy y certificado TLS automático.

## Resumen de comandos
```bash
export CONTENEDOR_DB_ORIGEN=$(docker ps --format '{{.Names}}' | grep db)
export NOMBRE_BD=prod_v01
export USUARIO_PG=odoo
export DIR_PROYECTO_ORIGEN=/opt/odoo19-docker/
export MIRESPALDO=$(date +%Y-%m-%d_%H-%M-%S)-$(hostname)-$NOMBRE_BD.dump
docker exec $CONTENEDOR_DB_ORIGEN \
  pg_dump -Fc -U $USUARIO_PG $NOMBRE_BD \
  -f /tmp/$MIRESPALDO

docker exec $CONTENEDOR_DB_ORIGEN ls -lh /tmp/
docker cp $CONTENEDOR_DB_ORIGEN:/tmp/$MIRESPALDO \
  $DIR_PROYECTO_ORIGEN
ls -lh $DIR_PROYECTO_ORIGEN/$MIRESPALDO

export VOLUMEN_WEB_ORIGEN=$(docker volume ls --format '{{.Name}}' | grep web)
docker volume inspect $VOLUMEN_WEB_ORIGEN
MOUNTPOINT=$(docker volume inspect $VOLUMEN_WEB_ORIGEN \
  --format '{{.Mountpoint}}')
sudo tar -czf ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz \
  -C / \
  "${MOUNTPOINT#/}/filestore/${NOMBRE_BD}/"
tar -tzf ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz | head -5
ls -lh ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz

tar -czf "/home/calu/$(date +%Y-%m-%d_%H-%M-%S)-$(hostname)-respaldo.tar.gz" \
    /etc/caddy/Caddyfile \
    /opt/odoo19-docker/config/odoo.conf \
    /opt/archivos

sftp usuario@SERVIDOR_ORIGEN
cd /opt/odoo19-docker/
get DIR_PROYECTO_ORIGEN/NOMBRE_BD.dump /ruta/local/
get DIR_PROYECTO_ORIGEN/filestore_FECHA.tar.gz /ruta/local/
exit

docker exec $CONTENEDOR_DB_ORIGEN rm /tmp/$MIRESPALDO
cd /opt/odoo19-docker/
sudo rm 2026...
```

## Índice

1. [Prerequisitos y variables del entorno](#1-prerequisitos-y-variables-del-entorno)
2. [Fase 1: Backup del servidor origen](#2-fase-1-backup-del-servidor-origen)
3. [Fase 2: Migración de la base de datos](#3-fase-2-migración-de-la-base-de-datos)
4. [Fase 3: Preparación del servidor destino](#4-fase-3-preparación-del-servidor-destino)
5. [Fase 4: Restauración de la base de datos migrada](#5-fase-4-restauración-de-la-base-de-datos-migrada)
6. [Fase 5: Restauración del filestore](#6-fase-5-restauración-del-filestore)
7. [Fase 6: Configuración del reverse proxy con TLS](#7-fase-6-configuración-del-reverse-proxy-con-tls)
8. [Verificación final](#8-verificación-final)
9. [Problemas conocidos y soluciones](#9-problemas-conocidos-y-soluciones)

---

## 1. Prerequisitos y variables del entorno

### 1.1 Convenciones del manual

A lo largo de este documento se usan las siguientes variables simbólicas. Sustitúyelas por los valores reales de tu entorno antes de ejecutar cualquier comando.

| Variable | Descripción | Ejemplo | Comando de asignación |
|---|---|---|---|
| `SERVIDOR_ORIGEN` | Hostname o IP del servidor origen | `banano2` | `SERVIDOR_ORIGEN=$(hostname)` *(ejecutar en origen)* |
| `SERVIDOR_DESTINO` | Hostname o IP del servidor destino | `maracuya` | `SERVIDOR_DESTINO=$(hostname)` *(ejecutar en destino)* |
| `NOMBRE_BD` | Nombre de la base de datos Odoo | `prod_v01` | `docker exec $CONTENEDOR_DB_DESTINO psql -U $USUARIO_PG -l` *(listar y elegir)* |
| `USUARIO_PG` | Usuario PostgreSQL de Odoo | `odoo` | `USUARIO_PG=$(cat stack.yml \| grep POSTGRES_USER \| awk -F': ' '{print $2}')` |
| `VERSION_ORIGEN` | Versión de Odoo origen | `18.0` | Manual — ver en Settings → About |
| `VERSION_DESTINO` | Versión de Odoo destino | `19.0` | Manual — ver en Settings → About |
| `CONTENEDOR_DB_ORIGEN` | Nombre del contenedor PostgreSQL en origen (Docker Compose) | `nombre_db` | `CONTENEDOR_DB_ORIGEN=$(docker ps --format '{{.Names}}' \| grep db)` *(en origen)* |
| `CONTENEDOR_ODOO_ORIGEN` | Nombre del contenedor Odoo en origen (Docker Compose) | `nombre_odoo` | `CONTENEDOR_ODOO_ORIGEN=$(docker ps --format '{{.Names}}' \| grep odoo)` *(en origen)* |
| `CONTENEDOR_DB_DESTINO` | Nombre del contenedor PostgreSQL en destino (Docker Swarm) | `odoo19_db.1.xxx` | `CONTENEDOR_DB_DESTINO=$(docker ps --format '{{.Names}}' \| grep db)` *(en destino)* |
| `CONTENEDOR_ODOO_DESTINO` | Nombre del contenedor Odoo en destino (Docker Swarm) | `odoo19_odoo.1.xxx` | `CONTENEDOR_ODOO_DESTINO=$(docker ps --format '{{.Names}}' \| grep odoo)` *(en destino)* |
| `STACK_DESTINO` | Nombre del stack Docker Swarm destino | `odoo19` | `STACK_DESTINO=$(docker stack ls --format '{{.Name}}')` |
| `VOLUMEN_WEB_ORIGEN` | Nombre del volumen web-data origen | `source_odoo-web-data` | `VOLUMEN_WEB_ORIGEN=$(docker volume ls --format '{{.Name}}' \| grep web)` *(en origen)* |
| `VOLUMEN_WEB_DESTINO` | Nombre del volumen web-data destino | `odoo19_odoo-web-data` | `VOLUMEN_WEB_DESTINO=$(docker volume ls --format '{{.Name}}' \| grep web)` *(en destino)* |
| `MOUNTPOINT_DESTINO` | Ruta del mountpoint del volumen web destino | `/var/lib/docker/volumes/...` | `MOUNTPOINT_DESTINO=$(docker volume inspect $VOLUMEN_WEB_DESTINO --format '{{.Mountpoint}}')` |
| `DIR_PROYECTO_ORIGEN` | Directorio del proyecto en origen | `/opt/odoo-v18/` | Manual — depende de tu instalación |
| `DIR_PROYECTO_DESTINO` | Directorio del proyecto en destino | `/opt/odoo19-docker/` | Manual — depende de tu instalación |
| `DOMINIO` | Dominio público del servicio | `mi-empresa.com` | Manual — definido en tu DNS |
| `EMAIL_ADMIN` | Email para certificado Let's Encrypt | `admin@mi-empresa.com` | Manual — definido en Caddyfile |

> Los comandos con `grep` asumen que solo hay un stack/volumen/contenedor de Odoo. Si hay varios, ajusta el filtro para ser más específico.


### 1.2 Herramientas necesarias

En el servidor origen:
- Docker con el stack de Odoo origen en funcionamiento
- Acceso al usuario con permisos `sudo` o `docker`

En el servidor destino:
- Docker con modo Swarm inicializado (`docker swarm init`)
- Caddy instalado como servicio systemd
- Puertos 80 y 443 abiertos en el firewall y en el proveedor de nube
- DNS del dominio apuntando a la IP pública del servidor destino

En la máquina de trabajo (opcional):
- Cliente `sftp` o `scp` para transferir archivos entre servidores

---

## 2. Fase 1: Backup del servidor origen

### 2.1 Dump de la base de datos

El dump debe hacerse **dentro del contenedor** usando `-f` y luego copiarse al host. La redirección `>` desde fuera del contenedor produce dumps corruptos.

```bash
# En SERVIDOR_ORIGEN
# Generar dump dentro del contenedor
MIRESPALDO=$(date +%Y-%m-%d_%H-%M-%S)-$(hostname)-$NOMBRE_BD.dump
docker exec $CONTENEDOR_DB_ORIGEN \
  pg_dump -Fc -U $USUARIO_PG $NOMBRE_BD \
  -f /tmp/$MIRESPALDO

# Para listarlo
docker exec $CONTENEDOR_DB_ORIGEN ls -lh /tmp/

# Copiar dump al host
docker cp $CONTENEDOR_DB_ORIGEN:/tmp/$MIRESPALDO \
  $DIR_PROYECTO_ORIGEN

# Verificar que el dump no es corrupto (debe ser > 1 MB)
ls -lh $DIR_PROYECTO_ORIGEN/$MIRESPALDO
```

> **Importante:** El formato `-Fc` (custom) es requerido por la web de upgrade.odoo.com. Si el archivo pesa pocos bytes (< 1 KB), el dump falló.

### 2.2 Backup del filestore

El filestore contiene imágenes, adjuntos y otros archivos binarios referenciados por la base de datos.

```bash
# En SERVIDOR_ORIGEN
# Verificar la ruta del filestore
docker volume inspect $VOLUMEN_WEB_ORIGEN

# La ruta Mountpoint del volumen contiene el filestore en:
# MOUNTPOINT/filestore/NOMBRE_BD/

# Crear el tar del filestore (preservando la ruta completa)
MOUNTPOINT=$(docker volume inspect $VOLUMEN_WEB_ORIGEN \
  --format '{{.Mountpoint}}')

sudo tar -czf ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz \
  -C / \
  "${MOUNTPOINT#/}/filestore/${NOMBRE_BD}/"

# Verificar contenido del tar
tar -tzf ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz | head -5
ls -lh ${DIR_PROYECTO_ORIGEN}/${MIRESPALDO}_filestore.tar.gz
```

> **Anota la ruta interna del tar** (el output de `head -5`). La necesitarás en la Fase 5 para calcular los `--strip-components`.

### 2.3 Transferir backups a la máquina de trabajo (opcional)

Si los servidores no tienen conectividad directa entre sí, descarga los backups a la máquina local:

```bash
# En la máquina de trabajo
sftp usuario@SERVIDOR_ORIGEN
get DIR_PROYECTO_ORIGEN/NOMBRE_BD.dump /ruta/local/
get DIR_PROYECTO_ORIGEN/filestore_FECHA.tar.gz /ruta/local/
exit
```

### 2.4 Borrar los respaldos generados dentro del contenedor

```bash
docker exec $CONTENEDOR_DB_ORIGEN rm /tmp/$MIRESPALDO
```

---

## 3. Fase 2: Migración de la base de datos

La migración del esquema se realiza a través del servicio oficial de Odoo.

### 3.1 Subir el dump a upgrade.odoo.com

1. Acceder a [https://upgrade.odoo.com](https://upgrade.odoo.com) con la cuenta de partner o cliente Enterprise.
2. Crear una nueva solicitud de migración:
   - **Versión origen:** `VERSION_ORIGEN`
   - **Versión destino:** `VERSION_DESTINO`
   - **Archivo:** subir `NOMBRE_BD.dump`
   - **Opción:** marcar "This database was moved" si el servidor cambia, o "This is a copy" si es una prueba.
   - **Neutralize:** activar solo si es una base de prueba (desactiva emails salientes, etc.)
3. Esperar a que el proceso termine. Para bases pequeñas (~20 MB) tarda pocos minutos; para bases grandes puede tardar horas.
4. Descargar el archivo resultante (formato `.zip`).

> **No descomprimas el zip.** El database manager de Odoo acepta el zip directamente.

---

## 4. Fase 3: Preparación del servidor destino

Esta fase asume que ya tienes un `stack.yml` y un `Dockerfile` para la nueva versión. Solo se documentan los pasos de inicialización.

### 4.1 Inicializar Docker Swarm (si no está activo)

```bash
# En SERVIDOR_DESTINO
docker swarm init
```

### 4.2 Crear el secret de PostgreSQL

```bash
# Generar y registrar el secret
openssl rand -base64 32 | docker secret create postgres_password -
```

### 4.3 Construir la imagen personalizada

```bash
# En SERVIDOR_DESTINO, dentro de DIR_PROYECTO_DESTINO
docker build -t odoo-enterprise:VERSION_DESTINO .
```

### 4.4 Desplegar el stack

```bash
docker stack deploy -c stack.yml STACK_DESTINO
```

### 4.5 Verificar que los servicios están corriendo

```bash
docker service ls
# Ambos servicios deben mostrar 1/1

docker service logs STACK_DESTINO_odoo --tail 30
# Debe mostrar "HTTP service running" y "Evented Service running"
```

---

## 5. Fase 4: Restauración de la base de datos migrada

### 5.1 Subir el zip migrado al servidor destino

```bash
# En la máquina de trabajo (o en SERVIDOR_ORIGEN)
sftp usuario@SERVIDOR_DESTINO
put /ruta/local/ARCHIVO_MIGRADO.zip /tmp/
exit
```

### 5.2 Restaurar via database manager web

1. Abrir `http://IP_SERVIDOR_DESTINO:8069/web/database/manager` en el navegador.
   > Usa la IP directa (no el dominio) mientras Caddy aún no está configurado.
2. Seleccionar **Restore Database**.
3. En el campo **File**, subir el archivo `ARCHIVO_MIGRADO.zip`.
4. En **Database Name**, escribir `NOMBRE_BD`.
5. Seleccionar la opción apropiada ("This database was moved" o "This is a copy").
6. Hacer clic en **Continue** y esperar a que termine.

### 5.3 Verificar la restauración

```bash
docker service logs ${STACK_DESTINO}_odoo --tail 20
# Debe mostrar: "RESTORE DB: NOMBRE_BD"
# Y luego: "loading 3XX modules..."
```

Acceder a `http://IP_SERVIDOR_DESTINO:8069/odoo` e iniciar sesión para confirmar que la base de datos cargó correctamente.

---

## 6. Fase 5: Restauración del filestore

### 6.1 Subir el tar al servidor destino

```bash
sftp usuario@SERVIDOR_DESTINO
put /ruta/local/filestore_FECHA.tar.gz /tmp/
exit
```

### 6.2 Determinar la ruta interna del tar

```bash
# En SERVIDOR_DESTINO
tar -tzf /tmp/filestore_FECHA.tar.gz | head -5
```

El output mostrará algo como:
```
var/lib/docker/volumes/VOLUMEN_WEB_ORIGEN/_data/filestore/NOMBRE_BD/
var/lib/docker/volumes/VOLUMEN_WEB_ORIGEN/_data/filestore/NOMBRE_BD/ab/
...
```

Cuenta la cantidad de componentes de ruta hasta llegar al directorio `NOMBRE_BD/` (sin incluirlo). En el ejemplo anterior son 6 componentes: `var`, `lib`, `docker`, `volumes`, `VOLUMEN_WEB_ORIGEN`, `_data`, `filestore`, `NOMBRE_BD` → son 7, pero como queremos el **contenido** de `NOMBRE_BD/`, el valor de `--strip-components` es la cantidad de directorios que preceden a los archivos reales.

> **Regla:** cuenta los `/` en la ruta del directorio raíz del tar. Ese número es el valor de `--strip-components`.

### 6.3 Verificar el mountpoint del volumen destino

```bash
docker volume inspect $VOLUMEN_WEB_DESTINO
# Anota el valor de "Mountpoint"
MOUNTPOINT_DESTINO=$(docker volume inspect $VOLUMEN_WEB_DESTINO \
  --format '{{.Mountpoint}}')
echo $MOUNTPOINT_DESTINO
```

### 6.4 Extraer el filestore

#### 6.4.1 Regenerar el tar del filestore en el servidor origen (si aún no lo hiciste así)

Si el tar de origen se generó con ruta absoluta (estructura interna tipo `var/lib/docker/volumes/.../_data/filestore/NOMBRE_BD/...`), re-genéralo con ruta plana antes de transferirlo. En el servidor origen:

```bash
# Obtener el mountpoint del volumen del filestore
MOUNTPOINT_ORIGEN=$(docker volume inspect $VOLUMEN_WEB_ORIGEN \
  --format '{{.Mountpoint}}')
echo "Mountpoint origen: $MOUNTPOINT_ORIGEN"

# Cambiar al directorio que contiene filestore/<NOMBRE_BD>/
cd ${MOUNTPOINT_ORIGEN}/filestore/

# Crear el tar de modo que su ruta interna empiece con NOMBRE_BD/
sudo tar -czf /tmp/filestore_FECHA.tar.gz ${NOMBRE_BD}/

# Verificar la estructura interna — debe mostrar entradas del tipo:
#   NOMBRE_BD/
#   NOMBRE_BD/ab/
#   NOMBRE_BD/ab/ab1234...
tar -tzf /tmp/filestore_FECHA.tar.gz | head -5
```

#### 6.4.2 Verificar el mountpoint del volumen destino

```bash
MOUNTPOINT_DESTINO=$(docker volume inspect $VOLUMEN_WEB_DESTINO \
  --format '{{.Mountpoint}}')
echo "Mountpoint destino: $MOUNTPOINT_DESTINO"
```

#### 6.4.3 Extraer el filestore

```bash
# Extraer dentro del directorio filestore/ del volumen destino
# (se creará automáticamente el subdirectorio NOMBRE_BD/)
sudo mkdir -p ${MOUNTPOINT_DESTINO}/filestore/
sudo tar -xzf /tmp/filestore_FECHA.tar.gz \
  -C ${MOUNTPOINT_DESTINO}/filestore/
```

> **Nota:** si el tar de origen ya se generó con ruta absoluta y no es posible regenerarlo, el comando alternativo es:
>
> ```bash
> sudo tar -xzf /tmp/filestore_FECHA.tar.gz \
>   --strip-components=N \
>   -C ${MOUNTPOINT_DESTINO}/filestore/
> ```
>
> donde `N` es el **número de componentes de ruta que preceden al directorio `NOMBRE_BD/` dentro del tar**. Calcúlalo así:
>
> ```bash
> tar -tzf /tmp/filestore_FECHA.tar.gz | grep -m1 "${NOMBRE_BD}/$"
> ```
>
> Cuenta los `/` a la izquierda de `NOMBRE_BD/` en la primera línea que aparezca; ese número es `N`. Ejemplo: si la salida es `var/lib/docker/volumes/VOL/_data/filestore/NOMBRE_BD/`, hay 6 barras antes de `NOMBRE_BD/` → `N=6`.

#### 6.4.4 Asignar permisos correctos

Dentro de la imagen oficial de Odoo el usuario `odoo` es UID/GID 101. Los archivos extraídos suelen quedar como `root:root`; debes normalizarlos para que Odoo pueda gestionar el filestore (p. ej. al ejecutar garbage collection de `ir_attachment`):

```bash
# Verificar primero el UID/GID correcto del usuario odoo dentro de la imagen
CID=$(sudo docker ps -q --filter label=com.docker.swarm.service.name={STACK_DESTINO}_odoo | head -n1)
sudo docker exec $CID id odoo
# Output esperado para Odoo 19: uid=100(odoo) gid=101(odoo)

sudo chown -R 100:101 ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/

sudo find ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/ -type d -exec chmod 755 {} \;
sudo find ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/ -type f -exec chmod 644 {} \;
```

#### 6.4.5 Verificar la extracción

Comprueba que los archivos están presentes y compara el conteo con el servidor origen:

```bash
# En SERVIDOR_DESTINO
sudo ls ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/ | head -10
sudo find ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/ -type f | wc -l

# Confirmar que NO existe una ruta anidada duplicada
# (este comando debe devolver vacío; si imprime algo, la extracción quedó mal)
sudo find ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/filestore 2>/dev/null
```

> Compara el conteo con el del servidor origen para confirmar que la transferencia fue completa:
> ```bash
> # En SERVIDOR_ORIGEN
> find $(docker volume inspect $VOLUMEN_WEB_ORIGEN --format '{{.Mountpoint}}')/filestore/${NOMBRE_BD}/ -type f | wc -l
> ```

Una diferencia significativa entre ambos conteos, o la existencia de una ruta `.../filestore/NOMBRE_BD/filestore/...`, indican que la extracción quedó mal. En ese caso, limpia el destino y repite desde 6.4.3:

```bash
sudo rm -rf ${MOUNTPOINT_DESTINO}/filestore/${NOMBRE_BD}/
```

#### 6.4.6 Validación funcional post-extracción

El conteo de archivos no detecta archivos vacíos (0 bytes) ni adjuntos registrados en la BD que apuntan a archivos inexistentes en el filestore. Una vez que Odoo esté levantado (tras el paso 6.5), corre esta verificación para detectar desajustes entre `ir_attachment.store_fname` y el filestore físico:

```bash
CID_DB=$(sudo docker ps -q --filter label=com.docker.swarm.service.name=${STACK_DESTINO}_db | head -n1)
CID_ODOO=$(sudo docker ps -q --filter label=com.docker.swarm.service.name=${STACK_DESTINO}_odoo | head -n1)

sudo docker exec $CID_DB psql -U $USUARIO_PG -d $NOMBRE_BD -tAc "
SELECT store_fname FROM ir_attachment 
WHERE store_fname IS NOT NULL
" | while read fname; do
  sudo docker exec $CID_ODOO test -f "/var/lib/odoo/filestore/${NOMBRE_BD}/$fname" \
    || echo "FALTA: $fname"
done
echo "=== Verificación terminada ==="
```

Si no imprime ningún "FALTA:", el filestore está íntegro y consistente con la BD. Si imprime faltantes, corrobora si están en una ruta anidada huérfana y usa `rsync` para moverlos a la ubicación correcta.



### 6.5 Reiniciar el servicio Odoo para recargar

```bash
docker service update --force ${STACK_DESTINO}_odoo
```

---

## 7. Fase 6: Configuración del reverse proxy con TLS

### 7.1 Prerequisito: DNS y firewall

Antes de configurar Caddy, asegúrate de que:

```bash
# El DNS del dominio apunta a la IP pública del servidor
curl -s https://api.ipify.org         # IP pública del servidor
curl -s "https://dns.google/resolve?name=DOMINIO&type=A" | grep -o '"data":"[^"]*"'
# Ambos deben coincidir

# Los puertos 80 y 443 están abiertos en el firewall
# (el método varía según proveedor: UFW, reglas de Azure/AWS/GCP, etc.)
```

### 7.2 Configurar el Caddyfile

Editar `/etc/caddy/Caddyfile`:

```caddyfile
DOMINIO {
    # Proxy principal hacia Odoo
    reverse_proxy localhost:8069

    # Proxy del websocket hacia el puerto evented de Odoo
    @websocket {
        path /websocket
    }
    reverse_proxy @websocket localhost:8072

    # Logs de acceso (opcional)
    log {
        output file /var/log/caddy/DOMINIO.access.log
    }
}
```

> Si tienes tanto `DOMINIO` como `www.DOMINIO`, agrega ambos al bloque o crea un bloque de redirección:
> ```caddyfile
> www.DOMINIO {
>     redir https://DOMINIO{uri} permanent
> }
> ```

### 7.3 Validar y recargar Caddy

```bash
# Formatear el Caddyfile
sudo caddy fmt --overwrite /etc/caddy/Caddyfile

# Validar la configuración
sudo caddy validate --config /etc/caddy/Caddyfile
# Debe mostrar "Valid configuration"

# Recargar (si Caddy ya estaba corriendo) o iniciar
sudo systemctl reload caddy
# o si es el primer inicio:
sudo systemctl restart caddy
```

### 7.4 Verificar la emisión del certificado TLS

```bash
sudo journalctl -u caddy -f
# Esperar el mensaje: "certificate obtained successfully"
```

Si el challenge falla con `Timeout during connect (likely firewall problem)`:
- Verificar que el puerto 80 y 443 están abiertos.
- Reiniciar Caddy después de abrir los puertos: `sudo systemctl restart caddy`.

### 7.5 Verificar HTTPS

```bash
curl -I https://DOMINIO
# Debe responder con HTTP/2 200 o una redirección al login de Odoo
```

---

## 8. Verificación final

### 8.1 Checklist post-migración

- [ ] `https://DOMINIO` carga el login de Odoo con candado verde (TLS válido)
- [ ] El login funciona con las credenciales del sistema origen
- [ ] Los módulos instalados aparecen en Ajustes → Módulos instalados
- [ ] Las imágenes, logos y avatares se muestran correctamente (filestore OK)
- [ ] Los documentos adjuntos se pueden abrir
- [ ] Las notificaciones en tiempo real funcionan (sin errores de websocket en consola del navegador)
- [ ] Los crons se ejecutan correctamente (revisar logs)
- [ ] El correo saliente está configurado correctamente (si aplica)

### 8.2 Comparar conteo de archivos del filestore

```bash
# En SERVIDOR_DESTINO
sudo find ${MOUNTPOINT_DESTINO}/filestore/NOMBRE_BD/ -type f | wc -l
```

Comparar con el conteo del servidor origen. Una diferencia significativa indica una transferencia incompleta.

### 8.3 Revisar logs en busca de errores críticos

```bash
docker service logs STACK_DESTINO_odoo --tail 50 2>&1 | grep -E "ERROR|CRITICAL" | grep -v websocket | grep -v ai_embedding
```

Los errores de `websocket` y `ai_embedding` son esperados y no críticos (ver sección 9).

---

## 9. Problemas conocidos y soluciones

### 9.1 Error de websocket: `Couldn't bind the websocket`

**Síntoma:** El log muestra `RuntimeError: Couldn't bind the websocket. Is the connection opened on the evented port (8072)?` de forma repetida.

**Causa:** Caddy está enviando las peticiones de websocket al puerto 8069 (werkzeug) en lugar del 8072 (gevent). Esto ocurre cuando el bloque `@websocket` no está configurado en el Caddyfile.

**Solución:** Agregar el bloque de websocket al Caddyfile (ver sección 7.2) y recargar Caddy.

### 9.2 Error `relation "ai_embedding" does not exist`

**Síntoma:** El log muestra errores sobre la tabla `ai_embedding` al ejecutar crons.

**Causa:** El módulo `ai` de Odoo Enterprise requiere la extensión `pgvector` en PostgreSQL. La imagen oficial `postgres:17` no la incluye.

**Solución:**
- Si no usas funciones de AI/embeddings, desactiva los crons relacionados en Ajustes → Técnico → Automatización → Acciones planificadas (deshabilitar "AI Embedding: Generate Embeddings" y similares).
- Si necesitas la funcionalidad, usa una imagen PostgreSQL que incluya `pgvector`.

### 9.3 `FileNotFoundError` en el filestore

**Síntoma:** El log muestra múltiples errores `FileNotFoundError: No such file or directory` para archivos en `/var/lib/odoo/filestore/NOMBRE_BD/`.

**Causa:** El tar del filestore estaba incompleto, o los `--strip-components` incorrectos hicieron que los archivos se extrajeron en la ruta equivocada.

**Solución:**
1. Verificar la ruta de extracción con `sudo ls` (requiere sudo por permisos Docker).
2. Comparar el conteo de archivos con el origen (ver sección 8.2).
3. Si es necesario, repetir la extracción limpiando primero el destino.

### 9.4 El dump pesa pocos bytes (dump corrupto)

**Síntoma:** El dump `.dump` pesa menos de 1 KB o exactamente 127 bytes.

**Causa:** Se usó redirección `>` desde fuera del contenedor, lo que captura la salida del shell en lugar de la salida del comando dentro del contenedor.

**Solución:** Usar siempre `-f /tmp/archivo` dentro del contenedor y luego `docker cp` (ver sección 2.1).

### 9.5 El challenge de Let's Encrypt falla con timeout

**Síntoma:** El log de Caddy muestra `Timeout during connect (likely firewall problem)` para los challenges `http-01` y `tls-alpn-01`.

**Causa:** Los puertos 80 y/o 443 no son accesibles desde internet. Puede ser el firewall del sistema operativo (UFW, iptables) o las reglas del proveedor de nube.

**Solución:**
1. Abrir los puertos 80 y 443 en todos los niveles de firewall.
2. Verificar con un servicio externo que los puertos son accesibles.
3. Reiniciar Caddy: `sudo systemctl restart caddy`.

### 9.6 El database manager web no responde al subir el zip

**Síntoma:** La petición POST a `/web/database/restore` hace timeout.

**Causa:** El zip puede ser grande y el timeout del servidor es corto, o el servicio Odoo está ocupado.

**Solución:** Es un falso negativo. Revisar los logs con `docker service logs STACK_DESTINO_odoo` — si aparece `RESTORING DB: NOMBRE_BD` seguido de `RESTORE DB: NOMBRE_BD`, la restauración fue exitosa.

### 9.7 AssetsLoadingError / PermissionError al editar posts o páginas

**Síntoma:** El editor del website/blog falla con `UncaughtPromiseError > 
AssetsLoadingError`. Los logs de Odoo muestran:
  - `PermissionError: [Errno 13] Permission denied: '/var/lib/odoo/filestore/
     NOMBRE_BD/checklist/XX/...'`
  - Peticiones `GET /web/assets/...` devolviendo HTTP 500.

**Causa:** El ownership del filestore fue asignado a un UID incorrecto durante
la migración. La imagen oficial de Odoo 19 usa UID 100 para el usuario odoo
(no 101 como versiones anteriores).

**Solución:**
```bash
    # Verificar UID real del usuario odoo en el contenedor
    docker exec <CID_odoo> id odoo   # debe decir uid=100(odoo) gid=101(odoo)
    
    # Corregir ownership
    MOUNT=$(docker volume inspect VOLUMEN_ODOO --format '{{.Mountpoint}}')
    sudo chown -R 100:101 $MOUNT/filestore/NOMBRE_BD/
    
    # Regenerar el checklist (staging del GC)
    sudo rm -rf $MOUNT/filestore/NOMBRE_BD/checklist
    sudo mkdir -p $MOUNT/filestore/NOMBRE_BD/checklist
    sudo chown 100:101 $MOUNT/filestore/NOMBRE_BD/checklist
    
    # Borrar bundles de assets corruptos para regenerar
    docker exec <CID_db> psql -U odoo -d NOMBRE_BD -c \
      "DELETE FROM ir_attachment WHERE url LIKE '/web/assets/%';"
    
    # Limpiar caché del navegador y probar
```
---

## Apéndice: Comandos de referencia rápida

```bash
# Ver estado del stack
docker service ls
docker stack ps STACK_DESTINO

# Reiniciar el servicio Odoo (aplica cambios de odoo.conf)
docker service update --force STACK_DESTINO_odoo
# por ejemplo: docker service update --force odoo19_odoo

# Ver logs en tiempo real
docker service logs -f STACK_DESTINO_odoo
docker service logs -f STACK_DESTINO_db

# Ver logs de Caddy
sudo journalctl -u caddy -f
sudo tail -f /var/log/caddy/DOMINIO.access.log

# Backup rápido desde el database manager web
# POST a: http://IP:8069/web/database/backup
# (También disponible desde la interfaz web del database manager)

# Verificar el certificado TLS
curl -vI https://DOMINIO 2>&1 | grep -E "subject|issuer|expire"
```

---

## Archivos y directorios a respaldar:

* /etc/caddy/Caddyfile
* /opt/archivos
* /opt/odoo19-docker/config/odoo.conf
* /var/log/caddy
---

[Despliegue de Odoo 19 Enterprise](https://github.com/noggalito/manuales/tree/main/aplicaciones/odoo) © 2026 by [Calú](https://github.com/calu777) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">
