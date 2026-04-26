# Sincronización React Native (SQLite) - .NET Core (PostgreSQL)

*Contenido extraído de: https://chat.deepseek.com/share/u75puy0pf4x2vmljn7*

---

Esta conversación compartida es generada por IA, solo para referencia.

recomiendame una solucion para una sincronizacion entre una app mobile hecha en react native con base de datos sqlite y una api .net core que usa postgres. La idea es realizar una sincronizacion como con dotmimsync

*(Pensó durante 68 segundos)*

Tenemos dos sistemas: una aplicación móvil en React Native con SQLite y una API en .NET Core con PostgreSQL. Queremos sincronizar datos entre ambos, similar a lo que hace Dotmim.Sync (una biblioteca de sincronización para .NET).

Dotmim.Sync está diseñado principalmente para entornos .NET, pero en el lado móvil tenemos React Native (JavaScript). Por lo tanto, necesitamos una solución que pueda trabajar en ambos lados.

**Opciones:**

1.  **Usar una solución similar a Dotmim.Sync pero con un enfoque multiplataforma**:
    *   Dotmim.Sync tiene un proveedor para SQLite y para PostgreSQL, pero está en .NET. Podríamos intentar usar Dotmim.Sync en el backend (API .NET Core) y en el móvil si usamos .NET MAUI, pero estamos en React Native.
    *   Dado que el móvil es React Native, necesitamos una biblioteca de sincronización que funcione con JavaScript/TypeScript:
        *   Podríamos usar una biblioteca como PouchDB (para el móvil) y CouchDB (en el servidor), pero nuestro backend ya está en PostgreSQL.
        *   Otra opción es WatermelonDB (para React Native) que tiene sincronización, pero requiere un backend específico (no es directamente con PostgreSQL).

2.  **Implementar una sincronización personalizada**:
    *   Diseñar un protocolo de sincronización (como un mecanismo de última modificación, marcas de tiempo, o un enfoque basado en operaciones).
    *   Esto implica crear endpoints en la API .NET Core para manejar la sincronización y lógica en la app móvil para enviar cambios y recibir actualizaciones.

3.  **Usar una herramienta o servicio de sincronización**:
    *   AWS AppSync (si estamos en AWS) puede sincronizar datos entre dispositivos y la nube, pero requiere un cambio en la infraestructura.
    *   Firebase Realtime Database o Cloud Firestore (si estamos en Google Cloud) también ofrecen sincronización, pero tendríamos que migrar a Firebase.

4.  **Dotmim.Sync para el backend y una implementación personalizada en React Native**:
    *   Dotmim.Sync expone endpoints web para la sincronización. Podríamos usar la misma lógica de sincronización de Dotmim.Sync en el backend y luego en React Native implementar el cliente que se comunica con esos endpoints.
    *   Examinando la opción 5:
        *   Dotmim.Sync tiene un servidor web (Dotmim.Sync.Server) que expone endpoints para la sincronización. Luego, tiene clientes para .NET (como Xamarin, Blazor, etc.). Pero no tiene un cliente oficial para JavaScript.
        *   Sin embargo, Dotmim.Sync usa un protocolo que podemos replicar. Otra alternativa es usar Dotmim.Sync solo en el backend y en el móvil usar una biblioteca que pueda sincronizar con un servidor que use el protocolo de Dotmim.Sync.
        *   Pero, ¿existe un cliente JavaScript para Dotmim.Sync? Hasta donde sé, no oficial. Pero podemos construir uno, ya que el protocolo está documentado.

5.  **GraphQL con suscripciones**:
    *   Podríamos usar GraphQL con suscripciones para la sincronización, pero esto no es exactamente igual a la sincronización bidireccional que hace Dotmim.Sync.

Dado que la pregunta es específica sobre una solución como Dotmim.Sync, y considerando que ya tenemos .NET Core en el backend, propongo:

*   **Backend (API .NET Core):**
    *   Usar Dotmim.Sync para PostgreSQL y exponer un controlador que herede de `Dotmim.Sync.Server.SyncController` (o similar) para manejar las solicitudes de sincronización.
*   **Frontend (React Native):**
    *   Implementar un cliente de sincronización que pueda comunicarse con el servidor Dotmim.Sync. Esto implica:
        *   Almacenar cambios locales en SQLite (con marcas de tiempo o versiones).
        *   Enviar estos cambios al servidor mediante una solicitud HTTP (con el formato que espera Dotmim.Sync).
        *   Recibir cambios del servidor y aplicarlos a la base de datos SQLite.
    *   Pero, ¿cómo implementar el cliente en React Native? Podemos usar una biblioteca para SQLite (como `react-native-sqlite-storage`) y luego construir manually la lógica de sincronización.
    *   Sin embargo, esto es complejo porque tendríamos que replicar la lógica de sincronización de Dotmim.Sync (detección de cambios, resolución de conflictos, etc.).
    *   Otra alternativa es usar `Dotmim.Sync.Client` en una capa nativa (por ejemplo, escribir un módulo nativo en .NET para React Native, o usar Xamarin y luego enlazar con React Native). Esto sería muy engorroso.

