# 🏗️ Arquitectura de Debug Multi-Proyecto

## Diagrama de Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                         VS Code IDE                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Debug Configurations                    │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  MIDASERP + Jupiter Auth (Multi-Debug) [COMPOUND]   │  │  │
│  │  │  ┌──────────────────┐  ┌──────────────────────────┐ │  │  │
│  │  │  │ MIDASERP.WebAPI  │  │ Jupiter.API.AuthService  │ │  │  │
│  │  │  └──────────────────┘  └──────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Build Tasks                            │  │
│  │  • build-all        • watch-midaserp                       │  │
│  │  • build-midaserp   • watch-jupiter-auth                   │  │
│  │  • build-jupiter    • clean-midaserp                       │  │
│  │                     • clean-jupiter-auth                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Compila y Ejecuta
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      .NET 8.0 Runtime                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┴──────────────┐
                ▼                            ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│   MIDASERP.WebAPI        │    │ Jupiter.API.AuthService  │
│                          │    │                          │
│  Port: 5020 (HTTP)       │    │  Port: 5021 (HTTP)       │
│  Port: 7190 (HTTPS)      │    │  Port: 6021 (HTTPS)      │
│                          │    │                          │
│  /swagger                │    │  /swagger                │
│  /api/v1/...             │    │  /api/auth/...           │
│                          │    │                          │
│  ┌────────────────────┐  │    │  ┌────────────────────┐  │
│  │ Controllers        │  │    │  │ Auth Strategies    │  │
│  │ • v1/...           │  │    │  │ • JWT              │  │
│  └────────────────────┘  │    │  │ • Firebase         │  │
│                          │    │  └────────────────────┘  │
│  ┌────────────────────┐  │    │                          │
│  │ Application Layer  │  │    │  ┌────────────────────┐  │
│  │ • MIDASERP.API     │  │    │  │ Application Layer  │  │
│  │   .Application     │  │    │  │ • Jupiter.Auth     │  │
│  └────────────────────┘  │    │  │   .Application     │  │
│                          │    │  └────────────────────┘  │
│  ┌────────────────────┐  │    │                          │
│  │ Persistence        │  │    │  ┌────────────────────┐  │
│  │ • MIDASERP.API     │  │    │  │ Identity           │  │
│  │   .Persistence     │  │    │  │ • MIDASERP.API     │  │
│  │ • DataAccess       │  │    │  │   .Identity        │  │
│  └────────────────────┘  │    │  └────────────────────┘  │
│                          │    │                          │
│  ┌────────────────────┐  │    │                          │
│  │ Identity           │  │    │                          │
│  │ • MIDASERP.API     │  │    │                          │
│  │   .Identity        │  │    │                          │
│  └────────────────────┘  │    │                          │
└──────────────────────────┘    └──────────────────────────┘
                │                            │
                └────────────┬───────────────┘
                             ▼
                ┌─────────────────────────┐
                │   PostgreSQL Database   │
                │                         │
                │  • MIDAS ERP Data       │
                │  • User Identity        │
                │  • Auth Data            │
                └─────────────────────────┘
```

## Flujo de Ejecución

### 1. Inicio de Debug (F5)

```
Usuario presiona F5
    │
    ├─> VS Code lee launch.json
    │
    ├─> Ejecuta preLaunchTask: "build-midaserp"
    │   └─> dotnet build API/MIDASERP.WebAPI/...
    │
    ├─> Ejecuta preLaunchTask: "build-jupiter-auth"
    │   └─> dotnet build API/Jupiter.API.AuthService/...
    │
    ├─> Inicia MIDASERP.WebAPI
    │   ├─> Puerto: 5020
    │   ├─> Abre: http://localhost:5020/swagger
    │   └─> Adjunta debugger
    │
    └─> Inicia Jupiter.API.AuthService
        ├─> Puerto: 5021
        ├─> Abre: http://localhost:5021/swagger
        └─> Adjunta debugger
```

### 2. Debugging Activo

```
┌─────────────────────────────────────────┐
│         Developer Actions               │
│  • Set breakpoints                      │
│  • Step through code                    │
│  • Inspect variables                    │
│  • Watch expressions                    │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│         VS Code Debugger                │
│  • Manages both debug sessions          │
│  • Handles breakpoints                  │
│  • Provides debug console                │
└─────────────────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
┌──────────────┐    ┌──────────────────┐
│ MIDASERP     │    │ Jupiter Auth     │
│ Debug        │    │ Debug            │
│ Session      │    │ Session          │
└──────────────┘    └──────────────────┘
```

### 3. Comunicación entre Servicios

```
┌──────────────────┐
│  Client/Browser  │
└──────────────────┘
         │
         │ HTTP Request
         ▼
