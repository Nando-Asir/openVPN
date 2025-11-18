# 🔐 Servidor OpenVPN Profesional en AWS

[![Debian](https://img.shields.io/badge/Debian-13%20Trixie-A81D33?logo=debian)](https://www.debian.org/)
[![OpenVPN](https://img.shields.io/badge/OpenVPN-2.6+-EA7E20?logo=openvpn)](https://openvpn.net/)
[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazon-aws)](https://aws.amazon.com/ec2/)
[![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-SSL-003A70?logo=letsencrypt)](https://letsencrypt.org/)

> Implementación completa de un servidor OpenVPN seguro en Amazon Web Services utilizando certificados SSL públicos (Let's Encrypt) para el servidor y PKI privada (Easy-RSA) para la autenticación de clientes.

---

## 📋 Tabla de Contenidos

- [Descripción del Proyecto](#-descripción-del-proyecto)
- [Características](#-características)
- [Arquitectura](#-arquitectura)
- [Requisitos Previos](#-requisitos-previos)
- [Instalación Rápida](#-instalación-rápida)
- [Documentación Completa](#-documentación-completa)
- [Uso](#-uso)
- [Troubleshooting](#-troubleshooting)
- [Contribuciones](#-contribuciones)
- [Autor](#-autor)
- [Licencia](#-licencia)

---

## 🎯 Descripción del Proyecto

Este proyecto implementa un **servidor OpenVPN de nivel empresarial** en una instancia EC2 de AWS con Debian 13, combinando las mejores prácticas de seguridad:

- **Certificados SSL públicos** (Let's Encrypt) para validación del servidor
- **PKI privada** (Easy-RSA) para autenticación robusta de clientes
- **NAT/Masquerading** con iptables para enrutamiento transparente
- **Renovación automática** de certificados SSL
- **Script automatizado** para generación de configuraciones de cliente

Este proyecto fue desarrollado como práctica del módulo **Servicios de Red e Internet** del ciclo formativo **ASIR** (Administración de Sistemas Informáticos en Red).

---

## ✨ Características

- ✅ **Seguridad de grado empresarial** con cifrado AES-256-CBC
- ✅ **Certificados SSL válidos** mediante Let's Encrypt
- ✅ **Generación automatizada de clientes** con script bash
- ✅ **Archivos .ovpn autónomos** (no requieren archivos adicionales)
- ✅ **Renovación automática** de certificados SSL vía cron
- ✅ **NAT/Masquerading** configurado para acceso completo a Internet
- ✅ **Firewall iptables** configurado y persistente
- ✅ **Compatibilidad multiplataforma**: Windows, Linux, macOS, Android, iOS

---

## 🏗️ Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERNET                               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  AWS Security Group  │
              │  - SSH (22/TCP)      │
              │  - OpenVPN (1194/UDP)│
              │  - HTTP (80/TCP)     │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   EC2 Instance       │
              │   Debian 13 Trixie   │
              │                      │
              │  ┌────────────────┐  │
              │  │   OpenVPN      │  │
              │  │   Server       │  │
              │  │                │  │
              │  │ • Let's Encrypt│  │
              │  │ • Easy-RSA PKI │  │
              │  │ • iptables NAT │  │
              │  └────────────────┘  │
              │                      │
              │   Interface: tun0    │
              │   Red: 10.8.0.0/24   │
              └──────────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼────┐          ┌────▼────┐
         │ Cliente │          │ Cliente │
         │    1    │          │    2    │
         │ Portátil│          │  Móvil  │
         └─────────┘          └─────────┘
```

---

## 📦 Requisitos Previos

### Infraestructura AWS
- ✅ Cuenta de AWS activa
- ✅ Instancia EC2 con Debian 13 "Trixie"
- ✅ IP Elástica (EIP) asignada a la instancia
- ✅ Par de claves SSH (archivo .pem)

### DNS
- ✅ Dominio propio o subdominio (ej: `vpn.tudominio.com`)
- ✅ Registro A apuntando a la EIP de tu instancia

### Configuración de Security Group
| Servicio | Protocolo | Puerto | Origen | Descripción |
|----------|-----------|--------|--------|-------------|
| SSH | TCP | 22 | Tu IP | Administración |
| OpenVPN | UDP | 1194 | 0.0.0.0/0 | Conexiones VPN |
| HTTP | TCP | 80 | 0.0.0.0/0 | Validación Let's Encrypt |

---

## 🚀 Instalación Rápida

### Paso 1: Conectar a la instancia

```bash
chmod 400 tu-clave-aws.pem
ssh -i tu-clave-aws.pem admin@TU_IP_PUBLICA
```

### Paso 2: Instalar dependencias

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa certbot iptables-persistent -y
```

### Paso 3: Configurar PKI privada

```bash
sudo mkdir -p /etc/openvpn/server/easy-rsa
sudo cp -r /usr/share/easy-rsa/* /etc/openvpn/server/easy-rsa/
cd /etc/openvpn/server/easy-rsa/
sudo ./easyrsa init-pki
sudo ./easyrsa build-ca
sudo ./easyrsa gen-dh
```

### Paso 4: Obtener certificado SSL

```bash
sudo certbot certonly --standalone -d vpn.tudominio.com
```

### Paso 5: Configurar OpenVPN

Sigue la [documentación completa](./GUIA_COMPLETA.md) para configurar el archivo `server.conf`, iptables, y el script de generación de clientes.

---

## 📖 Documentación Completa

Para instrucciones detalladas paso a paso, consulta:

📄 **[GUIA_COMPLETA.md](./GUIA_COMPLETA.md)** - Documentación técnica exhaustiva

---

## 💻 Uso

### Generar configuración de cliente

```bash
cd /etc/openvpn/server/
sudo ./gen_client.sh nombre_cliente
```

### Descargar el archivo .ovpn

```bash
# Desde tu máquina local
scp -i tu-clave-aws.pem admin@TU_IP:/etc/openvpn/server/easy-rsa/nombre_cliente.ovpn .
```

### Conectar desde el cliente

**Windows:**
1. Instala [OpenVPN GUI](https://openvpn.net/community-downloads/)
2. Copia el `.ovpn` a `C:\Program Files\OpenVPN\config\`
3. Conecta desde la aplicación

**Linux:**
```bash
sudo openvpn --config nombre_cliente.ovpn
```

**macOS:**
1. Instala [Tunnelblick](https://tunnelblick.net/)
2. Importa el archivo `.ovpn`

---

## 🔧 Troubleshooting

### El servidor no arranca

```bash
# Verificar logs
sudo journalctl -u openvpn-server@server -xe

# Verificar configuración
sudo openvpn --config /etc/openvpn/server/server.conf
```

### El cliente se conecta pero no hay Internet

```bash
# Verificar IP forwarding
sysctl net.ipv4.ip_forward

# Verificar reglas NAT
sudo iptables -t nat -L -v
```

### Problemas con certificados

```bash
# Verificar enlaces simbólicos
ls -la /etc/openvpn/server/server-le.*

# Renovar certificado manualmente
sudo certbot renew --force-renewal
```

### Verificar estado del servicio

```bash
sudo systemctl status openvpn-server@server
sudo cat /var/log/openvpn/openvpn-status.log
```

---

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor:

1. Haz fork del repositorio
2. Crea una rama para tu feature (`git checkout -b feature/mejora`)
3. Commit tus cambios (`git commit -am 'Añade nueva característica'`)
4. Push a la rama (`git push origin feature/mejora`)
5. Abre un Pull Request

---

## 👨‍💻 Autor

**Fernando** - Estudiante ASIR (Administración de Sistemas Informáticos en Red)

📧 Contacto: ferdurave@gmail.com  
🔗 LinkedIn: Fernando Durán  
📁 GitHub: [@Nando-Asir](https://github.com/Nando-Asir)

---

## 📄 Licencia

Este proyecto está bajo la Licencia MIT. Consulta el archivo [LICENSE](LICENSE) para más detalles.

---

## 🙏 Agradecimientos

- [OpenVPN Community](https://openvpn.net/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Debian Project](https://www.debian.org/)
- [Easy-RSA](https://github.com/OpenVPN/easy-rsa)

---

## 📊 Estado del Proyecto

🟢 **Activo** - Este proyecto se mantiene activamente y se aceptan contribuciones.

**Última actualización:** Noviembre 2024  
**Versión:** 1.0.0

---

<div align="center">

### ⭐ Si este proyecto te resultó útil, considera darle una estrella en GitHub

**[⬆ Volver arriba](#-servidor-openvpn-profesional-en-aws)**

</div>
