### Ejemplo práctico: descontar stock fuera de la creación del comprobante (sin tocar el código existente)  
A continuación tenés un ejemplo autocontenido de cómo reutilizar el flujo de stock del proyecto sin crear/modificar comprobantes ni editar archivos. La idea es construir un `Comprob` “mínimo”, una lista de `ComprobDetalle`, y llamar a `ArticuloService.MueveExistencias`, que ya hace todo: determina el signo con `TipoComprobanteService.MueveStock`, actualiza `ArticulosStock` y asienta `ArticulosMovim`.  
  
#### Dependencias y modelos a usar  
- Servicio: `ArticuloService` (resuelto por DI del proyecto).  
- Modelos: `Comprob`, `ComprobDetalle` de `MIDASERP.DataAccess.Models`.  
  
Usings típicos:  
```csharp  
using System;  
using System.Collections.Generic;  
using System.Threading.Tasks;  
using Microsoft.Extensions.DependencyInjection;  
using MIDASERP.API.BLL.Services; // ArticuloService  
using MIDASERP.DataAccess.Models; // Comprob, ComprobDetalle  
```  
  
---  
  
### 1) Método utilitario para mover existencias fuera del comprobante  
Este método lo podés invocar desde un job, un handler, un endpoint nuevo, etc. No requiere modificar nada del proyecto, solo usar los servicios ya registrados en DI.  
  
```csharp  
public static class StockHelper  
{  
    // items: lista de (IdArticulo, Cantidad) a restar    public static async Task DescontarStockAsync(        IServiceProvider serviceProvider,        long idTipoComprobante,       // p.ej. el tipo de comprobante “Factura de Venta” que RESTA stock        long situacionOrigen,         // depósito/ubicación desde donde sale el stock        long? situacionDestino,       // opcional: si fuera un traslado, a dónde entra        long idSucursal,        DateTime fechaMovimiento,        IEnumerable<(long idArticulo, decimal cantidad)> items,        long? idCliente = null,       // opcional, solo para enriquecer la observación del movimiento        string? nombreCliente = null  // opcional    )    {        // 1) Resolver el servicio que ya sabe mover existencias        using var scope = serviceProvider.CreateScope();        var articuloService = scope.ServiceProvider.GetRequiredService<ArticuloService>();  
        // 2) Armar un Comprob mínimo (solo los campos que MueveExistencias usa)        var comprob = new Comprob        {            IdComprob = 0, // no es necesario que exista en DB para el movimiento de stock            Situacionorigen = situacionOrigen,            Situaciondestino = situacionDestino,            Fecha = fechaMovimiento,            IdSucursal = idSucursal,            // Si querés una "Obs" más descriptiva en ArticulosMovim, seteá el cliente visible            IdClienteNavigation = nombreCliente is null ? null : new Persona { NombreVisible = nombreCliente }        };  
        // 3) Armar los detalles con IdArticulo y Cantidad        var detalles = new List<ComprobDetalle>();        foreach (var (idArticulo, cantidad) in items)        {            detalles.Add(new ComprobDetalle            {                IdArticulo = idArticulo,                Cantidad = cantidad            });        }  
        // 4) Ejecutar el flujo estándar de stock        await articuloService.MueveExistencias(            articulos: detalles,            comprob: comprob,            fecha: fechaMovimiento,            idSucursal: idSucursal,            idTipoFac: idTipoComprobante // define el signo (suma/resta) vía TipoComprobanteService.MueveStock        );    }}  
```  
  
Uso de ejemplo:  
```csharp  
// Ejemplo de uso (p.ej. dentro de un endpoint o un job)  
await StockHelper.DescontarStockAsync(  
    serviceProvider,    idTipoComprobante: 101,      // ID real del tipo de comprobante de venta que RESTA stock    situacionOrigen: 1,          // Depósito/ubicación origen    situacionDestino: null,      // Sin destino (no es traslado)    idSucursal: 1,    fechaMovimiento: DateTime.Now,    items: new[]    {        (idArticulo: 123L, cantidad: 2m),        (idArticulo: 456L, cantidad: 1m)    },    idCliente: 789,    nombreCliente: "Cliente SA");  
```  
  
Qué hace internamente este llamado (reutilizando tu código):  
- Determina el signo con `TipoComprobanteService.MueveStock(idTipoComprobante)`.  
- Verifica `ControlStock` del artículo (si es servicio, no mueve).  
- Respeta `ArticuloSituacionService.CuentaExistencia` para origen/destino.  
- Actualiza `ArticulosStock` y agrega asientos en `ArticulosMovim` con `Saldo` y `Obs`.  
  
---  
  
### 2) Variante “traslado” entre situaciones (origen y destino)  
Si en lugar de una venta querés trasferir stock entre situaciones (depósitos), podés usar un `idTipoComprobante` cuyo `MueveStock` haga el movimiento de salida/entrada según la lógica existente y setear `Situaciondestino`.  
  
```csharp  
await StockHelper.DescontarStockAsync(  
    serviceProvider,    idTipoComprobante: 205,     // un tipo de comprobante configurado como traslado    situacionOrigen: 1,         // Depósito A    situacionDestino: 2,        // Depósito B    idSucursal: 1,    fechaMovimiento: DateTime.Now,    items: new[] { (321L, 5m) },    nombreCliente: "Movimiento interno");  
```  
  
---  
  
### 3) Recomendaciones operativas  
- Transacciones: si hacés múltiples llamados o mezclás con otras operaciones críticas, envolvé en una transacción (p.ej., `TransactionScope`) en tu punto de orquestación.  
- Validaciones: hoy la validación para evitar stock negativo está comentada en `MueveExistencias`. Si te importa, validá antes de llamar o chequeá el saldo resultante y reaccioná.  
- Tipos de comprobante: usá el `idTipoComprobante` real del sistema para que el signo y la semántica sean consistentes con la configuración vigente.  
- Situaciones: asegurate de usar situaciones que “cuenten existencia”; caso contrario, no se actualizará `ArticulosStock`.  
  
---  
  
### 4) Parámetros mínimos que debés conocer  
- `idTipoComprobante`: clave para el signo (resta/suma).  
- `situacionOrigen` y (opcional) `situacionDestino`.  
- `idSucursal` y `fechaMovimiento`.  
- Por ítem: `idArticulo` y `cantidad`.  
- (Opcional) `nombreCliente` para enriquecer `Obs` en `ArticulosMovim`.  
  
---  
  
¿Querés que lo adapte a un ejemplo de endpoint (controller) o a un hosted service (job) con `IServiceProvider` y `TransactionScope` ya montados en tu solución? Puedo mostrártelo sin tocar el proyecto existente.