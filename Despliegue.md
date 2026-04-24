# Despliegue de Pokédex Angular en servidor propio con DuckDNS, Nginx y HTTPS

## 1. Objetivo

Documentar de forma completa el proceso técnico seguido para publicar la aplicación **Pokédex Angular** en internet usando un servidor propio en **Kali Linux**, con nombre público dinámico en **DuckDNS**, servicio web con **Nginx**, protección mediante **UFW** y cifrado **HTTPS** con **Let's Encrypt**.

Este documento incluye:

- comandos ejecutados,
- configuraciones aplicadas,
- decisiones técnicas,
- errores encontrados,
- soluciones implementadas,
- validaciones finales del entorno.

## 2. Información general del despliegue

| Elemento | Valor |
|---|---|
| Aplicación | Pokédex Angular |
| Tipo de despliegue | Sitio estático servido por Nginx |
| Dominio público | `mipokedex.duckdns.org` |
| URL final | `https://mipokedex.duckdns.org/` |
| Sistema operativo del servidor | Kali Linux |
| Servidor web | Nginx |
| Resolución DNS dinámica | DuckDNS |
| Certificados | Let's Encrypt |
| Firewall | UFW |
| IP local del servidor durante el despliegue | `192.168.1.9` |
| Interfaz WAN usada en el router | `ip2` |
| Calificación en Security Headers | `A` |

## 3. Arquitectura de publicación

```text
Build Angular
   |
   v
/var/www/pokedex
   |
   v
Nginx en Kali Linux
   |
   v
UFW permite 80/443
   |
   v
Router doméstico reenvía puertos a 192.168.1.9
   |
   v
DuckDNS apunta a la IP pública
   |
   v
Usuarios acceden por https://mipokedex.duckdns.org/
```

## 4. Pre-requisitos

Antes del despliegue se consideraron los siguientes elementos:

1. Proyecto Angular disponible localmente.
2. Build de producción generado.
3. Servidor Kali Linux con acceso a internet.
4. Router doméstico con posibilidad de configurar **port forwarding**.
5. Subdominio activo en DuckDNS.
6. Puertos 80 y 443 disponibles.
7. Permisos de administración sobre el sistema Linux y el router.

## 5. Verificación de red local

El primer paso fue identificar la IP local del servidor para usarla en el router.

```bash
ip -4 a | grep 'inet ' | grep -v 127
```

Resultado observado:

```text
192.168.1.9
```

Esta IP fue la usada después como destino interno para el reenvío de puertos.

## 6. Preparación del sistema

Se verificó que Nginx estuviera activo y luego se instalaron las herramientas necesarias.

```bash
sudo apt update
sudo apt install -y nginx certbot ufw
```

### Observación importante

En la guía inicial se intentó instalar:

```bash
python3-certbot-nginx
```

Sin embargo, en la versión de Kali utilizada ese paquete **no tenía candidato de instalación**. La solución fue usar:

```bash
certbot
```

y emitir el certificado mediante el modo **webroot**, evitando depender del plugin de Nginx.

## 7. Build de la aplicación Angular

El proyecto ya contaba con el código fuente y dependencias instaladas. Para un despliegue en la raíz del dominio se utiliza el siguiente build:

```bash
npm run build -- --configuration production --base-href=/
```

El resultado se genera en:

```text
dist/pokedex-angular/
```

## 8. Copia del build al directorio público

Se creó el directorio web y se copiaron los archivos compilados:

```bash
sudo mkdir -p /var/www/pokedex
sudo cp -r /home/kali/Desktop/pokedex-angular/dist/pokedex-angular/* /var/www/pokedex/
sudo chown -R www-data:www-data /var/www/pokedex
```

Con esto, Nginx pudo servir el contenido estático desde una ubicación estable del sistema.

## 9. Configuración de DuckDNS

Para mantener actualizado el dominio frente a cambios de IP pública, se creó un script de actualización automática.

### 9.1 Script de actualización

Archivo:

```text
/home/kali/duckdns/duck.sh
```

Contenido funcional:

```bash
#!/bin/bash
DOMAIN="mipokedex"
TOKEN="TU_TOKEN_AQUI"

IP=$(curl -s https://api.ipify.org)
RESULT=$(curl -s "https://www.duckdns.org/update?domains=${DOMAIN}&token=${TOKEN}&ip=${IP}")
echo "$(date) | IP: ${IP} | Resultado: ${RESULT}" >> ~/duckdns/duck.log
```

### 9.2 Automatización con cron

```bash
crontab -e
```

Entrada configurada:

```cron
*/5 * * * * /home/kali/duckdns/duck.sh >/dev/null 2>&1
```

### 9.3 Incidencia encontrada

Durante la revisión se detectó que la línea del `cron` había sido colocada erróneamente **dentro del script `duck.sh`**. Eso se corrigió separando responsabilidades:

- el script quedó solo con lógica Bash,
- la planificación quedó únicamente en `crontab`.

## 10. Configuración inicial de Nginx sobre HTTP

