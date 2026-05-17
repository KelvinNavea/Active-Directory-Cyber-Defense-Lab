# Laboratorio de Active Directory (Semana 2): Automatización y Hardening Avanzado (Tiering y AppLocker)

## 📌 Descripción General del Proyecto
Este repositorio detalla los despliegues prácticos realizados durante la **Semana 2** del laboratorio de Infraestructura Segura sobre el dominio `naveatech.local`. El objetivo principal de esta etapa fue migrar de una gestión e implementación manual de objetos hacia un entorno estandarizado mediante código, para posteriormente aplicar directivas de endurecimiento (**Hardening**) bajo estándares de seguridad defensiva de nivel corporativo.

La arquitectura final resultante implementa un modelo de **Control de Acceso Basado en Roles (RBAC)**, aislamiento de identidades por capas (**Tiering**) y la neutralización perimetral de la ejecución de binarios no autorizados en las estaciones de trabajo mediante **AppLocker**.

---

## 🛠️ Arquitectura y Componentes del Entorno
* **Controlador de Dominio (Tier 0):** Windows Server 2022 (`naveatech.local`)
* **Estación de Trabajo Cliente (Tier 2):** Windows 11 Enterprise (`CL-WKST-01`)
* **Estructura del Directorio:** Unidades Organizativas (OUs) segregadas por rol operativo y departamentos.

---

## 🚀 Fase 1: Automatización y Despliegue de Infraestructura (Día 1)

Para garantizar un entorno reproducible, ágil y libre de errores humanos de configuración en la interfaz gráfica, se desarrolló y ejecutó un script en **PowerShell** para automatizar la creación de la estructura base del dominio.

### 1. Funcionalidades del Script de Aprovisionamiento:
* Creación de las Unidades Organizativas jerárquicas esenciales (`OU_IT`, `OU_EQUIPOS`, `OU_RRHH`, `OU_VENTAS`).
* Creación y aprovisionamiento de grupos globales de seguridad para aplicar el modelo RBAC.
* Alta centralizada de cuentas de usuarios estándar y cuentas técnicas de soporte con contraseñas seguras preconfiguradas.

### 2. Código del Script de Automatización:

```powershell
Import-Module ActiveDirectory

# 1. Definir solo las nuevas OUs de departamentos que faltan
$OUs = @(
    "OU=OU_VENTAS,DC=naveatech,DC=local",
    "OU=OU_RRHH,DC=naveatech,DC=local",
    "OU=OU_IT,DC=naveatech,DC=local"
)

# Crear las OUs de departamentos
foreach ($OU in $OUs) {
    if (-not (Get-ADOrganizationalUnit -Filter "DistinguishedName -eq '$OU'")) {
        New-ADOrganizationalUnit -Name ($OU -split ",")[0].Replace("OU=","") -Path "DC=naveatech,DC=local" -ProtectedFromAccidentalDeletion $true
        Write-Host "[+] Creada la OU: $OU" -ForegroundColor Green
    }
}

# 2. Crear los Grupos de Seguridad (Roles) dentro de las nuevas OUs
New-ADGroup -Name "G_Ventas_Usuarios" -GroupScope Global -GroupCategory Security -Path "OU=OU_VENTAS,DC=naveatech,DC=local" -Description "Usuarios estándar de Ventas"
New-ADGroup -Name "G_RRHH_Usuarios" -GroupScope Global -GroupCategory Security -Path "OU=OU_RRHH,DC=naveatech,DC=local" -Description "Usuarios estándar de RRHH"
New-ADGroup -Name "G_IT_Soporte" -GroupScope Global -GroupCategory Security -Path "OU=OU_IT,DC=naveatech,DC=local" -Description "Personal técnico de soporte (Tier 2)"

# 3. Crear Usuarios de Prueba (Contraseña estándar de laboratorio: NaveaTech2026!)
$SecurePassword = ConvertTo-SecureString "NaveaTech2026!" -AsPlainText -Force

New-ADUser -Name "Pedro Ventas" -GivenName "Pedro" -Surname "Ventas" -SamAccountName "pedro.ventas" -UserPrincipalName "pedro.ventas@naveatech.local" -Path "OU=OU_VENTAS,DC=naveatech,DC=local" -AccountPassword $SecurePassword -Enable $true
New-ADUser -Name "Ana RRHH" -GivenName "Ana" -Surname "RRHH" -SamAccountName "ana.rrhh" -UserPrincipalName "ana.rrhh@naveatech.local" -Path "OU=OU_RRHH,DC=naveatech,DC=local" -AccountPassword $SecurePassword -Enable $true
New-ADUser -Name "Kelvin Soporte" -GivenName "Kelvin" -Surname "Soporte" -SamAccountName "knavea.tec" -UserPrincipalName "knavea.tec@naveatech.local" -Path "OU=OU_IT,DC=naveatech,DC=local" -AccountPassword $SecurePassword -Enable $true

# 4. Asignar los usuarios a sus respectivos grupos
Add-ADGroupMember -Identity "G_Ventas_Usuarios" -Members "pedro.ventas"
Add-ADGroupMember -Identity "G_RRHH_Usuarios" -Members "ana.rrhh"
Add-ADGroupMember -Identity "G_IT_Soporte" -Members "knavea.tec"

Write-Host "[+] Estructura departamental y usuarios de prueba creados con éxito." -ForegroundColor Cyan
```