Por lo tanto, considerando la complejidad, quizás la mejor opción es:

*   Usar una solución de sincronización que tenga cliente para React Native y servidor para .NET Core/PostgreSQL.

**Revisemos algunas opciones:**

*   **Supabase:** Ofrece una base de datos PostgreSQL en tiempo real con suscripciones. Tiene un cliente JavaScript. Podríamos usar Supabase en el backend (PostgreSQL) y en el móvil (React Native). Pero esto implica cambiar el backend a Supabase o autoalojar Supabase.
*   **Hasura:** Con GraphQL y suscripciones en tiempo real. Tiene cliente para React Native. Pero de nuevo, requiere cambiar la API.
*   **Replicar la base de datos PostgreSQL con SQLite en el móvil** usando una herramienta como `replicator` (pero no conozco una herramienta que haga esto automáticamente).

Dado que la pregunta es sobre una solución como Dotmim.Sync, y si queremos mantener el backend actual, sugiero:

**Implementar un protocolo de sincronización simple:**

*   Cada tabla tiene una columna `LastModified` (timestamp) y `IsDeleted` (booleano) para soft delete.
*   El móvil guarda los registros localmente y marca los cambios (insert, update, delete) con una marca de tiempo y un estado (por ejemplo, `SyncStatus`).
*   Al sincronizar, el móvil envía los cambios (registros con `SyncStatus` = pendiente) al servidor.
*   El servidor procesa los cambios (aplica los insert, update, delete) y luego devuelve los cambios del servidor que son más recientes que la última sincronización del móvil.
*   El móvil aplica los cambios del servidor y actualiza la última fecha de sincronización.

Este es un enfoque común y no requiere bibliotecas complejas. Se puede implementar en ambos lados.

**Pasos concretos:**

*   **Backend (API .NET Core):**
    *   Crear un endpoint para recibir cambios del cliente (con una lista de registros a aplicar).
    *   Aplicar esos cambios a la base de datos PostgreSQL (teniendo cuidado con los conflictos, por ejemplo, usando la marca de tiempo para decidir cuál gana, o el servidor siempre gana).
    *   Luego, devolver los cambios del servidor desde la última sincronización del cliente.
*   **Frontend (React Native):**
    *   Usar SQLite y en cada tabla tener columnas para el control de sincronización: `LastModified`, `IsDeleted`, `SyncStatus` (por ejemplo: 'pending', 'synced').
    *   Al hacer operaciones CRUD, marcar el registro con `SyncStatus='pending'` y actualizar `LastModified`.
    *   Al sincronizar, enviar todos los registros con `SyncStatus='pending'` al servidor.
    *   Recibir la respuesta del servidor y aplicar los cambios a la base de datos local (actualizar o insertar registros, y eliminar los que se hayan eliminado en el servidor).
    *   Marcar los registros enviados como `SyncStatus='synced'`.

**Consideraciones:**

*   **Conflictos:** En este diseño simple, el servidor podría sobrescribir los cambios del cliente si el mismo registro fue modificado en ambos lados. Para manejar conflictos, se puede implementar una estrategia (por ejemplo, el cliente gana, el servidor gana, o mezcla).

Si necesitas una sincronización más robusta (con resolución de conflictos, sincronización parcial, etc.), entonces es mejor usar una biblioteca como Dotmim.Sync, pero tendrías que construir un cliente JavaScript para Dotmim.Sync.

Dado el esfuerzo, recomendaría:

*   Si la sincronización es simple (sin conflictos complejos, sin grandes volúmenes de datos), implementa el enfoque personalizado.
*   Si necesitas las características de Dotmim.Sync (como resolución de conflictos, filtros, etc.), entonces considera usar Dotmim.Sync en el backend y busca o construye un cliente para React Native.

Existe un proyecto no oficial que intenta portar Dotmim.Sync a JavaScript: `Dotmim.Sync.Js` (pero no está completo y no sé si funciona con React Native).

Otra opción es usar Moor (ahora llamado Drift) para Flutter, pero estamos en React Native.

**Conclusión:**

Si puedes permitirte una sincronización simple (sin manejo de conflictos complejo), te recomiendo implementar tu propio mecanismo. De lo contrario, considera cambiar la aplicación móvil a .NET MAUI o Xamarin para usar Dotmim.Sync directamente, o busca otra solución de sincronización que tenga soporte para React Native y .NET Core.

**Recomendación final:**

Dada la popularidad de React Native y .NET Core, y si no quieres cambiar de tecnología, implementa un mecanismo de sincronización personalizado. Comienza con algo simple y luego agrega complejidad según sea necesario.

