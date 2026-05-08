# Pokédex Angular - Documentación del Proyecto y Publicación

**Autor:** Yeison Buelvas

## 1. Resumen ejecutivo

Este repositorio documenta el despliegue público y seguro de la aplicación **Pokédex Angular**, una web estática desarrollada con Angular para consultar especies Pokémon, estadísticas base, textos de la Pokédex y recursos visuales obtenidos desde servicios externos como **PokéAPI**, **PokéAPI GraphQL** y **assets.pokemon.com**.

La solución fue publicada usando la alternativa de **servidor propio expuesto a internet**, apoyada en **DuckDNS**, **Nginx**, **Let's Encrypt** y **UFW**, cumpliendo con el objetivo académico de dejar la aplicación disponible desde una URL pública y con encabezados HTTP de seguridad verificables.

## 2. Datos principales de la entrega

| Elemento | Valor |
|---|---|
| Nombre del proyecto | Pokédex Angular |
| Tipo de aplicación | Sitio web estático (HTML, CSS, JavaScript) |
| Framework base | Angular |
| URL pública | https://mipokedex.duckdns.org/ |
| Repositorio | https://github.com/yeisonbue/pokedex-angular |
| Plataforma elegida | Servidor propio en Kali Linux expuesto a internet |
| DNS dinámico | DuckDNS |
| Servidor web | Nginx |
| Certificado TLS | Let's Encrypt |
| Firewall | UFW |
| Resultado en Security Headers | A |

## 3. Contexto del caso de estudio

Pueblo Paleta Inc. solicitó a PumasLab el despliegue seguro y documentado de la aplicación Pokédex en una infraestructura pública. Dado que la actividad permitía dos caminos principales —Azure for Students o un servidor propio con Dynamic DNS— se seleccionó la segunda opción para demostrar control operativo completo sobre:

- publicación del build estático,
- exposición a internet mediante el router doméstico,
- configuración manual de cabeceras HTTP,
- emisión de certificados TLS,
- endurecimiento básico del servidor web.

Esta decisión permitió evidenciar competencias de despliegue real, resolución de incidencias y seguridad web aplicada.

## 4. Plataforma elegida y justificación

La solución se desplegó sobre un equipo propio con **Kali Linux** corriendo como servidor web. La elección se hizo por las siguientes razones:

1. Permitía publicar una aplicación estática sin necesidad de servicios PaaS adicionales.
2. Hacía posible controlar directamente Nginx, el firewall y el ciclo de certificados.
3. Facilitaba implementar encabezados HTTP de seguridad personalizados.
4. Aportaba una experiencia más completa de DevOps, al involucrar DNS dinámico, port forwarding y validación externa.

## 5. Paso a paso de creación de la solución pública elegida

Aunque no se utilizó una cuenta de nube tradicional como Azure, sí fue necesario crear y preparar los servicios públicos requeridos para exponer la aplicación.

### 5.1 Creación del repositorio en GitHub

1. Se inició sesión en GitHub.
2. Se creó un repositorio llamado **pokedex-angular**.
3. Se configuró el repositorio remoto del proyecto local:

```bash
git remote add origin https://github.com/yeisonbue/pokedex-angular
git branch -M main
git push -u origin main
```

4. El repositorio quedó disponible para alojar el código fuente y la documentación de la entrega.

### 5.2 Registro y uso de DuckDNS

DuckDNS fue el servicio público elegido para asociar un subdominio estable a una IP pública dinámica.

1. Se ingresó a https://www.duckdns.org/
2. Se inició sesión con un proveedor de identidad compatible.
3. Se creó el subdominio **mipokedex**.
4. El sistema asignó la URL pública:

```text
https://mipokedex.duckdns.org/
```

5. Se obtuvo el token de actualización del dominio.
6. Se automatizó la actualización de IP pública mediante un script en Kali Linux ejecutado con `cron`.

> Importante: por seguridad, el token real no debe publicarse en documentación ni repositorios públicos.

## 6. Configuraciones iniciales y parámetros aplicados

Para dejar trazabilidad clara del despliegue, estos fueron los parámetros principales configurados durante la publicación:

| Componente | Parámetro | Valor aplicado |
|---|---|---|
| Build Angular | `base-href` | `/` |
| DuckDNS | Subdominio | `mipokedex.duckdns.org` |
| Servidor | Web root | `/var/www/pokedex` |
| Nginx | `server_name` | `mipokedex.duckdns.org` |
| Router | Interfaz WAN | `ip2` |
| Router | Puertos reenviados | `80` y `443` hacia `192.168.1.9` |
| Firewall | Puertos permitidos | `80/tcp`, `443/tcp`, `OpenSSH` |
| Certbot | Método de validación | `webroot` |
| Certbot | Webroot | `/var/www/pokedex` |

Estos valores muestran exactamente dónde y cómo se aplicaron los parámetros operativos del despliegue, aun cuando la solución no depende de variables de entorno tradicionales como en plataformas PaaS.

