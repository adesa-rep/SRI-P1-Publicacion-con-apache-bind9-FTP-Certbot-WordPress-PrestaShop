# Preparación
## Alquiler de dominio y VPS
Para  el desarrollo de este proyecto se procede con el **alquiler de un dominio** en IONOS llamado `adesa-asir.com`. Posteriormente, se adquiere un servidor VPS tambien de IONOS con el SO Ubuntu Server 24.04.
## Gestion de IP y Credenciales
- **Identificación:** La IP asignada es la IP pública del servidor que se utilizará para trabajar.
- **Acceso Root:** Las credenciales de acceso inicial son el usuario **`root`** y la contraseña proporcionada en el panel del VPS.
## Configuración del Firewall Externo
El servidor tiene un firewall interno, pero hay una política de firewall **externa** gestionada por el proveedor:
- **El Firewall Interno (Dentro del Servidor):** El servidor (en este caso, el servidor Linux) tiene su propio firewall interno
- **La Política de Firewall Externa (Gestionada por el Proveedor):** Esta capa es un conjunto de reglas definido y gestionado a través del **panel de control del proveedor** (del VPS)

Si no se gestiona el firewall desde el panel de control del proveedor (en el apartado de `Red/Politicas de firewall`), las configuraciones internas en el servidor no tendrán efecto; es como si el tráfico estuviera bloqueado por defecto.

Para permitir un nuevo servicio (como un servidor DNS en el puerto 53, por protocolo UDP), se debe añadir la regla en la política de firewall del proveedor.
## Apuntar el Dominio al Servidor
Por defecto, el registro DNS Tipo A del dominio apunta a una IP que no es la del servidor VPS.

Asi dentro de la web de ionos debemos añadir un registro tipo A que apunte a nuestro VPS con IP `217.154.184.58`.

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI](000%20Assets/SRI%20proyecto%20con%20ionos/SRI.png)

Tras hacer esto, confirmamos propagación DNS:
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-10](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-10.png)
# Configuración del servidor DNS (BIND9)
## Estado actual
```bash
adesa@THINK-THINK:/mnt/c/Users/Adriana$ host -t NS adesa-asir.com
adesa-asir.com name server ns1017.ui-dns.de.
adesa-asir.com name server ns1118.ui-dns.com.
adesa-asir.com name server ns1121.ui-dns.biz.
adesa-asir.com name server ns1104.ui-dns.org.
```
Con esto sabemos que es IONOS quien gestiona los registros de `adesa-asir.com`
## Añadir NS personalizado
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-1](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-1.png)

Con esto se delega la autoridad DNS a nuestro propio servidor. Sin embargo, al hacer esto **No creamos el servidor autoritativo aquí**. Solo **apuntamos** a él y va al TLD (se propaga en los root servers).

Entonces, <span style="color:rgb(146, 208, 80)">¿Cuándo se crea el servidor autoritativo?</span> Solo cuando hacemos esto en nuestro VPS:
1. **Instalar Bind9** (o PowerDNS, NSD, etc.)
2. **Crear el archivo de zona** (db.adesa-asir.com)
3. **Configurar el SOA y los registros**
4. **Asegurar que ns1.adesa-asir.com resuelva a la IP de tu VPS**
## Abrir puerto 53 en el registrador IONOS
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-2](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-2.png)
## Instalación de bind9
```bash
sudo apt install bind9 bind9utils bind9-doc
```

- Verifica que el servicio esté activo:
```txt
systemctl status bind9
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-3](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-3.png)

## Resolvedor local y bind9
Por defecto Ubuntu server tiene su propio resolvedor local llamado `Network Name Resolution`. Si consultamos su estado veremos:
```bash
systemctl status systemd-resolved
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-4](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-4.png)

El archivo de configuración de este servicio esta en la ruta `/etc/resolv.conf`. Sin embargo, de nada sirve cambiar el `nameserver` en `/etc/resolv.conf` porque al reiniciar el servicio se sobrescribirá con la configuración establecida en `/run/systemd/resolve/stub-resolv.conf` ya que este archivo es gestionado por el servicio `systemd-resolved`.

Podría ser util cambiar el `nameserver` en `/etc/resolv.conf` para hacer pruebas “puntuales” ya que no es algo permanente.

De modo que para configurar otro servidor DNS, como en nuestro caso bind9, tenemos que indicar a `systemd-resolved` que reenvíe las consultas a bind9, debemos modificar el archivo `/etc/systemd/resolved.conf` y añadir `DNS=127.0.0.1`

Asi que modificamos el archivo de configuración:
```bash
nano /etc/systemd/resolved.conf
```
y añadimos al final:
```ini
DNS=127.0.0.1
```
Con esto le indicamos a `systemd-resolved` que _reenvie las consultas a bind9_.

Otra opción es deshabilitar el servicio `systemd-resolved` con el comando `systemctl stop systemd-resolved ` y luego `systemctl disable systemd-resolved `.