Espero que esto te ayude a decidir.

Para una sincronización robusta entre React Native (SQLite) y .NET Core (PostgreSQL) similar a Dotmim.Sync, te recomiendo implementar una solución personalizada con el siguiente enfoque:

## Arquitectura Recomendada

### 1. Backend (.NET Core + PostgreSQL)

**Dotmim.Sync en el servidor:**

```csharp
// Instala el paquete
// Dotmim.Sync.SqlServer y Dotmim.Sync.Server
// Para PostgreSQL: Dotmim.Sync.PostgreSQL

services.AddSyncServer<PostgreSQLSyncProvider>(connectionString, 
    options => options.Tables = new string[] { "Table1", "Table2" });
```

**Exponer endpoints de sincronización:**

```csharp
[Route("api/[controller]")]
public class SyncController : Controller
{
    [HttpPost]
    public async Task<SyncResult> Sync([FromBody] SyncRequest request)
    {
        // Usar Dotmim.Sync para manejar la sincronización
    }
}
```

### 2. Cliente React Native (SQLite)

**Opción A: Usar WatermelonDB (Recomendado)**

WatermelonDB tiene sincronización incorporada y es ideal para React Native:

```javascript
// Instalación
yarn add @nozbe/watermelondb @nozbe/watermelondb-sync

// Configuración de sincronización
import { synchronize } from '@nozbe/watermelondb/sync'

async function sync() {
  await synchronize({
    database,
    pullChanges: async ({ lastPulledAt, schemaVersion, migration }) => {
      const response = await fetch(
        `https://api.tuapp.com/sync/pull?lastPulledAt=${lastPulledAt}`
      )
      return response.json()
    },
    pushChanges: async ({ changes, lastPulledAt }) => {
      await fetch('https://api.tuapp.com/sync/push', {
        method: 'POST',
        body: JSON.stringify({ changes, lastPulledAt })
      })
    }
  })
}
```

**Opción B: Implementación Personalizada**

```javascript
// syncService.js
class SyncService {
  async sync() {
    // 1. Obtener cambios locales
    const localChanges = await this.getLocalChanges();
    
    // 2. Enviar al servidor
    const response = await fetch(`${API_URL}/sync`, {
      method: 'POST',
      body: JSON.stringify({
        clientId: DEVICE_ID,
        changes: localChanges,
        lastSync: await this.getLastSyncTimestamp()
      })
    });
    
    // 3. Aplicar cambios remotos
    const serverChanges = await response.json();
    await this.applyRemoteChanges(serverChanges);
    
    // 4. Marcar como sincronizado
    await this.markAsSynced(localChanges);
  }
}
```

### 3. Protocolo de Sincronización

Implementa un sistema basado en marcas de tiempo:

```csharp
// Modelo para cambios
public class SyncChange
{
    public string Table { get; set; }
    public string Operation { get; set; } // INSERT, UPDATE, DELETE
    public Dictionary<string, object> Data { get; set; }
    public DateTimeOffset Timestamp { get; set; }
    public string ClientId { get; set; }
}

// Response de sincronización
public class SyncResponse
{
    public List<SyncChange> Changes { get; set; }
    public DateTimeOffset LastSync { get; set; }
    public bool HasMore { get; set; }
}
```

### 4. Manejo de Conflictos

Implementa estrategias según tu negocio:

```csharp
public class ConflictResolver
{
    public object Resolve(object serverValue, object clientValue)
    {
        // Estrategias posibles:
        // 1. Última modificación gana
        // 2. Servidor siempre gana
        // 3. Fusión inteligente
        // 4. Intervención manual
    }
}
```

### 5. Tablas de Control

En ambas bases de datos:

```sql
-- Tabla de metadatos de sincronización
CREATE TABLE sync_metadata (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    last_sync TIMESTAMP,
    sync_token VARCHAR(255)
);

-- Triggers para capturar cambios
CREATE TRIGGER track_changes
AFTER INSERT OR UPDATE OR DELETE ON your_table
FOR EACH ROW EXECUTE FUNCTION sync_changes_function();
```

### 6. Paquetes Recomendados

**Backend (.NET Core):**

```xml
<PackageReference Include="Dotmim.Sync.Core" Version="1.0.0" />
<PackageReference Include="Dotmim.Sync.PostgreSQL" Version="1.0.0" />
<PackageReference Include="Dotmim.Sync.Server" Version="1.0.0" />
```

**Frontend (React Native):**

```json
{
  "dependencies": {
    "@nozbe/watermelondb": "^0.27.0",
    "@nozbe/watermelondb-sync": "^0.27.0",
    "react-native-sqlite-storage": "^6.0.0",
    "axios": "^1.4.0"
  }
}
```

### 7. Consideraciones Importantes

*   **Sincronización diferencial:** Sincroniza solo cambios desde la última sincronización
*   **Batch processing:** Maneja grandes volúmenes en lotes
*   **Offline-first:** Diseña para operación sin conexión
*   **Resolución de conflictos:** Define políticas claras
*   **Seguridad:** Implementa autenticación y autorización
*   **Compresión:** Comprime datos para optimizar transferencia

### 8. Alternativas Completas

Si prefieres una solución más integrada:

*   **AWS AppSync + Amplify:** Sincronización automática con GraphQL
*   **Firebase Realtime Database:** Sincronización en tiempo real
*   **CouchDB + PouchDB:** Base de datos con sincronización incorporada
*   **RxDB:** Base de datos JavaScript con sincronización

#### Ejemplo de Implementación Simple:

```javascript
// React Native - Sync Manager
class SyncManager {
  constructor() {
    this.isSyncing = false;
    this.syncInterval = 30000; // 30 segundos
  }

