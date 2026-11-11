# Matriz de interacciones — Cooperativa Ahorro Futuro

**Actividad:** GA-TL-01 — Modelado de comunicación entre procesos distribuidos  
**Escenario:** 4 — Sistema bancario: Cooperativa Ahorro Futuro  
**Grupo:** CMSZ  
**Responsable de la matriz:** C — Coordinación y transferencias

La matriz identifica las ocho interacciones principales del modelo. Cada flujo distingue el mecanismo de comunicación, el carácter síncrono o asíncrono, el protocolo o middleware y la justificación según los requisitos R1–R6.

| # | Origen | Destino | Tipo | Sync/Async | Protocolo / middleware | Justificación y requisito | Responsable |
|---|---|---|---|---|---|---|---|
| **1** | App/Web | Servicio de Transferencias | RPC | Síncrona | gRPC-Web sobre HTTP/2 | El socio requiere respuesta inmediata; contrato tipado y multiplexación para la meta de 300 TPS y p95 menor de 2 s (**R1**). La solicitud conserva una clave de idempotencia que apoya **R2**. | **C** |
| **2** | Servicio de Transferencias | Servicio Antifraude | RPC | Síncrona; deadline de **200 ms** | gRPC sobre HTTP/2, deadline y circuit breaker | El análisis debe anteceder al débito (**R3**). Ante una caída se aplica contingencia diferenciada, evitando detener indiscriminadamente todas las operaciones (**R6**). | **C** |
| 3 | Servicio de Transferencias | Core bancario | RPC | Síncrona | gRPC sobre HTTP/2, adaptador del core, idempotency key y pool de conexiones | Solo el core confirma saldos oficiales; la misma operación no genera doble débito (**R2**). | S |
| 4 | Cajero automático | Servicio de Cajeros | Sockets | Síncrona; timeout máximo de **15 s** | TCP, mensajes de formato fijo, framing length-prefix; mTLS opcional | El dispositivo heredado utiliza formato fijo y no entrega efectivo sin confirmación (**R4**). | M |
| 5 | Servicio de Cajeros | Core bancario | RPC | Síncrona; deadline de **10 s** | gRPC sobre HTTP/2 con idempotencia | El adaptador consulta al sistema de registro y confirma el débito antes de autorizar la entrega de efectivo (**R2**, **R4**). | M |
| 6 | Servicio de Transferencias | Red interbancaria | Sockets | Síncrona; timeout controlado | TCP y mensajes de formato fijo (supuesto: ISO 8583) | Se integra una red externa heredada, con estados de proceso, conciliación y compensación ante incertidumbre (**R6**). | S |
| 7 | Transferencias / Core | Servicio de Notificaciones | Mensajería | Asíncrona | AMQP 0-9-1 con RabbitMQ; reintentos y DLQ | El socio no espera el envío de SMS/correo; la cola absorbe picos y permite reintentos (**R1**, **R6**). | Z |
| 8 | Core / Transferencias | Servicio de Auditoría | Mensajería | Asíncrona | Kafka y outbox transaccional | El evento persiste aunque auditoría no esté disponible y se entrega posteriormente (**R5**). | Z |

## Detalle de los flujos bajo mi responsabilidad

### Flujo 1 — App/Web → Transferencias

- **Mensaje:** `TransferRequest{idempotency_key, cuenta_origen, cuenta_destino, monto, moneda}`.
- **Dirección:** el cliente inicia la solicitud y espera la respuesta del servicio de transferencias.
- **Requisitos:** R1 y R2.
- **Error o timeout:** el resultado se consulta o reintenta con la misma `idempotency_key`. Si el estado no se conoce, la operación se muestra como `PENDIENTE`; no se debe generar una nueva intención de débito.
- **Etiqueta para el diagrama:** `[1] TransferRequest · gRPC-Web/HTTP2 · sync · R1/R2`.

### Flujo 2 — Transferencias → Antifraude

- **Mensaje:** `FraudCheckRequest{op_id, cuenta, monto, beneficiario, canal}`; respuesta: `APROBAR`, `RECHAZAR` o `REVISAR`.
- **Dirección:** transferencias solicita la validación antes de ejecutar la transferencia.
- **Requisitos:** R3 y R6.
- **Error o deadline de 200 ms:** circuit breaker y política F2: operaciones de bajo monto sin alertas pueden continuar identificadas para revisión posterior; las operaciones de alto riesgo o monto alto quedan `PENDIENTE_REVISION`.
- **Etiqueta para el diagrama:** `[2] FraudCheck · gRPC/HTTP2 · sync · deadline 200 ms · R3/R6`.

## Manejo de fallos y decisiones de diseño

**F1 — Confirmación de débito perdida antes de llegar al cajero.** El cajero no dispensa sin confirmación. La operación se identifica con el mismo `op_id` para determinar el estado real, revertir si corresponde y conciliar cuando exista incertidumbre. El registro del dispensador evita revertir una entrega efectivamente realizada.

**F2 — Antifraude no disponible o lento.** Se utiliza un circuit breaker y una política de contingencia. El sistema no autoriza automáticamente todas las transferencias ni las bloquea indiscriminadamente. Las operaciones de bajo monto sin alertas se marcan para revisión y las de mayor riesgo permanecen pendientes. Toda decisión se deja registrada para auditoría.

**F3 — Doble clic en «Enviar».** La interfaz puede deshabilitar el botón para reducir repeticiones, pero la garantía es la `idempotency_key` por intención, la restricción de unicidad y el estado de la operación en el core. Repetir la misma solicitud retorna su resultado sin aplicar un segundo débito.

## Requisitos y supuestos

| Requisito | Respuesta del diseño |
|---|---|
| R1 | Objetivo de 300 TPS y p95 menor de 2 s; gRPC/HTTP2 y manejo asíncrono de eventos. |
| R2 | Core como registro oficial, `idempotency_key` y restricción única. |
| R3 | Deadline de 200 ms, circuit breaker y contingencia F2. |
| R4 | Sockets TCP para el cajero, formato fijo y timeout máximo de 15 s. |
| R5 | Outbox transaccional más Kafka y reintentos. |
| R6 | Aislamiento de fallas, réplicas, comprobaciones de salud y conciliación. |

Se asume un **adaptador gRPC frente al core heredado**, una **red interbancaria de formato fijo** y un **cajero heredado que habla TCP**. El modelado no representa una implementación final ni una medición de rendimiento. El uso de gRPC-Web para navegadores puede requerir un proxy o gateway compatible, que se considera parte de la frontera de entrada sin cambiar el flujo lógico número 1.

**Alcance:** esta matriz consolida el acuerdo grupal de ocho flujos; los detalles técnicos de los flujos 3–8 continúan siendo responsabilidad de sus respectivos integrantes.