---

## 🛡️ Fase 2: Hardening e Implementación Defensiva (Día 2)

Con la topología de red y los objetos de Active Directory desplegados de forma automatizada, se procedió al endurecimiento del entorno utilizando exclusivamente Directivas de Grupo (**GPOs**) para mitigar vectores de ataque reales.

### 1. Aislamiento de Privilegios (Tiering de Identidades y Restricción de Logueo)
El uso de cuentas con privilegios jerárquicos de **Tier 0** (como `Domain Admins` o el `Administrador` nativo del dominio) para tareas rutinarias en estaciones de trabajo de **Tier 2** expone credenciales críticas a técnicas de extracción de memoria (*Credential Dumping* vía `lsass.exe`). Si un equipo cliente se ve comprometido, un atacante podría extraer el hash del Administrador del Dominio, logrando el compromiso total de la infraestructura (movimiento lateral horizontal y vertical).

Para mitigar este vector de ataque, se aplicó una estrategia de Tiering simétrica dividida en dos directivas estrictas:

#### A. GPO de Restricción e Inhabilitación de Logueo (Tier 0 en Tier 2)
Para bloquear de forma mandatoria que las credenciales de alta jerarquía toquen las estaciones de trabajo, se configuró una directiva vinculada a la **`OU_EQUIPOS`**:

1. **Ruta de la directiva:** `Configuración del equipo` > `Directivas` > `Configuración de Windows` > `Configuración de seguridad` > `Directivas locales` > `Asignación de derechos de usuario`.
2. **Denegación de acceso local:** Se editó la directiva **"Denegar el inicio de sesión de forma local"** (Deny log on locally) y se añadieron explícitamente los grupos:
   * `NAVEATECH\Domain Admins`
   * `NAVEATECH\Enterprise Admins`
3. **Denegación de acceso remoto:** Se editó la directiva **"Denegar el inicio de sesión a través de Servicios de Escritorio remoto"** (Deny log on through Remote Desktop Services), añadiendo los mismos grupos de Tier 0.

**Resultado:** Si un Administrador del Dominio intenta iniciar sesión física o remotamente en la máquina cliente `CL-WKST-01`, el sistema operativo deniega el acceso de inmediato antes de generar cualquier proceso o token en memoria, neutralizando el vector de persistencia.

#### B. GPO de Delegación Técnico (Grupos Restringidos para Soporte Local)
Al quedar las cuentas de Tier 0 completamente bloqueadas para operar en la capa de usuario, se requirió delegar capacidades administrativas locales controladas para el personal de IT sin comprometer el dominio.

1. Se configuró la directiva `GPO_Configuracion_Soporte_Local` vinculada a la `OU_EQUIPOS`.
2. **Ruta de la directiva:** `Configuración del equipo` > `Directivas` > `Configuración de Windows` > `Configuración de seguridad` > `Grupos restringidos`.
3. Se añadió el grupo global del dominio: **`NAVEATECH\G_IT_Soporte`**.
4. En el apartado **"Este grupo es miembro de:"**, se inyectó el grupo local integrado de los endpoints: **`Administradores`**.

**Resultado:** Las identidades globales de administración permanecen aisladas en su capa. El personal de IT opera sobre el entorno cliente utilizando exclusivamente una cuenta técnica intermedia (`knavea.tec`) que posee privilegios administrativos locales limitados de forma estricta a ese parque de equipos.

### 2. Control de Ejecución Perimetral (AppLocker)

#### 📋 Contexto Operativo y Caso Real (Shadow IT)
La implementación de esta directiva surge como respuesta directa a incidentes comunes en entornos corporativos reales. En la experiencia de campo del área, se detectó el despliegue no autorizado de la suite **WPS Office** por parte de usuarios finales. Al tratarse de un software que permite la instalación e interacción dentro del espacio del perfil de usuario sin requerir elevación de privilegios de Administrador (evadiendo el control de UAC), el personal lograba ejecutar código ajeno al catálogo homologado por la compañía.

Esto introduce riesgos críticos de *Shadow IT*, fuga de información y potencial ejecución de vectores de persistencia si los binarios son descargados de fuentes no oficiales. Por lo tanto, la suite WPS Office se utiliza en este laboratorio **estrictamente como herramienta de prueba y simulación** para evaluar el comportamiento de los bloqueos perimetrales.