Tras hacer esto, si que podríamos cambiar directamente el nameserver en `/etc/resolv.conf` ya que no estaría vinculado con `systemd-resolved`.
### Zonas DNS
Una **zona DNS** es una porción del espacio de nombres DNS que está _bajo la autoridad de un servidor DNS específico_. Cada zona contiene información sobre un dominio (o subdominio) y sus registros DNS, como direcciones IP (registros A), servidores de nombres (NS), registros de correo (MX), etc. En BIND9, las zonas se definen en archivos de configuración y se almacenan en archivos de zona que contienen los detalles de los registros. Se encuentran en `/etc/bind/`.

#### Definir las zonas en `named.conf.local`
En el archivo `/etc/bind/named.conf.local` definimos dos tipos de zonas:
- Zona directa (Forward Zone): Traduce nombras de dominio a IPs
- Zona inversa (Reverse Zone): Traduce Ips a nombres (usada para registros PTR).

Por seguridad creamos una copia del archivo:
```bash
cp named.conf.local named.conf.local.bk
```

```bash
nano named.conf.local
```

Editamos:
```ini
# zona directa
zone "adesa-asir.com" {
   type master;
   file "/etc/bind/db.adesa-asir.com";
   allow-transfer {none;};
};


# Zona inversa
# 217.154.184.58 -->
zone "184.154.217.in-addr.arpa" {
   type master;
   file "/etc/bind/db.58";
   allow-transfer {none;};
};
```

- **Verifica la sintaxis:**
```bash
named-checkconf
```

Ahora procedemos con los archivos de zona que contienen los registros DNS.
#### Configurar la zona directa (`db.adesa-asir.com`)
Crea el archivo `/etc/bind/db.adesa-asir.com` copiando una plantilla (como `/etc/bind/db.empty`). Este archivo contiene los **registros DNS** que definen cómo se resuelve el dominio.

**Creando una copia**
```txt
cp db.empty db.adesa-asir.com
```

```ini
$TTL    86400
@       IN      SOA     ns1.adesa-asir.com. root.adesa-asir.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                          86400 )       ; Negative Cache TTL
;
@       IN      NS      ns1.adesa-asir.com.     ; Servidor de nombres
ns1     IN      A       217.154.184.58          ; IP del servidor de nombres
@       IN      A       217.154.184.58          ; IP del dominio principal
;
blog    IN      A       217.154.184.58          ; Subdominio blog
tienda  IN      A       217.154.184.58          ; Subdominio tienda
;
@       IN      TXT     "Servidor configurado por Adriana De Sa"        ; Registro de texto
blog    IN      TXT     "subdominio configurado por Adriana De Sa"
tienda  IN      TXT     "subdominio configurado por Adriana De Sa"
```

- **Verifica la sintaxis:**
```txt
named-checkzone adesa-asir.com /etc/bind/db.adesa-asir.com
```

```txt
root@ubuntu:/etc/bind# named-checkzone adesa-asir.com /etc/bind/db.adesa-asir.com
zone adesa-asir.com/IN: loaded serial 1
OK
```
#### Configurar la zona inversa (`db.58)
Creamos el archivo `/etc/bind/db.58` (hacemos una copia de db.127) para la zona inversa, que asocia IPs a nombres (registros PTR).

```txt
cp db.127 db.58
```

```bash
nano db.58
```

```ini
$TTL    604800
@       IN      SOA     ns1.adesa-asir.com. root.adesa-asir.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.adesa-asir.com.
58      IN      PTR     adesa-asir.com.         ; IP 217.154.184.58 -> adesa-asir.com
58      IN      PTR     blog.adesa-asir.com.
58      IN      PTR     tienda.adesa-asir.com.

```

- El registro **PTR** asocia el último octeto de la IP (`58`) al nombre `adesa-asir.com`.
  - Esto permite que, al consultar la IP `217.154.184.58`, se devuelva el nombre `adesa-asir.com`.

- **Verifica la sintaxis:**
```txt
named-checkzone 184.154.217.in-addr.arpa /etc/bind/db.58
```

```txt
root@ubuntu:/etc/bind# named-checkzone 184.154.217.in-addr.arpa /etc/bind/db.58
zone 184.154.217.in-addr.arpa/IN: loaded serial 1
OK
```
#### Reiniciar y probar el servidor
##### Reinicio bind9
```bash
systemctl restart bind9
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-6](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-6.png)
##### Pruebas con `host`:
- Verifica el servidor de nombres:
```txt
host -t NS adesa-asir.com
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-5](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-5.png)

- Verifica registros A:
```bash
host -t A adesa-asir.com
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-7](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-7.png)

- Verifica registro TXT:
```bash
host -t TXT adesa-asir.com
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-8](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-8.png)

