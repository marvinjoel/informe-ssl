# INFORME TÉCNICO DE INCIDENCIA Y RESOLUCIÓN SSL

**Proyecto:** Onix Sistema (onix.avantecds.es)
**Fecha:** 19 de Noviembre de 2025
**Plataforma:** DigitalOcean / CloudPanel / Nginx

## 1. RESUMEN EJECUTIVO
Se realizó la migración exitosa del certificado de seguridad SSL de "Autofirmado" a "Let's Encrypt" en el servidor de producción. Se solucionó un conflicto de configuración en Nginx que impedía la validación del dominio, eliminando las advertencias de seguridad en los navegadores.

## 2. DESCRIPCIÓN DEL PROBLEMA
**Estado Inicial:**
- El sitio web cargaba con advertencia de seguridad crítica: `NET::ERR_CERT_AUTHORITY_INVALID`.
- Causa: Uso de certificado "Self-Signed" (sin validez pública).

**Intento de Solución Automática (Fallido):**
- Al intentar generar un certificado válido mediante CloudPanel, el proceso fallaba.
- **Error:** `404 Not Found` en el desafío ACME (`.well-known/acme-challenge`).
- **Diagnóstico:** La autoridad certificadora (Let's Encrypt) intentaba leer el archivo de validación en el servidor, pero Nginx devolvía un error 404 porque buscaba en una ruta incorrecta.

## 3. ACCIONES TÉCNICAS REALIZADAS (TROUBLESHOOTING)

### Fase A: Verificación de Sistema y Permisos
Se descartaron problemas de permisos ejecutando los siguientes comandos en la terminal (SSH):

1. **Creación de estructura de directorios:**
   `sudo mkdir -p /var/www/letsencrypt`
2. **Asignación de permisos:**
   `sudo chown www-data:www-data /var/www/letsencrypt`
3. **Prueba de conectividad:**
   Se creó un archivo de prueba (`test.txt`) que siguió devolviendo Error 404, confirmando que el problema residía en la configuración del Virtual Host (Vhost), no en los permisos.

### Fase B: Corrección de Configuración Nginx (Vhost)
Se identificó que el bloque `server` (puerto 80) apuntaba a una ruta genérica incorrecta.

**Código Erróneo (Anterior):**
```
server {
    listen 80;
    listen [::]:80;
    server_name onix.avantecds.es;

    # --- ESTO ES LO QUE FALLABA ANTES ---
    # Obligamos a Nginx a mirar en la carpeta que creamos
    location ^~ /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        allow all;
        default_type "text/plain";
        try_files $uri =404;
    }
    # ------------------------------------

    # Todo lo demás, mándalo al HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name onix.avantecds.es;

    # Certificados (ahora usa los viejos, luego CloudPanel los cambia)
    ssl_certificate_key /etc/nginx/ssl-certificates/onix.avantecds.es.key;
    ssl_certificate /etc/nginx/ssl-certificates/onix.avantecds.es.crt;

    access_log /var/log/nginx/onix.avantecds.es.access.log;
    error_log /var/log/nginx/onix.avantecds.es.error.log;

    ssl_stapling off;
    ssl_stapling_verify off;

    # Archivos estáticos
    location ~* ^.+\.(css|js|jpg|jpeg|gif|png|ico|gz|svg|svgz|ttf|otf|woff|woff2|eot|mp4|ogg|ogv|webm|webp|zip|swf|map|mjs)$ {
        root /home/onixsistema/htdocs/onix.avantecds.es;
        expires max;
        access_log off;
    }

    location ~ /\.(ht|svn|git) {
        deny all;
    }

    # Tu aplicación (Proxy)
    location / {
        proxy_pass http://localhost:8090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}
```


```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/letsencrypt;
}
```

**Código Corregido (Implementado):**
Se modificó el Vhost para apuntar directamente a la carpeta raíz del aplicativo, permitiendo que Let's Encrypt encontrara el archivo de validación:

```nginx
location ^~ /.well-known/acme-challenge/ {
    root /home/onixsistema/htdocs/onix.avantecds.es;
    allow all;
    default_type "text/plain";
    try_files $uri =404;
}
```

### Fase C: Generación e Instalación del Certificado
Una vez corregida la configuración del Vhost y recargado Nginx (`sudo systemctl reload nginx`), se procedió a la instalación final:

1. Se accedió a la interfaz de **CloudPanel > Site > SSL/TLS**.
2. Se ejecutó la acción **"New Let's Encrypt Certificate"**.
3. **Resultado:** Al existir la ruta correcta en Nginx, la validación ACME fue exitosa. CloudPanel generó los archivos `.crt` y `.key` válidos y reemplazó automáticamente los antiguos certificados autofirmados.

## 4. RESULTADO FINAL
1. **Validación:** El servidor web ahora responde a las peticiones HTTPS con un certificado válido emitido por Let's Encrypt.
2. **Estado Actual:** El dominio `https://onix.avantecds.es` muestra el candado de seguridad verde y es accesible desde todos los navegadores sin advertencias.
