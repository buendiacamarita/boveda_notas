# 🚀 Guía de Configuración de Debug Multi-Proyecto

Esta guía te ayudará a configurar y ejecutar **MIDASERP.WebAPI** y **Jupiter.API.AuthService** simultáneamente en modo debug.

## ⚠️ IMPORTANTE: Breakpoints y Debugging

> **🔴 Los breakpoints SOLO funcionan cuando ejecutas con F5 (Debug Mode)**
> 
> - ✅ **F5 en VS Code** → Breakpoints funcionan
> - ❌ **`dotnet run`** → Breakpoints NO funcionan
> - ❌ **`./start-services.sh`** → Breakpoints NO funcionan
>
> **Si necesitas usar breakpoints, DEBES usar F5 con la configuración de debug.**
>
> 📖 Para más detalles, lee: [`DEBUG_VS_RUN.md`](DEBUG_VS_RUN.md)

## 📋 Requisitos Previos

### 1. Instalar .NET 8.0 SDK

**Opción A: Instalación Automática (Recomendado)**
```bash
cd /home/nestorbardel/Documentos/GrupoAgni/Repos/MIDAS.WebAPI/WebAPI
./.vscode/install-dotnet.sh
```

**Opción B: Instalación Manual**
```bash
sudo apt update
sudo apt install dotnet-sdk-8.0
```

Verifica la instalación:
```bash
dotnet --version
# Debería mostrar: 8.0.x
```

### 2. Instalar Extensiones de VS Code

Abre VS Code en este workspace y acepta instalar las extensiones recomendadas:
- **C#** (ms-dotnettools.csharp)
- **C# Dev Kit** (ms-dotnettools.csdevkit)
- **.NET Runtime Install Tool**

O instálalas manualmente desde la pestaña de extensiones (Ctrl+Shift+X).

### 3. Restaurar Paquetes NuGet

```bash
cd /home/nestorbardel/Documentos/GrupoAgni/Repos/MIDAS.WebAPI/WebAPI
dotnet restore
```

## 🎯 Cómo Ejecutar en Modo Debug

### Método 1: Debug Multi-Proyecto (Ambos a la vez)

1. Abre VS Code en la carpeta del workspace
2. Presiona `F5` o ve a **Run and Debug** (Ctrl+Shift+D)
3. Selecciona **"MIDASERP + Jupiter Auth (Multi-Debug)"**
4. Haz clic en el botón verde de play o presiona `F5`

✅ **Resultado:**
- MIDASERP.WebAPI se ejecutará en `http://localhost:5020`
- Jupiter.API.AuthService se ejecutará en `http://localhost:5021`
- Ambos abrirán automáticamente sus páginas de Swagger

### Método 2: Debug Individual

Selecciona una de estas configuraciones:
- **"MIDASERP.WebAPI"** - Solo MIDASERP
- **"Jupiter.API.AuthService"** - Solo Auth Service

## 🔧 Configuración Realizada

### Archivos Creados

```
.vscode/
├── launch.json          # Configuración de debug
├── tasks.json           # Tareas de build
├── settings.json        # Configuración del workspace
├── extensions.json      # Extensiones recomendadas
├── DEBUG_GUIDE.md       # Guía detallada de debug
└── install-dotnet.sh    # Script de instalación de .NET
```

### Cambios en Proyectos Existentes

**Jupiter.API.AuthService/Properties/launchSettings.json**
- ✅ Puerto HTTP cambiado de `5020` a `5021` (evita conflicto con MIDASERP)

### Puertos Configurados

| Proyecto | HTTP | HTTPS | Swagger |
|----------|------|-------|---------|
| **MIDASERP.WebAPI** | 5020 | 7190 | http://localhost:5020/swagger |
| **Jupiter.API.AuthService** | 5021 | 6021 | http://localhost:5021/swagger |

## 🛠️ Tareas Disponibles

Ejecuta desde Command Palette (Ctrl+Shift+P > "Tasks: Run Task"):

