# Flujo 4 — Cajero automático → Servicio de Cajeros

- **Tipo:** Sockets TCP
- **Sync/Async:** síncrona, timeout 15 s
- **Protocolo/middleware:** TCP con framing length-prefix, formato fijo y mTLS opcional
- **Mensaje:** `[len][ver][tipo][tarjeta][monto][cajero_id][op_id][crc]`
- **Requisitos:** R4, R6
- **Dueño:** M

## Por qué

R4 exige formato fijo y timeout de 15 s. El cajero es legacy; sockets TCP permiten
controlar el framing, la conexión persistente y el timeout. No se dispensa sin respuesta.

## Secuencia

1. El cajero genera `op_id` y envía `RetiroRequest` por TCP.
2. El servicio de cajeros valida tarjeta, monto, cajero y `op_id`.
3. El servicio llama al core por gRPC con el mismo `op_id` (flujo 5).
4. El core debita y responde OK; el servicio marca `DEBITADA`.
5. El servicio responde al cajero; el cajero dispensa.
6. Si no hay respuesta en 15 s, el cajero **NO** dispensa.
7. El servicio marca `DEBITADO_NO_ENTREGADO` y reversa/concilia.
8. Si el cajero sí dispensó, su journal lo registra y no se reversa.

## Si falla

Timeout: no dispensar; estado `EN_PROCESO`; consultar/revertir con `op_id`; conciliación.

## Estados

| Estado | Significado |
|---|---|
| `INICIADA` | El cajero envió la solicitud; todavía no hay débito. |
| `DEBITADA` | El core confirmó el débito; el cajero puede dispensar. |
| `ENTREGADA` | El cajero dispensó y confirmó. |
| `DEBITADO_NO_ENTREGADO` | El core debitó, pero el cajero no recibió respuesta. |
| `REVERSADA` | Se compensó el débito con el mismo `op_id`. |
| `EN_PROCESO` | Timeout o respuesta perdida; conciliación en curso. |

## F1 — El core debita, pero la respuesta no llega al cajero

Si el core debitó y la respuesta se perdió, el servicio marca
`DEBITADO_NO_ENTREGADO` y reversa con el mismo `op_id`. Si la reversa falla,
el job de conciliación reintenta. Si el cajero sí entregó, su journal lo
registra y no se reversa. El socio no pierde dinero y no hay doble débito.

## Qué dibujar

- Flecha gruesa ámbar desde Cajero hasta Servicio de Cajeros.
- Etiqueta: `[4] RetiroRequest · TCP · formato fijo · 15 s`.
- Cajero como actor/dispositivo, no como servicio.
- Nota: sin respuesta en 15 s **NO** se dispensa.