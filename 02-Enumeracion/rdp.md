# RDP — notas prácticas

**Autor:** Josema — [LinkedIn](https://www.linkedin.com/in/jose-manueltr/)
**Fuente:** Curso "El Rincón del Hacker" (Mario) — apuntes personales


> ⚠️ Uso legal: estos apuntes son solo para laboratorios y entornos autorizados. No los uses contra sistemas reales sin permiso.

---

## Objetivo

Identificar servicios RDP (Remote Desktop Protocol), comprobar configuración y autenticación (NLA), conectar a sesiones gráficas y, en entornos controlados, probar técnicas de enumeración y acceso.

---

## 1) Detección básica

```bash
# escanear el puerto RDP y detectar versión/servicio
nmap -sV -p 3389 --script rdp-enum-encryption,rdp-ntlm-info 10.0.8.11
```

* `rdp-enum-encryption` muestra las capas de seguridad y cifrado aceptadas.
* `rdp-ntlm-info` puede devolver información NTLM si el servicio lo permite.
* Si el puerto no es 3389, haz un barrido de puertos o usa `-p-`.

---

## 2) Identificar NLA (Network Level Authentication)

* Si RDP requiere **NLA**, no podrás iniciar la sesión gráfica sin autenticar primero (el handshake exige credenciales). Muchas máquinas actuales usan NLA por defecto.
* `rdp-enum-encryption` y `nmap -sV` suelen indicar si NLA está activo.

---

## 3) Conexión gráfica (herramientas recomendadas)

**FreeRDP (xfreerdp)** — recomendado por versatilidad:

```bash
# conexión básica (ignora certificado autofirmado)
xfreerdp /v:10.0.8.11 /u:mario /p:123123 /cert:ignore

# con redirección de unidades y clipboard
xfreerdp /v:10.0.8.11 /u:mario /p:123123 /cert:ignore /clipboard /drive:share,/path/to/local/dir

# forzar seguridad (ejemplo: RDP TLS)
xfreerdp /v:10.0.8.11 /u:mario /p:123123 /sec:tls /cert:ignore
```

**rdesktop** (antiguo, no recomendado para nuevas infraestructuras):

```bash
rdesktop -u mario -p 123123 10.0.8.11
```

**Remmina** (GUI) → práctico si trabajas con varias conexiones (soporta FreeRDP backend).

---

## 4) Fuerza bruta / pruebas de credenciales 

Herramientas: `ncrack`, `hydra`, `rdp-sec-checkers`.

Ejemplo con `ncrack`:

```bash
ncrack -p rdp://10.0.8.11 -u mario -P /usr/share/wordlists/rockyou.txt
```

Ejemplo con `hydra`:

```bash
hydra -t 4 -V -f -l mario -P /usr/share/wordlists/rockyou.txt rdp://10.0.8.11
```

* `-t` threads, `-V` verbose, `-f` stop on first valid.
* Fuerza bruta contra RDP es muy ruidoso y puede bloquear cuentas/activar defensa; usar solo en entornos autorizados.

---

## 5) Enumeración avanzada y post-conexión

### a) Credenciales válidas: qué mirar

* Revisar grupos locales (`net localgroup`), usuarios (`net user`), políticas de contraseña.
* Revisar tareas programadas, aplicaciones instaladas y servicios con privilegios.
* Montar unidad compartida o usar `winrm`/`powershell` para movimientos laterales si está permitido.

### b) Si no puedes conectar por NLA

* Prueba con usuarios diferentes o usa `rdp-ntlm-info` para ver si hay información útil.
* A veces un servicio RDP antiguo permite autenticación sin NLA (menos común en entornos parcheados).

---

## 6) Uso de Metasploit / módulos relacionados

* `auxiliary/scanner/rdp/rdp_*` → varios scripts de enumeración (ver la versión que tienes en Metasploit).
* `exploit/windows/rdp/*` → hay exploits históricos; en lab puedes probarlos si sabes que la versión es vulnerable.

Ejemplo rápido para enumerar con msfconsole:

```text
use auxiliary/scanner/rdp/rdp_encryption
set RHOSTS 10.0.8.11
run
```

---

## 7) Flujo de trabajo recomendado (rápido)

1. `nmap -sV -p 3389 --script rdp-enum-encryption,rdp-ntlm-info <host>` → detección y NLA.
2. Prueba conexión gráfica con `xfreerdp /cert:ignore` si tienes credenciales.
3. Si no tienes creds y es LAB: `ncrack`/`hydra` con wordlists controladas.
4. Con sesión: enumerar usuarios, grupos, tareas y servicios; exportar evidencia y limpiar trazas en lab.
5. Documentar todo: comando → salida → interpretación.

---

## 8) Buenas prácticas

* No subas contraseñas ni capturas con datos reales al repo. Usa placeholders.
* Anota si el host usa NLA y qué tipo de cifrado acepta (esto condiciona tácticas).
* Ten cuidado con redirecciones (clipboard, drives) cuando trabajes en entornos de pruebas: pueden exfiltrar datos locales.
* Si un exploit falla, asigna prioridad a enumeración: servicios, parches y políticas de grupo.

---

*Pequeño tip técnico:* si ves `TLS` como método de seguridad, FreeRDP con `/cert:ignore` suele ser suficiente para laboratorio; en entornos reales verifica certificados y evita ignorarlos.

---