- Verifica zonas inversas
```bash
host 217.154.184.58
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-9](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-9.png)

# Configuración del servidor Apache (Virtual Hosts)
El servidor Apache será el encargado de alojar los distintos sitios web del dominio.
```bash
sudo apt install apache2 -y
```

Verificamos estado del servicio:
```bash
systemctl status apache2
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-11](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-11.png)
## Creación de carpetas para sitios diferentes
```bash
root@ubuntu:~# mkdir -p /var/www/adesa-asir.com
root@ubuntu:~# mkdir -p /var/www/blog.adesa-asir.com
root@ubuntu:~# mkdir -p /var/www/tienda.adesa-asir.com
```

confirmamos creación de carpetas:
```bash
root@ubuntu:~# tree -L 1 /var/www/
/var/www/
├── adesa-asir.com
├── blog.adesa-asir.com
├── html
└── tienda.adesa-asir.com
```
## Asignación de Propiedad y herencia
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-12](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-12.png)

Apache corre bajo el usuario y grupo `www-data`. Es vital que este usuario sea el dueño para que WordPress y PrestaShop puedan escribir archivos (subir imágenes, instalar plugins, etc.).

Así que **cambiamos el propietario y grupo** de forma recursiva (-R):
```Bash
chown -R www-data:www-data /var/www/adesa-asir.com
chown -R www-data:www-data /var/www/blog.adesa-asir.com
chown -R www-data:www-data /var/www/tienda.adesa-asir.com
```

Tambien podríamos haber usado `sudo chown -R www-data:www-data /var/www/` para cambiar el dueño y el grupo a toda la raíz web.

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-13](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-13.png)

Ahora, aplicamos el bit de ID de grupo para que los archivos nuevos hereden el grupo `www-data` automáticamente.
```bash
sudo chmod -R g+s /var/www/
```
## Permisos “granulares”
Permiso de escritura para el grupo (**775** para carpetas y **664** para archivos).

Ajustar carpetas: lectura, escritura y ejecución para dueño y grupo
```bash
sudo find /var/www/ -type d -exec chmod 775 {} \;
```

Ajustar archivos: lectura y escritura para dueño y grupo (sin ejecución por segurida)
```bash
sudo find /var/www/ -type f -exec chmod 664 {} \;
```

Donde `find` realiza una búsqueda recursiva de carpetas `-type d` o de archivos `-type f`, y las coincidencias que encuentra las sustituye dinámicamente en el placeholder `{}`. Como usamos `-exec` para ejecutar un comando externo, debemos indicar dónde (o cuando) termina la lista de argumentos que se le asignan con un `;` pero como la shell puede interpretar el `;` como fin de la línea, debemos escapar el caracter con la `\`.

Verificamos permisos:
```bash
tree -p /var/www/
```

```txt
root@ubuntu:~# tree -p /var/www/
[drwxrwsr-x]  /var/www/
├── [drwxrwsr-x]  adesa-asir.com
├── [drwxrwsr-x]  blog.adesa-asir.com
├── [drwxrwsr-x]  html
│   └── [-rw-rw-r--]  index.html
└── [drwxrwsr-x]  tienda.adesa-asir.com
```
## Plan de acción
Hasta ahora tenemos las carpetas donde guardaremos los archivos “a servir” por apache en `/var/www`.

Y como los directorios y permisos ya están listos lo que demos hacer a continuación es:

1. [ ] **Crear los archivos de configuración:** Crear tres archivos `.conf` en `/etc/apache2/sites-available/`.
2. [ ] **Habilitar los sitios:** Usar `a2ensite`.
3. [ ] **Habilitar módulos críticos:** Usar `a2enmod rewrite`.
4. [ ] **Validar y Cargar:** `configtest` y `reload`.
## Creación de los Virtual Hosts
Los Virtual Hosts (VHosts) son una directiva de Apache que permite alojar múltiples dominios en una sola instancia del servicio y una sola dirección IP.
### Crear archivos de configuración
Los archivos de configuración de los Virtual Hosts definidos se guardan en `/etc/apache2/sites-available/` así que procedemos a crear aquí los tres archivos `.conf`:

**Para el sitio principal (`adesa-asir.com`)**
```bash
sudo nano /etc/apache2/sites-available/adesa-asir.com.conf
```

```ini
<VirtualHost *:80>
    ServerName adesa-asir.com
    ServerAlias www.adesa-asir.com
    DocumentRoot /var/www/adesa-asir.com

    <Directory /var/www/adesa-asir.com>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/adesa_error.log
    CustomLog ${APACHE_LOG_DIR}/adesa_access.log combined
</VirtualHost>
```

**Para el blog** (`blog.adesa-asir.com`)
```bash
nano /etc/apache2/sites-available/blog.adesa-asir.com.conf
```

```ini
<VirtualHost *:80>
    ServerName blog.adesa-asir.com
    DocumentRoot /var/www/blog.adesa-asir.com

    <Directory /var/www/blog.adesa-asir.com>
        Options -Indexes +FollowSymLinks
        # AllowOverride All permite que archivos .htaccess (usados por WordPress) cambien la configuracion
        AllowOverride All               
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/blog_error.log
    CustomLog ${APACHE_LOG_DIR}/blog_access.log combined
