# 🛡️ Cybersecurity Lab: Enterprise Infrastructure & Threat Hunting
## Semana 1: Despliegue del Núcleo de Active Directory e Identity Management (IAM)

### 📊 Topología de la Red
![Mapa de Red](../Mapa%20de%20red.jpg)

### 📋 Descripción General
En esta primera etapa, establecí los cimientos de una infraestructura corporativa segura. Se configuró el Directorio Activo (AD DS) bajo el dominio **NaveaTech.local**, aplicando principios de segmentación y hardening inicial.

---

### 🛠️ Detalles Técnicos

| Componente | Sistema Operativo | Dirección IP | Rol / Función |
| :--- | :--- | :--- | :--- |
| **DC-01** | Windows Server 2022 | `10.0.2.10` | Controlador de Dominio / DNS |
| **CL-WKST01** | Windows 11 Pro | `10.0.2.20` | Estación de Trabajo Cliente |

---

### 🚀 Hitos Alcanzados

#### ✅ Gestión de Identidades (IAM) y OUs
Se diseñó una estructura jerárquica de Unidades Organizativas (OUs) para separar activos y usuarios:
* **OU_ADMINS:** Control de cuentas con privilegios elevados.
* **OU_EQUIPOS:** Gestión de estaciones de trabajo.

#### ✅ Hardening Inicial
* Implementación del **Principio de Menor Privilegio (PoLP)**.
* Configuración de políticas de seguridad para cambio forzado de contraseña.
* Creación de cuenta administrativa nominal (`knavea`).

### 📸 Evidencia de Configuración
![Dominio Correcto](../Dominio%20correcto.png)
*Unión exitosa del cliente al dominio NaveaTech.local*

![Login Exitoso](../Inicio%20admin%20exitoso.jpg)
*Sesión iniciada con privilegios de Domain Admin*

---

### 🛡️ Enfoque Purple Team
* **Blue Team:** Configuración de GPOs restrictivas y auditoría de logs.
* **Red Team:** Preparación del entorno para futuras simulaciones de escalada de privilegios.

---
*Documentación para el proyecto de laboratorio de 180 días.*
