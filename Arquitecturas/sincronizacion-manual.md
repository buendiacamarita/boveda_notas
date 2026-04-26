# Sincronización Manual de Datos

## Cambios Realizados

### 1. Nuevo Servicio de Sincronización (`syncService.ts`)
- **Ubicación**: `src/infraestructure/services/syncService.ts`
- **Función**: Maneja la sincronización completa de todos los datos desde la API
- **Características**:
  - Itera por todas las páginas de productos para obtener el dataset completo
  - Sincroniza productos, clientes y puntos de venta
  - Proporciona información de progreso en tiempo real
  - Manejo robusto de errores

### 2. Hook de Sincronización (`useSyncData.ts`)
- **Ubicación**: `src/domain/hooks/useSyncData.ts`
- **Función**: Hook personalizado para manejar el estado de la sincronización
- **Características**:
  - Estados de carga, progreso y resultado
  - Función para iniciar sincronización
  - Función para resetear el estado

### 3. Modal de Progreso (`SyncModal.tsx`)
- **Ubicación**: `src/infraestructure/components/shared/SyncModal.tsx`
- **Función**: Interfaz visual para mostrar el progreso de sincronización
- **Características**:
  - Barra de progreso animada
  - Mensajes informativos por tipo de datos
  - Resultados finales con contadores
  - Manejo de errores con mensajes claros

### 4. Opción en Menú Drawer
- **Ubicación**: `stacks/AuthenticatedStack.tsx`
- **Función**: Agrega opción "Sincronizar datos" en el menú lateral
- **Características**:
  - Icono de sincronización
  - Ejecuta sincronización completa al presionar
  - Muestra modal de progreso

### 5. Modificación del Repositorio de Artículos
- **Ubicación**: `src/infraestructure/repositories/articulos.repository.tsx`
- **Mejoras**:
  - Mejor manejo de transacciones
  - Uso de `Promise.all` para inserción más eficiente
  - Logging mejorado para depuración
  - Manejo de errores más robusto

### 6. Cambios en Products.tsx
- **Modificación Principal**: Todas las operaciones ahora usan SQLite en lugar de la API
- **Comportamiento nuevo**: 
  - Carga productos desde base de datos local con paginación eficiente
  - Búsquedas se realizan en SQLite con paginación
  - Scroll infinito funciona con datos locales
  - Muestra mensaje informativo si no hay datos sincronizados
  - **LA API YA NO SE USA** para operaciones de listado/búsqueda de productos

### 7. Mejoras en el Repositorio de Productos
- **Ubicación**: `src/infraestructure/repositories/products.repository.tsx`
- **Nuevas funciones**:
  - `getProductsPaginated()`: Paginación local
  - `getProductsBySearchPaginated()`: Búsqueda con paginación
  - `getTotalProductsCount()`: Conteo total de productos
  - `getTotalProductsCountBySearch()`: Conteo de productos filtrados

## Cómo Usar

1. **Primera vez**: 
   - Abrir el menú lateral (hamburguesa)
   - Seleccionar "Sincronizar datos"
   - Esperar a que termine la sincronización

2. **Actualizaciones posteriores**:
   - Repetir el proceso cuando se necesiten datos actualizados
   - La sincronización obtiene TODOS los productos de la API

## Ventajas del Nuevo Sistema

1. **Control Manual**: El usuario decide cuándo sincronizar
2. **Sincronización Completa**: Se obtienen todos los datos, no solo una página
3. **Feedback Visual**: Progreso en tiempo real con información detallada
4. **Mejor Rendimiento**: 
   - No se sincroniza automáticamente en cada carga
   - Todas las operaciones de listado/búsqueda son locales (SQLite)
   - Paginación eficiente sin llamadas a la API
5. **Manejo de Errores**: Mensajes claros sobre problemas de conectividad
6. **Transacciones Optimizadas**: Inserción más eficiente en SQLite
7. **Funcionamiento Offline**: Una vez sincronizado, todo funciona sin conexión

## Archivos Creados/Modificados

### Nuevos Archivos:
- `src/infraestructure/services/syncService.ts`
- `src/domain/hooks/useSyncData.ts`
- `src/infraestructure/components/shared/SyncModal.tsx`

### Archivos Modificados:
- `stacks/AuthenticatedStack.tsx`
- `src/presentation/screens/pos/Products.tsx` (reescrito completamente)
- `src/infraestructure/repositories/articulos.repository.tsx`
- `src/infraestructure/repositories/products.repository.tsx` (nuevas funciones)

## Consideraciones Técnicas

- La sincronización se ejecuta en transacciones SQLite para garantizar integridad
- Se usa paginación para manejar grandes volúmenes de datos
- El progreso se reporta en tiempo real para mejorar UX
- Manejo robusto de errores de red y base de datos
