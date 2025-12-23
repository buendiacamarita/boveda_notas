### Qué querés lograr  
Explicar el flujo de datos que descuenta stock cuando se genera un comprobante de venta, y darte una guía para poder ejecutar ese mismo flujo, pero por fuera de la creación del comprobante.  
  
---  
  
### Visión general del flujo actual (venta)  
Puntos clave del recorrido de una solicitud HTTP hasta el descuento de stock:  
  
1) Endpoint HTTP  
- Archivo: `API/MIDASERP.WebAPI/Controllers/v1/Comprobante/ComprobanteController.cs`  
- Acción: `POST /v1/Comprobante/register`  
- Método: `Register(ComprobDTO comprobDto)`  
- Llama a: `ComprobanteCommand.RegisterComprobanteConPagos(comprobDto)`  
  
2) Orquestación de aplicación  
- Archivo: `Core/MIDASERP.API.Application/Features/Comprobante/Command/ComprobanteCommand.cs`  
- Método: `RegisterComprobanteConPagos(ComprobDTO dto)`  
  - Abre `TransactionScope`.  
  - Según `dto.id` crea o actualiza el `Comprob` vía `PostComprobante`/`UpdateComprobante`.  
  - Si el comprobante aún no está “impreso” (`Impresa == 0`), llama a `Register(comprob.IdComprob, dto.idTipoComprobante)`.  
  - Esa llamada es la que dispara movimientos de stock.  
  
3) Lógica de negocio (BLL) para registrar  
- Archivo: `Infraestructure/MIDASERP.API.BLL/Services/Comprobante/ComprobanteService.cs`  
- Método: `Register(long idReceipt, long idReceiptType)`  
  - Recupera el `Comprob` completo.  
  - Numera/actualiza datos del comprobante.  
  - Cancela importes relacionados si corresponde.  
  - Si hay detalles (`ComprobDetalles`), intenta mover stock: `TryMoveStock(comprob, idReceiptType)`.  
  - Luego procesa pagos y actualiza el estado del comprobante.  
  
4) Movimiento de stock (núcleo)  
- Archivo: `Infraestructure/MIDASERP.API.BLL/Services/Comprobante/ComprobanteService.cs`  
- Método: `TryMoveStock(Comprob comprob, long idFac)`  
  - Llama a: `ArticuloService.MueveExistencias(comprob.ComprobDetalles, comprob, comprob.Fecha, comprob.IdSucursal, idFac)`  
  
5) Cálculo y persistencia de existencias  
- Archivo: `Infraestructure/MIDASERP.API.BLL/Services/Articulos/ArticuloService.cs`  
- Método: `MueveExistencias(IEnumerable<ComprobDetalle> articulos, Comprob comprob, DateTime fecha, long idSucursal, long idTipoFac=0)`  
  - Para cada detalle:  
    - Obtiene el artículo (`GetByID`) y su stock en situación origen (`GetStockArticulo(idArticulo, comprob.Situacionorigen)`) y stock total/general.  
    - Determina el “signo” del movimiento con `TipoComprobanteService.MueveStock(idTipoFac)`. Para ventas típicas será negativo (descuento), para compras positivo (ingreso).  
    - Si el artículo controla stock (`art.ControlStock != 0`):  
      - Verifica si la situación de origen/destino “cuenta existencias” (`ArticuloSituacionService.CuentaExistencia`).  
      - Actualiza o crea registros en `ArticulosStock` para origen y/o destino según corresponda, ajustando `Cantidad`.  
    - Registra un movimiento en `ArticulosMovim` (kardex) con `Saldo` post-movimiento, `Desdesituacion`, `Haciasituacion`, y una `Obs` que referencia el comprobante y el cliente.  
  - `SaveChanges` al final.  
  
6) Tablas/modelos involucrados  
- `Comprob` y `ComprobDetalle`: cabecera y detalle del comprobante de venta.  
- `ArticulosStock`: stock por artículo y por situación (depósito/ubicación). Archivo: `Infraestructure/MIDASERP.DataAccess/Models/ArticulosStock.cs`.  
- `ArticulosMovim`: historial de movimientos (kardex).  
  
---  
  
### Datos que gobiernan el descuento de stock  
- Tipo de comprobante (`idTipoComprobante`): por medio de `TipoComprobanteService.MueveStock(id)` define si suma o resta (p. ej. factura de venta resta; nota de crédito podría sumar).  
- Situaciones de origen y destino (`comprob.Situacionorigen`, `comprob.Situaciondestino`):  
  - Se actualiza el stock de origen y/o destino solo si esa situación “cuenta existencias”.  
- Control de stock del artículo (`art.ControlStock`): si es 0 (servicio), no mueve stock.  
- Cantidad por detalle (`ComprobDetalle.Cantidad`).  
- Fecha del movimiento (usada para registros e historial).  
  
Nota: La validación de “stock no negativo” está comentada en `MueveExistencias` (no se bloquea stock negativo actualmente).  
  
---  
  
### Guía para ejecutar el descuento de stock fuera de la creación del comprobante  
Si querés replicar el descuento sin crear un `Comprob` completo, tenés dos caminos:  
  