</VirtualHost>
```

<span style="color:rgb(54, 211, 151)">IMPORTANTE</span> → `AllowOverride All` para WordPress

**Para el sitio de la tienda** (`tienda.adesa-asir.com`)
```bash
nano /etc/apache2/sites-available/tienda.adesa-asir.com.conf
```

```ini
<VirtualHost *:80>
    ServerName tienda.adesa-asir.com
    DocumentRoot /var/www/tienda.adesa-asir.com

    <Directory /var/www/tienda.adesa-asir.com>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/tienda_error.log
    CustomLog ${APACHE_LOG_DIR}/tienda_access.log combined
</VirtualHost>
```
<span style="color:rgb(54, 211, 151)">IMPORTANTE</span> → `AllowOverride All` para PrestaShop

```bash
root@ubuntu:~# tree /etc/apache2/sites-available/
/etc/apache2/sites-available/
├── 000-default.conf
├── adesa-asir.com.conf
├── blog.adesa-asir.com.conf
├── default-ssl.conf
└── tienda.adesa-asir.com.conf

1 directory, 5 files
```
### Activar sitios
Los archivos de configuración guardados en la carpeta `sites-available/` no tienen efecto por sí solos. Es necesario crear un **link simbólico** desde el archivo `adesa-asir.com.conf` a `/sites-enabled/` ya que apache solo “lee” lo que hay aquí.

Para hacer esto usamos el script `a2ensite` (o `a2dissite` para desactivar un sitio).

de forma masiva:
```bash
sudo a2ensite adesa-asir.com.conf blog.adesa-asir.com.conf tienda.adesa-asir.com.conf
```

o, de forma individual:
```bash
sudo a2ensite adesa-asir.com.conf 
sudo a2ensite blog.adesa-asir.com.conf 
sudo a2ensite tienda.adesa-asir.com.conf
```

nos devuelve:
```txt
root@ubuntu:/etc/apache2# a2ensite adesa-asir.com.conf blog.adesa-asir.com.conf tienda.adesa-asir.com.conf
Enabling site adesa-asir.com.
Enabling site blog.adesa-asir.com.
Enabling site tienda.adesa-asir.com.
To activate the new configuration, you need to run:
  systemctl reload apache2
root@ubuntu:/etc/apache2#
```

```bash
tree /etc/apache2/sites-enabled/
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-14](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-14.png)

desactivamos `000-default.conf`
```bash
a2dissite 000-default.conf
```
### Habilitar módulos
Como posteriormente vamos a instalar WordPress y PrestaShop, el **motor de reescritura** de URLs es obligatorio. Si no lo activamos, los enlaces de las webs darán error 404.

Así que habilitamos el módulo `mod_rewrite` que permite mapear URLs solicitadas a rutas reales en el sistema de archivos o a otras URLs e incluso redirigir todo el tráfico del puerto 80 al 443.

```bash
a2enmod rewrite
```

```txt
root@ubuntu:/etc/apache2# a2enmod rewrite
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2
```
### Validación de sintaxis
Antes de reiniciar el servicio, debemos comprobar que no hay errores tipográficos en los archivos creados. Este comando parsea todos los ficheros de configuración cargados.

```bash
sudo apache2ctl configtest
```
### Aplicar cambios
```bash
sudo systemctl reload apache2
```

Usamos ` reload` para que los procesos que están sirviendo páginas puedan terminar su trabajo y los nuevos carguen la configuración actualizada. Pero todo esto asegurando que si la configuración nueva falla, apache pueda seguir funcionando, mientras que si usamos `restart` si hay error el servidor se queda apagado.
### Confirmamos que cargan los dominios
Creamos archivos `index.html` en cada carpeta:
```bash
echo "<h1> adesa </h1>" > adesa-asir.com/index.html
echo "<h1> blog </h1>" > blog.adesa-asir.com/index.html
echo "<h1> tienda </h1>" > tienda.adesa-asir.com/index.html
```

Resultado:
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-15](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-15.png)

