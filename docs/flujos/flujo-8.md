# Flujo 8 — Core / Transferencias → Auditoría

- **Tipo:** Mensajería
- **Sync/Async:** asíncrona
- **Protocolo/middleware:** Kafka con transactional outbox; clave de partición `cuenta`
- **Mensaje:** `AuditEvent{op_id, tipo, actor, timestamp, payload_hash}`
- **Requisitos:** R5, R6
- **Dueño:** Z

## Por qué

R5 exige que toda operación quede auditada aunque el servicio de auditoría esté caído. Por eso
el evento no se manda directo al servicio: se guarda en una tabla outbox dentro de la misma
transacción que registra la operación. Si la operación se confirma, su evento también; si se
revierte, el evento tampoco queda.

**Trade-off:** ganamos R5 sin agregar una dependencia síncrona al camino del dinero, porque la
escritura va a la misma base de datos que ya participa en la operación. A cambio, sumamos un
componente (el relay), la entrega es *al menos una vez* (por eso el consumidor es idempotente)
y hay un retraso entre la operación y su registro en auditoría.

## Fuentes de auditoría

| Fuente | Qué audita | Cómo llega a Kafka |
|---|---|---|
| Outbox de la BD transaccional (Núcleo) | Transferencias, retiros, reversas | Misma transacción + relay |
| Outbox de Transferencias | Decisiones de antifraude que no llegan al core (`PENDIENTE_REVISION`, F2) | Misma transacción + relay |
| Servicio de Cuentas | Consultas de saldo | Directo, sin outbox (una consulta no escribe nada) |

## Secuencia

1. La operación y el `AuditEvent` se escriben en la misma transacción (outbox).
2. El relay lee el outbox y publica el evento en Kafka, con la cuenta como clave de partición.
3. El consumidor de auditoría guarda el evento de forma idempotente y descarta repetidos por
   `(op_id, tipo)`.
4. Si auditoría está caída, Kafka retiene y el consumidor sigue desde su último offset al volver.
5. Si Kafka no está disponible, el outbox conserva el evento y el relay reintenta.

## Si falla

| Qué cae | Qué pasa |
|---|---|
| Auditoría | Kafka retiene; el consumidor continúa desde su offset al volver. |
| Kafka | El evento espera en el outbox; el relay reintenta. |
| Relay | El outbox conserva el evento hasta que el relay vuelve. |

Ningún fallo bloquea la operación ni pierde el evento.

- **Repetidos por `(op_id, tipo)`, no solo por `op_id`.** En F1, la misma operación genera
  `DEBITO` y luego `REVERSA`. Si se deduplicara solo por `op_id`, se descartaría la reversa.
- **Partición por cuenta.** Kafka solo ordena dentro de una partición. Con la cuenta como clave,
  los eventos de un socio quedan en orden y las cuentas distintas se procesan en paralelo (R1).

## Supuestos

- La tabla outbox está en la BD transaccional del Núcleo. El adaptador del core registra la
  operación y el evento en la misma transacción, sin modificar el core heredado.
- R5 cubre todas las operaciones, incluidas las consultas de saldo.

## Qué dibujar

- Outbox como cilindro dentro de la BD transaccional; otro outbox junto a Transferencias.
- Línea gris fina continua: `[8] operación + AuditEvent · misma transacción` (Core → Outbox) y
  `[8] decisión antifraude · misma transacción` (Transferencias → su outbox).
- Línea gris fina continua: `[8] lee pendientes` desde los dos outbox hasta Relay outbox.
- Flecha punteada púrpura: `[8] AuditEvent · Kafka/outbox · async` (Relay outbox → Kafka).
- Flecha punteada púrpura: `[8] AuditEvent (CONSULTA) · Kafka · async` (Cuentas → Kafka).
- Flecha punteada púrpura: `[8] consume · Kafka · async · idempotente (op_id+tipo)` (Kafka → Auditoría).
- Nota: R5, auditoría aunque el servicio esté caído.
