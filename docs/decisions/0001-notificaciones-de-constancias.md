---
status: "proposed"
date: 2026-11-03
---

# Comunicación asíncrona mediante eventos persistidos para las notificaciones de constancias

## Context and Problem Statement

ConstanciaUV es un sistema que permite a estudiantes autorizados validar su identidad institucional (R1), generar y almacenar constancias digitales (R2), mantener un historial de constancias generadas (R3) y notificar por correo electrónico al estudiante cuando su constancia esté disponible (R4).

El equipo necesita decidir cómo organizar la comunicación entre el módulo de generación de constancias y el servicio de notificaciones por correo electrónico, considerando que dicho proveedor es externo al sistema (RT1). La solución debe garantizar que la generación y almacenamiento de constancias continúe operando sin detenerse si el servicio de correo falla o no está disponible (RT2), conservar las notificaciones pendientes para reintentar su envío posteriormente (RT3) y prevenir el envío de notificaciones duplicadas para una misma constancia sin presuponer que el proveedor externo garantice entrega única (RT4).

## Decision Drivers

* **Disponibilidad y tolerancia a fallos en la generación de constancias (originado por R2, RT1 y RT2):** El flujo principal de creación y almacenamiento de constancias no debe bloquearse ni fallar cuando el proveedor externo de correo electrónico presente caídas o problemas de red.
* **Idempotencia y confiabilidad en la entrega de notificaciones (originado por R4, RT3 y RT4):** El sistema debe asegurar el registro persistente de las notificaciones pendientes para su reintento y evitar que se emitan solicitudes de envío duplicadas para una misma constancia.

## Considered Options

* Alternativa A. Comunicación directa con recuperación de fallos
* Alternativa B. Procesamiento asíncrono mediante eventos persistidos

## Decision Outcome

Opción elegida: **"Alternativa B. Procesamiento asíncrono mediante eventos persistidos"**, porque desacopla por completo la generación y el almacenamiento de la constancia respecto a la disponibilidad del proveedor externo de correo, cumpliendo de forma natural con RT2 al no frenar ni bloquear la creación de constancias si el servicio de correo cae. Además, al registrar toda notificación como pendiente por defecto en el mismo flujo de persistencia (RT3), unifica la lógica de procesamiento en un único componente independiente, lo que centraliza el control de estado e idempotencia para prevenir solicitudes de notificación duplicadas (RT4) sin duplicar caminos de envío directo y de recuperación.

### Consequences

* **Bueno, porque (Beneficio):** Garantiza la continuidad de la generación y almacenamiento de constancias (R2, RT2) independientemente del estado del proveedor externo (RT1), y unifica en un solo flujo el manejo de envíos pendientes (RT3) y la prevención de duplicados (RT4).
* **Malo, porque (Costo o riesgo):** Introduce un componente adicional (procesador asíncrono de notificaciones) que debe implementarse, ejecutarse y monitorearse, además de que la notificación no se envía en el mismo instante transaccional de la creación de la constancia.
* **Neutral / Trade-off:** Se acepta una ligera demora eventual en el envío del correo electrónico y la complejidad operativa de un procesador independiente a cambio de desacoplar totalmente el flujo crítico de generación de constancias y centralizar la tolerancia a fallos e idempotencia en un solo mecanismo.

### Confirmation

En una implementación futura, el cumplimiento de esta decisión arquitectónica se comprobaría mediante las siguientes revisiones y pruebas:

* **Prueba de tolerancia a fallos en el servicio de correo (RT2 y RT3):** Simular la caída o el bloqueo de red hacia el proveedor externo de correo electrónico al momento de generar una constancia, verificando que la constancia digital se cree y almacene exitosamente (R2, R3) sin errores para el usuario y que el registro del evento de notificación quede persistido con estado `PENDIENTE`.
* **Revisión de diseño y prueba contra duplicados (RT4):** Revisar que el modelo de datos de notificaciones pendientes incluya una restricción de unicidad (`UNIQUE`) sobre el identificador de la constancia (`id_constancia`) junto con un control de estados (`PENDIENTE`, `EN_PROCESO`, `ENVIADA`), y diseñar pruebas de integración para verificar que el procesador ignore notificaciones ya registradas o enviadas ante reintentos o ejecuciones concurrentes.

## Pros and Cons of the Options

### Alternativa A. Comunicación directa con recuperación de fallos

Después de almacenar una constancia, la aplicación intenta solicitar directamente el envío de la notificación al proveedor externo; si este no responde, registra el envío pendiente para reintentarlo posteriormente.

* **Bueno, porque:** Permite el envío inmediato de la notificación cuando la red y el proveedor externo de correo se encuentran operativos.
* **Malo, porque:** Acopla temporalmente la solicitud de constancia a la respuesta o tiempos de espera (*timeouts*) del proveedor externo durante caídas del servicio de correo (afectando RT2).
* **Malo, porque:** Duplica la lógica de integración, control de errores y prevención de duplicados al mantener dos flujos distintos: el intento de envío directo inicial y el mecanismo posterior de reintento para pendientes (complejizando RT3 y RT4).

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

Después de almacenar una constancia, la aplicación registra únicamente una notificación pendiente; un componente independiente procesa posteriormente esos registros y se comunica con el proveedor externo.

* **Bueno, porque:** Desacopla totalmente la generación de la constancia respecto al servicio externo de correo en un solo flujo uniforme, evitando bloqueos por caídas del proveedor (RT1, RT2).
* **Bueno, porque:** Persiste todas las notificaciones como pendientes por defecto (RT3) y centraliza en un único procesador el control de reintentos y la prevención de duplicados (RT4).
* **Malo, porque:** Introduce una ligera demora entre el almacenamiento de la constancia y el envío efectivo del correo, y requiere un componente procesador adicional.

## More Information

Propuesta inicial elaborada para el análisis arquitectónico del sistema ConstanciaUV empleando la plantilla MADR 4.0.0. Queda sujeta a revisión ante posibles cambios en las condiciones de integración con el proveedor externo de notificaciones.
