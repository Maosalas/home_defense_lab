# 🏠 Home Defense Lab

Tu propio laboratorio de **ciberseguridad defensiva en casa** 🛡️💻  
Un entorno personal diseñado para practicar monitoreo, análisis y detección de amenazas, utilizando únicamente herramientas gratuitas y máquinas virtuales. Todo corre en un entorno controlado que simula escenarios reales de ataques, ideal para mejorar habilidades defensivas.

---

## 🔧 ¿Qué incluye el laboratorio?

- 🧠 **Splunk en Docker**  
  Para análisis de logs, visualización y creación de alertas personalizadas.

- 🪟 **Windows Server + Windows 10 VM**  
  Simulación de un dominio activo con políticas de seguridad configuradas.

- 🐧 **Ubuntu VM con rsyslog**  
  Simulación de un servidor web enviando eventos y tráfico realista.

- ⚔️ **Simulación de ataques con Atomic Red Team**  
  Basados en la matriz de MITRE ATT&CK para recrear técnicas utilizadas por atacantes reales.

---

## 🔍 Eventos y comportamientos monitoreados

- Ejecuciones de PowerShell
- Creación de procesos (Sysmon + Event ID 4688)
- Fallos de autenticación (brute force, password spraying)
- Movimientos laterales y cambios en el registro de Windows

---

## 💡 ¿Para qué sirve?

Este laboratorio me permite:

- Practicar detección en tiempo real
- Fortalecer el uso de herramientas SIEM (como Splunk)
- Prepararme para responder ante escenarios profesionales de ciberseguridad
- Hacerlo todo dentro de un entorno seguro y controlado

---

## ⚙️ 1. Entorno de Virtualización

## 🖥️ Especificaciones de Máquinas Virtuales

| Nombre de la VM     | Sistema Operativo         | Rol / Propósito                        | CPU     | RAM     | Disco       |
|---------------------|---------------------------|----------------------------------------|---------|---------|-------------|
| `host-ubuntu`       | Ubuntu 22.04              | Host principal / Docker + VirtualBox   | 4 vCPU  | 12 GB   | 100 GB SSD  |
| `splunk-docker`     | Contenedor Docker         | Instancia Splunk                       | Usa recursos del host                 |
| `dc-winserver2019`  | Windows Server 2019       | Controlador de Dominio (AD DS + DNS)   | 2 vCPU  | 4 GB    | 60 GB       |
| `win10-client`      | Windows 10 Enterprise     | Cliente unido a dominio `lab.local`    | 2 vCPU  | 4 GB    | 60 GB       |
| `ubuntu-web`        | Ubuntu 22.04              | Servidor web (Apache/SSH + rsyslog)    | 1 vCPU  | 2 GB    | 30 GB       |

> 💡 Notas:
> - Las VMs Windows pueden usar snapshots para restaurar tras simulaciones de ataque.


---

## 🐳 2. Instalación de Splunk en Docker
Dentro de la VM de Ubuntu vamos a crear un contenedor de Splunk con lo siguiente:
- **Imagen oficial:** `splunk/splunk:latest`
- **Comando de despliegue:**

```bash
docker run -d \
  -p 8000:8000 -p 9997:9997 \
  -e "SPLUNK_START_ARGS=--accept-license" \
  -e "SPLUNK_PASSWORD=tuPassword123" \
  -v ~/splunk-data/etc:/opt/splunk/etc \
  -v ~/splunk-data/var:/opt/splunk/var \
  --restart=always \
  --name splunk \
  splunk/splunk:latest
```
## 🪟 3. Máquinas Virtuales Windows

### 🖥️ Windows Server 2019 – Controlador de Dominio (DC)

Esta máquina se configura como controlador de dominio (DC) con Active Directory y DNS para crear el entorno corporativo simulado `lab.local`.

#### 🔧 Pasos de configuración:

1. **Instalar Windows Server 2019** en una nueva VM (mínimo 2 vCPU, 4 GB RAM, 60 GB disco).
2. Cambiar el nombre de host, por ejemplo: `dc-winserver2019`.
3. Asignar una IP estática (ej: `192.168.56.10`) y configurar DNS apuntando a sí misma.
4. Abrir el **Administrador del Servidor** y seleccionar:
   - **Agregar roles y características** → `Active Directory Domain Services (AD DS)` y `DNS Server`
5. Promocionar el servidor a **Controlador de Dominio**:
   - Crear un nuevo bosque con nombre de dominio: `lab.local`
6. Reiniciar el sistema y verificar la funcionalidad de AD y DNS.
7. Crear cuentas de usuario y grupos para pruebas (por ejemplo, `usuario1`, `ti-admin`, `marketing`).

