# Backlog y Release Plan — Sync Honor Smartwatch

## 1. Visión del Producto y Releases

| Release | Objetivo de negocio | Sprints | Funcionalidad clave |
|---|---|---|---|
| **Release 1 — MVP** | Recuperar sincronización básica de salud sin app oficial | Sprint 0, Sprint 1, Sprint 2, Sprint 3 | Emparejamiento BLE, sync manual de pasos y FC, dashboard, historial, exportación CSV |
| **Release 2 — Automatización** | Sincronización sin intervención del usuario | Sprint 4, Sprint 5 | Sync automática en background, notificaciones, sueño y entrenamientos |
| **Release 3 — Integración** | Centralizar datos en el ecosistema Android | Sprint 6 | Health Connect, multi-dispositivo, mejoras de UI |

---

## 2. Configuración del Proceso

- **Duración del Sprint:** 2 semanas
- **Definición de "Listo" (Definition of Ready):**
  - La historia tiene criterios de aceptación en formato Given/When/Then
  - Las dependencias técnicas están identificadas y resueltas o planificadas en sprint anterior
  - El diseño de pantalla o contrato de datos está definido
  - La historia está estimada en story points y priorizada
- **Definición de "Terminado" (Definition of Done):**
  - Código implementado y revisado (PR aprobado)
  - Pruebas unitarias para lógica de dominio con cobertura > 80 %
  - Pruebas manuales en dispositivo físico con smartwatch Honor
  - Sin regresiones en lint (`./gradlew lint`)
  - Documentación de README actualizada si hay cambios de configuración
  - Criterios de aceptación verificados y casos de prueba TC asociados ejecutados

---

## 3. Backlog por Sprint

### Sprint 0 — Setup e infraestructura

**Objetivo/Hito del sprint:** Proyecto Android compilable con arquitectura base, CI y esquema de base de datos.

**Tareas secuenciales:**
1. Crear proyecto Android con Kotlin, Compose, Hilt, Room, minSdk 26, targetSdk 34
2. Configurar estructura de paquetes: `data`, `domain`, `ui`, `ble`, `di`
3. Definir entidades Room (`PairedDevice`, `HealthMetric`, `SyncSession`) y DAOs con migración v1
4. Implementar `DataStoreManager` para configuración de app
5. Configurar GitHub Actions: build, lint, unit tests
6. Crear tema Material 3 y navegación base con `NavHost` (Dashboard, Pairing, Settings, History)

**Historias de Usuario del sprint:**

- **HU-001**: Como desarrollador, quiero un proyecto Android con arquitectura Clean + MVVM configurada, para construir features de forma estructurada.
  - **Criterios de Aceptación:**
    - Dado un repositorio nuevo, cuando ejecuto `./gradlew assembleDebug`, entonces la app compila sin errores.
    - Dado el módulo `domain`, cuando reviso sus dependencias, entonces no importa clases del framework Android.
    - Dado un push a `main`, cuando corre CI, entonces build y tests unitarios pasan en < 10 min.
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** ninguna

- **HU-002**: Como usuario, quiero que la app solicite los permisos de Bluetooth y ubicación necesarios, para poder escanear dispositivos BLE.
  - **Criterios de Aceptación:**
    - Dado Android 12+, cuando abro la app por primera vez, entonces se solicitan permisos `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` y `ACCESS_FINE_LOCATION`.
    - Dado que deniego un permiso crítico, cuando intento escanear, entonces veo un mensaje explicativo con enlace a Ajustes.
    - Dado que concedo todos los permisos, cuando llego a la pantalla de emparejamiento, entonces el escaneo puede iniciarse sin bloqueos.
  - **Estimación:** 3
  - **Prioridad:** Alta
  - **Dependencias:** HU-001

**Métricas del Sprint:**
- Velocity planeado: 8 story points
- Entregable(s) verificable(s): APK debug instalable, CI verde, esquema Room v1 creado

---

### Sprint 1 — Emparejamiento BLE

**Objetivo/Hito del sprint:** Descubrir y emparejar un smartwatch Honor vía BLE.

