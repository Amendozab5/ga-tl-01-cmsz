# Flujo N — Origen → Destino

- **Tipo:** Sockets / RPC (gRPC) / Mensajería
- **Sync/Async:** síncrona / asíncrona (con timeout si aplica)
- **Protocolo/middleware:** TCP, gRPC/HTTP2, AMQP, Kafka, etc.
- **Mensaje:** nombre y campos principales
- **Requisitos:** R1, R2, ...
- **Dueño:** C / M / S / Z

## Por qué

Explicar la elección citando el requisito y el trade-off: latencia, volumen,
respuesta inmediata, tolerancia a fallos o consistencia.

## Secuencia

1. ...
2. ...
3. ...

## Si falla

Timeout, reintento, idempotencia, compensación o conciliación.

## Qué dibujar

- Tipo de flecha y color.
- Etiqueta exacta.
- Nota o decisión que deba verse en el diagrama.