# Seguridad en la comunicación (Certbot / Let’s Encrypt)
Esta parte consiste en asegurar la comunicación entre cliente y servidor mediante HTTPS. Dado que tenemos un **VPS (Virtual Private Server)** y un **dominio real**, usaremos **Let's Encrypt** con **Certbot**.
## Requisitos Previos (DNS)
Nos aseguramos que nuestros registros DNS tipo A apuntan a la IP del VPS `217.154.184.58` (aunque esto lo hicimos al configurar [](#Configuración%20del%20servidor%20DNS%20(BIND9)#Configuración%20del%20servidor%20DNS%20(BIND9)#Pruebas%20con%20`host`|Bind9)):
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-16](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-16.png)
## Instalación de Certbot

```bash
sudo apt install certbot python3-certbot-apache
```

donde:

|**Componente**|**Tipo**|**Función**|
|---|---|---|
|**Certbot**|Aplicación / Cliente|Gestiona la comunicación con Let's Encrypt.|
|**Python3**|Entorno de ejecución|Lenguaje en el que está programada la herramienta.|
|**Plugin Apache**|Extensión|Permite a Certbot "meter mano" en los archivos `.conf` de Apache.|
|**Let's Encrypt**|Entidad Externa|La entidad que firma y valida tus certificados (gratis).|

Sin el plugin `python3-certbot-apache`, Certbot conseguiría el certificado, pero no sabría dónde guardarlo ni qué archivos de Apache tiene que tocar para que el candado verde aparezca.

Si ejecutamos (por comprobación):
```bash
systemctl status certbot
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-17](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-17.png)

Vemos que el proceso de renovación no se está ejecutando en este preciso instante ya que el servicio no se inicia solo sino que depende de un temporizador. Para comprobar que los certificados se renovarán solo, debemos consular el estado del **Timer**, no del servicio:

Ver la lista de todos los temporizadores y cuándo es la próxima ejecución:
```bash
systemctl list-timers | grep certbot
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-19](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-19.png)

Ver el estado detallado del temporizador:
```bash
systemctl status certbot.timer
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-18](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-18.png)
## Obtención e Instalación del Certificado
Certbot leerá los archivos de Virtual Host, verá los `ServerName` que configuramos antes y pedirá un certificado para cada uno. Para ello ejecutamos:
```bash
sudo certbot --apache -d adesa-asir.com -d blog.adesa-asir.com -d tienda.adesa-asir.com
```

con este comando y gracias al plugin `python3-certbot-apache`, indicamos con la bandera `-d` que trate lo que viene a continuación como un dominio. Pero **¿Qué hará Certbot durante este comando?**:

1. **Challenge:** Let's Encrypt enviará un reto al servidor para verificar que tú controlas el dominio.
2. **Retrieval:** Descargará el certificado y la clave privada a `/etc/letsencrypt/live/`.
3. **Deployment:** **Modificará automáticamente** tus archivos `.conf` añadiendo las rutas del certificado y creando una redirección de puerto 80 a 443.

**Nos pide datos para enviar el challenge y hacer el retrieval:**
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-20](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-20.png)

**Aquí vemos el deployment:**
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-21](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-21.png)
#### Comprobación de certificados
```bash
certbot certificates
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-34](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-34.png)

```toml
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: adesa-asir.com
    Serial Number: 5039647f5c3aab32f70547298e7b462cc68
    Key Type: ECDSA
    Domains: adesa-asir.com blog.adesa-asir.com tienda.adesa-asir.com
    Expiry Date: 2026-05-02 13:22:57+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/adesa-asir.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/adesa-asir.com/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

**NOTA:** Esto será útil para dar las rutas de los certificados.
## Verificación de la Renovación Automática
Los certificados de Let's Encrypt duran 90 días. Certbot instala un _timer_ en systemd para renovarlos solo cuando falten menos de 30 días.

**Probar que la renovación funciona (Simulación)**
```bash
sudo certbot renew --dry-run
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-22](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-22.png)

**Confirmamos la creación de los nuevos archivos `*-le-ssl.conf` en** `/etc/apache2/sites-available`:
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-23](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-23.png)

Si entramos en `https://blog.adesa-asir.com`, y hacemos click en el candado veremos “La conexión es segura”.
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-24](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-24.png)

Lo mismo veremos al probar los subdominios.
# Instalación de CMS: WordPress y Ecommerce (prestashop)
Esta es la etapa final del despliegue, donde transformamos la infraestructura técnica en servicios reales. Aquí es donde integramos las bases de datos, el intérprete de PHP y el servidor web.
## Instalación de paquetes
Instalamos mysql para crear las bases de datos que tanto WordPress como PrestaShop necesitan, PHP para poder interpretar el código PHP de WordPress, php-mysql para que PHP haga consultas mysql y otros paquetes que se describen a continuación pero estos son los indispensables para, al menos, realizar la instalación.

```bash
apt install mysql-server php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-zip php-intl php-bcmath
```