**Tareas secuenciales:**
1. Implementar `BleGattManager` con escaneo BLE filtrado por nombre (`Honor`, `HUAWEI`, `Band`, `Watch`)
2. Crear `ScannedDevice` model y Flow reactivo de resultados de escaneo con RSSI
3. Implementar conexión GATT: connect, discover services, guardar UUIDs descubiertos
4. Crear `PairingScreen` en Compose con lista de dispositivos y indicador de señal
5. Implementar `DeviceRepository` y persistencia de `PairedDevice` en Room
6. Manejar desconexión y reconexión automática (hasta 3 reintentos)

**Historias de Usuario del sprint:**

- **HU-003**: Como usuario, quiero escanear dispositivos Honor cercanos, para identificar mi smartwatch en una lista.
  - **Criterios de Aceptación:**
    - Dado permisos BLE concedidos, cuando pulso "Buscar dispositivos", entonces la lista muestra dispositivos con nombre, MAC y RSSI en < 10 s.
    - Dado que no hay dispositivos cercanos, cuando el escaneo termina, entonces veo mensaje "No se encontraron dispositivos" con opción de reintentar.
    - Dado un dispositivo con RSSI < -90 dBm, cuando aparece en la lista, entonces se muestra indicador de señal débil.
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** HU-002

- **HU-004**: Como usuario, quiero emparejar mi smartwatch seleccionándolo de la lista, para que la app recuerde mi dispositivo.
  - **Criterios de Aceptación:**
    - Dado un dispositivo en la lista, cuando lo selecciono y confirmo, entonces se establece conexión GATT y se guarda en Room con `is_active=true`.
    - Dado un emparejamiento exitoso, cuando regreso al dashboard, entonces veo el nombre del dispositivo y estado "Conectado" o "Emparejado".
    - Dado un fallo de conexión GATT, cuando ocurre timeout de 15 s, entonces veo error descriptivo y el dispositivo no se guarda.
  - **Estimación:** 8
  - **Prioridad:** Alta
  - **Dependencias:** HU-003

**Métricas del Sprint:**
- Velocity planeado: 13 story points
- Entregable(s): Emparejamiento funcional con Honor Band/Watch real

---

### Sprint 2 — Sincronización y captura de datos

**Objetivo/Hito del sprint:** Leer pasos y frecuencia cardíaca del reloj y persistirlos localmente.

**Tareas secuenciales:**
1. Implementar `HonorProtocolParser` con decodificación de payloads para pasos (UUID `0000fee7`) y FC (UUID `00002a37`)
2. Crear `SyncHealthDataUseCase` con máquina de estados: CONNECTING → READING → PARSING → SAVING → DONE
3. Implementar `HealthRepository` con inserción batch de métricas y deduplicación por `(device_mac, metric_type, recorded_at)`
4. Crear `SyncSession` logging con estados RUNNING/SUCCESS/FAILED
5. Implementar botón "Sincronizar ahora" en Dashboard con indicador de progreso
6. Escribir tests unitarios de parser con fixtures de bytes reales capturados de Honor Band 7

**Historias de Usuario del sprint:**

- **HU-005**: Como usuario, quiero sincronizar manualmente mis datos de salud, para actualizar pasos y frecuencia cardíaca al instante.
  - **Criterios de Aceptación:**
    - Dado un dispositivo emparejado y dentro de rango BLE, cuando pulso "Sincronizar ahora", entonces la sync completa en < 30 s y muestra resultado.
    - Dado una sync exitosa, cuando termina, entonces se crea `SyncSession` con status SUCCESS y `records_imported > 0`.
    - Dado que el reloj está fuera de rango, cuando intento sincronizar, entonces veo error "Dispositivo no encontrado" y `SyncSession` con status FAILED.
  - **Estimación:** 8
  - **Prioridad:** Alta
  - **Dependencias:** HU-004

