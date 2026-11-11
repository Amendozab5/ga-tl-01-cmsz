# Flujo 1 — App/Web → Servicio de Transferencias

**Actividad:** GA-TL-01 · Escenario 4 · Cooperativa Ahorro Futuro  
**Responsable:** C — Coordinación y transferencias

- **Origen:** aplicación móvil o cliente web del socio.
- **Destino:** servicio de transferencias.
- **Tipo:** RPC.
- **Sync/Async:** síncrona.
- **Protocolo/middleware:** gRPC-Web sobre HTTP/2, según la decisión de arquitectura.
- **Mensaje:** `TransferRequest{idempotency_key, cuenta_origen, cuenta_destino, monto, moneda}`.
- **Requisitos:** R1 y R2.

## Por qué se utiliza RPC

El socio necesita conocer el resultado de la transferencia al confirmar la operación. El contrato tipado de gRPC facilita definir el mensaje y su respuesta, mientras que HTTP/2 permite multiplexar solicitudes. Esta elección se relaciona con el requisito R1: capacidad prevista para 300 transacciones por segundo en días de pago y objetivo de que el 95 % complete en menos de dos segundos. Es una meta de diseño, no un rendimiento medido en este taller.

A diferencia de una cola asíncrona, la llamada RPC permite que la interfaz espere una respuesta de la operación principal. Como contrapartida, los errores de red y las demoras pueden dejar incierto el estado visto por el cliente. Por ello se mantiene la misma `idempotency_key` durante reintentos (R2). Para un navegador, el acceso gRPC-Web puede necesitar un gateway o proxy compatible; este elemento es un supuesto de despliegue, no una nueva interacción numerada.

## Secuencia

1. El socio confirma el envío desde la aplicación móvil o la banca web.
2. El cliente envía `TransferRequest` con una `idempotency_key` asociada a esa intención de transferencia.
3. El servicio de transferencias valida la sesión y los campos y realiza una comprobación preliminar del saldo.
4. Antes del movimiento, el servicio consulta antifraude mediante el **flujo 2**.
5. Si antifraude aprueba, solicita al core bancario la ejecución por el **flujo 3**.
6. El core confirma el débito y el crédito correspondientes, y conserva la garantía de idempotencia.
7. El servicio responde al cliente con el resultado y el comprobante, cuando exista confirmación.
8. Si llega un reintento con la misma `idempotency_key`, la operación se reconoce y no se efectúa un segundo débito.

## Si falla

Cuando la comunicación se interrumpe o vence el tiempo de espera, la interfaz no debe interpretar automáticamente que la transferencia falló o fue confirmada. Si no se dispone del estado final, se muestra `PENDIENTE` y cualquier consulta o reintento utiliza la misma `idempotency_key`.

En el caso **F3**, el socio pulsa dos veces «Enviar». Deshabilitar el botón reduce los duplicados visibles, pero el mecanismo decisivo es la unicidad de la operación en el core, junto con la clave de idempotencia. El segundo mensaje retorna el estado previamente registrado en lugar de causar otro movimiento.

## Qué dibujar en draw.io

- **Origen:** caja `Socio / App-Web` del carril **Canales**.
- **Destino:** caja `Servicio de Transferencias` del carril **Servicios**.
- **Flecha:** línea continua azul, desde el cliente al servicio.
- **Etiqueta:** `[1] TransferRequest · gRPC-Web/HTTP2 · sync · R1/R2`.
- **Nota:** `Reintentos con la misma idempotency_key; sin doble débito (F3)`.

## Justificación resumida para la defensa

El flujo 1 utiliza RPC síncrono porque el socio espera conocer el estado de su transferencia. gRPC-Web aporta un contrato definido y comunicación basada en HTTP/2, mientras que la clave de idempotencia impide que una solicitud repetida produzca dos débitos. Así se atienden R1 y R2.
