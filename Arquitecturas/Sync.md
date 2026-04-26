

# Sincronización React Native (SQLite) - .NET Core (PostgreSQL) - Resumen Detallado

  

## Visión General

  

Solución para sincronización bidireccional entre una aplicación móvil en React Native con SQLite y una API backend en .NET Core con PostgreSQL, similar al enfoque de Dotmim.Sync.

  

## Arquitectura de la Solución

  

**Diagrama del Sistema**

  

```text

┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐

│ React Native │ ◄─── │ Sync Service │ ───► │ .NET Core │

│ (SQLite) │ │ (Middleware) │ │ (PostgreSQL) │

└─────────────────┘ └─────────────────┘ └─────────────────┘

│ │ │

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