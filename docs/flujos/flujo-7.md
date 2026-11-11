# Flujo 7 — Transferencias / Core → Notificaciones

- **Tipo:** Mensajería
- **Sync/Async:** asíncrona
- **Protocolo/middleware:** AMQP 0-9-1 con RabbitMQ; reintentos y DLQ
- **Mensaje:** `MovimientoRegistrado{op_id, cuenta, monto, canal, tipo}`
- **Requisitos:** R1, R6
- **Dueño:** Z

## Por qué

El socio no necesita esperar a que se envíe el SMS, el correo o la push. RabbitMQ desacopla
el pico de 300 TPS de los días de pago (R1), permite reintentos y manda a la DLQ los mensajes
que no se pueden entregar. Si notificaciones está lenta o caída, la operación no se cae (R6).

El mensaje lo publica el relay del outbox, no Transferencias. Así, si un proceso cae justo
después de que el core confirma, el aviso no se pierde. Además, todo movimiento que pasa por el
core (transferencias, retiros en cajero y reversas) genera su notificación, como pide la
operación (a).

**Trade-off:** ganamos desacoplamiento y avisos que no se pierden. A cambio, operamos un broker
más, la notificación llega con algo de retraso y RabbitMQ no garantiza el orden entre avisos.
Como cada aviso trae su monto y su tipo, el desorden no confunde al socio.

## Secuencia

1. La operación y su evento se guardan en el outbox, en la misma transacción.
2. El relay publica `MovimientoRegistrado` en el exchange de RabbitMQ.
3. RabbitMQ enruta el mensaje a la cola de notificaciones.
4. El consumidor de notificaciones envía el SMS, el correo o la push.
5. Si el envío falla, se reintenta; agotados los reintentos, el mensaje pasa a la DLQ.
6. La operación **no espera** la notificación.

## Si falla

| Qué cae | Qué pasa |
|---|---|
| Notificaciones | El mensaje queda en la cola (durable, con mensajes persistentes) y se procesa al volver. |
| RabbitMQ | El evento espera en el outbox y el relay reintenta. |
| La cola se llena | Se agregan consumidores; la operación sigue porque está fuera del camino del dinero (R6). |

Los límites concretos (tamaño de cola, número de reintentos) se definen en la práctica con la
prueba de carga.

## Qué dibujar

- Flecha punteada púrpura desde Relay outbox hasta RabbitMQ.
- Etiqueta: `[7] MovimientoRegistrado · AMQP · async`.
- RabbitMQ y la DLQ como símbolos de cola o sobre, en el carril Eventos.
- `[7] consume · AMQP · async · reintentos` desde RabbitMQ hasta Notificaciones.
- `[7] agotados los reintentos → DLQ` desde RabbitMQ hasta DLQ.
- Nota: la operación no espera la notificación (R1/R6).
