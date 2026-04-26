# 🐛 Debug vs Run - Diferencias Importantes

## ⚠️ IMPORTANTE: Breakpoints solo funcionan con Debug

### ❌ `dotnet run` - SIN Breakpoints

Cuando ejecutas con `dotnet run` (desde terminal o script):

```bash
# Estos comandos NO permiten breakpoints
dotnet run
./start-services.sh
```

**Características:**
- ❌ **NO** puedes usar breakpoints
- ❌ **NO** puedes inspeccionar variables
- ❌ **NO** puedes hacer step-through
- ✅ Más rápido para iniciar
- ✅ Útil para pruebas rápidas
- ✅ Ver logs en consola

**Cuándo usar:**
- Pruebas rápidas sin necesidad de debug
- Ver logs de la aplicación
- Ejecutar en producción
- Testing de integración simple

### ✅ VS Code Debug (F5) - CON Breakpoints

Cuando ejecutas desde VS Code con F5:

```
VS Code > Run and Debug > F5
Selecciona: "MIDASERP + Jupiter Auth (Multi-Debug)"
```

**Características:**
- ✅ **SÍ** puedes usar breakpoints
- ✅ **SÍ** puedes inspeccionar variables
- ✅ **SÍ** puedes hacer step-through
- ✅ **SÍ** puedes evaluar expresiones
- ✅ **SÍ** puedes ver call stack
- ✅ Hot reload automático
- ⚠️ Un poco más lento para iniciar

**Cuándo usar:**
- Desarrollo activo
- Debugging de problemas
- Entender flujo de código
- Inspeccionar estado de variables
- **SIEMPRE que necesites breakpoints**

## 🔍 Comparación Detallada

| Característica | `dotnet run` | VS Code Debug (F5) |
|----------------|--------------|-------------------|
| **Breakpoints** | ❌ NO | ✅ SÍ |
| **Step Through** | ❌ NO | ✅ SÍ |
| **Inspeccionar Variables** | ❌ NO | ✅ SÍ |
| **Watch Expressions** | ❌ NO | ✅ SÍ |
| **Call Stack** | ❌ NO | ✅ SÍ |
| **Velocidad de inicio** | ✅ Rápido | ⚠️ Normal |
| **Ver logs** | ✅ SÍ | ✅ SÍ |
| **Hot Reload** | ⚠️ Con `dotnet watch` | ✅ Automático |
| **Multi-proyecto** | ⚠️ Manual (múltiples terminales) | ✅ Un solo F5 |

## 🎯 Cómo Usar Breakpoints Correctamente

### 1. Configuración Correcta (Ya está hecha)

Los archivos en `.vscode/launch.json` ya están configurados para adjuntar el debugger.

### 2. Establecer Breakpoints

```
1. Abre el archivo .cs donde quieres debuggear
2. Haz click en el margen izquierdo (junto al número de línea)
3. Aparecerá un punto rojo (breakpoint)
```

### 3. Iniciar Debug

```
Opción A: Presiona F5
Opción B: Run and Debug > Selecciona configuración > Play
```

### 4. Cuando el código llegue al breakpoint

```
✅ La ejecución se pausará
✅ Puedes ver valores de variables
✅ Puedes ejecutar paso a paso (F10, F11)
✅ Puedes evaluar expresiones en Debug Console
```

## 🚀 Flujo de Trabajo Recomendado

### Para Desarrollo con Debugging

```bash
# 1. Abre VS Code
code .

# 2. Establece breakpoints en tu código
# (Click en margen izquierdo de los archivos .cs)

# 3. Presiona F5
# Selecciona: "MIDASERP + Jupiter Auth (Multi-Debug)"

# 4. Los servicios inician CON debugger adjunto
# ✅ Breakpoints funcionarán
```

### Para Pruebas Rápidas (Sin Debug)

