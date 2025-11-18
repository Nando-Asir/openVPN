# Servidor OpenVPN en Debian 13 (AWS/EC2)

Este proyecto establece un servidor **OpenVPN** de alto rendimiento en una instancia de Debian 13 "Trixie" alojada en **Amazon Web Services (AWS)**. La seguridad se maximiza utilizando:

1. **Certificados X.509 públicos (Let's Encrypt)** para la autenticación del servidor.
2. **Una PKI privada (Easy-RSA)** para la autenticación segura de los clientes.

---

## ⚠️ Prerrequisitos y Consideraciones de Seguridad

- **Instancia AWS EC2:** Servidor Debian 13 operativo con una **IP Pública Elástica (EIP)**.
- **Dominio:** Un nombre de dominio (ej. **vpn.midominio.com**) apuntando a la EIP del servidor.
- **Acceso SSH:** Archivo **.pem** de tu par de claves AWS.

---

## 1️⃣ Configuración del Firewall Cloud (AWS Security Group)

Este es el paso **más crítico** en un entorno cloud. Si el Grupo de Seguridad bloquea el tráfico, el servicio no funcionará, aunque el firewall interno esté bien configurado.

Asegúrate de que las siguientes reglas de **Entrada (Inbound)** estén abiertas para el Grupo de Seguridad de tu instancia EC2:

| Servicio | Protocolo | Puerto | Origen        | Descripción                                    |
|----------|-----------|--------|---------------|------------------------------------------------|
| SSH      | TCP       | 22     | Mi IP o 0.0.0.0/0 | Acceso de administración inicial.          |
| OpenVPN  | UDP       | 1194   | 0.0.0.0/0     | Permite las conexiones VPN de clientes.        |
| HTTP     | TCP       | 80     | 0.0.0.0/0     | Temporalmente para la validación de Let's Encrypt. |

---

## 2️⃣ Acceso Inicial e Instalación de Paquetes

### 2.1 Acceso Remoto

Utiliza SSH desde PowerShell, CMD o WSL (en Windows) o tu terminal de Linux/macOS, asegurando el permiso correcto de la clave:

```bash
# 1. Asegurar la clave (SOLO en Linux/WSL/macOS)
chmod 400 /ruta/a/tu/clave-aws.pem

# 2. Conectar (ej. usando el usuario 'admin' o 'ubuntu')
ssh -i /ruta/a/tu/clave-aws.pem admin@X.X.X.X
```

### 2.2 Instalación de Componentes

Una vez conectado, actualiza el sistema e instala OpenVPN y las herramientas PKI/SSL:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openvpn easy-rsa certbot iptables-persistent -y
```

---

## 3️⃣ Configuración de la PKI Privada (Easy-RSA)

Esta PKI será la **Autoridad Certificadora (CA)** para emitir y firmar los certificados de **TODOS tus clientes**.

### 3.1 Preparar Easy-RSA

```bash
sudo mkdir -p /etc/openvpn/server/easy-rsa
sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/server/easy-rsa/
cd /etc/openvpn/server/easy-rsa/
sudo ./easyrsa init-pki
```

### 3.2 Crear la CA Raíz y Parámetros DH

```bash
# Crear la CA (establece una contraseña segura y un CN, ej. OpenVPN-ASIR-CA)
sudo ./easyrsa build-ca

# Generar parámetros Diffie-Hellman (tarda unos minutos)
sudo ./easyrsa gen-dh

# Mover archivos clave al directorio de configuración de OpenVPN
sudo cp pki/ca.crt /etc/openvpn/server/
sudo cp pki/dh.pem /etc/openvpn/server/
```

---

## 4️⃣ Certificado del Servidor (Let's Encrypt)

Obtendremos un certificado X.509 válido públicamente para el servidor.

### 4.1 Ejecutar Certbot

El puerto 80 debe estar abierto en el Grupo de Seguridad.

```bash
# Reemplaza vpn.midominio.com con tu dominio real
sudo certbot certonly --standalone -d vpn.midominio.com
```

Acepta los términos y proporciona una dirección de email.

### 4.2 Crear Enlaces Simbólicos

Esto es crucial para que OpenVPN acceda a los archivos y para la renovación automática.

```bash
# Definir variables (Ajustar si tu dominio es diferente)
LE_PATH=/etc/letsencrypt/live/vpn.midominio.com
OPENVPN_DIR=/etc/openvpn/server

# Crear enlaces simbólicos para la clave privada y el certificado completo
sudo ln -s $LE_PATH/privkey.pem $OPENVPN_DIR/server-le.key
sudo ln -s $LE_PATH/fullchain.pem $OPENVPN_DIR/server-le.crt
```

---

## 5️⃣ Fichero de Configuración del Servidor (server.conf)

Crear el archivo de configuración en `/etc/openvpn/server/server.conf`.

```bash
sudo nano /etc/openvpn/server/server.conf
```

### Contenido de **server.conf**

```conf
# ------------------------------------------------------------
# 1. Configuración Básica
# ------------------------------------------------------------
port 1194                       # Puerto por defecto
proto udp                       # UDP es más rápido que TCP para VPN
dev tun                         # Crea la interfaz de túnel
topology subnet                 # Asigna IPs de la subred
server 10.8.0.0 255.255.255.0   # Subred de clientes VPN
ifconfig-pool-persist ipp.txt

# ------------------------------------------------------------
# 2. Archivos de PKI y Cifrado
# ------------------------------------------------------------
# CA privada (para autenticar clientes)
ca ca.crt

# Certificado y Clave de SERVIDOR (de Let's Encrypt)
cert server-le.crt
key server-le.key

# Parámetros DH
dh dh.pem

# ------------------------------------------------------------
# 3. Push: Opciones para los clientes
# ------------------------------------------------------------
# Redirige todo el tráfico del cliente a través del túnel VPN
push "redirect-gateway def1 bypass-dhcp"

# DNS de Google (o tu DNS preferido)
push "dhcp-option DNS 8.8.8.8"

# ------------------------------------------------------------
# 4. Seguridad y Estabilidad
# ------------------------------------------------------------
cipher AES-256-CBC
auth SHA256
keepalive 10 120               # Ping cada 10s, timeout a los 120s
user nobody                    # Ejecutar el proceso con menos privilegios
group nogroup
persist-key                    # Previene la pérdida de conexión durante la reconexión
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
explicit-exit-notify 1
```

---

## 6️⃣ Configuración del Firewall Interno (IPTABLES) y NAT

En AWS, la interfaz pública es típicamente **eth0**. Usaremos **iptables** para configurar el firewall y NAT.

### 6.1 Habilitar IP Forwarding

```bash
sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sudo sysctl -p
```

### 6.2 Configurar IPTABLES y NAT (Masquerade)

Añadir las reglas permanentes de OpenVPN:

```bash
# 1. Permitir tráfico de entrada de OpenVPN (1194/UDP)
sudo iptables -A INPUT -p udp --dport 1194 -j ACCEPT

# 2. Permitir el tráfico de reenvío (FORWARD) a través del túnel
sudo iptables -A FORWARD -i tun0 -j ACCEPT
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

# 3. CRÍTICO: Configurar NAT (Masquerading) para salida a Internet
# Esto traduce 10.8.0.0/24 a la IP pública de eth0
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

# 4. Guardar la Persistencia de las Reglas
sudo netfilter-persistent save
```

### 6.3 Verificar las Reglas

```bash
# Para ver las reglas de la tabla FILTER
sudo iptables -L

# Para ver las reglas de la tabla NAT (Masquerading)
sudo iptables -t nat -L
```

---

## 7️⃣ Gestión del Servicio y Automatización

### 7.1 Iniciar y Habilitar OpenVPN

```bash
sudo systemctl start openvpn-server@server
sudo systemctl enable openvpn-server@server
sudo systemctl status openvpn-server@server
```

### 7.2 Automatizar la Renovación del Certificado

El cron job intentará renovar el certificado y recargará el servicio si lo consigue, cargando el nuevo certificado X.509.

```bash
sudo crontab -e
```

Añadir la siguiente línea al crontab:

```bash
0 */12 * * * /usr/bin/certbot renew --quiet && systemctl reload openvpn-server@server
```

---

## 8️⃣ Script de Generación de Clientes

Este script crea el par de clave/certificado para cada usuario y genera el archivo **.ovpn** autoconfigurado.

### Fichero: **gen_client.sh**

```bash
#!/bin/bash
# ----------------------------------------------------------------
# Script para automatizar la generación de archivos de cliente
# Autor: Fernando (Estudiante ASIR)
# ----------------------------------------------------------------

CLIENT_NAME=$1
EASY_RSA_DIR="/etc/openvpn/server/easy-rsa"
OPENVPN_SERVER_DOMAIN="vpn.midominio.com"  # <-- ¡CAMBIA ESTO!

if [ -z "$CLIENT_NAME" ]; then
    echo "Uso: $0 <nombre_cliente>"
    echo "Ejemplo: $0 fernando-movil"
    exit 1
fi

echo "Iniciando generación de PKI para el cliente: $CLIENT_NAME..."

# Generar clave y certificado
cd $EASY_RSA_DIR
sudo ./easyrsa gen-req $CLIENT_NAME nopass
sudo ./easyrsa sign-req client $CLIENT_NAME

# Capturar contenido de los archivos clave
CA_CRT=$(sudo cat $EASY_RSA_DIR/pki/ca.crt)
CLIENT_CRT=$(sudo cat $EASY_RSA_DIR/pki/issued/$CLIENT_NAME.crt)
CLIENT_KEY=$(sudo cat $EASY_RSA_DIR/pki/private/$CLIENT_NAME.key)

# Crear archivo de configuración .ovpn
echo "Creando archivo $CLIENT_NAME.ovpn..."

cat <<EOF > $CLIENT_NAME.ovpn
client
dev tun
proto udp
remote $OPENVPN_SERVER_DOMAIN 1194
resolv-retry infinite
nobind
persist-key
persist-tun
cipher AES-256-CBC
auth SHA256
verb 3

<ca>
$CA_CRT
</ca>

<cert>
$CLIENT_CRT
</cert>

<key>
$CLIENT_KEY
</key>
EOF

echo "✅ Archivo $CLIENT_NAME.ovpn creado en $EASY_RSA_DIR."
echo "¡Transfiérelo al cliente y conéctate!"
```

### Uso del Script

1. Mueve el script al directorio principal de OpenVPN y dale permisos de ejecución:

```bash
sudo mv gen_client.sh /etc/openvpn/server/
cd /etc/openvpn/server/
sudo chmod +x gen_client.sh
```

2. Ejecuta para cada nuevo cliente:

```bash
sudo ./gen_client.sh cliente_portatil
```

El archivo `cliente_portatil.ovpn` se creará en ese directorio. Tendrás que descargarlo de la instancia para usarlo.

---

## 9️⃣ Transferir el Archivo .ovpn al Cliente

### Desde Linux/macOS/WSL:

```bash
scp -i /ruta/a/tu/clave-aws.pem admin@X.X.X.X:/etc/openvpn/server/easy-rsa/cliente_portatil.ovpn .
```

### Desde Windows (PowerShell):

```powershell
scp -i C:\ruta\a\tu\clave-aws.pem admin@X.X.X.X:/etc/openvpn/server/easy-rsa/cliente_portatil.ovpn .
```

---

## 🔟 Conectar el Cliente

### En Windows:
1. Instala **OpenVPN GUI**.
2. Copia el archivo `.ovpn` a `C:\Program Files\OpenVPN\config\`.
3. Ejecuta OpenVPN GUI y conecta.

### En Linux:
```bash
sudo openvpn --config cliente_portatil.ovpn
```

### En macOS:
1. Instala **Tunnelblick**.
2. Importa el archivo `.ovpn`.
3. Conecta desde la aplicación.

### En Android / iOS:
1. Instala la app `OpenVPN Connect`.
2. Transfiere el archivo `.ovpn` al dispositivo.
3. Cuando abras la app, sube el archivo y ya te conectarás a la VPN.

> [!WARNING]
> En el archivo .ovp que transfieres al dispositivo android o iOS, es necesario modificar la línea *tls-auth [ta.key] 1* y cambiarla por *key-direction 1*.

---

## 🎯 Verificación de la Conexión

Una vez conectado, verifica tu IP pública:

```bash
curl ifconfig.me
```

Debería mostrar la IP pública de tu servidor AWS, confirmando que todo el tráfico pasa por la VPN.

---

## 📚 Recursos Adicionales

- [Documentación oficial de OpenVPN](https://openvpn.net/community-resources/)
- [Easy-RSA en GitHub](https://github.com/OpenVPN/easy-rsa)
- [Let's Encrypt](https://letsencrypt.org/)

---

## 🛡️ Consideraciones de Seguridad

- **Mantén actualizado el sistema:** `sudo apt update && sudo apt upgrade`
- **Monitoriza los logs:** `sudo tail -f /var/log/openvpn/openvpn-status.log`
- **Revoca certificados comprometidos:** `sudo ./easyrsa revoke <nombre_cliente>`
- **Cambia el puerto por defecto** si experimentas bloqueos de ISP
- **Implementa fail2ban** para proteger contra ataques de fuerza bruta SSH

---

## 📝 Licencia

Este proyecto es de código abierto y está disponible bajo la licencia MIT.

---

<div align="center">

### ⭐ Si este proyecto te resultó útil, considera darle una estrella en GitHub

**[⬆ Volver al README](README.md)**

</div>