| **Paquete**              | **Función Técnica**                              | **Necesario para...**                                                       |
| ------------------------ | ------------------------------------------------ | --------------------------------------------------------------------------- |
| **`mysql-server`**       | RDBMS (Sistema Gestor de BBDD).                  | Almacenar tablas, usuarios, posts y pedidos.                                |
| **`php`**                | Intérprete de lenguaje de scripting.             | Ejecutar el código fuente del lado del servidor.                            |
| **`libapache2-mod-php`** | SAPI (Server Application Programming Interface). | Integrar PHP como un proceso dentro del hilo de Apache.                     |
| **`php-mysql`**          | Driver de conectividad PDO/MySQLi.               | Permitir que PHP haga consultas SQL a la base de datos.                     |
| **`php-curl`**           | Librería de transferencia de datos URL.          | Que WordPress pueda conectar con servidores externos (ej. actualizaciones). |
| **`php-gd`**             | Graphics Draw Library.                           | Procesamiento de imágenes (miniaturas, marcas de agua).                     |
| **`php-mbstring`**       | Multi-Byte String.                               | Gestionar alfabetos no latinos y caracteres especiales (tildes, UTF-8).     |
| **`php-xml`**            | Parser de XML/DOM.                               | Leer configuraciones y feeds RSS.                                           |
| **`php-zip`**            | Compresión de archivos.                          | Instalar plugins y temas desde el panel de control.                         |
| **`php-intl`**           | Internationalization extension.                  | Formatos de moneda, fechas y números locales (crítico en PrestaShop).       |
| **`php-bcmath`**         | Arbitrary Precision Mathematics.                 | Cálculos financieros precisos sin errores de redondeo (vital para tiendas). |
## Configuración de MySQL
En el paso anterior instalamos MySQL server, así que ahora procedemos con su configuración para WordPress y PrestaShop.
### Accedemos a la consola
```bash
mysql -u root
```

Al ejecutar esto nos encontramos con que el **usuario root no tiene contraseña**. Esto es porque en las versiones actuales de **MySQL/MariaDB** en Ubuntu, el usuario `root` utiliza por defecto un plugin llamado `auth_socket` o `unix_socket`. Esto significa que no te pide contraseña porque "confía" en que ya eres `root` en el sistema Linux. Sin embargo, para mayor seguridad y para facilitar ciertas herramientas, vamos a ponerle una contraseña robusta.

**Ejecutamos un script de seguridad** que elimina configuraciones inseguras de fábrica (usuarios anónimos, bases de datos de test, etc.).

```bash
mysql_secure_installation
```
Nos encontraremos con un asistente y debemos responder:
- _Enter current password for root:_ Pulsa **Enter** (está vacío).
- _Switch to unix_socket authentication?_ Pulsa **n** (ya que queremos usar contraseña).
- _Change the root password?_ Pulsa **y**.
- _Remove anonymous users?_ **y**.
- _Disallow root login remotely?_ **y**.
- _Remove test database?_ **y**.
- _Reload privilege tables now?_ **y**.

Para asegurarnos de que siempre pida la clave, entramos en MySQL y ejecutamos:
```mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Tu_Nueva_Password_Aqui';

FLUSH PRIVILEGES;
```

```bash
alter user 'root'@'localhost' identified with mysql_native_password by 'pswforadesadomain';
```
Tras esto, al salir y entrar, nos pedirá la nueva contraseña.
### Creamos BBDD y Usuario para WordPress
```mysql
# 1. Acceder al monitor de MySQL 
mysql -u root 

# 2. Crear BBDD y Usuario para WordPress 
CREATE DATABASE wp_blog; 
CREATE USER 'adesa_wp'@'localhost' IDENTIFIED BY 'adesa_wp_topsecret';
GRANT ALL PRIVILEGES ON wp_blog.* to 'adesa_wp'@'localhost';
```
### Creamos BBDD y Usuario para PrestaShop
```mysql
# 3. Crear BBDD y Usuario para WordPress 
CREATE DATABASE ps_tienda; 
CREATE USER 'adesa_ps'@'localhost' IDENTIFIED BY 'adesa_ps_topsecret';
GRANT ALL PRIVILEGES ON ps_tienda.* to 'adesa_ps'@'localhost';
```
### Aplicar cambios y salir
```sql
FLUSH PRIVILEGES; 
EXIT;
```
## Despliegue de WordPress (blog.adesa-asir.com)
Vamos a descargar la última versión directamente al servidor.
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-25](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-25.png)

Enlace `https://es.wordpress.org/latest-es_ES.zip` (2026-02-01).
### Ir a una carpeta temporal y descargar
```bash
cd /tmp 
wget https://es.wordpress.org/latest-es_ES.zip
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-26](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-26.png)
### Descomprimir
```bash
unzip latest-es_ES.zip
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-27](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-27.png)
### Mover a la carpeta del Virtual Host
```bash
sudo cp -r wordpress/* /var/www/blog.adesa-asir.com/
```
### Ajustar permisos para que Apache (www-data) pueda trabajar
```bash
sudo chown -R www-data:www-data /var/www/blog.adesa-asir.com/
```
### Probamos el blog
Accedemos a `https://blog.adesa-asir.com` a traves del [enlace](https://blog.adesa-asir.com).
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-28](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-28.png)

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-29](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-29.png)

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-30](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-30.png)

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-31](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-31.png)

**IMPORTANTE (tener en cuenta tras hacer click en “Realizar instalación”)**:
Es fundamental entender la separación de capas en una aplicación web.
- **Capa de Datos:** El usuario `adesa_wp` que creaste en MariaDB solo sirve para que el código PHP de WordPress se autentique contra el servicio de base de datos.
- **Capa de Aplicación:** El usuario que defines en esta pantalla ("Hola") se guardará dentro de una tabla de la base de datos (habitualmente `wp_users`). Es el usuario con rol de **Administrador** dentro del CMS.

