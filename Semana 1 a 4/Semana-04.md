# Laboratorio de Active Directory (Semana 4): Protección de Infraestructura y FSRM 🛡️

## 📌 Descripción General del Proyecto
Este repositorio documenta las medidas de *hardening* y seguridad aplicadas al dominio `naveatech.local`. Durante la semana 4, nos enfocamos en proteger la integridad lógica del Active Directory y en implementar un sistema de control y auditoría de archivos mediante FSRM.

## 🛡️ Sábado: Blindaje y Recuperación en Active Directory
Implementamos configuraciones críticas para proteger la estructura del dominio frente a errores humanos y posibles ataques internos.

*   **Protección de Objetos (Accidental Deletion):** Configuramos las Unidades Organizativas (OU) con la propiedad "Proteger objeto contra eliminación accidental" habilitada. Esto añade una barrera de seguridad necesaria para evitar que una estructura completa del dominio sea borrada por error o por un usuario con privilegios que no debería realizar esa acción.
*   **Activación de la Papelera de Reciclaje (AD Recycle Bin):** Habilitamos la característica de Papelera de Reciclaje de Active Directory. Esto es fundamental para la recuperación de objetos borrados (usuarios, grupos, equipos) sin tener que recurrir a procesos de restauración autoritativa más complejos y riesgosos.

## 📂 Domingo: Filtrado de Archivos (FSRM) y Auditoría
Para asegurar que los servidores de archivos se utilicen únicamente para fines productivos y autorizados, implementamos el Administrador de recursos del servidor de archivos (FSRM).

*   **Objetivo:** Restringir el almacenamiento de archivos potencialmente peligrosos o no autorizados en carpetas sensibles de RRHH.
*   **Implementación:**
    *   Se configuraron plantillas de filtrado para bloquear un grupo completo de extensiones ejecutables y de scripts, incluyendo **`.exe` (ejecutables), `.ps1` (PowerShell) y `.vbs` (Visual Basic Script)**, entre otros.
    *   Se configuró la **Auditoría de filtros** para registrar cualquier intento de violación.
*   **Validación de Auditoría:** 
    *   Al realizar pruebas de copia de diversos tipos de archivos bloqueados, el sistema denegó el acceso exitosamente en todos los casos.
    *   Se capturó el **Event ID 8215 (Origen: SRMSVC)** en el Visor de Eventos (**Registros de Aplicaciones**).
    *   El log detalla claramente el usuario responsable (`NAVEATECH\ana.rrh`), la ruta intentada y el archivo bloqueado (por ejemplo, `PruebaSeguridad.ps1`), proporcionando una trazabilidad completa del intento de acceso.

## 🎯 Conclusión
Al finalizar esta semana, hemos alcanzado hitos importantes en nuestra arquitectura defensiva:

*   **Integridad:** Protección física y lógica de las OUs y objetos de AD.
*   **Control:** Implementación de políticas de filtrado de archivos mediante FSRM para prevenir la ejecución de binarios y scripts no permitidos en el servidor.
*   **Detección:** Configuración de auditoría detallada, logrando centralizar y registrar intentos de acceso no autorizado con un nivel de detalle granular.

¡La infraestructura de `naveatech.local` es cada vez más resiliente y segura!

---