- **HU-006**: Como usuario, quiero que la app capture y almacene mis pasos diarios, para consultarlos sin conexión a Internet.
  - **Criterios de Aceptación:**
    - Dado una sync exitosa con datos de pasos, cuando consulto Room, entonces existen registros `MetricType.STEPS` con timestamp del día actual.
    - Dado pasos ya sincronizados, cuando ejecuto otra sync del mismo día, entonces no se duplican registros (upsert por clave compuesta).
    - Dado 0 pasos en el reloj, cuando sincronizo, entonces se guarda valor 0 sin error.
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** HU-005

- **HU-007**: Como usuario, quiero que la app capture mi frecuencia cardíaca, para monitorear mi ritmo cardíaco diario.
  - **Criterios de Aceptación:**
    - Dado una sync exitosa, cuando hay mediciones de FC en el reloj, entonces se almacenan registros `MetricType.HEART_RATE` con valor en bpm y timestamp.
    - Dado una FC fuera de rango fisiológico (> 220 o < 30), cuando se parsea, entonces el registro se descarta y se loguea advertencia sin crashear.
    - Dado el reloj sin sensor activo, cuando sincronizo, entonces la sync termina con status SUCCESS y 0 registros de FC (sync parcial aceptable).
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** HU-005

**Métricas del Sprint:**
- Velocity planeado: 18 story points
- Entregable(s): Sync manual de pasos + FC verificada en hardware real

---

### Sprint 3 — Dashboard, historial y exportación (cierre MVP)

**Objetivo/Hito del sprint:** Release 1 funcional de punta a punta: ver datos, historial y exportar CSV.

**Tareas secuenciales:**
1. Crear `DashboardScreen` con tarjetas de pasos del día, FC promedio/última, timestamp última sync
2. Implementar `HistoryScreen` con lista de `SyncSession` ordenada por fecha descendente
3. Crear `CsvExporter` que genera archivo en cache y lanza `Intent.ACTION_SEND`
4. Implementar `SettingsScreen` con toggle de unidades y botón "Desemparejar dispositivo"
5. Añadir empty states para dashboard sin datos y sin dispositivo emparejado
6. Pruebas de integración end-to-end del flujo completo en emulador + dispositivo físico

**Historias de Usuario del sprint:**

- **HU-008**: Como usuario, quiero ver un dashboard con mis métricas del día, para conocer mi actividad de un vistazo.
  - **Criterios de Aceptación:**
    - Dado datos sincronizados hoy, cuando abro el dashboard, entonces veo total de pasos, última FC y hora de última sincronización.
    - Dado sin datos del día, cuando abro el dashboard, entonces veo estado vacío con CTA "Sincronizar ahora".
    - Dado una sync en progreso, cuando estoy en el dashboard, entonces veo indicador de carga y botón deshabilitado.
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** HU-006, HU-007

- **HU-009**: Como usuario, quiero consultar el historial de sincronizaciones, para detectar fallos de conexión.
  - **Criterios de Aceptación:**
    - Dado al menos una sync previa, cuando abro Historial, entonces veo lista con fecha, duración, registros importados y estado (éxito/error).
    - Dado una sync fallida, cuando la veo en historial, entonces se muestra el mensaje de error almacenado en `error_message`.
    - Dado sin historial, cuando abro la pantalla, entonces veo mensaje "Aún no hay sincronizaciones".
  - **Estimación:** 3
  - **Prioridad:** Media
  - **Dependencias:** HU-005

- **HU-010**: Como usuario, quiero exportar mis datos de salud a CSV, para analizarlos externamente o respaldarlos.
  - **Criterios de Aceptación:**
    - Dado métricas almacenadas, cuando pulso "Exportar CSV", entonces se genera archivo con columnas `fecha,tipo,valor,unidad,dispositivo` y se abre selector de compartir.
    - Dado sin métricas, cuando pulso exportar, entonces veo mensaje "No hay datos para exportar".
    - Dado un rango de 30 días de datos, cuando exporto, entonces el archivo se genera en < 5 s y pesa < 1 MB.
  - **Estimación:** 5
  - **Prioridad:** Alta
  - **Dependencias:** HU-006, HU-007