Primero se preparó una configuración básica para responder por HTTP y servir el sitio.

Archivo del sitio:

```text
/etc/nginx/sites-available/pokedex
```

Se desactivó el sitio por defecto:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Se activó el nuevo sitio:

```bash
sudo ln -sf /etc/nginx/sites-available/pokedex /etc/nginx/sites-enabled/pokedex
```

Validación:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 11. Apertura de puertos en el firewall

Se configuró UFW con el mínimo necesario:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow OpenSSH
sudo ufw --force enable
```

Con esto quedaron habilitados:

- HTTP (`80/tcp`)
- HTTPS (`443/tcp`)
- administración remota por SSH

## 12. Port forwarding en el router

La aplicación no fue accesible públicamente hasta abrir el paso desde el router hacia el servidor interno.

### 12.1 Identificación de la interfaz WAN correcta

En el panel del router se revisaron las interfaces disponibles:

| Interfaz | Estado | Tipo | Dirección |
|---|---|---|---|
| `ip2` | Up | PPPoE | `179.33.194.56` |
| `ip3` | Up | DHCP | `11.94.142.195` |
| `ip4` | Up | DHCP | `10.124.191.45` |

Se eligió **`ip2`** porque coincidía con la IP pública utilizada por DuckDNS.

### 12.2 Reglas creadas

#### Regla HTTP

| Campo | Valor |
|---|---|
| Adm.State | Enable |
| Interface | `ip2` |
| Protocol | TCP |
| Remote IP | vacío o `0.0.0.0` |
| External port | `80` a `80` |
| Internal IP | `192.168.1.9` |
| Internal port | `80` |

#### Regla HTTPS

| Campo | Valor |
|---|---|
| Adm.State | Enable |
| Interface | `ip2` |
| Protocol | TCP |
| Remote IP | vacío o `0.0.0.0` |
| External port | `443` a `443` |
| Internal IP | `192.168.1.9` |
| Internal port | `443` |

### 12.3 Comportamiento observado

Antes del port forwarding, el dominio no respondía desde fuera. Tras aplicar las reglas, el sitio quedó accesible por HTTP y fue posible continuar con el certificado TLS.

## 13. Emisión del certificado HTTPS

Una vez disponible el dominio públicamente por HTTP, se generó el certificado con Certbot usando `webroot`.

```bash
sudo certbot certonly \
  --webroot \
  -w /var/www/pokedex \
  -d mipokedex.duckdns.org \
  -m correo2yb@gmail.com \
  --agree-tos \
  --no-eff-email \
  --non-interactive
