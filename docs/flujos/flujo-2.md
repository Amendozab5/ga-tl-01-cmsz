# Flujo 2 — Servicio de Transferencias → Servicio Antifraude

**Actividad:** GA-TL-01 · Escenario 4 · Cooperativa Ahorro Futuro  
**Responsable:** C — Coordinación y transferencias

- **Origen:** servicio de transferencias.
- **Destino:** servicio antifraude.
- **Tipo:** RPC.
- **Sync/Async:** síncrona, con deadline de **200 ms**.
- **Protocolo/middleware:** gRPC sobre HTTP/2, deadline de 200 ms y circuit breaker.
- **Mensaje:** `FraudCheckRequest{op_id, cuenta, monto, beneficiario, canal}`.
- **Respuesta:** `APROBAR`, `RECHAZAR` o `REVISAR`.
- **Requisitos:** R3 y R6.

## Por qué se utiliza RPC

La revisión de riesgo debe realizarse antes de solicitar un débito en el core. El requisito R3 establece una respuesta del servicio antifraude en menos de 200 ms. Una llamada gRPC síncrona permite asociar la autorización al procesamiento de la transferencia y definir un deadline explícito.

La dependencia síncrona ofrece una decisión inmediata, pero puede afectar la disponibilidad si antifraude deja de responder. Por ello se propone un **circuit breaker** que limite las llamadas repetidas a un servicio inestable y active una política de contingencia controlada. Esta decisión contribuye a R6, cuya meta de disponibilidad es 99,95 % para transferencias y retiros; no constituye por sí sola una garantía de ese porcentaje.

## Secuencia

1. Transferencias recibe una solicitud del canal del socio mediante el **flujo 1**.
2. El servicio envía `FraudCheckRequest` con `op_id` y los datos necesarios para evaluar el movimiento.
3. Antifraude aplica sus reglas y emite `APROBAR`, `RECHAZAR` o `REVISAR` dentro del deadline de 200 ms.
4. Si responde `APROBAR`, transferencias puede continuar al core mediante el **flujo 3**.
5. Si responde `RECHAZAR`, se cancela el intento antes del débito. Si responde `REVISAR`, la operación se mantiene pendiente de revisión.
6. Si antifraude falla, tarda más de 200 ms o el circuit breaker está abierto, se aplica la política F2 y se registra la decisión.

## Si falla — F2: antifraude caído o lento

La política definida para el escenario evita dos extremos: detener indiscriminadamente todas las transferencias y aprobar indiscriminadamente operaciones sin evaluación.

| Situación | Decisión de contingencia | Seguimiento |
|---|---|---|
| Monto bajo y sin alertas previas | Permitir la continuación bajo contingencia | Marcar para revisión posterior y dejar registro auditable. |
| Monto alto o riesgo elevado | Mantener en `PENDIENTE_REVISION` | No ejecutar el débito mientras no exista decisión autorizada. |
| Respuesta explícita `RECHAZAR` | Rechazar | Conservar el resultado para auditoría. |
| Respuesta `REVISAR` | Mantener pendiente | Enviar a revisión del proceso correspondiente. |

El umbral que separa «monto bajo» y «monto alto» y las reglas para identificar alertas **no están definidos en la actividad**. Deben establecerse mediante política institucional; no se inventan importes ni porcentajes para este taller.

El circuit breaker permite detener temporalmente nuevas solicitudes a una dependencia que está fallando. Las operaciones que siguen un camino de contingencia se identifican expresamente y se registran para auditoría posterior. El servicio de auditoría no debe convertirse en un bloqueo síncrono de la transferencia, de acuerdo con R5 y el flujo 8.

## Qué dibujar en draw.io

- **Origen:** `Servicio de Transferencias`.
- **Destino:** `Servicio Antifraude`.
- **Flecha:** línea continua azul entre servicios.
- **Etiqueta:** `[2] FraudCheck · gRPC/HTTP2 · sync · 200 ms · R3/R6`.
- **Nota junto al flujo:** `F2: circuit breaker; bajo monto sin alertas → revisión posterior; alto riesgo → PENDIENTE_REVISION`.
- **Advertencia:** el servicio antifraude no debe representarse como cola, porque su decisión forma parte de una llamada síncrona previa al débito.

## Justificación resumida para la defensa

El flujo 2 utiliza gRPC síncrono porque la transferencia debe comprobar el riesgo antes del débito. El deadline de 200 ms responde a R3. Si antifraude se cae, el circuit breaker y la contingencia controlada permiten mantener algunas operaciones de bajo riesgo sin aceptar todas indiscriminadamente, atendiendo también R6.