**Métricas del Sprint:**
- Velocity planeado: 13 story points
- Entregable(s): **Release 1 (MVP)** publicable como APK sideload

---

### Sprint 4 — Sincronización automática en background

**Objetivo/Hito del sprint:** Sync periódica sin intervención del usuario.

**Historias de Usuario del sprint:**

- **HU-011**: Como usuario, quiero configurar sincronización automática cada 15/30/60 minutos, para mantener mis datos actualizados sin abrir la app.
  - **Criterios de Aceptación:**
    - Dado intervalo configurado en 30 min, cuando pasan 30 min con dispositivo en rango, entonces WorkManager ejecuta sync automáticamente.
    - Dado sync automática desactivada, cuando pasa el intervalo, entonces no se ejecuta ninguna sync.
    - Dado el teléfono en modo ahorro de batería, cuando WorkManager intenta sync, entonces respeta restricciones y reintenta según política.
  - **Estimación:** 8
  - **Prioridad:** Alta
  - **Dependencias:** HU-005

- **HU-012**: Como usuario, quiero recibir notificación del resultado de cada sincronización, para saber si hubo errores sin abrir la app.
  - **Criterios de Aceptación:**
    - Dado una sync automática exitosa, cuando termina, entonces recibo notificación silenciosa con "X registros sincronizados".
    - Dado una sync fallida, cuando termina, entonces recibo notificación con prioridad alta y mensaje de error.
    - Dado notificaciones desactivadas en ajustes de la app, cuando sync termina, entonces no se muestra notificación.
  - **Estimación:** 5
  - **Prioridad:** Media
  - **Dependencias:** HU-011

**Métricas del Sprint:**
- Velocity planeado: 13 story points
- Entregable(s): Sync background funcional

---

### Sprint 5 — Sueño y entrenamientos

**Objetivo/Hito del sprint:** Ampliar tipos de métricas capturadas.

**Historias de Usuario del sprint:**

- **HU-013**: Como usuario, quiero sincronizar datos de sueño, para registrar mis horas de descanso.
  - **Criterios de Aceptación:**
    - Dado datos de sueño en el reloj, cuando sincronizo, entonces se almacenan registros con duración total y hora inicio/fin en `metadata_json`.
    - Dado sin datos de sueño, cuando sincronizo, entonces no se generan registros de tipo SLEEP sin error.
    - Dado sueño sincronizado, cuando veo el dashboard, entonces aparece tarjeta "Sueño anoche: X h Y min".
  - **Estimación:** 8
  - **Prioridad:** Media
  - **Dependencias:** HU-005

- **HU-014**: Como usuario, quiero sincronizar sesiones de entrenamiento, para llevar registro de mis actividades físicas.
  - **Criterios de Aceptación:**
    - Dado entrenamientos en el reloj, cuando sincronizo, entonces se almacenan registros con tipo, duración y calorías en metadata.
    - Dado múltiples entrenamientos del día, cuando sincronizo, entonces cada sesión se guarda como registro independiente.
    - Dado entrenamientos sincronizados, cuando exporto CSV, entonces incluyen filas de tipo WORKOUT.
  - **Estimación:** 8
  - **Prioridad:** Media
  - **Dependencias:** HU-005

**Métricas del Sprint:**
- Velocity planeado: 16 story points
- Entregable(s): **Release 2** con métricas extendidas

---

### Sprint 6 — Health Connect e integración

**Objetivo/Hito del sprint:** Publicar datos en Health Connect de Android.

**Historias de Usuario del sprint:**

- **HU-015**: Como usuario, quiero que mis pasos y FC se publiquen en Health Connect, para centralizarlos con Google Fit y otras apps.
  - **Criterios de Aceptación:**
    - Dado permisos de Health Connect concedidos, cuando sincronizo pasos, entonces se insertan registros en `StepsRecord` de Health Connect.
    - Dado permisos denegados, cuando sincronizo, entonces los datos se guardan solo localmente sin error bloqueante.
    - Dado datos ya publicados, cuando re-sincronizo el mismo día, entonces Health Connect actualiza sin duplicar.
  - **Estimación:** 8
  - **Prioridad:** Baja
  - **Dependencias:** HU-006, HU-007

