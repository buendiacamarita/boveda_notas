# 📦 Resumen de Configuración - Debug Multi-Proyecto

## ✅ Archivos Creados

### Configuración de VS Code (.vscode/)
```
.vscode/
├── 📄 launch.json           - Configuración de debug (individual y multi-proyecto)
├── 📄 tasks.json            - Tareas de build, watch y clean
├── 📄 settings.json         - Configuración del workspace
├── 📄 extensions.json       - Extensiones recomendadas
├── 📄 DEBUG_GUIDE.md        - Guía detallada de debugging
└── 📄 install-dotnet.sh     - Script de instalación de .NET SDK
```

### Scripts de Utilidad (raíz del proyecto)
```
├── 📄 README_DEBUG_SETUP.md - Guía completa de configuración
├── 📄 start-services.sh     - Iniciar ambos servicios desde terminal
└── 📄 stop-services.sh      - Detener todos los servicios
```

## 🔧 Cambios Realizados

### Jupiter.API.AuthService
- ✅ Puerto HTTP cambiado: `5020` → `5021`
- 📁 Archivo: `API/Jupiter.API.AuthService/Properties/launchSettings.json`
- 🎯 Razón: Evitar conflicto de puertos con MIDASERP.WebAPI

## 🚀 Configuraciones de Debug Disponibles

### 1. Multi-Debug (Recomendado)
**Nombre:** `MIDASERP + Jupiter Auth (Multi-Debug)`
- ✅ Ejecuta ambos proyectos simultáneamente
- ✅ Breakpoints funcionan en ambos
- ✅ Consolas separadas para cada servicio
- ✅ Detiene ambos con un solo click

### 2. Debug Individual - MIDASERP.WebAPI
**Nombre:** `MIDASERP.WebAPI`
- Puerto: `http://localhost:5020`
- Swagger: `http://localhost:5020/swagger`

### 3. Debug Individual - Jupiter.API.AuthService
**Nombre:** `Jupiter.API.AuthService`
- Puerto: `http://localhost:5021`
- Swagger: `http://localhost:5021/swagger`

## 🛠️ Tareas Configuradas

### Build
- `build-all` - Compila ambos proyectos
- `build-midaserp` - Solo MIDASERP.WebAPI
- `build-jupiter-auth` - Solo Jupiter.API.AuthService

### Watch (Hot Reload)
- `watch-midaserp` - MIDASERP con recarga automática
- `watch-jupiter-auth` - Jupiter Auth con recarga automática

### Clean
- `clean-midaserp` - Limpia MIDASERP
- `clean-jupiter-auth` - Limpia Jupiter Auth

## 📊 Puertos Configurados

| Servicio | HTTP | HTTPS | Swagger |
|----------|------|-------|---------|
| MIDASERP.WebAPI | 5020 | 7190 | http://localhost:5020/swagger |
| Jupiter.API.AuthService | 5021 | 6021 | http://localhost:5021/swagger |

## 🎯 Cómo Usar

### Opción 1: VS Code Debug (Recomendado para desarrollo)
```bash
# 1. Abre VS Code en el workspace
code .

# 2. Presiona F5 y selecciona:
#    "MIDASERP + Jupiter Auth (Multi-Debug)"

# 3. ¡Listo! Ambos servicios se ejecutarán con debug
```

### Opción 2: Terminal (Rápido, sin debug)
```bash
# Iniciar ambos servicios
./start-services.sh

# Detener todos los servicios
./stop-services.sh
```

### Opción 3: Manual
```bash
# Terminal 1 - MIDASERP
cd API/MIDASERP.WebAPI
dotnet run

# Terminal 2 - Jupiter Auth
cd API/Jupiter.API.AuthService
dotnet run
```

## 📋 Checklist de Instalación

- [ ] Instalar .NET 8.0 SDK
  ```bash
  ./.vscode/install-dotnet.sh
  ```

- [ ] Instalar extensiones de VS Code
  - Abre VS Code y acepta las extensiones recomendadas
  - O instala manualmente: C#, C# Dev Kit

- [ ] Restaurar paquetes NuGet
  ```bash
  dotnet restore
  ```

- [ ] Probar la configuración
  ```bash
  # Compilar todo
  dotnet build
  
  # O usar la tarea de VS Code
  # Ctrl+Shift+P > Tasks: Run Task > build-all
  ```

- [ ] Ejecutar en debug
  - Presiona F5 en VS Code
  - Selecciona "MIDASERP + Jupiter Auth (Multi-Debug)"

## 🐛 Troubleshooting Rápido

### "dotnet: command not found"
```bash
./.vscode/install-dotnet.sh
```

### "Puerto en uso"
```bash
./stop-services.sh
```

### "Error de compilación"
```bash
dotnet clean
dotnet restore
dotnet build
```

### "Breakpoints no funcionan"
```bash
# En VS Code:
# Ctrl+Shift+P > Developer: Reload Window
```

## 📚 Documentación

- **Guía Completa:** `README_DEBUG_SETUP.md`
- **Guía de Debug:** `.vscode/DEBUG_GUIDE.md`
- **Configuración de Launch:** `.vscode/launch.json`
- **Tareas:** `.vscode/tasks.json`

## 🎉 Próximos Pasos

1. ✅ Instala .NET 8.0 SDK (si aún no lo tienes)
2. ✅ Abre VS Code en este workspace
3. ✅ Acepta instalar las extensiones recomendadas
4. ✅ Ejecuta `dotnet restore`
5. ✅ Presiona F5 y selecciona "MIDASERP + Jupiter Auth (Multi-Debug)"
6. ✅ ¡Comienza a desarrollar! 🚀

---

**Creado:** 2025-12-09  
**Versión:** 1.0  
**Proyectos:** MIDASERP.WebAPI + Jupiter.API.AuthService