  async startAutoSync() {
    setInterval(() => {
      if (this.hasNetworkConnection()) {
        this.performSync();
      }
    }, this.syncInterval);
  }

  async performSync() {
    try {
      // Obtener cambios pendientes
      const pendingChanges = await db.getPendingChanges();
      
      // Enviar al servidor
      const response = await api.sync({
        changes: pendingChanges,
        lastSync: await this.getLastSync()
      });

      // Aplicar cambios del servidor
      await db.applyChanges(response.serverChanges);
      
      // Marcar como sincronizado
      await db.markAsSynced(pendingChanges);
      
      // Actualizar última sincronización
      await this.updateLastSync(response.timestamp);
    } catch (error) {
      console.error('Sync failed:', error);
    }
  }
}
```

Esta arquitectura te dará una sincronización robusta y confiable similar a Dotmim.Sync, pero adaptada al stack React Native + .NET Core.

---

# Sincronización React Native (SQLite) - .NET Core (PostgreSQL) - Resumen Detallado

## Visión General

Solución para sincronización bidireccional entre una aplicación móvil en React Native con SQLite y una API backend en .NET Core con PostgreSQL, similar al enfoque de Dotmim.Sync.

## Arquitectura de la Solución

**Diagrama del Sistema**

```text
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   React Native  │ ◄─── │   Sync Service  │ ───► │   .NET Core     │
│     (SQLite)    │      │   (Middleware)  │      │   (PostgreSQL)  │
└─────────────────┘      └─────────────────┘      └─────────────────┘
         │                        │                        │
         └────────────────────────┴────────────────────────┘
                    Sincronización Bidireccional
```

### 1. Backend (.NET Core + PostgreSQL)

#### Configuración con Dotmim.Sync

**Instalación de Paquetes NuGet**

```xml
<PackageReference Include="Dotmim.Sync.Core" Version="1.0.0" />
<PackageReference Include="Dotmim.Sync.PostgreSQL" Version="1.0.0" />
<PackageReference Include="Dotmim.Sync.Web.Server" Version="1.0.0" />
```

**Configuración en `Program.cs`**

```csharp
using Dotmim.Sync;
using Dotmim.Sync.PostgreSQL;
using Dotmim.Sync.Web.Server;

var builder = WebApplication.CreateBuilder(args);

// Configuración del servicio de sincronización
builder.Services.AddSyncServer<PostgreSQLSyncProvider>(
    connectionString: builder.Configuration.GetConnectionString("PostgreSQL"),
    options: new string[] { "Customers", "Orders", "Products" }
);

var app = builder.Build();

// Mapeo del endpoint de sincronización
app.MapSyncServer("/api/sync");

app.Run();
```

#### Controlador de Sincronización Personalizado

```csharp
[ApiController]
[Route("api/[controller]")]
public class SyncController : ControllerBase
{
    private readonly WebServerAgent _syncAgent;

    public SyncController(WebServerAgent syncAgent)
    {
        _syncAgent = syncAgent;
    }

    [HttpPost("pull")]
    public async Task<IActionResult> PullChanges([FromBody] SyncRequest request)
    {
        try
        {
            var syncResult = await _syncAgent.HandleRequestAsync(
                HttpContext,
                request.ScopeName,
                default
            );
            
            return Ok(new SyncResponse
            {
                Changes = syncResult.Changes,
                LastSync = DateTimeOffset.UtcNow,
                Conflicts = syncResult.Conflicts
            });
        }
        catch (Exception ex)
        {
            return StatusCode(500, new { error = ex.Message });
        }
    }

    [HttpPost("push")]
    public async Task<IActionResult> PushChanges([FromBody] SyncRequest request)
    {
        var syncResult = await _syncAgent.HandleRequestAsync(
            HttpContext,
            request.ScopeName,
            default
        );
        
        return Ok(new { success = true, timestamp = DateTimeOffset.UtcNow });
    }
}

