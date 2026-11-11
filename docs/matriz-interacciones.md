# Matriz de interacciones

> **Dueño:** C. Borrador inicial armado por M para arrancar; C debe revisar, ajustar
> y commitear la versión final. Cada fila debe tener tipo, sync/async, protocolo/middleware
> y una justificación que cite R1–R6.

| # | Origen | Destino | Tipo | Sync/Async | Protocolo / middleware | Justificación (R1–R6) | Dueño |
|---|---|---|---|---|---|---|---|
| 1 | App/Web | Transferencias | RPC | Síncrona | gRPC-Web / HTTP2 | Respuesta inmediata y contrato tipado; p95 < 2 s (R1). | C |
| 2 | Transferencias | Antifraude | RPC | Síncrona · 200 ms | gRPC / HTTP2 + circuit breaker | R3 exige decidir antes de ejecutar; deadline de 200 ms. | C |
| 3 | Transferencias | Core | RPC | Síncrona | gRPC / HTTP2 + idempotencia | R2: saldo oficial y débito único; consistencia fuerte. | S |
| 4 | Cajero | Servicio de Cajeros | Sockets | Síncrona · 15 s | TCP + formato fijo | R4 exige formato fijo y timeout; no dispensar sin respuesta. | M |
| 5 | Servicio de Cajeros | Core | RPC | Síncrona · 10 s | gRPC / HTTP2 + idempotencia | El adaptador confirma el débito antes de dispensar (R2/R4). | M |
| 6 | Transferencias | Red interbancaria | Sockets | Síncrona · timeout | TCP + ISO 8583 (supuesto) | Red externa legacy; timeout, compensación y conciliación (R6). | S |
| 7 | Transferencias / Core | Notificaciones | Mensajería | Asíncrona | AMQP / RabbitMQ | El socio no espera; absorbe picos y reintenta (R1/R6). | Z |
| 8 | Core / Transferencias | Auditoría | Mensajería | Asíncrona | Kafka + outbox | R5: auditoría aunque el servicio esté caído. | Z |
