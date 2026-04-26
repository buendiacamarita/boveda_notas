La opción "Segura": Scaffold a una carpeta temporal (Staging)

Si tienes mucho código personalizado en tu DbContext actual y no quieres que se borre, sigue este truco profesional:  
Genera la tabla nueva en una carpeta aparte:  

```
dotnet ef dbcontext scaffold "tu_cadena_de_conexion" Microsoft.EntityFrameworkCore.SqlServer --table NuevaTabla --output-dir ModelosTemp --context-dir ContextoTemp  
```
##### Copia y pega:  
- Mueve la clase `NuevaTabla.cs` generada a tu carpeta de modelos real
- Copia la línea del public virtual `DbSet<NuevaTabla>` de ContextoTemp a tu DbContext real
- Copia la configuración de OnModelCreating (Fluent API) correspondiente a la nueva tabla