```

Ubicación del certificado:

```text
/etc/letsencrypt/live/mipokedex.duckdns.org/fullchain.pem
/etc/letsencrypt/live/mipokedex.duckdns.org/privkey.pem
```

### Incidencia encontrada

El primer intento falló porque se usó un correo de ejemplo inválido. Let's Encrypt rechazó el registro del ACME account. La solución fue repetir el comando con un correo válido.

## 14. Configuración final de Nginx con redirección y seguridad

Una vez emitido el certificado, se dejó una configuración final con:

- redirección de HTTP a HTTPS,
- soporte TLS,
- reglas para SPA con `try_files`,
- encabezados de seguridad.

Configuración aplicada:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name mipokedex.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/pokedex;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name mipokedex.duckdns.org;

    root /var/www/pokedex;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/mipokedex.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mipokedex.duckdns.org/privkey.pem;

    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; img-src 'self' data: https://assets.pokemon.com; font-src 'self' data: https://fonts.gstatic.com; connect-src 'self' https://pokeapi.co https://beta.pokeapi.co; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; upgrade-insecure-requests" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()" always;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

## 15. Verificaciones realizadas

### 15.1 Validación local del sitio

```bash
curl -I http://localhost
```

### 15.2 Validación de redirección HTTP -> HTTPS

```bash
curl -I http://mipokedex.duckdns.org
```

Resultado esperado:

```text
HTTP/1.1 301 Moved Permanently
Location: https://mipokedex.duckdns.org/
```

### 15.3 Validación de encabezados sobre HTTPS

```bash
curl -I https://mipokedex.duckdns.org
```

Se verificó la presencia de:

- `Content-Security-Policy`
- `Strict-Transport-Security`
- `X-Content-Type-Options`
- `X-Frame-Options`
- `Referrer-Policy`
- `Permissions-Policy`

### 15.4 Prueba de renovación del certificado

```bash
sudo certbot renew --dry-run
```

Resultado observado:

```text
Congratulations, all simulated renewals succeeded
```

## 16. Resultado del escaneo de seguridad

Se evaluó el sitio en:

```text
https://securityheaders.com/
```

Resultado obtenido:

```text
Calificación A
```

Encabezados detectados por el escáner:

- `Content-Security-Policy`
- `Strict-Transport-Security`
- `X-Content-Type-Options`
- `X-Frame-Options`
- `Referrer-Policy`
- `Permissions-Policy`

## 17. Auditoría adicional con SSL Labs

Como verificación complementaria de la seguridad HTTPS, se ejecutó una auditoría sobre el dominio público usando **Qualys SSL Labs**.

### 17.1 Resultado general

| Campo | Valor |
|---|---|
| Host evaluado | `mipokedex.duckdns.org` |
| IP evaluada | `179.33.194.56` |
| Resultado global | **A+** |
| Protocolos detectados | TLS 1.2, TLS 1.3 |
| HSTS | Presente |
| Forward Secrecy | Activa |

### 17.2 Hallazgos técnicos

Hallazgos positivos confirmados por SSL Labs:

1. El servidor no expone SSLv3, TLS 1.0 ni TLS 1.1.
2. No se detectaron vulnerabilidades clásicas como **Heartbleed**, **POODLE**, **FREAK**, **LOGJAM** o **BEAST**.
3. No se observó soporte para **RC4**.
4. La política **HSTS** fue reconocida correctamente.
5. La capa TLS obtuvo una calificación **A+**, superior incluso a la nota alcanzada en Security Headers, que evalúa otra dimensión de seguridad.

Hallazgos de compatibilidad:

- SSL Labs reportó fallos de handshake en clientes muy antiguos, por ejemplo:
  - Android 2.3.7 y varias versiones 4.x antiguas
  - Internet Explorer 6, 7, 8 y 10
  - Java 6u45 y 7u25
  - Safari 5.1.9 y 6.0.4

Este hallazgo no implica una vulnerabilidad; representa una **consecuencia del endurecimiento moderno de TLS**.

### 17.3 Mejora evaluada durante la auditoría

Se intentó habilitar **OCSP stapling** en Nginx como mejora adicional. Sin embargo, el servidor reportó que el certificado actual no publica una URL de responder OCSP, por lo que esa opción fue ignorada. La configuración fue revertida para evitar advertencias innecesarias.

### 17.4 Mejoras recomendadas a partir de la auditoría

1. **Mantener la configuración actual de TLS**, ya que el resultado A+ confirma un nivel de endurecimiento adecuado.
2. **Eliminar `'unsafe-inline'` de la CSP** para mejorar la seguridad del lado navegador y aspirar a A+ en Security Headers.
3. **Definir explícitamente un conjunto más reducido de suites TLS 1.2** si se desea un endurecimiento criptográfico más estricto, evaluando el impacto en compatibilidad.
4. **Valorar si se requiere compatibilidad con clientes heredados**; en caso afirmativo, habría que estudiar una estrategia específica, asumiendo posibles concesiones de seguridad.

## 18. Errores encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| El paquete `python3-certbot-nginx` no estaba disponible | Diferencia de paquetes en Kali | Se usó `certbot` con validación `webroot` |
| El dominio no respondía desde fuera | Faltaba port forwarding en el router | Se abrieron puertos 80 y 443 hacia `192.168.1.9` |
| El script DuckDNS tenía una línea de cron dentro del archivo Bash | Error de configuración manual | Se limpió `duck.sh` y se movió la programación a `crontab` |
| Certbot rechazó el primer intento | Correo inválido para el registro ACME | Se repitió el comando con un correo válido |
| El sitio obtenía calificación B inicialmente | No había redirección a HTTPS ni política final completa | Se agregó TLS, redirección y más encabezados |
| El sitio quedó en A y no en A+ | La CSP mantiene `'unsafe-inline'` en `script-src` | Se documentó como mejora futura remover o hashear el script inline |

## 19. Consideraciones de seguridad

La aplicación quedó razonablemente protegida para el alcance de la actividad, pero es importante reconocer que publicar un servidor doméstico en internet implica riesgos adicionales:

1. La IP pública del hogar queda expuesta.
2. Un error de configuración del router puede abrir servicios no deseados.
3. El sistema debe mantenerse actualizado.
4. El firewall debe limitarse a los puertos estrictamente necesarios.
5. Los secretos, como el token de DuckDNS, no deben compartirse.

## 20. Mejoras futuras

Para elevar el resultado de **A** a **A+**, la mejora más directa es ajustar la CSP:

1. eliminar el script inline heredado en `index.html`, o
2. reemplazar `'unsafe-inline'` con un `nonce` o un `sha256`.

También se podrían incorporar:

- escaneo con **OWASP ZAP**,
- seguimiento periódico con **SSL Labs**,
- monitoreo del servicio,
- endurecimiento extra de TLS y logging.

## 21. Conclusión

El despliegue fue completado con éxito. La aplicación quedó publicada en internet, accesible mediante HTTPS y protegida con cabeceras de seguridad relevantes. Además del resultado funcional, el proceso permitió resolver problemas reales de infraestructura, compatibilidad de paquetes, DNS dinámico, apertura de puertos y configuración segura de Nginx.

En términos académicos y técnicos, la solución demuestra una integración efectiva entre **desarrollo frontend**, **operación de infraestructura**, **seguridad web** y **documentación profesional**.