A) Reutilizar directamente `MueveExistencias` preparando un “comprobante mínimo”  
- Requerido:  
  - `idTipoComprobante` (para que `MueveStock` determine el signo).  
  - `Situacionorigen` y, opcionalmente, `Situaciondestino`.  
  - `Fecha` del movimiento y `IdSucursal`.  
  - Lista de ítems: para cada uno, `IdArticulo` y `Cantidad`.  
- Pasos:  
  1. Construí un objeto `Comprob` mínimo con: `IdComprob` (puede ser 0 si no lo usás en `Obs`), `Situacionorigen`, `Situaciondestino`, `Fecha`, `IdSucursal`, y opcionalmente `IdClienteNavigation` si querés que la observación del movimiento tenga nombre visible.  
  2. Construí una colección de `ComprobDetalle` con `IdArticulo` y `Cantidad`.  
  3. Llamá a `ArticuloService.MueveExistencias(detalles, comprob, fecha, idSucursal, idTipoComprobante)`.  
- Qué hace por vos:  
  - Calcula el signo (suma/resta) con `TipoComprobanteService.MueveStock`.  
  - Actualiza `ArticulosStock` en origen/destino según “cuenta existencias”.  
  - Inserta los renglones en `ArticulosMovim` con `Saldo` y `Obs` formateada.  
  
B) Implementar un flujo propio usando los mismos componentes  
- Si preferís no depender de `Comprob`, replicá la lógica base:  
  1. Determinar el signo: `signo = TipoComprobanteService.MueveStock(idTipoComprobante)`.  
  2. Por cada (idArticulo, cantidad):  
     - Verificar `ControlStock` del artículo (saltear si es servicio).  
     - Leer stock actual en origen (`GetStockArticulo(idArticulo, situacionOrigen)`) y stock general.  
     - Calcular `nuevoStockOrigen = stockOrigen + signo * cantidad` y `nuevoStockGeneral = stockGeneral + signo * cantidad`.  
     - Si `CuentaExistencia(situacionOrigen)` → upsert en `ArticulosStock` para origen.  
     - Si hay `situacionDestino` y `CuentaExistencia(situacionDestino)` → upsert en `ArticulosStock` para destino (si fuera un traspaso, el signo en destino suele ser el contrario).  
     - Registrar movimiento en `ArticulosMovim` con: artículo, fecha, cantidad, saldo post, desde/hacia situación, y una observación.  
  3. `SaveChanges`.  
- Beneficio: desacoplás de `Comprob`, pero mantenés la misma semántica de stock.  
  
---  
  
### Señales y condiciones que tenés que respetar  
- El “signo” del movimiento no lo hardcodees: usá `TipoComprobanteService.MueveStock(idTipoComprobante)` porque el proyecto ya codifica allí la política (venta/nota de crédito/compra/traslado/presupuesto/pedido, etc.).  
- Situaciones que cuentan existencias: consultá `ArticuloSituacionService.CuentaExistencia(idSituacion)` antes de tocar `ArticulosStock`.  
- Servicios: si `art.ControlStock == 0`, no mueven stock.  
- Concurrencia/consistencia: encapsulá los movimientos en una transacción si vas a realizar múltiples ítems o pasos dependientes.  
  
---  
  
### Dónde mirar/corroborrar en el código  
- Controller de comprobantes: `API/MIDASERP.WebAPI/Controllers/v1/Comprobante/ComprobanteController.cs`.  
- Orquestación de registro: `Core/.../Features/Comprobante/Command/ComprobanteCommand.cs` → métodos `RegisterComprobanteConPagos`, `Register`.  
- Registro BLL y gatillo de stock: `Infraestructure/.../Services/Comprobante/ComprobanteService.cs` → métodos `Register`, `TryMoveStock`.  
- Movimiento de existencias: `Infraestructure/.../Services/Articulos/ArticuloService.cs` → método `MueveExistencias` (núcleo del descuento/ingreso), y helpers como `GetStockArticulo`.  
- Modelos de stock: `Infraestructure/MIDASERP.DataAccess/Models/ArticulosStock.cs` y `ArticulosMovim`.  
  
---  
  
### Resumen corto  
- La resta de stock en una venta sucede cuando se registra el comprobante: `Register` → `TryMoveStock` → `ArticuloService.MueveExistencias`.  
- `MueveExistencias` calcula el signo según el tipo de comprobante, ajusta `ArticulosStock` por situación (si corresponde), y asienta `ArticulosMovim`.  
- Para hacerlo fuera del comprobante, reusá `MueveExistencias` construyendo un `Comprob` mínimo y una lista de `ComprobDetalle`; o bien replicá la misma lógica consultando `MueveStock`, `CuentaExistencia` y actualizando `ArticulosStock`/`ArticulosMovim` dentro de una transacción.  
  
Si querés, te armo un ejemplo mínimo (sin cambiar el proyecto) de cómo instanciar el `Comprob` y los `ComprobDetalle` para llamar directamente a `MueveExistencias`. ¿Preferís esa aproximación o una guía para un servicio nuevo desacoplado de `Comprob`?