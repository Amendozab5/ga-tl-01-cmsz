# Flujo 3 — Transferencias → Core

* **Tipo:** RPC (gRPC)
* **Sync/Async:** síncrona
* **Protocolo/middleware:** gRPC/HTTP2 hacia el adaptador gRPC del core, con idempotency key y pool de conexiones
* **Mensaje:** `PostTransaction{op\_id, débito, crédito, monto}`
* **Requisitos:** R2, R1
* **Dueño:** S

## Por qué

El saldo oficial vive en el core: es el sistema de registro (R2). Transferencias necesita una
confirmación síncrona antes de responder al socio por el flujo 1, y el débito y el crédito deben
quedar en una sola transacción: consistencia fuerte. gRPC/HTTP2 aporta contrato tipado y
deadline; el pool de conexiones reutiliza las conexiones con el adaptador para sostener el pico
de 300 TPS de los días de pago (R1).

La idempotencia evita el doble débito (R2): el core aplica una restricción única sobre `op\_id`.
Si llega otra vez el mismo `op\_id`, devuelve el resultado original y no debita de nuevo.

**Trade-off:** ganamos consistencia fuerte y débito único. A cambio, la transferencia depende de
que el core responda (está en el camino del dinero) y un timeout deja el estado incierto, por lo
que hay que consultar por `op\_id` y conciliar.

## Secuencia

1. Con la aprobación de antifraude (flujo 2), Transferencias envía `PostTransaction` con `op\_id`,
las cuentas de débito y crédito y el monto.
2. El core valida la restricción única de `op\_id`.
3. Si `op\_id` es nuevo, debita y acredita en una sola transacción. En esa misma transacción quedan
en el outbox los eventos de la operación, que publican los flujos 7 y 8.
4. Si `op\_id` ya existe, devuelve el resultado original sin debitar otra vez.
5. El core responde OK o error; Transferencias actualiza el estado de la operación y responde al
canal (flujo 1).

## Si falla

|Situación|Qué pasa|
|-|-|
|Timeout o respuesta perdida|La operación queda `PENDIENTE`; Transferencias consulta al core por `op\_id` antes de decidir.|
|Reintento|Se reenvía `PostTransaction` con el mismo `op\_id`; el core lo reconoce y devuelve el resultado original (R2).|
|Estado todavía incierto|La conciliación (cada minuto, supuesto del equipo) compara el estado con el core usando el mismo `op\_id`.|

**F3 (doble clic en «Enviar»):** las dos solicitudes llevan la misma intención y llegan al core con
el mismo `op\_id`; la restricción única hace que la segunda devuelva el mismo resultado, no un
segundo débito.

## Supuestos

* El core heredado expone un adaptador gRPC; el equipo no lo implementa, solo lo modela.
* La restricción única de `op\_id` y la tabla outbox están en la BD transaccional del Núcleo; el
adaptador escribe la operación y sus eventos en la misma transacción (supuesto compartido con
los flujos 7 y 8).

## Qué dibujar

* Flecha sólida azul desde Servicio de Transferencias hasta Núcleo Bancario (Core).
* Etiqueta: `\[3] PostTransaction · gRPC · sync · idempotente`.
* Core como sistema de registro y BD transaccional como cilindro.
* Marca R2 junto a la flecha.