┌──────────────────────────┐
│   MIDASERP.WebAPI        │
│   (Port 5020)            │
│                          │
│   Necesita autenticar    │
│   usuario...             │
└──────────────────────────┘
         │
         │ HTTP Request
         │ POST /api/auth/login
         ▼
┌──────────────────────────┐
│ Jupiter.API.AuthService  │
│ (Port 5021)              │
│                          │
│ Valida credenciales      │
│ Genera JWT token         │
└──────────────────────────┘
         │
         │ Response: JWT
         ▼
┌──────────────────────────┐
│   MIDASERP.WebAPI        │
│   Recibe token           │
│   Continúa proceso       │
└──────────────────────────┘
         │
         │ Response
         ▼
┌──────────────────┐
│  Client/Browser  │
└──────────────────┘
```

## Estructura de Proyectos

```
MIDAS.WebAPI/WebAPI/
│
├── .vscode/                          # Configuración de VS Code
│   ├── launch.json                   # Debug configurations
│   ├── tasks.json                    # Build tasks
│   ├── settings.json                 # Workspace settings
│   ├── extensions.json               # Recommended extensions
│   ├── DEBUG_GUIDE.md               # Debug guide
│   └── install-dotnet.sh            # .NET installer
│
├── API/
│   ├── MIDASERP.WebAPI/             # Main ERP API
│   │   ├── Controllers/
│   │   ├── Properties/
│   │   │   └── launchSettings.json  # Port: 5020
│   │   └── MIDASERP.WebAPI.csproj
│   │
│   └── Jupiter.API.AuthService/     # Authentication Service
│       ├── AuthStrategies/
│       ├── Controllers/
│       ├── Properties/
│       │   └── launchSettings.json  # Port: 5021 (MODIFICADO)
│       └── Jupiter.API.AuthService.csproj
│
├── Core/                            # Application Logic
│   ├── MIDASERP.API.Application/
│   ├── Jupiter.Auth.Application/
│   └── ...
│
├── Infraestructure/                 # Data Access & Identity
│   ├── MIDASERP.API.Persistence/
│   ├── MIDASERP.API.Identity/
│   └── ...
│
├── start-services.sh                # Start both services
├── stop-services.sh                 # Stop all services
├── README_DEBUG_SETUP.md            # Complete setup guide
└── SETUP_SUMMARY.md                 # Quick summary
```

## Ventajas de esta Configuración

### ✅ Desarrollo Eficiente
- **Un solo F5**: Inicia ambos servicios simultáneamente
- **Breakpoints compartidos**: Debug en ambos proyectos a la vez
- **Hot Reload**: Cambios reflejados automáticamente

### ✅ Debugging Avanzado
- **Consolas separadas**: Output independiente para cada servicio
- **Control individual**: Pausa/continúa cada servicio por separado
- **Variables compartidas**: Inspecciona estado en ambos servicios

### ✅ Productividad
- **Tareas automatizadas**: Build, watch, clean con un click
- **Scripts útiles**: Inicio/parada desde terminal
- **Documentación completa**: Guías y troubleshooting

### ✅ Mantenibilidad
- **Configuración versionada**: Todo en .vscode/
- **Puertos sin conflicto**: 5020 y 5021
- **Extensiones recomendadas**: Setup consistente en equipo

## Tecnologías Utilizadas

| Componente | Tecnología | Versión |
|------------|-----------|---------|
| Runtime | .NET | 8.0 |
| IDE | Visual Studio Code | Latest |
| Debugger | C# Extension | Latest |
| Database | PostgreSQL | - |
| API Framework | ASP.NET Core | 8.0 |
| Auth | JWT + Firebase | - |
| ORM | Entity Framework Core | 8.0 |

## Referencias

- [VS Code Multi-target Debugging](https://code.visualstudio.com/docs/editor/debugging#_multitarget-debugging)
- [.NET Debug Configuration](https://github.com/dotnet/vscode-csharp/blob/main/debugger-launchjson.md)
- [ASP.NET Core Launch Settings](https://docs.microsoft.com/aspnet/core/fundamentals/environments)

---

**Última actualización:** 2025-12-09  
**Configurado por:** Antigravity AI Assistant