public class SyncRequest
{
    public string ScopeName { get; set; }
    public SyncState SyncState { get; set; }
    public string ClientId { get; set; }
}

public class SyncResponse
{
    public SyncState Changes { get; set; }
    public DateTimeOffset LastSync { get; set; }
    public List<SyncConflict> Conflicts { get; set; }
}
```

### 2. Cliente React Native (SQLite)

#### Opción A: WatermelonDB (Recomendada)

**Instalación**

```bash
npm install @nozbe/watermelondb @nozbe/watermelondb-sync
# o
yarn add @nozbe/watermelondb @nozbe/watermelondb-sync
```

**Configuración de la Base de Datos**

```javascript
// database/schema.js
import { appSchema, tableSchema } from '@nozbe/watermelondb'

export default appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'customers',
      columns: [
        { name: 'server_id', type: 'string', isOptional: true },
        { name: 'name', type: 'string' },
        { name: 'email', type: 'string' },
        { name: 'last_sync', type: 'number' },
        { name: 'is_dirty', type: 'boolean' },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' }
      ]
    }),
    tableSchema({
      name: 'sync_metadata',
      columns: [
        { name: 'last_pulled_at', type: 'number' },
        { name: 'last_pulled_version', type: 'number' }
      ]
    })
  ]
})
```

**Servicio de Sincronización**

```javascript
// services/SyncService.js
import { synchronize } from '@nozbe/watermelondb/sync'
import { database } from '../database'
import NetInfo from '@react-native-community/netinfo'
import AsyncStorage from '@react-native-async-storage/async-storage'

class SyncService {
  constructor() {
    this.isSyncing = false
    this.syncInterval = null
    this.lastSyncKey = 'last_sync_timestamp'
  }

  async initialize() {
    // Configurar listener de conexión
    NetInfo.addEventListener(state => {
      if (state.isConnected && !this.isSyncing) {
        this.performSync()
      }
    })

    // Sincronización periódica cada 5 minutos
    this.syncInterval = setInterval(() => {
      if (!this.isSyncing) {
        this.performSync()
      }
    }, 5 * 60 * 1000)
  }

  async performSync() {
    if (this.isSyncing) return

    this.isSyncing = true

    try {
      await synchronize({
        database,
        pullChanges: async ({ lastPulledAt, schemaVersion, migration }) => {
          const response = await fetch(
            `${API_URL}/api/sync/pull`,
            {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${await this.getToken()}`
              },
              body: JSON.stringify({
                lastPulledAt,
                schemaVersion,
                clientId: await this.getClientId()
              })
            }
          )

          if (!response.ok) {
            throw new Error(`Pull failed: ${response.status}`)
          }

          const { changes, timestamp } = await response.json()
          
          // Guardar timestamp de última sincronización
          await AsyncStorage.setItem(
            this.lastSyncKey,
            timestamp.toString()
          )

          return { changes, timestamp }
        },
        pushChanges: async ({ changes, lastPulledAt }) => {
          const response = await fetch(
            `${API_URL}/api/sync/push`,
            {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${await this.getToken()}`
              },
              body: JSON.stringify({
                changes,
                lastPulledAt,
                clientId: await this.getClientId()
              })
            }
          )

          if (!response.ok) {
            throw new Error(`Push failed: ${response.status}`)
          }
        },
        sendCreatedAsUpdated: true,
        log: __DEV__ ? console.log : null,
        migrationsEnabledAtVersion: 1
      })

      console.log('Sincronización completada exitosamente')
    } catch (error) {
      console.error('Error en sincronización:', error)
      // Implementar reintentos con backoff exponencial
      await this.retrySync()
    } finally {
      this.isSyncing = false
    }
  }

  async getClientId() {
    let clientId = await AsyncStorage.getItem('client_id')
    if (!clientId) {
      clientId = DeviceInfo.getUniqueId()
      await AsyncStorage.setItem('client_id', clientId)
    }
    return clientId
  }

  async getToken() {
    // Implementar lógica para obtener token de autenticación
    return await AsyncStorage.getItem('auth_token')
  }

  async retrySync() {
    // Lógica de reintento con backoff exponencial
    const maxRetries = 3
    let retryCount = 0
    let delay = 1000

    while (retryCount < maxRetries) {
      await new Promise(resolve => setTimeout(resolve, delay))
      
      try {
        await this.performSync()
        break
      } catch (error) {
        retryCount++
        delay *= 2
        console.log(`Reintento ${retryCount} después de ${delay}ms`)
      }
    }
  }

  cleanup() {
    if (this.syncInterval) {
      clearInterval(this.syncInterval)
    }
  }
}

