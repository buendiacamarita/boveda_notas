✅ ### Reporte: Implementación de Movimiento de Stock sin Comprobante

Este reporte detalla cómo desacoplar la lógica de movimiento de existencias de la entidad `Comprobante`, permitiendo realizar ajustes de stock, transferencias internas o ingresos/egresos directos de manera independiente.

---

#### 1. Análisis de Dependencias Actuales

El método actual `ArticuloService.MueveExistencias` tiene dependencias críticas con la entidad `Comprob`:

- **Situaciones:** Utiliza `comprob.Situacionorigen` y `comprob.Situaciondestino`.
- **Signo del Movimiento:** Depende de `idTipoFac` para determinar si suma o resta mediante `TipoComprobanteService`.
- **Referencia en Kardex:** El objeto `ArticulosMovim` requiere un `IdComprob` (aunque es nulable en la base de datos).
- **Descripción:** Las observaciones del movimiento se construyen usando datos del cliente y número del comprobante.

---

#### 2. Propuesta Técnica: Abstracción del Movimiento de Stock

Para implementar esto, se recomienda crear un nuevo método en `ArticuloService` (o un servicio dedicado `StockService`) que utilice un DTO genérico.

##### A. Definición del DTO de Entrada

```
public class StockMovementRequest
{
    public long IdSucursal { get; set; }
    public long? IdSituacionOrigen { get; set; }
    public long? IdSituacionDestino { get; set; }
    public string Observaciones { get; set; }
    public DateTime Fecha { get; set; } = DateTime.Now;
    public List<StockMovementDetail> Detalles { get; set; }
}

public class StockMovementDetail
{
    public long IdArticulo { get; set; }
    public decimal Cantidad { get; set; }
    public decimal Signo { get; set; } // 1 para ingreso, -1 para egreso
}
```

##### B. Implementación del Nuevo Método en `ArticuloService`

Se propone la creación de un método `MueveExistenciasManual` que reutilice la lógica interna de actualización de tablas pero sin el objeto `Comprob`.

```
public async Task MueveExistenciasManual(StockMovementRequest request)
{
    long nuevoIdMovim = (await _model.Set<ArticulosMovim>().MaxAsync(m => (long?)m.IdMovim) ?? 0) + 1;

    foreach (var detalle in request.Detalles)
    {
        var art = await GetByID(detalle.IdArticulo);
        if (art.ControlStock == 0) continue; // Si no controla stock, omitir

        var stockGeneral = await GetStockArticulo(detalle.IdArticulo);

        // 1. Actualizar Stock en Situación Origen (Resta si hay origen)
        if (request.IdSituacionOrigen.HasValue && await _articuloSituacionService.CuentaExistencia(request.IdSituacionOrigen))
        {
            await AjustarStockFisico(detalle.IdArticulo, request.IdSituacionOrigen.Value, -detalle.Cantidad);
        }

        // 2. Actualizar Stock en Situación Destino (Suma si hay destino)
        if (request.IdSituacionDestino.HasValue && await _articuloSituacionService.CuentaExistencia(request.IdSituacionDestino))
        {
            await AjustarStockFisico(detalle.IdArticulo, request.IdSituacionDestino.Value, detalle.Cantidad);
        }

        // 3. Registrar en Kardex (ArticulosMovim)
        var movimiento = new ArticulosMovim
        {
            IdMovim = nuevoIdMovim++,
            IdArticulo = detalle.IdArticulo,
            IdComprob = null, // Sin comprobante vinculado
            Cantidad = detalle.Cantidad,
            Fecha = request.Fecha,
            Desdesituacion = request.IdSituacionOrigen,
            Haciasituacion = request.IdSituacionDestino,
            Saldo = (decimal)stockGeneral + (detalle.Signo * detalle.Cantidad),
            Estado = 1,
            Obs = request.Observaciones,
            CreadoPor = UserContext.CurrentUserId
        };

        await _model.Set<ArticulosMovim>().AddAsync(movimiento);
    }
    
    await _model.SaveChangesAsync();
}

// Método auxiliar para evitar duplicación de código
private async Task AjustarStockFisico(long idArticulo, long idSituacion, decimal cantidadAjuste)
{
    var stockRecord = await _model.Set<ArticulosStock>()
        .FirstOrDefaultAsync(m => m.IdArticulo == idArticulo && m.IdSituacion == idSituacion);

    if (stockRecord != null)
    {
        stockRecord.Cantidad += cantidadAjuste;
        _model.Update(stockRecord);
    }
    else
    {
        await _model.Set<ArticulosStock>().AddAsync(new ArticulosStock
        {
            IdArticulo = idArticulo,
            IdSituacion = idSituacion,
            Cantidad = cantidadAjuste,
            Estado = 1,
            FechaCreado = DateTime.Now
        });
    }
}
```

---

#### 3. Beneficios de esta Implementación

1. **Versatilidad:** Permite realizar ajustes de inventario por rotura, pérdida o transferencias entre depósitos sin necesidad de simular una venta o compra.
2. **Mantenibilidad:** Se extrae la lógica de "Ajuste de Stock" a un método privado (`AjustarStockFisico`) que puede ser invocado tanto por el flujo de comprobantes como por el flujo manual.
3. **Trazabilidad:** El Kardex (`ArticulosMovim`) sigue funcionando correctamente, permitiendo filtrar movimientos que no tienen un `IdComprob` asociado para auditorías de stock manual.

#### 4. Consideraciones Adicionales

- **Permisos:** Se debería restringir el uso de este método manual a roles de usuario con permisos de "Administrador de Stock".
- **Validaciones:** Antes de mover, validar que la situación de origen tenga stock suficiente si el sistema no permite stock negativo.
- **Transaccionalidad:** Asegurar que todo el movimiento (múltiples artículos) se ejecute dentro de una misma transacción de base de datos para evitar inconsistencias.