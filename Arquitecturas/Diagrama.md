# Diagrama de Arquitectura - Ubicación GPS del Dispositivo

  

## Flujo de Datos Completo

  

```

┌─────────────────────────────────────────────────────────────────────┐

│ UI LAYER (Presentación) │

├─────────────────────────────────────────────────────────────────────┤

│ │

│ ┌──────────────────────┐ ┌─────────────────────────────┐ │

│ │ React Component │────────▶│ useDeviceLocation Hook │ │

│ │ (DeviceLocation │ │ - location │ │

│ │ Example.tsx) │ │ - isLoading │ │

│ │ │ │ - error │ │

│ │ - Button │ │ - getCurrentLocation() │ │

│ │ - Display coords │ │ - clearLocation() │ │

│ └──────────────────────┘ └─────────────────────────────┘ │

│ │ │

│ │ useContext │

│ ▼ │

│ ┌──────────────────────────────┐ │

│ │ DependencyProvider │ │

│ │ (Context) │ │

│ │ - getDeviceLocation │ │

│ └──────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

│

│ Dependency Injection

▼

┌─────────────────────────────────────────────────────────────────────┐

│ DOMAIN LAYER (Dominio) │

├─────────────────────────────────────────────────────────────────────┤

│ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ Use Case: GetDeviceLocation │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ execute() │ │ │

│ │ │ 1. Check permissions │ │ │

│ │ │ 2. Request permissions if needed │ │ │

│ │ │ 3. Get current location │ │ │

│ │ │ 4. Return DeviceLocation │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ └──────────────────────────────────────────────────────────────┘ │

│ │ │

│ │ Uses │

│ ▼ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ Repository Interface: IDeviceLocationRepository │ │

│ │ - getCurrentLocation(): Promise<DeviceLocation> │ │

│ │ - requestLocationPermission(): Promise<Permission> │ │

│ │ - checkLocationPermission(): Promise<Permission> │ │

│ └──────────────────────────────────────────────────────────────┘ │

│ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ Interfaces │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ DeviceLocation │ │ │

│ │ │ - latitude: number │ │ │

│ │ │ - longitude: number │ │ │

│ │ │ - altitude: number | null │ │ │

│ │ │ - accuracy: number | null │ │ │

│ │ │ - heading: number | null │ │ │

│ │ │ - speed: number | null │ │ │

│ │ │ - timestamp: number │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ DeviceLocationPermission │ │ │

│ │ │ - granted: boolean │ │ │

│ │ │ - canAskAgain: boolean │ │ │

│ │ │ - status: 'granted' | 'denied' | 'undetermined' │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ └──────────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

│

│ Implements

▼

┌─────────────────────────────────────────────────────────────────────┐

│ DATA LAYER (Datos) │

├─────────────────────────────────────────────────────────────────────┤

│ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ Repository Implementation: DeviceLocationRepository │ │

│ │ - Implements IDeviceLocationRepository │ │

│ │ - Delegates to DataSource │ │

│ └──────────────────────────────────────────────────────────────┘ │

│ │ │

│ │ Uses │

│ ▼ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ DataSource Interface: IDeviceLocationDataSource │ │

│ │ - getCurrentLocation(): Promise<DeviceLocation> │ │

│ │ - requestLocationPermission(): Promise<Permission> │ │

│ │ - checkLocationPermission(): Promise<Permission> │ │

│ └──────────────────────────────────────────────────────────────┘ │

│ │ │

│ │ Implements │

│ ▼ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ DataSource Implementation: DeviceLocationDataSource │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ getCurrentLocation() │ │ │

│ │ │ - Check permissions │ │ │

│ │ │ - Call expo-location API │ │ │

│ │ │ - Map response to DeviceLocation │ │ │

│ │ │ - Return coordinates │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ requestLocationPermission() │ │ │

│ │ │ - Call expo-location permission API │ │ │

│ │ │ - Return permission status │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ │ ┌────────────────────────────────────────────────────────┐ │ │

│ │ │ checkLocationPermission() │ │ │

│ │ │ - Call expo-location permission check │ │ │

│ │ │ - Return current permission status │ │ │

│ │ └────────────────────────────────────────────────────────┘ │ │

│ └──────────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

│

│ Uses

▼

┌─────────────────────────────────────────────────────────────────────┐

│ EXTERNAL DEPENDENCIES │

├─────────────────────────────────────────────────────────────────────┤

│ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ expo-location │ │

│ │ - getCurrentPositionAsync() │ │

│ │ - requestForegroundPermissionsAsync() │ │

│ │ - getForegroundPermissionsAsync() │ │

│ └──────────────────────────────────────────────────────────────┘ │

│ │ │

│ │ Accesses │

│ ▼ │

│ ┌──────────────────────────────────────────────────────────────┐ │

│ │ Native Device GPS │ │

│ │ - iOS Location Services │ │

│ │ - Android Location Services │ │

│ └──────────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────────┘

```

  

## Estructura de Archivos

  

```

src/

├── domain/ # Capa de Dominio

│ ├── interfaces/

│ │ └── DeviceLocation.ts # Interfaces de datos

│ ├── repositories/

│ │ └── IDeviceLocationRepository.ts # Contrato del repositorio

│ └── use-cases/

│ └── GetDeviceLocation.ts # Lógica de negocio

│

├── data/ # Capa de Datos

│ ├── data-sources/

│ │ └── device/

│ │ ├── IDeviceLocationDataSource.ts # Interface DataSource

│ │ └── DeviceLocationDataSource.ts # Implementación

│ └── repositories/

│ └── DeviceLocationRepository.ts # Implementación repositorio

│

└── ui/ # Capa de Presentación

├── hooks/

│ └── useDeviceLocation.tsx # Hook personalizado

├── components/

│ └── DeviceLocationExample.tsx # Componente de ejemplo

└── providers/

└── DependencyProvider.tsx # Inyección de dependencias

```

  

## Principios de Clean Architecture Aplicados

  

### 1. **Dependency Rule** ✅

- Las dependencias apuntan hacia adentro (hacia el dominio)

- El dominio no conoce la UI ni la Data layer

- La UI y Data dependen del dominio

  

### 2. **Separation of Concerns** ✅

- **Domain**: Lógica de negocio pura

- **Data**: Acceso a datos y APIs externas

- **UI**: Presentación y gestión de estado de UI

  

### 3. **Dependency Inversion** ✅

- Se usan interfaces (IDeviceLocationRepository, IDeviceLocationDataSource)

- Las implementaciones concretas dependen de abstracciones

- Facilita testing y cambio de implementaciones

  

### 4. **Single Responsibility** ✅

- Cada clase tiene una única responsabilidad

- Use Case: Lógica de negocio

- Repository: Mediación

- DataSource: Acceso a datos

- Hook: Estado de UI

  

## Ventajas de esta Arquitectura

  

1. **Testeable**: Cada capa se puede testear independientemente

2. **Mantenible**: Cambios en una capa no afectan a las demás

3. **Escalable**: Fácil agregar nuevas funcionalidades

4. **Reutilizable**: Los componentes son independientes

5. **Flexible**: Fácil cambiar implementaciones (ej: cambiar expo-location por otra librería)