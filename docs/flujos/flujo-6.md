# Flujo 6 — Transferencias → Red interbancaria

* **Tipo:** Sockets
* **Sync/Async:** síncrona, con timeout
* **Protocolo/middleware:** TCP con mensajes de formato fijo (supuesto: ISO 8583)
* **Mensaje:** `0200/0210 · op\\\_id, monto, beneficiario, banco destino`
* **Requisitos:** R6, R2
* **Dueño:** S

## Por qué

La red interbancaria es externa y legacy: no se controla su latencia ni se puede cambiar su
protocolo. Por eso no se usa gRPC: se habla el formato fijo que la red expone (supuesto: ISO 8583
sobre TCP, con el mensaje 0200 como solicitud y 0210 como respuesta). El timeout corto evita que
Transferencias quede bloqueado por una red lenta: si no hay respuesta, la operación queda
`EN\\\_PROCESO` y se resuelve después por conciliación (R6). Si la red rechaza, el débito se
compensa con una reversa que usa el mismo `op\\\_id`, sin dejar al socio con un débito sin
transferencia (R2).

**Trade-off:** ganamos integración con un sistema que no controlamos sin bloquear el servicio. A
cambio, hay que manejar framing y formato fijo, el socio puede ver un estado intermedio
(`EN\\\_PROCESO`) y se necesita un proceso de conciliación.

## Secuencia

1. El débito de la cuenta del socio ya quedó registrado en el core por el flujo 3; por eso un
rechazo se corrige con una reversa.
2. Transferencias envía el mensaje 0200 con `op\\\_id`, monto, beneficiario y banco destino.
3. La red responde 0210 con aprobación o rechazo.
4. Si aprueba, se confirma la operación y se notifica al socio.
5. Si rechaza, se reversa el débito con el mismo `op\\\_id`.
6. Si no responde dentro del timeout, la operación queda `EN\\\_PROCESO`, se reintenta con el mismo
`op\\\_id` y se concilia.

## Si falla

|Situación|Qué pasa|
|-|-|
|Rechazo de la red (0210 negativo)|Reversa del débito con el mismo `op\\\_id` (compensación).|
|Timeout|Estado `EN\\\_PROCESO`; reintento con el mismo `op\\\_id`; conciliación.|
|Estado todavía incierto|La conciliación (cada minuto, supuesto del equipo) usa el mismo `op\\\_id` hasta saber si la red aplicó o no la transferencia.|

## Supuestos

* La red interbancaria usa mensajes de formato fijo: ISO 8583 sobre TCP.
* La conciliación corre cada minuto y usa el mismo `op\\\_id`.
* La red reconoce un reintento con el mismo `op\\\_id` como la misma operación, de modo que no se
aplica dos veces.
* El valor exacto del timeout no está definido en el escenario; se fija en la práctica.

## Qué dibujar

* Flecha gruesa ámbar desde Servicio de Transferencias hasta Red interbancaria.
* Etiqueta: `\\\[6] 0200/0210 · TCP/ISO 8583 · timeout`.
* Red interbancaria como sistema externo, en gris, en el carril Externo.
* Supuesto ISO 8583 en la leyenda o en una nota.

