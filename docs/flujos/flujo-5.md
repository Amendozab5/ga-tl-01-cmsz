# Flujo 5 — Servicio de Cajeros → Core

- **Tipo:** RPC (gRPC)
- **Sync/Async:** síncrona, timeout 10 s
- **Protocolo/middleware:** gRPC/HTTP2 con idempotencia y deadline 10 s
- **Mensaje:** `DebitRequest{op_id, tarjeta, monto, cajero_id}`
- **Requisitos:** R2, R4
- **Dueño:** M

## Por qué

El servicio de cajeros actúa como adaptador: recibe el formato fijo por socket y llama
al core por gRPC. El débito debe confirmarse antes de que el cajero entregue dinero;
la idempotencia protege reintentos.

## Secuencia

1. El servicio de cajeros recibe el `RetiroRequest` por TCP.
2. Traduce el formato fijo a `DebitRequest` y llama al core por gRPC.
3. El core valida `op_id`, debita y responde con estado y saldo.
4. El servicio responde al cajero antes de los 15 s.
5. Si el core no responde en 10 s, el servicio responde `NO_DISPENSAR`.
6. El cajero no entrega dinero; se marca `EN_PROCESO` y se concilia.

## Si falla

Si el core no responde en 10 s, el cajero no dispensa; se revierte o concilia.

## Contrato gRPC (modelo)

```proto
rpc Debit(DebitRequest) returns (DebitResponse);

message DebitRequest {
  string op_id;
  string tarjeta_token;
  int64 monto;
  string cajero_id;
}

message DebitResponse {
  Status status;
  int64 saldo;
  string tx_id;
}

enum Status {
  OK;
  ALREADY_PROCESSED;
  INSUFFICIENT_FUNDS;
  TIMEOUT;
}
```

## Qué dibujar

- Flecha sólida azul desde Servicio de Cajeros hasta Core.
- Etiqueta: `[5] DebitRequest · gRPC · 10 s · idempotente`.
- Marca el adaptador: TCP fijo entra, gRPC sale.
- Relaciona esta flecha con F1 en una nota roja.
