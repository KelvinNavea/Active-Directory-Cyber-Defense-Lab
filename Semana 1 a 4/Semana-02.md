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

### 1. Aislamiento de Privilegios (Tiering mediante Grupos Restringidos)
El uso de cuentas de alto privilegio como *Administrador del Dominio* (Tier 0) para tareas cotidianas de soporte técnico en puestos de usuario (Tier 2) expone credenciales críticas a ataques de extracción en memoria (Credential Dumping vía LSASS). 

Para mitigar este riesgo y aislar los privilegios:
1. Se configuró la directiva `GPO_Configuracion_Soporte_Local` vinculada directamente a la `OU_EQUIPOS`.
2. Dentro del Editor de GPO, se navegó a: `Configuración del equipo` > `Directivas` > `Configuración de Windows` > `Configuración de seguridad` > `Grupos restringidos`.
3. Se añadió el grupo global del dominio: **`NAVEATECH\G_IT_Soporte`**.
4. En la sección **"Este grupo es miembro de:"**, se configuró el grupo nativo del sistema local: **`Administradores`** (mapeado desde la carpeta integrada `naveatech.local/Builtin`).

**Resultado:** Al aplicar `gpupdate /force`, cualquier máquina unida a la `OU_EQUIPOS` inyecta automáticamente al grupo de técnicos como administradores locales. Esto permite realizar tareas de soporte completo sin comprometer las llaves maestras del dominio en estaciones de trabajo desprotegidas.

### 2. Control de Ejecución Perimetral (AppLocker)
**El problema del "Gris de Seguridad":** Por defecto, un usuario sin privilegios administrativos no puede escribir en `C:\Program Files`. Sin embargo, muchas aplicaciones no autorizadas e instaladores de malware modernos evaden esta restricción instalándose en el espacio exclusivo del perfil de usuario: `C:\Users\<Usuario>\AppData\Local` o ejecutándose directamente desde la carpeta de descargas. Al tener control sobre su propia ruta, el usuario ejecuta software sin requerir elevación de UAC.

**Solución mediante AppLocker:**
1. **Configuración del Servicio Base:** Se forzó el arranque automático en las estaciones de trabajo del servicio encargado de interceptar los procesos: **Identidad de aplicación** (`AppIDSvc`), configurado a través de GPO en el nodo de `Servicios del sistema`.
2. **Definición de Reglas Ejecutables:** En el nodo `Directivas de control de aplicaciones` > `AppLocker` > `Reglas ejecutables`, se generaron las **Reglas Predeterminadas** para permitir la ejecución legítima del sistema operativo en:
   * `%PROGRAMFILES%\*`
   * `%WINDIR%\*`
3. **Modo de Cumplimiento:** Se modificaron las propiedades de AppLocker para pasar del estado de auditoría a la exigencia perimetral, marcando la opción de **Aplicar reglas** (Enforce rules) de manera mandatoria.

---

## 🔬 Escenarios de Verificación y Auditoría de Roles (RBAC)

Para comprobar la efectividad y el cumplimiento cruzado de las directivas en el laboratorio, se auditaron las estaciones de trabajo bajo dos perfiles con alcances totalmente opuestos:

### Caso A: Validación del Rol Técnico (`knavea.tec`)
* **Comportamiento observado:** El usuario inicia sesión en la máquina cliente (`CL-WKST-01`). Al estar integrado dentro de los administradores locales por la directiva de grupos restringidos, el sistema operativo le permite interactuar con instaladores tradicionales.
* **Resultado:** Puede ejecutar e instalar software corporativo (como la suite WPS Office) de forma abierta y transparente. AppLocker valida que la identidad operativa del técnico cuenta con privilegios administrativos locales, permitiendo las tareas rutinarias de soporte técnico.

### Caso B: Validación del Rol de Usuario Estándar (`ana.rrhh`)
* **Comportamiento observado:** La usuaria descarga un binario, instalador portable o software no homologado que intenta instalarse de manera silenciosa dentro de su espacio de usuario (`Downloads` o carpetas de `AppData`).
* **Resultado:** El motor del kernel intercepta la llamada del proceso. Al validar que la ruta del ejecutable pertenece al perfil de usuario y no a los directorios protegidos por el administrador (`Windows` o `Program Files`), AppLocker bloquea el binario en seco antes de tocar la memoria, arrojando el mensaje nativo de Windows:
  > 🛑 **"El administrador del sistema bloqueó esta aplicación."**

---

## 🎯 Conclusiones del Laboratorio
El cierre de la Semana 2 consolida un entorno de Directorio Activo bajo criterios estrictos de **Privilegio Mínimo** y **Reducción de la Superficie de Ataque**:
1. Las identidades se encuentran segregadas y los permisos de soporte local están centralizados sin riesgo de escalada de privilegios horizontal ni vertical.
2. Se neutraliza por completo la capacidad de ejecutar binarios arbitrarios en el espacio de usuario final, blindando la red interna contra técnicas comunes de ingeniería social y malware persistente.

---