**RECOMENDACIONES**

| **Campo**             | **Recomendación Técnica**                                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Nombre de usuario** | Usa algo como `admin_adriana` o tu nombre. Evita usar "admin" a secas por seguridad.                                                  |
| **Contraseña**        | WordPress te sugiere una muy fuerte. **Cópiala y guárdala**, porque es la que te pedirá para entrar a `blog.adesa-asir.com/wp-admin`. |
| **Tu correo**         | Pon uno real, ya que ahí te llegará el enlace para recuperar la clave si se te olvida.                                                |

Por seguridad he realizado este cambio desde la interfaz de WordPress y lo he confirmado en la BBDD:

```sql
mysql -u root -p

use wp_blog;

select id, user_login, user_email from wp_users;
```

```diff
+----+-------------+------------+
| id | user_login  | user_email |
+----+-------------+------------+
|  2 | adesa_admin | a@2.com    |
+----+-------------+------------+
1 row in set (0.00 sec)
```

**Dashboard**
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-32](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-32.png)
## Despliegue de PrestaShop (tienda.adesa-asir.com)
En lugar de usar el comando `wget` como hicimos con WordPress usaremos un servidor FTP **VSFTP** (Very Secure FTP Daemon).
## Instalación y Seguridad de vsftpd
### Instalación
```bash
apt install vsftpd -y
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-33](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-33.png)
### Configuración de archivo `/etc/vsftpd.conf`
Ahora editamos la configuración: `sudo nano /etc/vsftpd.conf`.
```toml
# Accesos y permisos
listen=NO 
listen_ipv6=YES 
anonymous_enable=NO
local_enable=YES
write_enable=YES
#local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES

# ENJAULAMIENTO (CHROOT)
chroot_local_user=YES
#allow_writeable_chroot=YES

# Configuración SSL (Requisito: SSL = YES)
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO 
ssl_ciphers=HIGH

# Rutas de los certificados de produccion (LET's ENCRYPT)
rsa_cert_file=/etc/letsencrypt/live/adesa-asir.com/fullchain.pem
rsa_private_key_file=/etc/letsencrypt/live/adesa-asir.com/privkey.pem

# RANGO DE PUERTOS PASIVOS (Para el Firewall del VPS) 
pasv_min_port=40000 
pasv_max_port=50000

```

**Bloque 1: Gestión de Usuarios y Permisos**

| **Directiva**                    | **Significado Técnico**                                                                                                                                                                                                                                                                         |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`local_enable=YES`**           | Permite que los usuarios definidos en `/etc/passwd` (usuarios del sistema Linux) puedan iniciar sesión. Imprescindible para que poder usar el usuario `adesa_ftp` que vamos a crear.                                                                                                            |
| **`write_enable=YES`**           | Habilita los comandos del protocolo FTP que modifican el sistema de archivos (STOR, DELE, RNFR, etc.). Sin esto, solo podrían descargar. Sin esto, no podrían subir el `.zip` de PrestaShop ni crear carpetas mediante FileZilla (es decir, no podriamos “escribir/editar” desde fuera/cliente. |
| **`chroot_local_user=YES`**      | **Seguridad Crítica:** Enjaula al usuario en su `home`. No le permite hacer `cd ..` para ver el resto del servidor. Evita que un usuario FTP pueda "escalar" directorios y ver archivos críticos como `/etc/shadow` o configuraciones de otros sitios.                                          |
| **`allow_writeable_chroot=YES`** | Permite que el usuario tenga permisos de escritura en la raíz de su jaula. Sin esto, `vsftpd` daría error por seguridad al intentar conectar.                                                                                                                                                   |

**Bloque 2: Cifrado y Túnel SSL (FTPS)** → cumple requisito de **SSL = YES**

| **Directiva**                    | **Significado Técnico**                                                                                                |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **`ssl_enable=YES`**             | Activa el soporte para conexiones cifradas mediante el protocolo TLS/SSL.                                              |
| **`allow_anon_ssl=NO`**          | Prohíbe que los usuarios anónimos (si existieran) usen cifrado, reservando el túnel seguro para usuarios autenticados. |
| **`force_local_data_ssl=YES`**   | Obliga a que la transferencia de archivos (el canal de datos) sea cifrada.                                             |
| **`force_local_logins_ssl=YES`** | Obliga a que el envío del usuario y la contraseña sea cifrado. Imprescindible para evitar el "sniffing" de claves.     |
| **`ssl_tlsv1=YES`**              | Habilita el protocolo TLS v1.0 o superior (el estándar actual seguro).                                                 |
| **`ssl_sslv2/v3=NO`**            | **Hardening:** Desactiva versiones antiguas y vulnerables de SSL (ataques como POODLE). Solo permitimos TLS.           |

| **Parámetro**     | **Valor de Producción** | **Razón Técnica**                                                           |
| ----------------- | ----------------------- | --------------------------------------------------------------------------- |
| **Certificado**   | `fullchain.pem`         | Incluye el certificado del dominio y la cadena intermedia de Let's Encrypt. |
| **Clave Privada** | `privkey.pem`           | Clave RSA/ECDSA generada por Certbot.                                       |
| **Cifrado**       | `HIGH`                  | Solo permite algoritmos de cifrado fuertes (AES-256, etc.).                 |
| **Seguridad**     | `TLSv1`                 | Bloquea protocolos obsoletos vulnerables a ataques como POODLE.             |

### Reinicio y Validación del Servicio
**Reinicio**
```bash
sudo systemctl restart vsftpd
```

**Verificar estado**
```bash
sudo systemctl status vsftpd
```

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-35](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-35.png)

Dado que lo mas probable es que el error sea por los certificados, verificamos que en el archivo de configuración no hallan espacios en blando al final de la línea y también los permisos de las carpetas que los contienen.

los archivos en `/etc/letsencrypt/live/` son solo **enlaces simbólicos** (accesos directos). Los archivos reales están en `/etc/letsencrypt/archive/`, y esos archivos suelen tener permisos tan restrictivos que solo el usuario `root` puede leerlos. Revisando los permisos vemos que, efectivamente, `privkey1.pem` solo puede ser leído por root:
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-36](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-36.png)

**Corregimos los permisos**
```bash
chmod 755 -R /etc/letsencrypt/archive/

