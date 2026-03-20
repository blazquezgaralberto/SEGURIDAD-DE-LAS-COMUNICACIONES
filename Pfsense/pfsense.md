# 🔥 pfSense — Firewall, Portal Cautivo, Proxy y VPN

> **Asignatura:** Seguridad en las Comunicaciones  
> **Máster en Ciberseguridad**

---

## 📋 Índice

1. [Introducción](#1-introducción)
2. [Preparación del entorno de máquinas virtuales](#2-preparación-del-entorno-de-máquinas-virtuales)
3. [Configuración de pfSense](#3-configuración-de-pfsense)
4. [Configuración de un portal cautivo](#4-configuración-de-un-portal-cautivo)
5. [Proxy transparente con Squid y SquidGuard](#5-proxy-transparente-con-squid-y-squidguard)
   - 5.1 [Squid](#51-squid)
   - 5.2 [SquidGuard](#52-squidguard)
   - 5.3 [pfBlockerNG — Bloqueo de anuncios](#53-pfblockerng--bloqueo-de-anuncios)
6. [Bloqueo de tráfico](#6-bloqueo-de-tráfico)
7. [VPN con OpenVPN](#7-vpn-con-openvpn)
8. [Descarga del archivo de configuración XML](#8-descarga-del-archivo-de-configuración-xml)

---

## 1. Introducción

### Enunciado

Instalación y configuración de **pfSense** como router y cortafuegos, desplegando sobre él las siguientes tecnologías:

1. Preparación de la VM pfSense + VM cliente Linux (Kali). pfSense actúa como router/firewall entre WAN y LAN.
2. Configuración básica de pfSense: conectividad LAN→WAN, bloqueo WAN→LAN.
3. **Portal cautivo** con autenticación basada en tickets (Vouchers).
4. **Proxy transparente** con Squid + SquidGuard: bloqueo de redes sociales, compras y música. *(Extra: pfBlockerNG para bloquear anuncios.)*
5. **Bloqueo de tráfico** por categorías con monitorización en tiempo real.
6. **VPN de acceso remoto** con OpenVPN (usuarios WAN → recursos LAN).

---

## 2. Preparación del entorno de máquinas virtuales

Se configuran **dos adaptadores de red** en la VM de pfSense en VMware:

| Adaptador | Red | Función |
|-----------|-----|---------|
| Adaptador 1 | VMnet0 (Bridge) | WAN — conectividad a Internet |
| Adaptador 2 | VMnet1 | LAN — red interna con DHCP |

![Añadir adaptador de red 2 pfsense](assets/image1.png)

![Redes configuradas en VMWare](assets/image2.png)

Se desmarca la opción "Use local DHCP service" para VMnet1, permitiendo que sea pfSense quien gestione el DHCP de la LAN:

![Desmarcar use local DHCP service para VMnet1](assets/image3.png)

> Esto simula un entorno real donde pfSense es el router de la red interna, otorgando IPs y filtrando tráfico entre WAN y LAN.

Se instala la máquina pfSense:

![Instalar máquina pfsense](assets/image4.png)

Al iniciar, se verifica que WAN y LAN están correctamente asignadas:

![Inicio pfsense](assets/image5.png)

Se ajusta la dirección IP de la LAN para tener una identificación clara de ambas redes:

![Configuración IP red LAN pfsense](assets/image6.png)

**Configuración del cliente Kali Linux:**

Se establece el adaptador de red de Kali en VMnet1 para que esté en el mismo segmento LAN que pfSense:

![Configuración máquina Kali](assets/image7.png)

Kali realiza una solicitud DHCP y pfSense responde asignándole una IP del rango LAN:

![IP máquina Kali](assets/image8.png)

**Verificación de conectividad:**

```bash
ping 10.0.1.1        # Ping hacia pfSense LAN
ping 8.8.8.8         # Ping hacia Internet
```

![Ping Kali hacia pfsense](assets/image9.png)

![Ping Kali hacia internet](assets/image10.png)

Se accede a la administración web de pfSense desde Kali dirigiéndose a la IP LAN de pfSense:

![Administración pfsense desde Kali](assets/image11.png)

> Credenciales por defecto: `admin / pfsense`

---

## 3. Configuración de pfSense

Se revisan y verifican las configuraciones básicas:

![Configuración básica inicial pfsense](assets/image12.png)

Las interfaces WAN y LAN ya fueron configuradas durante la instalación:

![Configuración WAN](assets/image13.png)

![Configuración LAN](assets/image14.png)

Se verifica la resolución DNS con un ping a Google:

```bash
ping google.com
```

![Ping Google.com](assets/image15.png)

**Reglas de firewall por defecto:**

pfSense aplica el **principio de mínimo privilegio** por defecto:
- ✅ Permite tráfico saliente desde LAN (regla "pass all").
- ❌ Bloquea todo tráfico entrante desde WAN.

**Reglas LAN:**

![Reglas firewall LAN — Allow LAN to any](assets/image16.png)

**Reglas WAN:**

![Reglas firewall WAN — Block private/bogon networks](assets/image17.png)

**NAT saliente:**

Se verifica que el modo "Automatic Outbound NAT" esté activo, lo que traduce las IPs internas de la LAN a la IP pública de la WAN:

![Reglas firewall NAT — Automatic Outbound NAT](assets/image18.png)

---

## 4. Configuración de un portal cautivo

Se configura un **portal cautivo** sobre la interfaz LAN que redirige a todos los dispositivos a una página de autenticación. Se usa el sistema de **Vouchers** de pfSense para generar códigos temporales de acceso.

Se accede a la administración de portales cautivos:

![Portal cautivo pfSense](assets/image19.png)

Se crea un nuevo portal cautivo:

![Creación portal cautivo](assets/image20.png)

**Configuración del portal:**

Parámetros clave: activar "Enable Captive Portal", seleccionar la interfaz LAN, configurar tiempo de inactividad y duración máxima de sesión:

![Configuración portal cautivo](assets/image21.png)

**Método de autenticación:**

![Configuración autenticación portal cautivo](assets/image22.png)

*(Opcional) Si se usara autenticación por base de datos de usuarios:*

![Administración de usuarios](assets/image23.png)

![Privilegios de portal cautivo para usuarios](assets/image24.png)

En este caso se usa **autenticación basada en Vouchers (tickets)**:

![Configuración autenticación portal cautivo — Vouchers](assets/image25.png)

**Configuración de Vouchers:**

Se activa la generación de tickets y se pulsa "Generate new keys":

![Configuración opciones de Voucher](assets/image26.png)

Se crea un nuevo rollo de **50 tickets**:

![Creación nuevo rollo de tickets](assets/image27.png)

![Voucher Roll añadido](assets/image28.png)

**Prueba del portal cautivo:**

Al intentar navegar desde Kali, aparece la pantalla de autenticación:

![Login portal cautivo](assets/image29.png)

Se introduce uno de los códigos del voucher roll:

![Listado voucher roll creado](assets/image30.png)

![Login con voucher code](assets/image31.png)

Tras la autenticación exitosa, se redirige al usuario a la URL configurada en "After authentication redirection URL":

![Portal UEM redireccionado](assets/image32.png)

![Campo after authentication redirection URL](assets/image33.png)

✅ **Portal cautivo con autenticación por tickets funcionando correctamente.**

---

## 5. Proxy transparente con Squid y SquidGuard

Se instala y configura **Squid** como proxy transparente en la LAN (todo el tráfico HTTP/HTTPS pasa obligatoriamente por él sin configuración en los clientes), y **SquidGuard** como filtro de URL por categorías.

Se instalan ambos paquetes desde el gestor de paquetes de pfSense:

![Instalación de squid y squidguard](assets/image34.png)

### 5.1 Squid

Se guarda la configuración de caché local (valores por defecto):

![Configuración Squid local caché](assets/image35.png)

**Configuración general:** activar el proxy, seleccionar interfaz LAN:

![Configuración general Squid](assets/image36.png)

Se marca la opción para permitir automáticamente el tráfico LAN sin reglas ACL adicionales:

![Configuración general Squid 2](assets/image37.png)

**Filtrado HTTPS con SSL Splice All** — permite interceptar tráfico HTTPS de forma transparente sin instalar certificados en los clientes:

![Configuración general Squid SSL](assets/image38.png)

**Transparent proxy settings** — activado apuntando a la LAN:

![Transparent proxy settings](assets/image39.png)

**Creación del certificado CA para Squid** (necesario para interceptar HTTPS):

![Authorities CA_SQUID](assets/image40.png)

![Certificates — certificado para Squid](assets/image41.png)

Se verifica que el servicio Squid esté activo:

![Status servicio Squid](assets/image42.png)

**Verificación de funcionamiento:**

Desde el monitor de Squid se puede ver en tiempo real todo el tráfico que pasa por el proxy:

![Squid Monitor tabla de accesos](assets/image43.png)

![Squid Monitor tabla caché](assets/image44.png)

Se verifica también en los logs del sistema:

```bash
cat /var/squid/logs/access.log
```

![Logs pfsense /var/squid/logs](assets/image45.png)

**Prueba de blacklist manual:**

Se añade `yahoo.com` a la ACL blacklist de Squid:

![Squid ACL Blacklist](assets/image46.png)

Al intentar acceder, Squid deniega la conexión:

![Error intento acceso yahoo.com](assets/image47.png)

En los logs se puede ver la entrada de denegación:

![Denegación de acceso log squid](assets/image48.png)

---

### 5.2 SquidGuard

Se activa SquidGuard:

![Activación de SquidGuard](assets/image49.png)

Se especifica la URL de una lista negra pública para descargar categorías de bloqueo:

![Blacklist SquidGuard](assets/image50.png)

![Descarga de blacklist](assets/image51.png)

**Configuración de Common ACL:**

Se establece el acceso por defecto como `Allowed` y se deniegan las categorías específicas:

![SquidGuard control common ACL](assets/image52.png)

| Categoría bloqueada | Ejemplos |
|---|---|
| **Redes sociales** | Facebook, Twitter, Instagram |
| **Compras** | Amazon, eBay, Zara |
| **Música y audio** | YouTube, Spotify |

![Denegar acceso a redes sociales](assets/image53.png)

![Denegar acceso a compras](assets/image54.png)

![Denegar acceso a música](assets/image55.png)

Se personaliza el mensaje de error cuando se deniega el acceso:

![Personalizar proxy denied error](assets/image56.png)

Se verifica que el servicio SquidGuard esté activo:

![Status servicio SquidGuard](assets/image57.png)

**Pruebas de bloqueo:**

![Prueba acceso denegado — YouTube](assets/image58.png)

![Prueba acceso denegado — Facebook](assets/image59.png)

![Prueba acceso denegado — Zara](assets/image60.png)

✅ **SquidGuard bloqueando correctamente las tres categorías.**

---

### 5.3 pfBlockerNG — Bloqueo de anuncios

Se instala el paquete **pfBlockerNG**:

![Instalación paquete pfblockerng](assets/image61.png)

Tras la instalación aparece una nueva entrada en el menú Firewall:

![Firewall → pfBlockerNG](assets/image62.png)

Se verifica la integración con el DNS Resolver en `Services → DNS Resolver`:

![Custom options DNS Resolver](assets/image63.png)

**Activación del módulo DNSBL:**

![Activar DNSBL](assets/image64.png)

Se selecciona la interfaz LAN y se configura DNSBL:

![Configuración DNSBL](assets/image65.png)

Se activa "Deny Outbound" para bloquear consultas DNS a dominios de anuncios:

![Configuración DNSBL 2](assets/image66.png)

**Creación del grupo de bloqueo de anuncios** con una lista pública de dominios de publicidad:

![Creación de lista de bloqueo](assets/image67.png)

![Creación grupo de bloqueo de anuncios](assets/image68.png)

Se fuerza la actualización de pfBlockerNG para descargar las listas:

![Forzar update de pfBlocker](assets/image69.png)

**Resultado — as.com sin anuncios:**

![Acceso as.com sin anuncios](assets/image70.png)

> Los espacios en blanco muestran dónde deberían aparecer los anuncios bloqueados.

Dashboard de pfBlockerNG:

![Dashboard pfBlockerNG](assets/image71.png)

Logs detallados de dominios bloqueados:

![AnunciosFilter logs generados](assets/image72.png)

✅ **pfBlockerNG bloqueando anuncios correctamente.**

---

## 6. Bloqueo de tráfico

Se monitoriza en tiempo real el tráfico bloqueado por SquidGuard, mostrando la IP del cliente y la categoría de bloqueo.

Desde la interfaz de logs de SquidGuard se filtran los accesos bloqueados:

![Filtrado logs squidGuard blocked](assets/image73.png)

Cada entrada del log incluye:
- **Timestamp** — fecha y hora del intento
- **IP cliente** — dispositivo que realizó la solicitud
- **URL solicitada** — dominio al que intentó acceder
- **Respuesta** — módulo de filtrado que lo bloqueó

**Prueba con curl:**

```bash
curl -I https://amazon.com
```

![Respuesta denegación curl amazon](assets/image74.png)

**Ejemplos de bloqueo — Redes sociales (3 ejemplos):**

![Denegación de acceso de redes sociales](assets/image75.png)

**Ejemplos de bloqueo — Compras (3 ejemplos):**

![Denegación acceso páginas de compras](assets/image76.png)

**Ejemplos de bloqueo — Música y vídeo (3 ejemplos):**

![Denegación acceso música](assets/image77.png)

**Monitorización en tiempo real desde la VM pfSense:**

```bash
tail -f /var/squidGuard/log/block.log
```

![Log squidGuard de pfsense](assets/image78.png)

![Contenido block.log](assets/image79.png)

---

## 7. VPN con OpenVPN

Se despliega un servidor **OpenVPN de acceso remoto** en pfSense para que usuarios desde la WAN puedan conectarse de forma segura a la red LAN.

**Instalación del paquete OpenVPN:**

![Instalar paquete openvpn](assets/image80.png)

**Creación de la Autoridad Certificadora (CA):**

OpenVPN requiere una CA interna para firmar el certificado del servidor y los de los usuarios:

![Creación CA_OPENVPN](assets/image81.png)

**Creación del certificado del servidor:**

![Creación openVpnCert](assets/image82.png)

**Configuración con el Wizard de pfSense:**

pfSense incluye un wizard que automatiza la creación del servidor OpenVPN y las reglas de firewall necesarias:

![VPN → OpenVPN](assets/image83.png)

![Configuración wizard openVPN](assets/image84.png)

Se seleccionan los certificados creados:

![Configurar Certificate Authority](assets/image85.png)

![Configurar Certificate](assets/image86.png)

**Configuración general del servidor VPN** (protocolo UDP, puerto 1194, interfaz WAN):

![Configuración general openVPN](assets/image87.png)

**Configuración del túnel** — red virtual para clientes VPN y red local LAN (10.0.1.0/24):

![Configuración del túnel openVPN](assets/image88.png)

**Reglas de firewall generadas automáticamente:**

`Firewall → Rules → WAN` — abre el puerto UDP 1194 para que los clientes puedan iniciar el túnel:

![Regla WAN generada](assets/image89.png)

`Firewall → Rules → OpenVPN` — permite todo el tráfico hacia la LAN una vez autenticado:

![Regla OpenVPN generada](assets/image90.png)

**Creación de usuarios con certificados individuales:**

![Creación de usuarios utilizando el certificado](assets/image91.png)

**Exportación del perfil de cliente (.ovpn):**

Usando OpenVPN Client Export se descarga el archivo de configuración del cliente:

![OpenVPN Client Export](assets/image92.png)

![Export clientes VPN](assets/image93.png)

![Archivo user1vpn.ovpn](assets/image94.png)

**Conexión desde el cliente:**

```bash
openvpn --config user1vpn.ovpn
```

![Ejecución comando openvpn](assets/image95.png)

La VPN asigna una nueva IP al cliente dentro de la red del túnel:

![Dirección IP proporcionada por la VPN](assets/image96.png)

**Verificación de conectividad:**

```bash
ping 10.0.1.1      # Gateway LAN
ping 10.8.0.1      # Gateway del túnel VPN
ping 10.0.1.X      # Dispositivo dentro de la LAN
```

![Ping LAN](assets/image97.png)

![Ping tunel vpn](assets/image98.png)

![Ping dirección dentro de la LAN](assets/image99.png)

✅ **Confirmación de funcionamiento correcto:**
- Túnel VPN establecido correctamente.
- Enrutamiento configurado: el tráfico llega desde el cliente VPN a la LAN.
- Reglas de firewall permiten la comunicación.

---

## 8. Descarga del archivo de configuración XML

Se exporta la configuración completa de pfSense en formato XML desde `Diagnostics → Backup & Restore`:

![Descarga config.xml](assets/image100.png)

> El archivo `config.xml` contiene toda la configuración de pfSense y es útil para **respaldos** y **migraciones** entre instancias.
