# Guía de Implementación Backend (ASP.NET Core + PostgreSQL)

Esta guía detalla cómo implementar la sincronización incremental utilizando el campo existente `fecha_modificado` en PostgreSQL.

## 1. Requisito en Base de Datos

Se asume que la tabla de artículos (y cualquier otra a sincronizar) cuenta con el siguiente campo:
- **Campo**: `fecha_modificado`
- **Tipo**: `TIMESTAMP` (o `TIMESTAMPTZ`)
- **Formato esperado**: `yyyy-MM-dd HH:mm:ss.ffffff`

> [!IMPORTANT]
> Es vital que el Backend actualice este campo automáticamente cada vez que se realice un `UPDATE` o `INSERT` sobre el registro.

---

## 2. Implementación en ASP.NET Core

### Paso 2.1: Endpoint de "Novedades"

En lugar de una tabla de logs, consultaremos directamente la tabla de artículos filtrando por la fecha enviada por la app móvil.

**Ejemplo en `ArticulosController.cs`:**

```csharp
[HttpGet("novedades")]
public async Task<IActionResult> GetNovedades([FromQuery] DateTime since)
{
    try 
    {
        // 1. Consultar artículos modificados después de la fecha 'since'
        // El framework (EF Core) se encarga de la conversión de tipos
        var articulosActualizados = await _context.Articulos
            .Where(a => a.FechaModificado > since)
            // Asegúrate de incluir las mismas relaciones que en el listado completo
            .Include(a => a.Alicuota) 
            .Include(a => a.Categoria)
            .ToListAsync();

        // 2. Retornar la lista (si está vacía, retornar array vacío [])
        return Ok(articulosActualizados);
    }
    catch (Exception ex)
    {
        return StatusCode(500, "Error al obtener novedades: " + ex.Message);
    }
}
```

### Paso 2.2: Consideraciones de Formato de Fecha

La App móvil envía la fecha en formato **ISO 8601 (UTC)** (ejemplo: `2023-12-18T13:45:00.000Z`). 

ASP.NET Core mapea automáticamente este string a un objeto `DateTime`. Sin embargo, para asegurar precisión milimétrica con el campo `timestamp` de Postgres, verifica que en tu `Program.cs` o configuración de JSON no se esté perdiendo la precisión de microsegundos (`ffffff`).

---

## 3. Resumen de Flujo Simplificado

1. **Móvil (SQLite)**: Consulta su `synchronization_log` y obtiene la fecha de su última sincronización exitosa.
2. **Móvil (API Call)**: Llama a `GET /api/Articulo/novedades?since=2023-12-18T10:00:00Z`.
3. **Backend (Postgres)**: Ejecuta `SELECT * FROM articulos WHERE fecha_modificado > '2023-12-18 10:00:00'`.
4. **Backend (JSON)**: Devuelve solo los registros que cambiaron.
5. **Móvil (Sync)**: Realiza el `INSERT OR REPLACE` en su base de datos local.

---

## 4. Limitaciones de este enfoque

> [!WARNING]
> **Eliminaciones**: Este método no detecta registros que fueron eliminados físicamente (`DELETE`) de la base de datos de PostgreSQL. 
> - Para manejar esto sin tablas de log, se recomienda usar **Borrado Lógico** (una columna `estado` o `activo` que cambie a `false` y actualice la `fecha_modificado`).
