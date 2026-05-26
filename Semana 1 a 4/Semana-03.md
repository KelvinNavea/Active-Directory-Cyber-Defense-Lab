# Laboratorio de Active Directory (Semana 3): Hardening, Forense y Centralización

## 📌 Descripción General del Proyecto
Este repositorio documenta la implementación técnica de una arquitectura defensiva avanzada sobre el dominio `naveatech.local`. Durante la semana 3, nos enfocamos en dos pilares fundamentales: el control de ejecución en el endpoint y la centralización de telemetría para la auditoría de seguridad.

---

## 🛡️ Sábado: Control de Aplicaciones con AppLocker
Implementamos políticas de control de ejecución para mitigar riesgos de software no autorizado o potencialmente malicioso.

* **Objetivo:** Restringir la ejecución de archivos ejecutables (`.exe`) en rutas no controladas (específicamente en la carpeta Descargas del usuario `NAVEATECH\ana.rrhh`).
* **Validación:** Se realizó una prueba de ejecución de un binario no permitido. El sistema disparó correctamente el **Event ID 8004**, confirmando el bloqueo de ejecución.
* **Análisis:** Identificamos la ruta de origen, el usuario y el nombre del ejecutable bloqueado, demostrando la eficacia de AppLocker como herramienta de prevención en el endpoint.

---

## 📡 Domingo: Centralización de Telemetría (WEC/WEF)
Para mejorar la capacidad de respuesta, configuramos una arquitectura de **Windows Event Forwarding (WEF)**. Configuramos un modelo de suscripción iniciada por el origen donde los clientes reportan eventos específicos al Servidor Colector (WEC).

* **Arquitectura:** Servidor Colector (Windows Server 2022) y Cliente (Windows 11).
* **Comunicación:** WinRM sobre puerto 5985.
* **Validación:** Ejecutamos el comando `wecutil es` en el servidor para confirmar que la suscripción está activa.

### Eventos Críticos Centralizados
Hemos configurado la suscripción para capturar los siguientes IDs de eventos, fundamentales para el análisis de seguridad:

* **Event ID 4740 (Bloqueo de cuentas):** Crucial para detectar ataques de fuerza bruta. Al centralizarlo, podemos identificar patrones de múltiples bloqueos en diferentes equipos del dominio.
* **Event ID 4756 (Añadido a grupo de seguridad con privilegios):** Se registra cuando un usuario es agregado a grupos de dominio sensibles (ej: *Domain Admins*). Es vital para detectar intentos de escalada de privilegios o persistencia por parte de un atacante.
* **Event ID 8004 (Bloqueo de AppLocker):** Al centralizar este evento, obtenemos visibilidad inmediata de todos los intentos de ejecución bloqueados en toda la infraestructura.

---

## 🎯 Conclusión
Al finalizar esta semana, hemos logrado un flujo de trabajo defensivo completo:
1. **Prevención:** Bloqueo de binarios mediante políticas de AppLocker.
2. **Visibilidad:** Centralización de logs para un monitoreo proactivo.
3. **Detección:** Capacidad de correlacionar eventos críticos en un solo punto de auditoría.

¡La infraestructura de `naveatech.local` es ahora mucho más robusta y fácil de auditar!