export default new SyncService()
```

#### Opción B: Implementación Personalizada con SQLite

**Instalación de Dependencias**

```bash
npm install react-native-sqlite-storage @react-native-async-storage/async-storage axios
```

**Modelo de Sincronización**

```javascript
// models/SyncModel.js
export class SyncModel {
  static async getPendingChanges() {
    // Obtener registros marcados como sucios (is_dirty = true)
    return await db.executeSql(
      `SELECT * FROM customers WHERE is_dirty = 1 
       UNION ALL
       SELECT * FROM orders WHERE is_dirty = 1`
    )
  }

  static async markAsSynced(table, ids) {
    await db.executeSql(
      `UPDATE ${table} SET is_dirty = 0 WHERE id IN (${ids.join(',')})`
    )
  }

  static async getLastSyncTimestamp() {
    const result = await db.executeSql(
      'SELECT timestamp FROM sync_metadata ORDER BY timestamp DESC LIMIT 1'
    )
    return result.rows.length > 0 ? result.rows.item(0).timestamp : null
  }

  static async updateSyncTimestamp(timestamp) {
    await db.executeSql(
      'INSERT OR REPLACE INTO sync_metadata (id, timestamp) VALUES (1, ?)',
      [timestamp]
    )
  }
}
```

**Controlador de Sincronización**

```javascript
// controllers/SyncController.js
import axios from 'axios'
import SyncModel from '../models/SyncModel'

class SyncController {
  constructor() {
    this.batchSize = 100
    this.apiClient = axios.create({
      baseURL: API_URL,
      timeout: 30000
    })
  }

  async sync() {
    const lastSync = await SyncModel.getLastSyncTimestamp()
    const pendingChanges = await SyncModel.getPendingChanges()

    if (pendingChanges.length === 0 && lastSync) {
      // Solo pull si no hay cambios locales
      await this.pullChanges(lastSync)
    } else {
      // Push primero, luego pull
      await this.pushChanges(pendingChanges)
      await this.pullChanges(lastSync)
    }
  }

  async pushChanges(changes) {
    const batches = this.createBatches(changes, this.batchSize)
    
    for (const batch of batches) {
      try {
        await this.apiClient.post('/api/sync/push', {
          changes: batch,
          clientId: await this.getClientId(),
          timestamp: Date.now()
        })

        await SyncModel.markAsSynced(
          batch.map(item => ({ table: item.table, id: item.id }))
        )
      } catch (error) {
        console.error('Error en push batch:', error)
        throw error
      }
    }
  }

  async pullChanges(lastSync) {
    try {
      const response = await this.apiClient.post('/api/sync/pull', {
        lastSync,
        clientId: await this.getClientId()
      })

      const { changes, conflicts, timestamp } = response.data
      
      await this.applyRemoteChanges(changes)
      await this.resolveConflicts(conflicts)
      await SyncModel.updateSyncTimestamp(timestamp)
    } catch (error) {
      console.error('Error en pull:', error)
      throw error
    }
  }

  async applyRemoteChanges(changes) {
    for (const change of changes) {
      switch (change.operation) {
        case 'INSERT':
        case 'UPDATE':
          await db.executeSql(
            `INSERT OR REPLACE INTO ${change.table} 
             (${Object.keys(change.data).join(',')}) 
             VALUES (${Object.keys(change.data).map(() => '?').join(',')})`,
            Object.values(change.data)
          )
          break
        case 'DELETE':
          await db.executeSql(
            `DELETE FROM ${change.table} WHERE id = ?`,
            [change.id]
          )
          break
      }
    }
  }

  createBatches(array, size) {
    const batches = []
    for (let i = 0; i < array.length; i += size) {
      batches.push(array.slice(i, i + size))
    }
    return batches
  }
}
```

### 3. Protocolo de Sincronización

**Estructura de Mensajes**

```json
{
  "sync_request": {
    "client_id": "device-uuid-12345",
    "last_sync": "2024-01-15T10:30:00Z",
    "changes": [
      {
        "table": "customers",
        "operation": "UPDATE",
        "data": {
          "id": 1,
          "name": "Nuevo Nombre",
          "email": "nuevo@email.com",
          "updated_at": "2024-01-15T10:25:00Z"
        },
        "timestamp": "2024-01-15T10:25:30Z"
      }
    ]
  },
  "sync_response": {
    "changes": [
      {
        "table": "orders",
        "operation": "INSERT",
        "data": {
          "id": 100,
          "customer_id": 1,
          "total": 150.75
        }
      }
    ],
    "conflicts": [],
    "last_sync": "2024-01-15T10:35:00Z",
    "has_more": false
  }
}
```

### 4. Manejo de Conflictos

**Estrategias de Resolución**

```csharp
// Backend: ConflictResolver.cs
public class ConflictResolver
{
    public SyncRow ResolveConflict(SyncRow serverRow, SyncRow clientRow)
    {
        var serverTime = serverRow["updated_at"] as DateTime?;
        var clientTime = clientRow["updated_at"] as DateTime?;
        
        // Estrategia: Última modificación gana
        if (clientTime > serverTime)
        {
            return clientRow;
        }
        
        // Estrategia: Servidor gana para datos críticos
        if (IsCriticalData(serverRow.TableName))
        {
            return serverRow;
        }
        
        // Estrategia: Fusión para campos específicos
        return MergeRows(serverRow, clientRow);
    }
    
