# Sistema de Integraciones - Documentación Técnica

## Índice
1. [Visión General](#visión-general)
2. [Estructura de Proyectos](#estructura-de-proyectos)
3. [Arquitectura Core (Abstract)](#arquitectura-core-abstract)
4. [Modelo de Datos (DTOs)](#modelo-de-datos-dtos)
5. [Servicios Principales](#servicios-principales)
6. [Transformers (Mapeo BLL ↔ DTO)](#transformers-mapeo-bll--dto)
7. [Pipeline de Importación](#pipeline-de-importación)
8. [Pipeline de Exportación](#pipeline-de-exportación)
9. [Pipeline de Eliminación](#pipeline-de-eliminación)
10. [Preview de Datos](#preview-de-datos)
11. [Clientes de Integración (Clients)](#clientes-de-integración-clients)
12. [Factories de Clientes](#factories-de-clientes)
13. [Instanciador (Resolución Dinámica)](#instanciador-resolución-dinámica)
14. [Automatización de Tareas](#automatización-de-tareas)
15. [Cómo Agregar una Nueva Integración](#cómo-agregar-una-nueva-integración)
16. [Cómo Agregar un Nuevo Tipo de Dato](#cómo-agregar-un-nuevo-tipo-de-dato)
17. [Diagrama de Flujo](#diagrama-de-flujo)

---

## Visión General

El sistema de integraciones permite a MIDAS sincronizar datos (artículos, comprobantes, personas, cotizaciones, etc.) con plataformas externas como **Tienda Nube**, **Mercado Libre** y **Midas API**. 

La arquitectura está diseñada con los siguientes principios:
- **Agnóstica al proveedor**: el core no sabe nada de Tienda Nube, MercadoLibre, etc.
- **Basada en genéricos**: cada operación se parametriza con `<DTOType, BLLType>`.
- **Auto-descubrimiento**: los transformers, pre-importers, post-importers y selectors se descubren automáticamente por reflexión al escanear el assembly.
- **Extensible**: para agregar un nuevo proveedor o tipo de dato, basta con implementar las interfaces correspondientes.

**Target**: .NET Framework 4.8.1, C# 7.3

---

## Estructura de Proyectos

```
Jupiter.Integrations.Core/
├── Jupiter.Integrations.Core.Abstract/    ← Core: interfaces, servicios, transformers, DTOs
│   ├── Common/                            ← Interfaces de clientes (IImportable, IExportable, IDelete)
│   ├── Models/                            ← DTOs genéricos (ArticuloDTO, ComprobanteDTO, etc.)
│   ├── Mappers/                           ← Mappers BLL → DTO para exportación
│   └── Services/                          ← Servicios de Import/Export/Delete/Preview
│       ├── Transformers/                  ← Mapeo bidireccional DTO ↔ BLL
│       ├── PreImport/                     ← Lógica previa a importar (dependencias)
│       ├── PostImport/                    ← Lógica posterior a importar
│       ├── ExportSelector/                ← Selección de items a exportar
│       └── ImportSelector/                ← Parámetros de filtro para importar
│
├── Jupiter.Integrations.Core.Automation/  ← Ejecución automática de tareas
│   └── AutoTaskRunner/                    ← Runners para import/export automáticos
│
Jupiter.Integrations.Clients/
├── Jupiter.Integrations.Clients.TiendaNube/   ← Cliente Tienda Nube
├── Jupiter.Integrations.Clients.MercadoLibre/ ← Cliente Mercado Libre
├── Jupiter.Integrations.Clients.MidasAPI/     ← Cliente Midas API (cotizaciones)
└── Jupiter.Integrations.Clients.Abaco/        ← Cliente Abaco (vacío por ahora)
```

---

## Arquitectura Core (Abstract)

### Clase Base: `AbstractIntegrationService`
**Archivo**: `Services/AbstractIntegrationService.cs`

Clase abstracta de la que heredan todos los servicios (Import, Export, Delete, Preview). 

**Responsabilidad principal**: mantener un diccionario estático (`Factory`) que mapea pares `(Type DTOType, Type BLLType)` → `IGenericWrapper` (transformer). Este diccionario se puebla automáticamente por reflexión al instanciar cualquier servicio por primera vez.

```
Factory[( ArticuloDTO, Articulo )]         → ArticuloTransformer
Factory[( ComprobanteConArticulosDTO, ComprobanteFacturable )] → ComprobanteTransformer
Factory[( ClienteDTO, Persona )]           → ClienteTransformer
...etc
```

**Método clave**:
- `GetTransformer<TModelBase, TBll>(servicio)` — busca el transformer apropiado y le asigna la cuenta de integración.

---

## Modelo de Datos (DTOs)

### `BaseModelDTO`
**Archivo**: `Models/BaseModelDTO.cs`

Clase base de todos los DTOs de integración:

| Propiedad             | Tipo     | Descripción                                           |
|-----------------------|----------|-------------------------------------------------------|
| `LocalId`             | `long`   | ID del elemento en la base de datos local de MIDAS    |
| `IntegrationServiceId`| `long`   | ID de la `CuentaIntegracion` asociada                 |
| `RemoteId`            | `string` | ID del elemento en el sistema remoto (ej: ID de TiendaNube) |

### DTOs Concretos Disponibles

| DTO                        | Namespace                        | Uso                          |
|----------------------------|----------------------------------|------------------------------|
| `ArticuloDTO`              | `Models.Articulo`                | Productos/artículos          |
| `ComprobanteConArticulosDTO`| `Models.Comprobante`            | Órdenes/ventas con detalles  |
| `ClienteDTO`               | `Models.Personas`                | Clientes                     |
| `VendedorDTO`              | `Models.Personas`                | Vendedores                   |
| `FormaPagoDTO`             | `Models.FormaPago`               | Formas de pago               |
| `PagoCobroDTO`             | `Models.Pagos`                   | Pagos individuales           |
| `DetalleArticuloDTO`       | `Models.DetalleArticulo`         | Líneas de detalle de comprobante |
| `MonedaDTO`                | `Models.Moneda`                  | Monedas                      |
| `CotizacionDTO`            | `Models.Moneda`                  | Cotizaciones de moneda       |
| `CajaDTO`                  | `Models.Caja`                    | Cajas                        |
| `ImagenDTO`                | `Models.Common`                  | Imágenes (base64)            |

### `IntegrationServiceDTO`
**Archivo**: `Models/IntegrationServiceDTO.cs`

Configuración que se pasa a los clientes al instanciarlos:

| Propiedad           | Descripción                                    |
|---------------------|------------------------------------------------|
| `IdIntegrationService` | ID de la cuenta                             |
| `BaseEndPoint`      | URL base del servicio (ej: `https://api.tiendanube.com`) |
| `EndPoint`          | Ruta relativa del recurso (ej: `products`)     |
| `CustomParammeters` | Parámetros de autenticación (token, store_id, etc.) |
| `AditionalSettings` | Configuraciones extra por motor                |

### `Instanciador` (DTOs)
**Archivo**: `Models/Instanciador.cs`

Utilidad que resuelve un nombre de DTO (string) a su `Type` real. Se usa en automatización para resolver tipos dinámicamente desde configuración de texto plano.

---

## Servicios Principales

Todos los servicios heredan de `AbstractIntegrationService` y tienen acceso al método `GetTransformer<>()`.

### 1. `ImportService`
**Archivo**: `Services/ImportService.cs`

Servicio de importación. Trae datos de un sistema remoto, los transforma a BLL y los guarda en la BD local.

**Factories auto-descubiertas** (por reflexión):
- `PreImporterFactory` — acciones previas a importar (ej: importar dependencias)
- `PostImporterFactory` — acciones posteriores a importar (ej: imprimir comprobante)
- `ImportSelectorsFactory` — determina los parámetros de consulta al importar

**Métodos principales**:

| Método | Descripción |
|--------|-------------|
| `PullFromClientAsync<ImportModel, TipoBll>()` | Trae items del remoto, filtra los ya importados, e importa los nuevos |
| `ImportData<ImportModel, TipoBll>()` | Importa una colección de DTOs. Ejecuta pre-import → transform → guardar → asociar → post-import |

### 2. `ExportService`
**Archivo**: `Services/ExportService.cs`

Servicio de exportación. Toma datos locales (BLL), los transforma a DTO y los envía al remoto.

**Factory auto-descubierta**:
- `ExportSelectorFactory` — determina qué items BLL se deben exportar

**Métodos principales**:

| Método | Descripción |
|--------|-------------|
| `PushItemsAsync<TBaseModel, TipoBLL>(... items ...)` | Exporta una lista específica de items |
| `PushItemsAsync<TBaseModel, TipoBLL>(... sin items ...)` | Usa el `ExportSelector` para determinar automáticamente qué exportar |

### 3. `DeleteService`
**Archivo**: `Services/DeleteService.cs`

Elimina items del sistema remoto y des-asocia la relación local. **NO elimina el objeto local**.

### 4. `PreviewImport`
**Archivo**: `Services/PreviewImport.cs`

Dado un conjunto de DTOs, busca si ya tienen un elemento local asociado y les asigna el `LocalId`. Se usa para mostrar al usuario una vista previa antes de importar.

---

## Transformers (Mapeo BLL ↔ DTO)

### Interfaz: `IGenericTransformer<ImportModel, BLLType>`
**Archivo**: `Services/Transformers/IGenericTransformer.cs`

Define el contrato bidireccional de transformación:

| Método | Dirección | Descripción |
|--------|-----------|-------------|
| `Import(connection, item)` | DTO → BLL | Transforma un DTO a un objeto BLL (crea o busca existente) |
| `Import(connection, items)` | DTO[] → BLL[] | Versión en lote |
| `Export(item)` | BLL → DTO | Transforma un objeto BLL a DTO para enviar al remoto |
| `Export(items)` | BLL[] → DTO[] | Versión en lote |
| `Preview(connection, item)` | DTO → DTO (enriquecido) | Matchea el DTO con datos locales (asigna `LocalId`) |
| `SetServicioIntegracion(servicio)` | — | Asigna la cuenta de integración activa |

### Clase Base: `GenericTransformerBase<ImportModel, BllType>`
**Archivo**: `Services/Transformers/GenericTransformerBase.cs`

Implementa la parte no-genérica (`IGenericWrapper`) haciendo cast entre `object` y los tipos concretos. Los transformers concretos heredan de esta clase.

### Wrapper: `IGenericWrapper`
**Archivo**: `Services/Transformers/IGenericWrapper.cs`

Interfaz no-genérica que permite almacenar cualquier transformer en el diccionario estático. Expone:
- `ImportModelType` → `typeof(DTOType)`
- `ImportType` → `typeof(BLLType)`

### Transformers Implementados

| Transformer | DTO | BLL | Archivo |
|-------------|-----|-----|---------|
| `ArticuloTransformer` | `ArticuloDTO` | `Articulo` | `Transformers/Articulo/` |
| `ComprobanteVentaOnlineTransformer` | `ComprobanteConArticulosDTO` | `ComprobanteVentaOnline` | `Transformers/Comprobante/` |
| `ComprobanteConArticulosTransformer` | `ComprobanteConArticulosDTO` | `ComprobanteConArticulos` | `Transformers/Comprobante/` |
| `ClienteTransformer` | `ClienteDTO` | `Persona` | `Transformers/Persona/` |
| `FormaPagoTransformer` | `FormaPagoDTO` | `FormaDePago` | `Transformers/FormaPago/` |
| `DetalleComprobanteTransformer` | `DetalleArticuloDTO` | (detalle) | `Transformers/DetalleArticulo/` |
| `MonedaTransformer` | `MonedaDTO` | (moneda) | `Transformers/Moneda/` |
| `CotizacionTransformer` | `CotizacionDTO` | (cotización) | `Transformers/Cotizacion/` |
| `CajaTransformer` | `CajaDTO` | (caja) | `Transformers/Caja/` |
| `PagoCobroTransformer` | `PagoCobroDTO` | (pago) | `Transformers/PagoCobro/` |

### Ejemplo: `ArticuloTransformer`

```
Import: ArticuloDTO → busca si ya existe vía IntegrationHandler.FindByRemoteIdAsync
         → si existe: lo devuelve
         → si no existe y el motor permite crear: crea nuevo Articulo BLL
         → si no permite crear: lanza excepción

Export: Articulo BLL → ArticuloDTO usando ArticuloToExportDTO mapper
         → incluye precio de lista, existencias, imágenes en base64, etc.

Preview: ArticuloDTO → busca la ID local y la asigna al DTO
```

---

## Pipeline de Importación

### Flujo completo de `ImportData`:

```
1. Agrupar items por IntegrationServiceId
   └─ Para cada cuenta de integración:
      2. Obtener transformer para (DTOType, BLLType)
      3. Obtener items ya asociados (remoteId → localItem)
      4. Filtrar: solo items nuevos (no asociados aún)
      5. Para cada item nuevo:
         ├─ 5a. PreImport (si existe): importar dependencias recursivamente
         │       Ejemplo para Comprobante:
         │       ├── Importar Cliente (si no existe)
         │       ├── Importar FormaPago faltantes
         │       └── Importar Articulos faltantes
         ├─ 5b. Transform: DTO → BLL (transformer.Import)
         ├─ 5c. Guardar en BD (elemento.Guardar()) dentro de transacción
         ├─ 5d. Asociar: IntegrationHandler.UpsertAssociatedItemAsync
         │       (vincula BLL.Id ↔ RemoteId para esa cuenta)
         └─ 5e. PostImport (si existe): lógica posterior
```

### Pre-Importers

| Pre-Importer | Para DTO | Qué hace |
|--------------|----------|----------|
| `ComprobanteVentaOnlinePreImporter` | `ComprobanteConArticulosDTO` | Antes de importar un comprobante, importa recursivamente: Cliente, Formas de Pago y Artículos faltantes |

Interfaz: `IGenericPreImporter<TImport>` con método `PreImportAsync(connection, importItem, servicio, progress)`.

### Post-Importers

Interfaz: `IGenericPostImporter<TImport>` con método `PostImportAsync(connection, importItem, servicio, progress)`.

Actualmente el `ComprobanteVentaOnlinePostImport` está **comentado** (estaba pensado para imprimir el comprobante automáticamente después de importar).

### Import Selectors

| Selector | Para BLL | Qué hace |
|----------|----------|----------|
| `ComprobanteImportSelector` | `ComprobanteFacturable` | Genera `QueryParameters` con la fecha del último comprobante importado para pedir solo los nuevos |

Interfaz: `IGenericImportSelector<BllType>` con método `GetImportParameters(connection, cuenta)`.

---

## Pipeline de Exportación

### Flujo completo de `PushItemsAsync`:

```
1. Obtener transformer para (DTOType, BLLType)
2. Para cada item BLL:
   ├─ 2a. Transform: BLL → DTO (transformer.Export)
   ├─ 2b. Enviar al remoto: exportClient.PushItemAsync(dto)
   │       → retorna SuccessOperationResult con Message = RemoteId generada
   └─ 2c. Asociar: IntegrationHandler.UpsertAssociatedItemAsync
          (vincula BLL.Id ↔ nueva RemoteId)
```

### Export Selectors

| Selector | Para BLL | Qué hace |
|----------|----------|----------|
| `ArticuloExportSelector` | `Articulo` | Determina qué artículos exportar (actualmente la lógica está comentada, devuelve colección vacía) |

Interfaz: `IExportSelector<BllType>` con método `GetItemsToExport(connection, cuenta)`.

---

## Pipeline de Eliminación

### Flujo de `DeleteItemsAsync`:

```
1. Obtener transformer
2. Para cada item BLL:
   ├─ 2a. Transform: BLL → DTO (para obtener RemoteId)
   ├─ 2b. Enviar delete al remoto: deleteClient.DropItemAsync(dto)
   └─ 2c. Des-asociar: IntegrationHandler.DeAssociateAsync
```

---

## Preview de Datos

`PreviewImport.PreviewData<ImportModel, TipoBll>()` toma una colección de DTOs y, para cada uno, busca si tiene un elemento local asociado. Devuelve los DTOs enriquecidos con `LocalId` (> 0 si ya existe en MIDAS, 0 si es nuevo). Se usa para UI de confirmación antes de importar.

---

## Clientes de Integración (Clients)

Los clientes son las clases que hablan con las APIs externas. Implementan una o más de estas interfaces:

### Interfaces de Cliente

| Interfaz | Método | Dirección |
|----------|--------|-----------|
| `IImportableClient<T>` | `PullItemsAsync()` / `PullItemsAsync(QueryParameters)` | Remoto → MIDAS |
| `IExportableClient<T>` | `PushItemAsync(T item)` | MIDAS → Remoto |
| `IDeleteClient<T>` | `DropItemAsync(T item)` | Eliminar en remoto |
| `ICustomActionClient<T>` | `PerformAction(items, actionName, endpoint)` | Acción personalizada |

Todas exponen `long IdIntegrationService` para identificar la cuenta asociada.

### Tienda Nube

| Cliente | DTOs | Interfaces | Archivo |
|---------|------|------------|---------|
| `ProductoTiendaNubeClient` | `ArticuloDTO` | Import, Export, Delete | `Clients/ProductoTiendaNubeClient.cs` |
| `ComprobanteTiendaNubeClient` | `ComprobanteConArticulosDTO` | Import | `Clients/ComprobanteTiendaNubeClient.cs` |

Heredan de `BaseTiendaNubeClient<TRequest, TResponse>` que maneja:
- `HttpClient` con headers custom (`Midas-Token`, `Midas-User-Agent`, `Midas-Store-Id`)
- CRUD genérico: `GetAllAsync`, `GetByIdAsync`, `CreateAsync`, `UpdateAsync`, `DeleteAsync`
- Serialización con Newtonsoft.Json

**Doble mapeo**: DTO genérico → Modelo específico TiendaNube (y viceversa) usando mappers en `Mappers/Articulos/Export.cs` y `Import.cs`.

### Mercado Libre

| Cliente | DTOs | Interfaces |
|---------|------|------------|
| `ArticuloClient` | `ArticuloDTO` | Import, Export |
| `PaymentClient` | (pagos ML) | — |

Base: `MercadoLibreClient` con autenticación OAuth (refresh token).

### Midas API

| Cliente | DTOs | Interfaces |
|---------|------|------------|
| `CotizacionClient` | `CotizacionDTO` | Import |

---

## Factories de Clientes

Cada proveedor tiene un **factory estático** que mapea `(DTOType, InterfaceType)` → instancia de cliente.

| Factory | Método | Proveedores |
|---------|--------|-------------|
| `TiendaNubeClientFactory` | `InferirTiendaNubeClient<DTOType, TClient>(config)` | Tienda Nube |
| `MercadoLibreClientFactory` | `InferirMLClient<DTOType, TClient>(config)` | Mercado Libre |
| `MidasWebApiFactory` | `InferirMidasApiClient<DTOType, TClient>(config)` | Midas API |

Ejemplo de registro en `TiendaNubeClientFactory`:
```
(ArticuloDTO, IImportableClient<>) → ProductoTiendaNubeClient
(ArticuloDTO, IExportableClient<>) → ProductoTiendaNubeClient
(ComprobanteConArticulosDTO, IImportableClient<>) → ComprobanteTiendaNubeClient
```

---

## Instanciador (Resolución Dinámica)

**Archivo**: `Jupiter.Integrations.Core.Automation/Instanciador.cs`

Switch central que, dada una `CuentaIntegracion`, resuelve qué factory usar según el código del motor:

| Código del Motor | Factory |
|------------------|---------|
| `"tienda-nube"` / `"tienda-nube-local"` | `TiendaNubeClientFactory` |
| `"mercado-libre"` / `"mercado-libre-local"` | `MercadoLibreClientFactory` |
| `"midas-api"` / `"midas-api-local"` | `MidasWebApiFactory` |

**Métodos**:
- `InferirImportableClient<DTOType, BLLType>(cuenta)` → `IImportableClient<DTOType>`
- `InferirExportableClient<DTOType, BLLType>(cuenta)` → `IExportableClient<DTOType>`
- `InferirDeleteClient<DTOType, BLLType>(cuenta)` → `IDeleteClient<DTOType>`

Cada método construye un `IntegrationServiceDTO` con la configuración de la cuenta y se lo pasa al factory correspondiente.

---

## Automatización de Tareas

**Proyecto**: `Jupiter.Integrations.Core.Automation`

### `AutoTaskHandler`
**Archivo**: `AutoTaskRunner/AutoTaskHandler.cs`

Punto de entrada. Recibe un string de comando y ejecuta la tarea:

```csharp
await AutoTaskHandler.ExecuteTask("Import 15 ComprobanteFacturable ComprobanteConArticulosDTO");
```

### Formato del Comando
```
<TipoTarea> <IdCuentaIntegracion> <TipoBLL> <TipoDTO>
```

### `AutoTaskIntegrationType` (enum)
```
None = 0, Import = 1, Export = 2, Delete = 3, CustomAction = 4
```

### `AutoTaskRunnerFactory`
Parsea el primer token del comando para determinar el tipo y crea el runner apropiado:
- `Import` → `AutoImportTask`
- (Export, Delete, CustomAction → por implementar)

### `AutoImportTask`
**Archivo**: `AutoTaskRunner/Runners/AutoImportTask.cs`

1. Parsea los parámetros: `IdCuenta`, `TipoBLL` (vía `BLL.Instanciador`), `TipoDTO` (vía `Models.Instanciador`)
2. Usa reflexión para invocar `Instanciador.InferirImportableClient<DTOType, BLLType>(cuenta)`
3. Usa reflexión para invocar `ImportService.PullFromClientAsync<DTOType, BLLType>(connection, client, cuenta)`

> **Nota**: `AutoExportTask` existe pero tiene `NotImplementedException`.

---

## Cómo Agregar una Nueva Integración

Ejemplo: agregar un proveedor "WooCommerce".

### Paso 1: Crear el proyecto del cliente
```
Jupiter.Integrations.Clients/
└── Jupiter.Integrations.Clients.WooCommerce/
    ├── Clients/
    │   ├── BaseWooCommerceClient.cs          ← HTTP client base
    │   └── ProductoWooCommerceClient.cs      ← Implementa IImportableClient<ArticuloDTO>, IExportableClient<ArticuloDTO>
    ├── Models/
    │   └── WooCommerceProductResponse.cs     ← Modelo de respuesta de la API
    ├── Mappers/
    │   ├── Import.cs                         ← WooCommerceResponse → ArticuloDTO
    │   └── Export.cs                         ← ArticuloDTO → WooCommerceRequest
    └── WooCommerceClientFactory.cs           ← Factory: (DTOType, InterfaceType) → cliente
```

### Paso 2: Registrar en el `Instanciador`
En `Jupiter.Integrations.Core.Automation/Instanciador.cs`, agregar un case al switch:

```csharp
case "woocommerce":
    return WooCommerceClientFactory.InferirClient<DTOType, IImportableClient<DTOType>>(config);
```

### Paso 3: Configurar la cuenta en BD
Crear una `CuentaIntegracion` con `MotorIntegracion.Codigo = "woocommerce"` y los endpoints/credenciales correspondientes.

> **No hace falta tocar el core**: los transformers, pre-importers, etc. ya existen para `ArticuloDTO` → `Articulo`. Solo se necesita el cliente nuevo.

---

## Cómo Agregar un Nuevo Tipo de Dato

Ejemplo: agregar soporte para importar/exportar "Categorías".

### Paso 1: Crear el DTO
```csharp
// Models/Categoria/CategoriaDTO.cs
public class CategoriaDTO : BaseModelDTO
{
    public string Nombre { get; set; }
    public long? CategoriaPadreId { get; set; }
}
```

### Paso 2: Crear el Transformer
```csharp
// Services/Transformers/Categoria/CategoriaTransformer.cs
internal class CategoriaTransformer : GenericTransformerBase<CategoriaDTO, Jupiter.BLL.Articulos.Categoria>
{
    // Implementar Import, Export, Preview
}
```

> Al heredar de `GenericTransformerBase`, se auto-registra en el `Factory` de `AbstractIntegrationService` al primer uso (descubrimiento por reflexión).

### Paso 3: (Opcional) Crear PreImporter, ExportSelector, ImportSelector
Solo si el tipo necesita lógica extra (dependencias, filtrado, etc.).

### Paso 4: Agregar al cliente del proveedor
En el factory del proveedor, registrar el nuevo par:
```csharp
{ (typeof(CategoriaDTO), typeof(IImportableClient<>)), config => new CategoriaTiendaNubeClient(config) }
```

---

## Diagrama de Flujo

### Importación
```
┌─────────────┐     PullItemsAsync()     ┌──────────────────┐
│  API Remota  │ ◄────────────────────── │  IImportableClient│
│ (TiendaNube) │ ────────────────────── │  (ProductoClient)  │
└─────────────┘   List<TNResponse>       └────────┬─────────┘
                                                   │ List<ArticuloDTO>
                                                   ▼
                                          ┌────────────────┐
                                          │  ImportService  │
                                          │  .ImportData()  │
                                          └───────┬────────┘
                                                  │
                              ┌────────────────────┼────────────────────┐
                              ▼                    ▼                    ▼
                     ┌──────────────┐    ┌─────────────────┐   ┌──────────────┐
                     │ PreImporter  │    │   Transformer    │   │ PostImporter │
                     │ (dependencias│    │ DTO → BLL        │   │ (acciones    │
                     │  recursivas) │    │ .Import()        │   │  posteriores)│
                     └──────────────┘    └────────┬────────┘   └──────────────┘
                                                  │
                                                  ▼
                                         ┌─────────────────┐
                                         │  elemento.Guardar│ (en transacción)
                                         └────────┬────────┘
                                                  │
                                                  ▼
                                         ┌──────────────────────┐
                                         │ IntegrationHandler   │
                                         │ .UpsertAssociatedItem│
                                         │ (BLL.Id ↔ RemoteId)  │
                                         └──────────────────────┘
```

### Exportación
```
┌──────────────┐                          ┌──────────────────┐
│ ExportSelector│  GetItemsToExport()     │  Transformer     │
│ (qué exportar)│ ──────────────────────► │  BLL → DTO       │
└──────────────┘   List<BLL>              │  .Export()        │
                                          └────────┬─────────┘
                                                   │ ArticuloDTO
                                                   ▼
                                          ┌──────────────────┐     PushItemAsync()     ┌─────────────┐
                                          │  ExportService    │ ──────────────────────► │  API Remota  │
                                          │  .PushItemsAsync()│ ◄──────────────────── │ (TiendaNube) │
                                          └──────────────────┘   RemoteId generada     └─────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────────┐
                                          │ IntegrationHandler   │
                                          │ .UpsertAssociatedItem│
                                          └──────────────────────┘
```

---

## Conceptos Clave para el Desarrollador

### `IntegrationHandler` (BLL)
Clase de `Jupiter.BLL.Integraciones` que gestiona las asociaciones `(ElementoBLL, CuentaIntegracion, RemoteId)`. Métodos importantes:
- `UpsertAssociatedItemAsync` — crea o actualiza la asociación
- `FindByRemoteIdAsync<BLL, TKey>` — busca un BLL por su RemoteId
- `GetAllRelatedWithRemoteId<BLL, TKey>` — lista todos los elementos asociados
- `GetAllRelatedIdByType<BLL, TKey>` — lista solo los RemoteIds
- `DeAssociateAsync` — elimina la asociación
- `ListarElementosAsociadosAsync` — lista los BLL asociados a una cuenta

### `CuentaIntegracion` (BLL)
Representa una cuenta configurada con un motor de integración. Propiedades clave:
- `MotorIntegracion` — define endpoints, permisos, código del motor
- `AccessConfig` — credenciales (token, store_id, etc.)
- `MotorIntegracion.Codigo` — string que identifica el proveedor ("tienda-nube", "mercado-libre", etc.)

### Reflexión y Auto-Descubrimiento
El sistema usa reflexión para descubrir implementaciones automáticamente al escanear el assembly de `AbstractIntegrationService`. **Cualquier clase que implemente las interfaces correspondientes y esté en ese assembly se registra automáticamente**. No hace falta registrar nada manualmente en ningún contenedor DI.

### OperationResult (Framework)
Patrón de resultado usado en todo el sistema:
- `SuccessOperationResult` — éxito, `.Message` puede contener la RemoteId generada
- `FailureOperationResult` — error, `.Message` y `.Exception`
- `CancelOperationResult` — operación cancelada (ej: no hay items para importar)
