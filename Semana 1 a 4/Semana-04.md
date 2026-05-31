# Laboratorio de Active Directory (Semana 4): Protección de Infraestructura y FSRM 🛡️

## 📌 Descripción General del Proyecto
Este repositorio documenta las medidas de *hardening* y seguridad aplicadas al dominio `naveatech.local`. Durante la semana 4, nos enfocamos en proteger la integridad lógica del Active Directory y en implementar un sistema de control y auditoría de archivos mediante FSRM.

## 🛡️ Sábado: Blindaje y Recuperación en Active Directory
Implementamos configuraciones críticas para proteger la estructura del dominio frente a errores humanos y posibles ataques internos.

*   **Protección de Objetos (Accidental Deletion):** Configuramos las Unidades Organizativas (OU) con la propiedad "Proteger objeto contra eliminación accidental" habilitada. Esto añade una barrera de seguridad necesaria para evitar que una estructura completa del dominio sea borrada por error o por un usuario con privilegios que no debería realizar esa acción.
*   **Activación de la Papelera de Reciclaje (AD Recycle Bin):** Habilitamos la característica de Papelera de Reciclaje de Active Directory. Esto es fundamental para la recuperación de objetos borrados (usuarios, grupos, equipos) sin tener que recurrir a procesos de restauración autoritativa más complejos y riesgosos.

## 📂 Domingo: Filtrado de Archivos (FSRM) y Auditoría
Para asegurar que los servidores de archivos se utilicen únicamente para fines productivos y autorizados, implementamos el Administrador de recursos del servidor de archivos (FSRM) y configuramos el acceso para los usuarios.

*   **Configuración del Entorno:**
    *   Creamos una carpeta compartida en el servidor en la ruta `\\SRV-AD-01\RecursosCompartidos\RRHH`.
    *   Realizamos el mapeo de red de esta carpeta en el equipo del usuario (cliente) como la **unidad de red Z:**.
*   **Implementación de Seguridad:**
    *   Se configuraron plantillas de filtrado para bloquear un grupo completo de extensiones ejecutables y de scripts críticos, incluyendo específicamente **`.exe` (ejecutables), `.ps1` (scripts de PowerShell) y `.vbs` (Visual Basic Scripts)**, entre otros.
    *   Se configuró la **Auditoría de filtros** para registrar cualquier intento de violación.
*   **Validación de Auditoría:** 
    *   Realizamos pruebas de copia de diversos archivos bloqueados desde la unidad Z: utilizando la cuenta de usuario `NAVEATECH\ana.rrh`.
    *   El sistema denegó el acceso exitosamente, y el evento fue capturado en el Visor de Eventos (**Registros de Aplicaciones**) bajo el **Event ID 8215 (Origen: SRMSVC)**.
    *   Esta validación confirma que, aunque mi cuenta de soporte TI posee privilegios para administrar la política, el usuario final está correctamente restringido por las reglas implementadas.

## 🎯 Conclusión
Al finalizar esta semana, hemos alcanzado hitos importantes en nuestra arquitectura defensiva:

*   **Integridad:** Protección física y lógica de las OUs y objetos de AD.
*   **Control:** Implementación de políticas de filtrado de archivos mediante FSRM para prevenir la ejecución de binarios y scripts no permitidos (.exe, .ps1, .vbs) en el servidor.
*   **Detección:** Configuración de auditoría detallada, logrando centralizar y registrar intentos de acceso no autorizado con un nivel de detalle granular.

¡La infraestructura de `naveatech.local` es cada vez más resiliente y segura!

---
*Documentado por: Kelvin Navea (Onsite IT Technician | Support Analyst)*