    private SyncRow MergeRows(SyncRow serverRow, SyncRow clientRow)
    {
        var mergedRow = serverRow.Clone();
        
        foreach (var column in clientRow.Table.Columns)
        {
            var clientValue = clientRow[column.ColumnName];
            var serverValue = serverRow[column.ColumnName];
            
            // Mantener el valor más reciente si existe
            if (clientValue != null && 
                (serverValue == null || 
                 clientRow["updated_at"] > serverRow["updated_at"]))
            {
                mergedRow[column.ColumnName] = clientValue;
            }
        }
        
        return mergedRow;
    }
}
```

### 5. Tablas de Control en Base de Datos

**PostgreSQL (Backend)**

```sql
-- Tabla para metadatos de sincronización
CREATE TABLE sync_metadata (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    last_sync_timestamp TIMESTAMPTZ,
    sync_token VARCHAR(255),
    client_id VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tabla para seguimiento de cambios
CREATE TABLE sync_changes (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    record_id INTEGER NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    change_data JSONB,
    client_id VARCHAR(255),
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    is_applied BOOLEAN DEFAULT FALSE,
    conflict_resolved BOOLEAN DEFAULT FALSE
);

-- Función de trigger para capturar cambios
CREATE OR REPLACE FUNCTION track_changes_function()
RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'DELETE') THEN
        INSERT INTO sync_changes (table_name, record_id, operation, change_data)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', row_to_json(OLD));
    ELSIF (TG_OP = 'UPDATE') THEN
        INSERT INTO sync_changes (table_name, record_id, operation, change_data)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', row_to_json(NEW));
    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO sync_changes (table_name, record_id, operation, change_data)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', row_to_json(NEW));
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Aplicar trigger a tablas específicas
CREATE TRIGGER track_customer_changes
AFTER INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW EXECUTE FUNCTION track_changes_function();
```

**SQLite (Frontend)**

```sql
-- Tabla de metadatos local
CREATE TABLE IF NOT EXISTS sync_metadata (
    id INTEGER PRIMARY KEY,
    last_sync_timestamp INTEGER,
    sync_token TEXT,
    client_id TEXT
);

-- Columna adicional en tablas de datos
ALTER TABLE customers ADD COLUMN is_dirty BOOLEAN DEFAULT 0;
ALTER TABLE customers ADD COLUMN sync_timestamp INTEGER;
ALTER TABLE customers ADD COLUMN client_id TEXT;

-- Índices para optimización
CREATE INDEX idx_customers_dirty ON customers(is_dirty);
CREATE INDEX idx_customers_sync ON customers(sync_timestamp);
```

### 6. Optimizaciones y Mejores Prácticas

**Compresión de Datos**

```javascript
// Frontend: Compresión antes de enviar
import pako from 'pako'

async function sendCompressedData(data) {
  const jsonString = JSON.stringify(data)
  const compressed = pako.gzip(jsonString)
  
  await fetch(`${API_URL}/api/sync`, {
    method: 'POST',
    headers: {
      'Content-Encoding': 'gzip',
      'Content-Type': 'application/json'
    },
    body: compressed
  })
}

// Backend: Descompresión
[HttpPost("sync")]
public async Task<IActionResult> Sync([FromBody] byte[] compressedData)
{
    var jsonString = Decompress(compressedData);
    var request = JsonSerializer.Deserialize<SyncRequest>(jsonString);
    // Procesar solicitud
}
```

**Manejo Offline**

```javascript
// OfflineQueue.js
class OfflineQueue {
  constructor() {
    this.queue = []
    this.isOnline = true
    this.maxQueueSize = 1000
  }

  async addToQueue(operation) {
    if (this.queue.length >= this.maxQueueSize) {
      // Eliminar operaciones más antiguas
      this.queue.shift()
    }
    
    this.queue.push({
      ...operation,
      timestamp: Date.now(),
      id: uuid.v4()
    })
    
    await this.saveQueue()
    
    if (this.isOnline) {
      await this.processQueue()
    }
  }

  async processQueue() {
    while (this.queue.length > 0 && this.isOnline) {
      const operation = this.queue[0]
      
      try {
        await this.executeOperation(operation)
        this.queue.shift()
        await this.saveQueue()
      } catch (error) {
        console.error('Error procesando operación:', error)
        break
      }
    }
  }