#### 🔧 Solución e Implementación mediante AppLocker:
1. **Configuración del Servicio Base:** Se forzó el arranque automático en las estaciones de trabajo del servicio encargado de interceptar los procesos: **Identidad de aplicación** (`AppIDSvc`), configurado a través de GPO en el nodo de `Servicios del sistema`.
2. **Definición de Reglas Ejecutables:** En el nodo `Directivas de control de aplicaciones` > `AppLocker` > `Reglas ejecutables`, se generaron las **Reglas Predetermiandas** para permitir la ejecución legítima del sistema operativo en:
   * `%PROGRAMFILES%\*`
   * `%WINDIR%\*`
3. **Modo de Cumplimiento:** Se modificaron las propiedades de AppLocker para pasar del estado de auditoría a la exigencia perimetral, marcando la opción de **Aplicar reglas** (Enforce rules) de manera mandatoria.

### 3. Directivas de Bloqueo de Cuenta y Expiración de Contraseñas (Account Policies)
Para mitigar vectores de ataque por fuerza bruta, diccionarios o rociado de contraseñas (*Password Spraying*), se modificaron las directivas predeterminadas del dominio para forzar un ciclo de vida seguro de las credenciales y la suspensión temporal de identidades bajo sospecha de compromiso.

Estas configuraciones se aplicaron directamente sobre la **`Default Domain Policy`** en el nodo: `Configuración del equipo` > `Directivas` > `Configuración de Windows` > `Configuración de seguridad` > `Directivas de cuenta`.

#### A. Directiva de Bloqueo de Cuenta (*Account Lockout Policy*)
* **Umbral de bloqueo de cuenta:** Se estableció en **`4` intentos de inicio de sesión no válidos**. Si un atacante o usuario erra la contraseña más de 4 veces consecutivas, la cuenta se inhabilita automáticamente en el controlador de dominio.
* **Duración del bloqueo de cuenta:** Configurado en **`30` minutos**. Una vez bloqueada, la cuenta permanecerá inactiva durante este período a menos que un administrador de IT la desbloquee manualmente antes.
* **Restablecer el contador de bloqueos de cuenta después de:** Configurado en **`30` minutos**. Determina cuánto tiempo debe pasar entre intentos fallidos para que el contador interno de Windows vuelva a cero.

#### B. Directiva de Contraseñas (*Password Policy*)
* **Vigencia máxima de la contraseña:** Se estableció un tiempo estimado de **`60` días** para la expiración. Al cumplirse este plazo, el sistema operativo fuerza al usuario a renovar sus credenciales de forma mandatoria para mitigar la persistencia de contraseñas potencialmente filtradas.
* **Vigencia mínima de la contraseña:** Configurado en **`1` día** (evita que el usuario cambie la contraseña 24 veces seguidas el mismo día para volver a usar su clave vieja).
* **Historial de contraseñas:** Se configuró recordar las últimas **`24` contraseñas**, bloqueando la reutilización inmediata de credenciales anteriores.
---

## 🔬 Escenarios de Verificación y Auditoría de Roles (RBAC)

Para comprobar la efectividad y el cumplimiento cruzado de las directivas en el laboratorio, se auditaron las estaciones de trabajo bajo dos perfiles con alcances totalmente opuestos, utilizando la suite mencionada como elemento de control:

### Caso A: Validación del Rol Técnico (`knavea.tec`)
* **Comportamiento observado:** El usuario inicia sesión en la máquina cliente (`CL-WKST-01`). Al estar integrado dentro de los administradores locales por la directiva de grupos restringidos, el sistema operativo le otorga capacidades de gestión sobre el endpoint.
* **Resultado:** Puede ejecutar e instalar software corporativo (como la suite WPS Office, utilizada aquí como binario de prueba homologado para soporte) de forma abierta y transparente. AppLocker valida que la identidad operativa del técnico cuenta con privilegios administrativos locales, permitiendo las tareas rutinarias de soporte técnico.

### Caso B: Validación del Rol de Usuario Estándar (`ana.rrhh`)
* **Comportamiento observado:** La usuaria descarga el instalador de la suite de uso no autorizado, el cual intenta procesar y ejecutar sus hilos silenciosamente dentro de su espacio de usuario (`Downloads` o carpetas de `AppData`).
* **Resultado:** El motor del kernel intercepta la llamada del proceso. Al validar que la ruta del ejecutable pertenece al perfil de usuario y no a los directorios protegidos y autorizados por el administrador (`Windows` o `Program Files`), AppLocker bloquea el binario en seco antes de tocar la memoria, arrojando el mensaje nativo de Windows:
  > 🛑 **"El administrador del sistema bloqueó esta aplicación."**

---

## 🎯 Conclusiones del Laboratorio
El cierre de la Semana 2 consolida un entorno de Directorio Activo bajo criterios estrictos de **Privilegio Mínimo** y **Reducción de la Superficie de Ataque**:
1. Las identidades se encuentran segregadas y los permisos de soporte local están centralizados sin riesgo de escalada de privilegios horizontal ni vertical.
2. Se neutraliza por completo la capacidad de ejecutar binarios arbitrarios en el espacio de usuario final, blindando la red interna contra técnicas comunes de ingeniería social y malware persistente.

---
