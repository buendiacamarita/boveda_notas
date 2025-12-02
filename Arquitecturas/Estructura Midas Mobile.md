# Estructura del Proyecto MIDAS.CRM.Mobile

Este documento describe la estructura de directorios y la arquitectura del proyecto `MIDAS.CRM.Mobile`. El proyecto está construido utilizando **React Native** con **Expo** y sigue una arquitectura basada en **Clean Architecture** para separar responsabilidades y facilitar la escalabilidad y el mantenimiento.

## Directorios Principales

### Raíz del Proyecto
- **`.expo/`**: Archivos de configuración y caché generados por Expo.
- **`app/`**: Contiene la estructura de rutas y navegación de la aplicación (Expo Router).
- **`assets/`**: Recursos estáticos como imágenes, fuentes e iconos.
- **`src/`**: Código fuente principal de la aplicación, organizado por capas.
- **`App.tsx` / `index.js`**: Puntos de entrada de la aplicación (gestionados por Expo Router en `app/`).
- **`app.json`**: Configuración de Expo.
- **`package.json`**: Dependencias y scripts del proyecto.
- **`tsconfig.json`**: Configuración de TypeScript.

---

### `app/` (Expo Router)
Este directorio maneja la navegación basada en archivos.
- **`(app)/`**: Grupo de rutas protegidas o principales de la aplicación (e.g., dashboard, formularios).
- **`Login.tsx`**: Pantalla de inicio de sesión.
- **`_layout.tsx`**: Layout raíz que define la estructura global de navegación (e.g., Stack, Drawer).
- **`index.tsx`**: Punto de entrada inicial (redirección o pantalla de carga).

---

### `src/` (Código Fuente)
El código se organiza siguiendo los principios de Clean Architecture.

#### `src/core/`
Contiene utilidades y configuraciones base del núcleo de la aplicación.

#### `src/data/` (Capa de Datos)
Responsable de la obtención y persistencia de datos.
- **`data-sources/`**: Implementaciones concretas para obtener datos.
  - **`remote/`**: Fuentes de datos remotas (API REST, servicios externos).
  - **`device/`**: Fuentes de datos del dispositivo (GPS, sensores, almacenamiento local).
  - **`local/`**: Base de datos local (SQLite).
- **`repositories/`**: Implementaciones de los repositorios definidos en la capa de dominio. Actúan como mediadores entre las fuentes de datos y el dominio.

#### `src/domain/` (Capa de Dominio)
Contiene la lógica de negocio y las definiciones del sistema. Es independiente de la UI y de frameworks externos.
- **`interfaces/`**: Definiciones de interfaces generales.
- **`models/`**: Modelos de datos y entidades del dominio (e.g., `User`, `Visit`).
- **`repositories/`**: Interfaces que definen los contratos de los repositorios.
- **`use-cases/`**: Casos de uso que encapsulan la lógica de negocio específica (e.g., `LoginUser`, `GetVisits`).

#### `src/ui/` (Capa de Presentación)
Contiene todo lo relacionado con la interfaz de usuario.
- **`components/`**: Componentes reutilizables de la UI (botones, inputs, tarjetas).
- **`hooks/`**: Custom hooks de React para lógica de UI y conexión con stores/casos de uso.
- **`mappers/`**: Transformadores de datos entre la capa de UI y el dominio.
- **`providers/`**: Context Providers de React (e.g., AuthProvider, ThemeProvider).
- **`stores/`**: Manejo de estado global (e.g., Zustand).
- **`styles/`**: Archivos de estilos globales, temas y constantes de diseño.

#### `src/types/`
Definiciones de tipos de TypeScript compartidos en toda la aplicación.

---

## Flujo de Datos Típico

1. **UI (`src/ui`)**: Un componente o hook solicita una acción (e.g., iniciar sesión).
2. **Use Case (`src/domain/use-cases`)**: La UI invoca un caso de uso.
3. **Repository Interface (`src/domain/repositories`)**: El caso de uso interactúa con una interfaz de repositorio abstracta.
4. **Repository Implementation (`src/data/repositories`)**: La implementación concreta del repositorio decide de dónde obtener los datos (API, caché, BD).
5. **Data Source (`src/data/data-sources`)**: Realiza la petición real a la API o base de datos.

---

## Funcionalidad de Ubicación GPS del Dispositivo

### Descripción
El sistema incluye un circuito completo para obtener las coordenadas GPS del dispositivo siguiendo la arquitectura Clean Architecture. Esta funcionalidad gestiona automáticamente los permisos de ubicación y proporciona acceso a las coordenadas del dispositivo.

### Archivos Implementados

#### Domain Layer (Dominio)
- **`src/domain/interfaces/DeviceLocation.ts`**: Interfaces para la ubicación del dispositivo y permisos.
- **`src/domain/repositories/IDeviceLocationRepository.ts`**: Contrato del repositorio de ubicación del dispositivo.
- **`src/domain/use-cases/GetDeviceLocation.ts`**: Caso de uso para obtener coordenadas GPS.

#### Data Layer (Datos)
- **`src/data/data-sources/device/IDeviceLocationDataSource.ts`**: Interface del DataSource.
- **`src/data/data-sources/device/DeviceLocationDataSource.ts`**: Implementación usando expo-location.
- **`src/data/repositories/DeviceLocationRepository.ts`**: Implementación del repositorio.

#### UI Layer (Presentación)
- **`src/ui/hooks/useDeviceLocation.tsx`**: Hook personalizado para usar en componentes React.
- **`src/ui/components/DeviceLocationExample.tsx`**: Componente de ejemplo.

### Ejemplo de Uso

```typescript
import { useDeviceLocation } from '@/src/ui/hooks/useDeviceLocation';

const MyComponent = () => {
    const { location, isLoading, error, getCurrentLocation } = useDeviceLocation();

    const handleGetLocation = async () => {
        const coords = await getCurrentLocation();
        if (coords) {
            console.log('Latitud:', coords.latitude);
            console.log('Longitud:', coords.longitude);
        }
    };

    return (
        <Button onPress={handleGetLocation} disabled={isLoading}>
            Obtener Ubicación
        </Button>
    );
};
```

### Dependencias
- **expo-location**: Librería de Expo para acceder al GPS del dispositivo.
-