📺 [Guía en video: Cómo configurar un DC en Server 2019](https://www.youtube.com/watch?v=h3sxduUt5a8)

## 🔐 Configuración de Políticas de Grupo (GPOs)

Desde el servidor DC, abrir la consola de **Group Policy Management (GPMC)** y configurar las siguientes GPOs para aplicar a la OU de los clientes (por ejemplo, `Computadoras`):

- Activar auditoría de eventos:
  - **Account Logon**, **Logon Events**, **Object Access**, **Process Creation**
- Habilitar **PowerShell Logging**:
  - Script Block Logging
  - Module Logging
  - Transcription
- Instalar y configurar Sysmon automáticamente (usando script GPO o manualmente).
- Aplicar restricciones como desactivar macros o RDP según el caso de uso.

---

### 💻 Windows 10 Enterprise – Cliente de Dominio

Esta máquina simula un endpoint corporativo. Requiere estar unido al dominio para recibir las políticas del DC.

#### 🔧 Pasos de configuración:

1. Instalar **Windows 10 Enterprise** en una VM (mínimo 2 vCPU, 4 GB RAM, 60 GB disco).
2. Asignar nombre de host, por ejemplo: `win10-client`.
3. Asignar IP estática y DNS apuntando al DC (ej: `192.168.56.10`).
4. Unir la máquina al dominio:
   - Sistema → Configuración avanzada → Nombre de equipo → `lab.local`
   - Reiniciar al finalizar la unión.
5. Iniciar sesión con una cuenta de dominio para validar políticas GPO aplicadas.

## 🧩 Instalación de Sysmon en Windows 10

#### 1. Descargar Sysmon:
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

#### 2. Descargar configuración recomendada:
[https://github.com/SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)

#### 3. Instalar:

```cmd
sysmon.exe -accepteula -i sysmonconfig-export.xml

```
## 📥 Instalación de Splunk Universal Forwarder (Windows Server y Windows 10)

El **Splunk Universal Forwarder** permite enviar eventos del sistema Windows al servidor Splunk central. Se debe instalar tanto en el **Windows Server (DC)** como en el **cliente Windows 10**.

### 🔧 Pasos de instalación:

1. Descargar el instalador desde el sitio oficial:  
   🔗 [https://www.splunk.com/en_us/download/universal-forwarder.html](https://www.splunk.com/en_us/download/universal-forwarder.html)

2. Ejecutar el instalador como administrador y seguir el asistente:

   - Aceptar la licencia de uso.
   - Elegir **instalación personalizada** (recomendada).
   - Establecer la contraseña para el usuario `splunk`.
   - Activar la opción de reenvío (**"Enable forwarding"**) e ingresar la IP de tu servidor Splunk (ej. `192.168.56.20`) y puerto `9997`.
   - En *Monitored Inputs*, seleccionar:
     - `Security`
     - `System`
     - `Application` (opcional)

3. Finalizar la instalación y reiniciar el servicio si es necesario:

```powershell
Get-Service SplunkForwarder
Restart-Service SplunkForwarder
```

## 🐧 4. Ubuntu VM – Servidor Web Simulado

Esta máquina simula un servidor Linux típico expuesto en red. Envia eventos mediante `rsyslog` a la instancia Splunk para análisis y correlación.

### 🛠️ Configuración básica

- **Sistema operativo:** Ubuntu 22.04 LTS
- **Nombre de host:** `ubuntu-web`
- **Recursos mínimos recomendados:** 1 vCPU, 2 GB RAM, 30 GB disco

### 🔧 Servicios instalados:

```bash
sudo apt update && sudo apt install apache2 openssh-server rsyslog -y
```
## ⚔️ 5. Simulación de Ataques – Atomic Red Team

Para validar la detección de eventos en el laboratorio, se simulan ataques reales utilizando técnicas documentadas en MITRE ATT&CK.

### 🧪 Herramienta utilizada:

🔗 [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) – Red Canary

### 🛠️ Instalación en PowerShell:

En la máquina Windows 10 unida al dominio (o también en el DC si deseas):

```powershell
Install-Module -Name Invoke-AtomicRedTeam -Force
```
▶️Pruebas ejecutadas:
```bash
Invoke-AtomicTest T1059.001  # PowerShell execution
Invoke-AtomicTest T1112      # Registry modification
Invoke-AtomicTest T1003      # Credential access
```
```bash
Get-AtomicTechnique | Where-Object {$_.SupportedPlatforms -contains "windows"}
```
---
## 📊 Consultas SPL en Splunk

A continuación, se presentan las consultas SPL utilizadas para validar la correcta recolección de eventos en el laboratorio y detectar actividades sospechosas relacionadas con procesos, autenticaciones fallidas, ejecución de PowerShell y más. Estas búsquedas están alineadas con las simulaciones realizadas mediante Atomic Red Team y la configuración defensiva del entorno.

```spl
# Procesos creados (Seguridad - EventCode 4688)
index=* EventCode=4688

# Procesos creados (Sysmon - EventCode 1)
index=* sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational EventCode=1

# Comandos ejecutados por PowerShell
index=* EventCode=4688 Process_Command_Line="*powershell*"

# Script Block Logging de PowerShell
index=* EventCode=4104

# Combinación de eventos de PowerShell para detección avanzada
index=* EventCode=4104 OR EventCode=4688 Process_Command_Line="*powershell*"

# Fallos de inicio de sesión
index=* EventCode=4625

# Acceso a archivos sensibles como SAM (indicador de Credential Dumping)
index=* EventCode=11 file_path="*sam*"

# Estadísticas por host, fuente y tipo de log
index=* | stats count by host, source, sourcetype

# Eventos más frecuentes por EventCode
index=* | stats count by EventCode

# Comandos más ejecutados (línea de comandos)
index=* | top Process_Command_Line

# Hosts que generan más eventos
index=* | top host
