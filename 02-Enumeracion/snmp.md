# SNMP — notas prácticas

**Autor:** Josema — [LinkedIn](https://www.linkedin.com/in/jose-manueltr/)
**Fuente:** Curso "El Rincón del Hacker" (Mario) — apuntes personales

---

## Objetivo

Detectar y explotar (en laboratorio) servicios SNMP que suelen comunicarse por UDP. Enumerar información expuesta mediante community strings, obtener datos del sistema y posibles pistas (usuarios, rutas, configuraciones, subdominios, etc.).

---

## 1) Detección UDP (puerto 161)

SNMP usa UDP (puerto 161). Al escanear UDP ten en cuenta que es más lento y que los puertos pueden aparecer como `open|filtered` si no responden.

Comando usado (ejemplo):

```bash
sudo nmap -sU --top-ports 200 --min-rate 5000 10.0.8.14
```

Salida relevante (ejemplo):

```
PORT      STATE  SERVICE
161/udp   open   snmp
... (otros puertos)
```

**Notas:**

* `-sU` escaneo UDP.
* `--top-ports 200` limita a los 200 puertos UDP más comunes.
* `--min-rate 5000` acelera el escaneo (ajustar según red/lab).

Si ves `161/udp open` sospecha SNMP y pasa a enumeración.

---

## 2) ¿Qué es una "community string"?

SNMPv1/v2c usan *community strings* como "contraseñas" muy simples: `public`, `private`, u otras personalizadas. Con la community correcta puedes leer (GET/GETNEXT) información del agente SNMP.

* `public` suele dar lectura (RO = read-only).
* `private` a veces da escritura (RW = read-write) — peligroso si está mal configurado.

No confundir con SNMPv3, que usa usuario/clave y cifrado (más seguro).

---

## 3) Enumerar community strings (tool: onesixtyone)

`onesixtyone` es rápido para probar listas de community strings.

```bash
onesixtyone -c /usr/share/wordlists/rockyou.txt 10.0.8.14
```

Salida de ejemplo:

```
10.0.8.14 [security] Linux chain 5.10.0-23-amd64 #1 SMP Debian ... x86_64
```

Aquí hemos identificado `security` como community válida (ejemplo). Anota la community para el siguiente paso.

---

## 4) Obtener MIBs / datos con `snmpwalk`

Con la community válida usamos `snmpwalk` (v2c en este ejemplo):

```bash
snmpwalk -v 2c -c security 10.0.8.14
```

Salida (fragmento):

```
iso.3.6.1.2.1.1.1.0 = STRING: "Linux chain 5.10.0-23-amd64 ..."
iso.3.6.1.2.1.1.4.0 = STRING: "Blue <blue@chaincorp.nyx>"
iso.3.6.1.2.1.1.5.0 = STRING: "Chain"
iso.3.6.1.2.1.1.6.0 = STRING: "VulNyx.com"
... (más OIDs y MIB info)
```

**Interpretación:**

* `sysDescr` (1.3.6.1.2.1.1.1.0) → info del SO y versión del kernel.
* `sysContact` (1.3.6.1.2.1.1.4.0) → contacto (correo) — pista de la organización.
* `sysName`, `sysLocation` → más metadatos útiles.

Cuidado: puede exponerse información sensible (nombres de host, rutas, imágenes de boot, software instalado).

---

## 5) ¿Qué buscar en la salida?

* Usuarios o cuentas listadas en MIBs (pueden dar nombres útiles).
* Nombres de dominios/subdominios (`VulNyx.com`, `chaincorp.nyx`) para pivotar a web/host discovery.
* Rutas de boot o config que indiquen versiones vulnerables.
* Vistos OIDs relativos a interfaces, tablas, procesos que den pistas para escalada o superficie de ataque.

---

## 6) Escaneo web asociado / fuzzing de subdominios (ejemplo con wfuzz)

Si `snmpwalk` devuelve dominios o nombres, a veces hay servicios HTTP asociados. Un ejemplo de fuzzing para subdominios usando `wfuzz`:

```bash
wfuzz -c -z file,/usr/share/wordlists/wfuzz/general/spanish.txt --hc 404 --hl 11 -H "Host: FUZZ.chaincorp.nyx" http://10.0.8.14/
```

Notas sobre la salida: si no encuentra nada puede ser por:

* wordlist inadecuada para subdominios.
* el servicio no está corriendo en ese host o requiere ruta distinta.
* errores de TLS/SSL si fueras a HTTPS (wfuzz advierte si pycurl no está compilado con OpenSSL).

Mejoras: usar wordlists específicas de subdominios, ajustar `--hc/--sc` para filtrar respuestas útiles, y lanzar peticiones con `Host:` correctos.

---

## 7) Buenas prácticas y precauciones

* **Solo** en laboratorio. La enumeración SNMP es ruidosa y puede estar auditada.
* No pruebes RW (write) sin permiso — puede cambiar configuraciones.
* Añade `snmp` y wordlists a tu carpeta `resources/` del repo si te sirven (sin datos sensibles).
* Documenta: comando → salida → interpretación (como siempre).

---

## 8) Cheatsheet rápido (comandos mínimos)

```bash
# detectar SNMP (UDP)
sudo nmap -sU --top-ports 200 --min-rate 5000 10.0.8.14

# buscar community strings con onesixtyone
onesixtyone -c /usr/share/wordlists/rockyou.txt 10.0.8.14

# enumerar toda la MIB con snmpwalk (v2c)
snmpwalk -v 2c -c security 10.0.8.14

# fuzzing de subdominios (ejemplo)
wfuzz -c -z file,/usr/share/wordlists/wfuzz/general/spanish.txt --hc 404 -H "Host: FUZZ.chaincorp.nyx" http://10.0.8.14/
```

---

*Status:* listo para pegar en `04-Explotacion-Host/snmp.md`. ¿Quieres que haga también la versión "cheatsheet" minimal con solo 4-6 líneas?
