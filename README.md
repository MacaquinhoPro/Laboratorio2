# Práctica FreeRTOS — Sensores Touch + Colas + Mutex

## Estructura del repositorio

```
FreeRTOS_Touch_Practice/
├── FreeRTOS_Touch_Practice.ino   ← Sketch principal (importar en Arduino IDE)
├── PREGUNTAS_Y_RESPUESTAS.md     ← Respuestas con código de ejemplo
└── README.md                     ← Este archivo
```

## Hardware requerido

- ESP32 (cualquier variante con pines touch: DOIT DevKit, TTGO, etc.)
- Dos conductores o almohadillas capacitivas conectadas a GPIO4 y GPIO15
- Cable USB para programación y monitoreo serial

## Parámetros ajustables (definidos en el `.ino`)

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| `SENSOR_PIN_1` | 4 | Pin touch del sensor 1 (T0) |
| `SENSOR_PIN_2` | 15 | Pin touch del sensor 2 (T3) |
| `TOUCH_THRESHOLD` | 40 | Umbral de detección (< = toque) |
| `QUEUE_LENGTH` | 10 | Capacidad de cada cola |
| `TASK_PERIOD_MS` | 200 | Período de muestreo en ms |

## Salida Serial (JSON)

```json
{"sensor":1,"raw":23,"touched":true,"ts":1042}
{"sensor":2,"raw":68,"touched":false,"ts":1045}
```

## Arquitectura del sistema

```
[Sensor 1] → TaskSensor(p1) → Queue1 → TaskSerial(p1) ─┐
                                                         ├→ [Puerto Serial]
[Sensor 2] → TaskSensor(p2) → Queue2 → TaskSerial(p2) ─┘
                                              ↕ Mutex (acceso exclusivo)
```

---

## Arquitectura hipotética del proyecto final

> Aplica los conceptos de FreeRTOS a un proyecto más complejo, por ejemplo un sistema de monitoreo ambiental con display, almacenamiento SD y envío WiFi.

### Componentes y tareas

| Tarea | Función | Prioridad |
|---|---|---|
| `TaskSensors` | Lee temperatura, humedad, CO2 (mismo patrón: fn reutilizable + params) | Alta (3) |
| `TaskDisplay` | Consume cola de datos y actualiza pantalla OLED | Media (2) |
| `TaskWiFi` | Consume cola de datos y publica por MQTT/HTTP | Media (2) |
| `TaskSDLogger` | Consume cola y escribe CSV en SD card | Baja (1) |
| `TaskBLE` | Consume cola BLE y notifica a app móvil | Media (2) |

### Colas

| Cola | Productor | Consumidores |
|---|---|---|
| `xEnvQueue` | TaskSensors | TaskDisplay, TaskWiFi, TaskSDLogger, TaskBLE |
| `xAlertQueue` | TaskSensors (cuando valor > umbral) | TaskWiFi (alerta push) |
| `xCommandQueue` | UART / BLE (comandos externos) | TaskSensors (cambiar umbral) |

### Recursos compartidos (mutex necesario)

| Recurso | Tareas que comparten | Control |
|---|---|---|
| Bus SPI (SD + Display) | TaskDisplay, TaskSDLogger | `xSPIMutex` |
| Buffer WiFi | TaskWiFi, TaskBLE | `xNetMutex` |
| Configuración EEPROM | Todas | `xConfigMutex` |

### Diagrama de arquitectura hipotética

```
                        ┌─────────────┐
[Sensores] → TaskSensors │  xEnvQueue  │──→ TaskDisplay  → [OLED]
            (fn shared)  │  (fan-out)  │──→ TaskWiFi     → [MQTT/HTTP]
                        └─────────────┘──→ TaskSDLogger  → [SD Card]
                               │        └→ TaskBLE       → [App móvil]
                         xAlertQueue
                               ↓
                          TaskWiFi (push alert)

[UART/BLE] → xCommandQueue → TaskSensors (ajustar umbrales)

Mutex:
  xSPIMutex    → comparten TaskDisplay y TaskSDLogger (bus SPI)
  xNetMutex    → comparten TaskWiFi y TaskBLE
  xConfigMutex → todas las tareas (lectura/escritura de configuración)
```

Esta arquitectura escala bien porque:
1. Las tareas productoras y consumidoras están desacopladas por colas.
2. Agregar un nuevo consumidor (ej: Telegram bot) solo requiere suscribirse a `xEnvQueue`.
3. El mutex por bus evita colisiones en hardware compartido.
4. La prioridad diferenciada garantiza que los sensores siempre son leídos aunque el WiFi esté lento.