**Métricas del Sprint:**
- Velocity planeado: 8 story points
- Entregable(s): **Release 3** con integración Health Connect

---

## 4. Resumen de Release Plan

| Release | Sprints incluidos | Fecha estimada | Funcionalidad entregada | Hito de negocio |
|---|---|---|---|---|
| Release 1 (MVP) | Sprint 0 – Sprint 3 | 2026-08-08 | Emparejamiento BLE, sync manual pasos+FC, dashboard, historial, export CSV | Usuario recupera datos básicos sin app oficial |
| Release 2 | Sprint 4 – Sprint 5 | 2026-09-19 | Sync automática, notificaciones, sueño, entrenamientos | Operación autónoma y datos completos |
| Release 3 | Sprint 6 | 2026-10-03 | Health Connect | Integración con ecosistema Android de salud |

*Fechas asumiendo inicio Sprint 0 el 2026-07-07.*

---

## 5. Backlog priorizado (vista tabular consolidada)

| ID | Historia | Sprint | Prioridad | Story Points | Estado |
|---|---|---|---|---|---|
| HU-001 | Proyecto Android con Clean + MVVM configurada | Sprint 0 | Alta | 5 | Pendiente |
| HU-002 | Permisos BLE y ubicación | Sprint 0 | Alta | 3 | Pendiente |
| HU-003 | Escanear dispositivos Honor cercanos | Sprint 1 | Alta | 5 | Pendiente |
| HU-004 | Emparejar smartwatch desde lista | Sprint 1 | Alta | 8 | Pendiente |
| HU-005 | Sincronizar manualmente datos de salud | Sprint 2 | Alta | 8 | Pendiente |
| HU-006 | Capturar y almacenar pasos diarios | Sprint 2 | Alta | 5 | Pendiente |
| HU-007 | Capturar frecuencia cardíaca | Sprint 2 | Alta | 5 | Pendiente |
| HU-008 | Dashboard con métricas del día | Sprint 3 | Alta | 5 | Pendiente |
| HU-009 | Historial de sincronizaciones | Sprint 3 | Media | 3 | Pendiente |
| HU-010 | Exportar datos a CSV | Sprint 3 | Alta | 5 | Pendiente |
| HU-011 | Sincronización automática configurable | Sprint 4 | Alta | 8 | Pendiente |
| HU-012 | Notificaciones de resultado de sync | Sprint 4 | Media | 5 | Pendiente |
| HU-013 | Sincronizar datos de sueño | Sprint 5 | Media | 8 | Pendiente |
| HU-014 | Sincronizar sesiones de entrenamiento | Sprint 5 | Media | 8 | Pendiente |
| HU-015 | Publicar en Health Connect | Sprint 6 | Baja | 8 | Pendiente |

**Total:** 89 story points | **MVP (Release 1):** 52 story points (Sprint 0–3)

---

## 6. Riesgos y Dependencias Técnicas

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| UUIDs GATT de Honor cambian entre modelos/firmware | Alta | Alto | Mantener tabla de UUIDs por `model_id`; parser con fallback a servicio genérico `0xFEE7`; tests con fixtures de múltiples modelos |
| Android mata conexión BLE en background (Doze) | Media | Alto | Foreground Service durante sync activa; WorkManager con `setExpedited` para sync programada |
| Reloj requiere autenticación propietaria no documentada | Media | Alto | Capturar tráfico BLE con nRF Connect en fase Sprint 1; documentar handshake en wiki interna |
| Permisos BLE denegados permanentemente por el usuario | Media | Medio | Pantalla educativa con enlace a Ajustes; detectar `shouldShowRequestPermissionRationale` |
| Parser recibe payload corrupto | Baja | Medio | Validación de checksum; sync parcial con status PARTIAL; no crashear app |
| Health Connect no disponible en dispositivo | Baja | Bajo | Feature flag; ocultar opción si `HealthConnectClient.getSdkStatus() != AVAILABLE` |
