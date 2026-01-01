Aquí tienes el resumen técnico de los endpoints implementados para la integración con React Native. Estos endpoints permiten gestionar el registro de dispositivos y la sincronización incremental de
  movimientos de stock.

  ---

  1. Gestión de Dispositivos (SyncDispositivo)
  Controla el estado de sincronización de cada terminal móvil.

   * `POST /api/v1/SyncDispositivo` (Upsert)
       * Uso: Al iniciar la app, el móvil se registra o actualiza su información (nombre, ID).
       * Body: SyncDispositivoDTO (DeviceId, NombreDispositivo).
   * `GET /api/v1/SyncDispositivo/{deviceId}`
       * Uso: Consultar el estado actual del móvil en el servidor (ej. cuál fue el último log que procesó).
   * `POST /api/v1/SyncDispositivo/confirm`
       * Uso: Cierre del ciclo de sincronización. El móvil informa al servidor que ha procesado correctamente los logs hasta el ID enviado.
       * Body: { "deviceId": "string", "lastLogId": long }

  ---

  2. Logs de Movimientos (LogMovimientosStock)
  Provee la información de los cambios de stock ocurridos en el sistema.

   * `GET /api/v1/LogMovimientosStock/updates?device_id={XYZ}`
       * Uso: Sincronización Incremental. Es el endpoint principal que llamará el móvil periódicamente.
       * Lógica: La API busca el ultimoIdLogProcesado del dispositivo y devuelve únicamente los registros de LogMovimientosStock que tengan un ID mayor a ese valor.
       * Retorno: Lista de LogMovimientosStockDTO.
   * `GET /api/v1/LogMovimientosStock/articulo/{idArticulo}`
       * Uso: Consultar el historial de logs específicos de un solo artículo.
   * `POST /api/v1/LogMovimientosStock`
       * Uso: Registro manual de un log desde la API (aunque la mayoría se generan automáticamente por procesos de stock).

  ---

  Flujo de Integración Recomendado para React Native

   1. Registro/Handshake: La App llama a POST /SyncDispositivo enviando su UniqueId.
   2. Consulta de Cambios: La App llama a GET /LogMovimientosStock/updates?device_id=... cada X minutos.
   3. Procesamiento Local: El móvil guarda los logs recibidos en su base de datos local (SQLite/Realm).
   4. Confirmación: Una vez guardados localmente, el móvil llama a POST /SyncDispositivo/confirm enviando el ID del último log recibido. Esto evita que el servidor vuelva a enviar los mismos datos en la
      siguiente consulta.

  ---

  Estructura de Datos (DTOs)
  Para las interfaces en TypeScript:

    1 interface SyncDispositivoDTO {
    2   deviceId: string;
    3   ultimoIdLogProcesado?: number;
    4   fechaUltimaSync?: string;
    5   nombreDispositivo?: string;
    6 }
    7
    8 interface LogMovimientosStockDTO {
    9   idLog: number;
   10   idArticulo: number;
   11   cantidadMovimiento: number;
   12   stockNuevo: number;
   13   fechaMovimiento: string;
   14   creadoPor: string;
   15 }

  ¿Listo para pasar a la parte de React Native o necesitas algún ajuste adicional en la API?