**Evidencia:** [Ver captura de la configuración de port mapping en el router](https://drive.google.com/file/d/1MKOpaW02ilWm5HA9LvEUu8el87RsdcKD/view?usp=sharing)

## 7. Arquitectura de la solución

```text
Usuario en Internet
        |
        v
mipokedex.duckdns.org
        |
        v
IP pública del router
        |
        v
Port forwarding 80/443
        |
        v
Kali Linux (192.168.1.9)
        |
        v
Nginx -> /var/www/pokedex
```

## 8. Seguridad implementada

Se configuraron los siguientes controles de seguridad en Nginx:

| Encabezado | Objetivo |
|---|---|
| `Content-Security-Policy` | Restringir orígenes permitidos para scripts, estilos, imágenes, fuentes y conexiones |
| `Strict-Transport-Security` | Forzar navegación sobre HTTPS |
| `X-Content-Type-Options: nosniff` | Evitar interpretación insegura de tipos MIME |
| `X-Frame-Options: DENY` | Bloquear carga del sitio en iframes |
| `Referrer-Policy: no-referrer` | Reducir fuga de información en solicitudes salientes |
| `Permissions-Policy` | Deshabilitar APIs del navegador no necesarias |

Además:

- se habilitó **TLS** con Let's Encrypt,
- se configuró redirección automática de **HTTP a HTTPS**,
- se abrió únicamente lo necesario en el firewall (`80`, `443`, `OpenSSH`),
- se validó la renovación automática del certificado.

**Evidencia:** [Ver captura de los headers y la configuración aplicada](https://drive.google.com/file/d/1H7J3lz1SiNSZkvHuRXbMuU6iDoUJpZSI/view?usp=sharing)

## 9. Resultado del despliegue

La aplicación quedó accesible públicamente en:

```text
https://mipokedex.duckdns.org/
```

El análisis en **securityheaders.com** arrojó una calificación **A**, confirmando una mejora significativa en la postura de seguridad del sitio.

**Evidencia:** [Ver captura del resultado en Security Headers](https://drive.google.com/file/d/12P8wyjJ2WpCKw3AYr51RS3V792eSzMtO/view?usp=drive_link)

### Auditoría adicional de TLS con SSL Labs

Como validación complementaria de la capa HTTPS, se realizó una auditoría con **Qualys SSL Labs** sobre `https://mipokedex.duckdns.org/`.

Resultado resumido:

| Elemento | Resultado |
|---|---|
| Herramienta | SSL Labs |
| Calificación TLS | **A+** |
| Protocolos habilitados | TLS 1.2 y TLS 1.3 |
| HSTS | Presente |
| RC4 | No soportado |
| Heartbleed / POODLE / FREAK / LOGJAM / BEAST | No detectadas |

Hallazgos principales:

1. La capa TLS quedó endurecida con una configuración moderna y sin protocolos obsoletos.
2. El sitio presenta **compatibilidad limitada con clientes muy antiguos**, lo cual es esperable al restringir el servicio a TLS modernos.
3. La mejora pendiente más importante sigue estando en la **CSP del navegador**, no en la capa TLS.

**Evidencia:** [Ver captura de la auditoría en SSL Labs](https://drive.google.com/file/d/1tNjd3UznMcyKDm74IBwDplBvmzQZsWik/view?usp=drive_link)

### Advertencia restante para llegar a A+

La única observación importante restante es el uso de `'unsafe-inline'` dentro de la política de `script-src` en la **Content-Security-Policy**. Este punto está asociado a un script inline heredado en `index.html`. Para obtener **A+**, se debe:

1. eliminar ese script inline si ya no es necesario en el entorno Nginx, o
2. reemplazar la excepción por un `nonce` o un `sha256` específico.

## 10. Evidencias propias de la entrega

Como respaldo del proceso realizado, se incorporaron evidencias propias del despliegue y de las validaciones finales:

| Evidencia | Enlace |
|---|---|
| Captura del resultado en Security Headers | [Ver captura](https://drive.google.com/file/d/12P8wyjJ2WpCKw3AYr51RS3V792eSzMtO/view?usp=drive_link) |
| Captura de auditoría adicional SSL Labs | [Ver captura](https://drive.google.com/file/d/1tNjd3UznMcyKDm74IBwDplBvmzQZsWik/view?usp=drive_link) |
| Captura de headers/configuración aplicada | [Ver captura](https://drive.google.com/file/d/1H7J3lz1SiNSZkvHuRXbMuU6iDoUJpZSI/view?usp=sharing) |

## 11. Archivos de documentación incluidos

Este repositorio entrega dos documentos principales:

- **README.md**: visión general del proyecto, solución elegida, arquitectura, seguridad y resultados.
- **Despliegue.md**: procedimiento técnico detallado, comandos, configuración, incidentes y soluciones aplicadas.

## 12. Reflexión técnica

Este ejercicio permitió comprobar que el despliegue y la seguridad web no son actividades separadas. Publicar un sitio en internet sin HTTPS, sin firewall y sin políticas de seguridad adecuadas expone al sistema a riesgos evitables. Entre los aprendizajes principales destacan:

1. **La accesibilidad pública no es suficiente**: una aplicación puede estar “en línea” y aun así ser insegura.
2. **Los encabezados HTTP sí impactan la seguridad real**: ayudan a reducir vectores comunes como XSS, clickjacking y fuga de información.
3. **La operación de infraestructura importa**: detalles como el port forwarding, el DNS dinámico y la renovación automática del certificado son críticos para la continuidad del servicio.
4. **Los problemas reales rara vez son lineales**: durante el despliegue aparecieron incidencias de paquetes, validación de correo para Certbot, red doméstica, compatibilidad de políticas CSP y decisiones de endurecimiento TLS.

## 13. Referencias

- Angular: https://angular.io/
- PokéAPI: https://pokeapi.co/
- DuckDNS: https://www.duckdns.org/
- Let's Encrypt: https://letsencrypt.org/
- Security Headers: https://securityheaders.com/
- Nginx: https://nginx.org/
- UFW: https://wiki.ubuntu.com/UncomplicatedFirewall

## 14. Estado final

La entrega cumple con los objetivos principales del caso:

- aplicación publicada en una URL pública,
- documentación técnica detallada,
- HTTPS funcional,
- cabeceras de seguridad implementadas,
- resultado verificable en una herramienta externa de análisis.

Para el detalle completo del procedimiento, revisar **Despliegue.md**.