```bash
# Opción 1: Script
./start-services.sh

# Opción 2: Manual
dotnet run --project API/MIDASERP.WebAPI/MIDASERP.WebAPI.csproj

# ❌ Breakpoints NO funcionarán
# ✅ Útil solo para ver logs o probar endpoints
```

## 🔧 Alternativa: Adjuntar Debugger a Proceso Existente

Si ya tienes un proceso corriendo con `dotnet run`, puedes adjuntar el debugger:

### Paso 1: Inicia el proceso
```bash
dotnet run --project API/MIDASERP.WebAPI/MIDASERP.WebAPI.csproj
```

### Paso 2: Adjunta debugger en VS Code
```
1. Ctrl+Shift+P
2. Escribe: ".NET: Attach to Process"
3. Selecciona el proceso "MIDASERP.WebAPI"
```

### Paso 3: Ahora los breakpoints funcionarán
```
✅ El debugger está adjunto
✅ Breakpoints activados
```

**Nota:** Este método es más complicado. Es mejor usar F5 directamente.

## 📊 Diagrama de Flujo

```
┌─────────────────────────────────────┐
│ ¿Necesitas usar breakpoints?        │
└─────────────────────────────────────┘
                │
        ┌───────┴────────┐
        │                │
       SÍ               NO
        │                │
        ▼                ▼
┌──────────────┐  ┌──────────────┐
│ USA F5       │  │ USA dotnet   │
│ (VS Code     │  │ run o script │
│  Debug)      │  │              │
│              │  │              │
│ ✅ Breakpoints│  │ ❌ Sin debug  │
│ ✅ Variables  │  │ ✅ Más rápido │
│ ✅ Step-thru  │  │ ✅ Solo logs  │
└──────────────┘  └──────────────┘
```

## 💡 Consejos

### ✅ HACER
- Usar F5 para desarrollo diario
- Establecer breakpoints antes de iniciar
- Usar "MIDASERP + Jupiter Auth (Multi-Debug)" para ambos proyectos
- Inspeccionar variables en Debug Console
- Usar Step Over (F10) y Step Into (F11)

### ❌ NO HACER
- Usar `dotnet run` si necesitas breakpoints
- Esperar que breakpoints funcionen sin debugger adjunto
- Ejecutar múltiples instancias manualmente (usa la config multi-debug)

## 🎓 Atajos de Teclado Útiles

| Acción | Atajo | Descripción |
|--------|-------|-------------|
| **Iniciar Debug** | `F5` | Inicia con debugger adjunto |
| **Detener Debug** | `Shift+F5` | Detiene la sesión de debug |
| **Restart** | `Ctrl+Shift+F5` | Reinicia el debug |
| **Continue** | `F5` | Continúa hasta siguiente breakpoint |
| **Step Over** | `F10` | Ejecuta línea actual |
| **Step Into** | `F11` | Entra en función |
| **Step Out** | `Shift+F11` | Sale de función actual |
| **Toggle Breakpoint** | `F9` | Activa/desactiva breakpoint |

## 📚 Resumen

### Para usar breakpoints en VS Code:

1. ✅ **DEBES** usar la configuración de debug (F5)
2. ✅ **NO** puedes usar `dotnet run` directamente
3. ✅ La configuración en `.vscode/launch.json` ya está lista
4. ✅ Presiona F5 y selecciona "MIDASERP + Jupiter Auth (Multi-Debug)"
5. ✅ Establece breakpoints y debuggea normalmente

### El script `start-services.sh`:

- ❌ **NO** permite breakpoints
- ✅ **SÍ** es útil para pruebas rápidas sin debug
- ✅ **SÍ** muestra logs en consola
- ⚠️ Úsalo solo cuando NO necesites debuggear

---

**Conclusión:** Si necesitas breakpoints, **SIEMPRE** usa F5 en VS Code con las configuraciones que creé. El script `start-services.sh` es solo para ejecución rápida sin debugging.
