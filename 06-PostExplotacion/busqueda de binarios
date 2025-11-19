🔥 Enumeración de binarios SUID/SGID y detección de binarios vulnerables

Cuando estoy buscando escalada de privilegios en Linux, una de las primeras cosas que hago es revisar los binarios con permisos SUID/SGID. No todos serán útiles, pero algunos pueden dar una vía directa a root si tienen funciones peligrosas, capabilities raras o llamadas inseguras.

Aquí dejo los métodos que uso para buscarlos y filtrarlos.

✅ Búsqueda clásica de binarios SUID
find / -perm -4000 -type f 2>/dev/null

✅ Binarios SGID (también pueden dar juego)
find / -perm -2000 -type f 2>/dev/null

✅ Listar SUID con permisos completos (más visual)
find / -type f -perm -u=s -exec ls -l {} \; 2>/dev/null

✅ Buscar SUID fuera de rutas “normales” (los sospechosos)

Los más peligrosos suelen estar fuera de /usr/bin o /bin.

find / -perm -4000 -not -path "/usr/*" 2>/dev/null

✅ Revisar capabilities del sistema (muy importante)

Algunos binarios sin SUID pueden tener capabilities que permiten escalada.

getcap -r / 2>/dev/null


Capabilities peligrosas:

cap_setuid

cap_dac_read_search

cap_sys_admin

Si un binario como python, perl, tar o openssl tiene capabilities raras, huele a privesc.

✅ Usar herramientas de auditoría (linpeas, LinEnum)
linpeas (el rey para privesc):
curl -sL https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

LinEnum:
wget https://raw.githubusercontent.com/rebootroot/LinEnum/master/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh

✅ Revisar binarios sospechosos con strings

Para ver si llaman a programas sin ruta absoluta (PATH hijacking):

strings /ruta/del/binario | grep "/bin"
strings /ruta/del/binario | grep -i "exec"
strings /ruta/del/binario | grep -i "sh"


Si veo llamadas como:

system("cp");
system("cat");


→ posible escalada modificando el PATH.

✅ Buscar archivos escritos por root pero editables por el usuario

Muy común en máquinas vulnerables:

find / -writable -type f 2>/dev/null


Scripts en cron con permisos abiertos = root en bandeja.

✅ Comprobación del PATH (para hijacking)
echo $PATH


Si incluye rutas como:

/tmp
/home/user


es un regalo.

🎯 Comprobar binarios encontrados en GTFOBins

Una vez localizo un binario SUID, lo meto en GTFOBins para ver si tiene explotación directa:

📌 https://gtfobins.github.io

Ejemplo rápido:

gtfobins env


Si aparece en GTFOBins como “SUID”, lo pruebo.

✔️ Conclusión rápida

Con esto cubro prácticamente todas las vías habituales de escalada basadas en SUID, SGID, capabilities y PATH hijacking. Estos comandos son los que uso siempre que entro en una máquina Linux porque dan resultados rápidos y directos.