### Tareas de Build
- **build-all** - Compila ambos proyectos
- **build-midaserp** - Compila solo MIDASERP.WebAPI
- **build-jupiter-auth** - Compila solo Jupiter.API.AuthService

### Tareas de Watch (Hot Reload)
- **watch-midaserp** - MIDASERP con recarga automática
- **watch-jupiter-auth** - Jupiter Auth con recarga automática

### Tareas de Limpieza
- **clean-midaserp** - Limpia build de MIDASERP
- **clean-jupiter-auth** - Limpia build de Jupiter Auth

## 🐛 Debugging Avanzado

### Breakpoints
- Establece breakpoints en cualquier archivo .cs de ambos proyectos
- Los breakpoints funcionan simultáneamente en ambos servicios
- Cada servicio tiene su propia consola de debug

### Consolas de Debug
Cuando ejecutas en modo multi-debug verás:
- **DEBUG CONSOLE**: Salida de debug combinada
- **TERMINAL**: Una terminal para cada proyecto
- **PROBLEMS**: Errores y advertencias de compilación

### Variables de Entorno
Ambos proyectos usan:
```
ASPNETCORE_ENVIRONMENT=Development
```

Puedes modificar esto en `.vscode/launch.json` si necesitas diferentes configuraciones.

## 📝 Comandos Útiles

### Compilar Manualmente
```bash
# Compilar MIDASERP.WebAPI
dotnet build API/MIDASERP.WebAPI/MIDASERP.WebAPI.csproj

# Compilar Jupiter.API.AuthService
dotnet build API/Jupiter.API.AuthService/Jupiter.API.AuthService.csproj

# Compilar toda la solución
dotnet build MIDASERP.WebAPI.sln
```

### Ejecutar sin Debug
```bash
# MIDASERP.WebAPI
cd API/MIDASERP.WebAPI
dotnet run

# Jupiter.API.AuthService
cd API/Jupiter.API.AuthService
dotnet run
```

### Limpiar y Reconstruir
```bash
# Limpiar
dotnet clean

# Restaurar paquetes
dotnet restore

# Reconstruir
dotnet build
```

## 🔍 Troubleshooting

### Error: "dotnet: command not found"
**Solución:**
```bash
./.vscode/install-dotnet.sh
```

### Error: "Puerto en uso"
**Solución:**
```bash
# Ver qué proceso usa el puerto
lsof -i :5020
lsof -i :5021

# Matar el proceso (reemplaza PID con el número del proceso)
kill -9 PID
```

### Error: "No se pueden restaurar los paquetes"
**Solución:**
```bash
# Limpiar caché de NuGet
dotnet nuget locals all --clear

# Restaurar de nuevo
dotnet restore
```

### Los breakpoints no funcionan
**Solución:**
1. Verifica que estés compilando en modo Debug (no Release)
2. Recarga VS Code: Ctrl+Shift+P > "Developer: Reload Window"
3. Limpia y reconstruye: `dotnet clean && dotnet build`

### Error de compilación
**Solución:**
```bash
# Ver detalles del error
dotnet build --verbosity detailed

# Limpiar y reconstruir
dotnet clean
dotnet restore
dotnet build
```

## 📚 Recursos Adicionales

- [Documentación de .NET](https://docs.microsoft.com/dotnet/)
- [Debugging en VS Code](https://code.visualstudio.com/docs/editor/debugging)
- [ASP.NET Core](https://docs.microsoft.com/aspnet/core/)

## 🎉 ¡Listo!

Ahora puedes:
1. ✅ Ejecutar ambos proyectos simultáneamente
2. ✅ Depurar con breakpoints en ambos servicios
3. ✅ Ver logs separados para cada proyecto
4. ✅ Hot reload con las tareas de watch
5. ✅ Acceder a Swagger de ambos servicios

---

**Próximos Pasos:**
1. Instala .NET 8.0 SDK si aún no lo has hecho
2. Restaura los paquetes con `dotnet restore`
3. Presiona `F5` y selecciona "MIDASERP + Jupiter Auth (Multi-Debug)"
4. ¡Comienza a desarrollar! 🚀