chmod 644 -R /etc/letsencrypt/archive/adesa-asir.com/*
```

Sin embargo, el error resultó ser un espacio en blanco al final de una de las lineas de configuración del archivo `/etc/vsftpd.conf`.

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-37](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-37.png)
## Creamos usuario FTP
Necesitamos un usuario de sistema que "aterrice" directamente en la carpeta de la tienda.
### Crear usuario sin acceso a shell por seguridad
```bash
sudo adduser adriana_ftp --shell /bin/false --home /var/www/tienda.adesa-asir.com
```
### Darle la propiedad de la carpeta
```bash
sudo chown -R adriana_ftp:www-data /var/www/tienda.adesa-asir.com
sudo chmod -R 775 /var/www/tienda.adesa-asir.com
```
#### Comando 1: Creación del Usuario de Sistema
`sudo adduser adriana_ftp --shell /bin/false --home /var/www/tienda.adesa-asir.com`

- **`adduser`**: Script de alto nivel para crear un usuario, su grupo y su directorio.
- **`--shell /bin/false`**: Establece una _nologin shell_. Si el usuario intenta conectar por SSH, el sistema rechazará la conexión inmediatamente. Es una medida de **Hardening** esencial en servicios FTP.
- **`--home [ruta]`**: Define el `HOME_DIR`. Al conectar por FTP, `vsftpd` leerá esta ruta y, gracias al `chroot` que configuramos en el `.conf`, el usuario no podrá subir de nivel en el árbol de directorios (`/`).
#### Comando 2: Gestión de Propiedad (Ownership)
`sudo chown -R adriana_ftp:www-data /var/www/tienda.adesa-asir.com`
- **`chown -R`**: Cambia el propietario de forma **recursiva** (afecta a todas las carpetas y archivos internos).
- **`adriana_ftp:www-data`**: Asigna al usuario `adriana_ftp` como dueño y al grupo `www-data` (el grupo de Apache/Nginx) como grupo propietario. Esto permite que el servidor web también pueda leer lo que tú subas.
#### Comando 3: Gestión de Permisos (Permissions)
`sudo chmod -R 775 /var/www/tienda.adesa-asir.com`
- **`7 (Dueño)`**: `adriana_ftp` puede Leer, Escribir y Ejecutar (subir el instalador de PrestaShop).
- **`7 (Grupo)`**: El grupo `www-data` también tiene control total.
- **`5 (Otros)`**: El resto del mundo solo puede Leer y Ejecutar (necesario para que la web sea visible públicamente).

### Usando como cliente WinSCP
Dado que la conexión es FTPS:
![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-41](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-41.png)


![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-38](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-38.png)

descomprimimos el archivo con `unzip prestashop_edition_basic_version_9.0.2-2.1.zip`. Y al acceder a URL `tienda.adesa-asir.com` empieza el instalador:

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-39](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-39.png)

![000%20Assets/SRI%20proyecto%20con%20ionos/SRI-40](000%20Assets/SRI%20proyecto%20con%20ionos/SRI-40.png)

<span style="color:rgb(255, 0, 0)">NO HE PODIDO CONTINUAR CON LA INSTALACION PORQUE MI VPS SOLO TIENE 1GB DE RAM Y NO ES SUFICIENTE PARA PRESTASHOP</span> 