  async saveQueue() {
    await AsyncStorage.setItem(
      'offline_queue',
      JSON.stringify(this.queue)
    )
  }
}
```

### 7. Monitoreo y Logging

**Backend - Middleware de Logging**

```csharp
public class SyncLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<SyncLoggingMiddleware> _logger;

    public SyncLoggingMiddleware(RequestDelegate next, ILogger<SyncLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var clientId = context.Request.Headers["Client-Id"].FirstOrDefault();
        
        try
        {
            await _next(context);
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Sync completed - Client: {ClientId}, Duration: {Duration}ms, Status: {StatusCode}",
                clientId, stopwatch.ElapsedMilliseconds, context.Response.StatusCode
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Sync failed - Client: {ClientId}, Duration: {Duration}ms",
                clientId, stopwatch.ElapsedMilliseconds
            );
            throw;
        }
    }
}
```

**Frontend - Estadísticas de Sincronización**

```javascript
// SyncStats.js
class SyncStats {
  constructor() {
    this.stats = {
      totalSyncs: 0,
      successfulSyncs: 0,
      failedSyncs: 0,
      totalDataTransferred: 0,
      averageSyncDuration: 0,
      lastSyncTimestamp: null
    }
  }

  recordSync(startTime, endTime, dataSize, success) {
    const duration = endTime - startTime
    
    this.stats.totalSyncs++
    
    if (success) {
      this.stats.successfulSyncs++
      this.stats.totalDataTransferred += dataSize
      this.stats.averageSyncDuration = 
        (this.stats.averageSyncDuration * (this.stats.successfulSyncs - 1) + duration) / 
        this.stats.successfulSyncs
      this.stats.lastSyncTimestamp = new Date()
    } else {
      this.stats.failedSyncs++
    }

    this.saveStats()
  }

  async saveStats() {
    await AsyncStorage.setItem('sync_stats', JSON.stringify(this.stats))
  }

  getStats() {
    return {
      ...this.stats,
      successRate: this.stats.totalSyncs > 0 ? 
        (this.stats.successfulSyncs / this.stats.totalSyncs) * 100 : 0
    }
  }
}
```

### 8. Pruebas

**Tests de Sincronización**

```javascript
// __tests__/SyncService.test.js
describe('SyncService', () => {
  let syncService
  let mockDatabase
  let mockApi

  beforeEach(() => {
    mockDatabase = {
      executeSql: jest.fn(),
      getPendingChanges: jest.fn()
    }
    
    mockApi = {
      post: jest.fn()
    }
    
    syncService = new SyncService(mockDatabase, mockApi)
  })

  test('debe sincronizar cambios pendientes', async () => {
    mockDatabase.getPendingChanges.mockResolvedValue([
      { id: 1, table: 'customers', data: { name: 'Test' } }
    ])
    
    mockApi.post.mockResolvedValue({
      data: { success: true, timestamp: '2024-01-15T10:30:00Z' }
    })
    
    await syncService.sync()
    
    expect(mockApi.post).toHaveBeenCalledWith('/api/sync/push', expect.any(Object))
    expect(mockDatabase.markAsSynced).toHaveBeenCalled()
  })

  test('debe manejar errores de red', async () => {
    mockApi.post.mockRejectedValue(new Error('Network error'))
    
    await expect(syncService.sync()).rejects.toThrow('Network error')
  })
})
```

### 9. Configuración de Deployment

**Docker Compose para Backend**

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "5000:5000"
    environment:
      - ConnectionStrings__PostgreSQL=Host=postgres;Database=sync_db;Username=user;Password=pass
      - ASPNETCORE_ENVIRONMENT=Production
    depends_on:
      - postgres
  
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=sync_db
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

### 10. Scripts de Mantenimiento

**Limpieza de Datos Antiguos**

```sql
-- PostgreSQL: Limpiar cambios antiguos
CREATE OR REPLACE PROCEDURE cleanup_old_sync_data()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Eliminar cambios aplicados hace más de 30 días
    DELETE FROM sync_changes 
    WHERE timestamp < NOW() - INTERVAL '30 days'
    AND is_applied = TRUE;
    
    -- Eliminar metadatos de clientes inactivos
    DELETE FROM sync_metadata 
    WHERE last_sync_timestamp < NOW() - INTERVAL '90 days';
    
    COMMIT;
END;
$$;

-- Programar tarea (usando pg_cron)
SELECT cron.schedule('cleanup-sync-data', '0 2 * * *', 
    'CALL cleanup_old_sync_data()');
```

## Conclusión

Esta solución proporciona una sincronización robusta y escalable entre React Native y .NET Core, con características como:

✅ Sincronización bidireccional

✅ Manejo offline

✅ Resolución de conflictos

✅ Optimización de red

✅ Monitoreo y logging

✅ Pruebas automatizadas

✅ Mantenimiento automático

La implementación puede ajustarse según los requisitos específicos de cada proyecto, pero esta arquitectura sirve como base sólida para la mayoría de los casos de uso de sincronización.
