Borrar cache de paquetes nuget

```
dotnet nuget locals all --clear
```

Como Hacer un contexto EF desde linea de comandos

```csharp
dotnet ef dbcontext scaffold "Server=localhost,1433; TrustServerCertificate=True; Database=CsharpDB;User=sa;Password=Lola_100314!;" Microsoft.EntityFrameworkCore.SqlServer
```

En el caso que sea con autenticacion de windows agregar Trusted_Connection=True

sepuede agregar —outputdir para especificar donde y —force

Como hacer una migracion con EF

```csharp
dotnet ef migrations add InitDb
```

Como crear la base de datos despues de la migracion con EF

```csharp
dotnet ef database update   
```

Buscar paquetes con linea de comandos

```csharp
dotnet nuget search <nombre-del-paquete>
```