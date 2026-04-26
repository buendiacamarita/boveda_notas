# Resumen de la Estructura del Proyecto MIDASERP

Este documento detalla la arquitectura y organización de carpetas del proyecto MIDASERP.

## 1. Capa de Aplicación (Core)
**Ruta:** `Core/MIDASERP.API.Application`

Esta capa contiene la lógica de negocio, las interfaces y la implementación del patrón CQRS (Command Query Responsibility Segregation).

*   **Features/:** Organizado por entidad (ej. `Articulo`, `Caja`, `Persona`). Cada carpeta contiene:
    *   **Command/:** Contiene los comandos para operaciones de escritura (Create, Update, Delete). Comúnmente agrupados en archivos como `{Entidad}Command.cs`.
    *   **Queries/:** Contiene las consultas para operaciones de lectura. Comúnmente agrupados en archivos como `{Entidad}Query.cs`.
*   **DTOS/:** Objetos de transferencia de datos utilizados para comunicar las distintas capas.
*   **Mappings/:** Configuraciones de AutoMapper para la conversión entre modelos y DTOs.
*   **Interfaces/:** Definiciones de contratos para servicios y repositorios.

## 2. Capa de Negocio e Infraestructura (BLL)
**Ruta:** `Infraestructure/MIDASERP.API.BLL`

En este proyecto, los servicios actúan fundamentalmente como repositorios.

*   **Services/:** Organizado por módulos. Los servicios (ej. `ArticuloService.cs`) heredan de `GenericRepository<TEntity, MidasContext>`.
    *   **Verificación:** Se ha confirmado que estos servicios interactúan directamente con el `MidasContext` y encapsulan la lógica de acceso a datos, funcionando como la capa de persistencia lógica.

## 3. Capa de Acceso a Datos (Persistence / Models)
**Ruta:** `Infraestructure/MIDASERP.DataAccess`

Esta capa contiene la definición del modelo de datos y el contexto de Entity Framework.

*   **Models/:** Contiene las entidades que representan las tablas de la base de datos (ej. `Articulo.cs`, `Alicuota.cs`).
*   **MidasContext.cs:** El DbContext principal que gestiona la conexión y el mapeo con la base de datos PostgreSQL.

## 4. Controladores (API)
**Ruta:** `API/MIDASERP.WebAPI/Controllers`

Esta capa contiene los puntos de entrada de la API que exponen la funcionalidad al exterior.

*   **v1/:** Los controladores están versionados y se encuentran generalmente en esta subcarpeta. Cada controlador (ej. `ArticuloController.cs`) suele inyectar el `IMediator` para enviar comandos y consultas a la capa `Core`.

## 5. Otros Componentes
*   **Jupiter.APIGateway:** Puerta de enlace para las peticiones a la API.
*   **API/:** Diferentes servicios WebAPI (AuthService, CurrencyQuotation, etc.).
*   **openspec/:** Directorio para la gestión de especificaciones y cambios (OpenSpec).
