# 🔌 05 — Contratos API

[← Volver al Índice](README.md) · [← Modelos de Datos](04-modelos-datos.md)

---

## Resumen

Contratos REST API en formato JSON estándar para integrar el módulo **Fix Academy** con Dragon App, independientemente del lenguaje.

**Base URL:** `/api/fix-academy/v1`

---

## 📋 Índice de Endpoints

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `POST` | `/parser/parse` | Parsear texto crudo de práctica |
| `POST` | `/parser/parse-screenshot` | 📸 Parsear captura de pantalla de app móvil GPRO |
| `POST` | `/validator/validate` | Validar estrategia completa |
| `POST` | `/entries` | Crear registro de carrera para alumno |
| `GET` | `/entries/{id}` | Obtener registro específico |
| `GET` | `/races/{raceId}/entries` | Todos los registros de una carrera |
| `GET` | `/students/{id}/history` | Historial de un alumno |
| `PUT` | `/entries/{id}/review` | Mentor revisa un registro |
| `POST` | `/baselines` | Crear/Actualizar Base de Oro |
| `GET` | `/baselines/{trackId}` | Obtener Base de Oro de una pista |
| `GET` | `/dashboard/race/{raceId}` | Dashboard comparativo |
| `GET` | `/dashboard/student/{id}` | Dashboard evolución alumno |
| `POST` | `/calculator/sync-baseline` | 🆕 Auto-generar Base de Oro desde calculadora |
| `GET` | `/calculator/compare/{entryId}` | 🆕 Comparar setup del alumno vs calculado |
| `POST` | `/calculator/link` | 🆕 Vincular setup calculado con entry |
| `GET` | `/calculator/track-sync` | 🆕 Estado de sincronización de pistas |

---

## 🔍 Parser

### `POST /parser/parse`

Recibe texto crudo y retorna datos parseados.

**Request:**

```json
{
  "rawText": "Practice results\nLap  Time       Mistake  Net time\n1    1:22.456   0.120s   1:22.576\n\nSetup used:\nFront wing: 45\nRear wing: 38\nEngine: 680\nBrakes: 320\nGear: 140\nSuspension: 72\n\nTyres used: Extra Soft\nFuel load: 35 lts\n\nWeather: 28°C, Humidity 45%, Clouds: 10%\n\nDriver feedback:\n\"The car understeers in slow corners. I need more front wing.\nThe brakes feel too soft.\"",
  "studentId": 1,
  "raceId": 5
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "setup": {
      "frontWing": 45,
      "rearWing": 38,
      "engine": 680,
      "brakes": 320,
      "gear": 140,
      "suspension": 72
    },
    "strategy": {
      "tyreCompound": "Extra Soft",
      "fuelLoad": 35.0
    },
    "conditions": {
      "temperature": 28,
      "humidity": 45,
      "clouds": 10
    },
    "performance": {
      "laps": [
        {
          "lapNumber": 1,
          "time": "1:22.456",
          "mistake": 0.120,
          "netTime": "1:22.576"
        }
      ],
      "bestLap": "1:22.456"
    },
    "feedback": {
      "raw": "The car understeers in slow corners...",
      "suggestions": [
        {
          "component": "FrontWing",
          "action": "INCREASE",
          "reason": "Understeer detected",
          "priority": "HIGH"
        },
        {
          "component": "Brakes",
          "action": "INCREASE",
          "reason": "Brakes too soft",
          "priority": "MEDIUM"
        }
      ]
    },
    "parseConfidence": 0.95,
    "warnings": []
  }
}
```

**Response `422 Unprocessable`:**

```json
{
  "success": false,
  "error": {
    "code": "PARSE_FAILED",
    "message": "No se pudo reconocer el formato del texto",
    "details": [
      "No se encontró el campo 'FrontWing'",
      "No se encontró el campo 'Engine'"
    ]
  }
}
```

---

### `POST /parser/parse-screenshot` 📸

Recibe una captura de pantalla de la app móvil de GPRO y extrae los datos via OCR.

**Request:** `multipart/form-data`

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `screenshot` | `File` | ✅ | Imagen PNG, JPG o WebP. Máx 10 MB |
| `studentId` | `INT` | ✅ | ID del alumno |
| `raceId` | `INT` | ✅ | ID de la carrera |

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "setup": {
      "frontWing": 45,
      "rearWing": 38,
      "engine": 680,
      "brakes": 320,
      "gear": 140,
      "suspension": 72
    },
    "strategy": {
      "tyreCompound": "Extra Soft",
      "fuelLoad": 35.0
    },
    "conditions": {
      "temperature": 28,
      "humidity": 45
    },
    "performance": {
      "bestLap": "1:22.456"
    },
    "feedback": {
      "raw": "The car understeers...",
      "suggestions": []
    },
    "parseConfidence": 0.88,
    "source": "SCREENSHOT",
    "ocrConfidence": 0.91,
    "warnings": [
      "⚠️ Confianza OCR media (91%). Verifica los valores."
    ]
  }
}
```

**Response `422 Unprocessable` (OCR falla):**

```json
{
  "success": false,
  "error": {
    "code": "OCR_FAILED",
    "message": "No se pudo leer la captura con suficiente confianza",
    "ocrConfidence": 0.45,
    "suggestion": "Intenta con una captura más nítida o pega el texto manualmente"
  }
}
```

**Response `400 Bad Request` (formato inválido):**

```json
{
  "success": false,
  "error": {
    "code": "INVALID_IMAGE",
    "message": "Formato de imagen no soportado. Usa PNG, JPG o WebP.",
    "maxSizeMB": 10
  }
}
```

---

## 🛡️ Validador

### `POST /validator/validate`

Valida la estrategia completa de un alumno.

**Request:**

```json
{
  "setup": {
    "frontWing": 45,
    "rearWing": 38,
    "engine": 680,
    "brakes": 320,
    "gear": 140,
    "suspension": 72
  },
  "strategy": {
    "tyreCompound": "Extra Soft",
    "fuelLoad": 30.0,
    "pitStops": [
      {
        "stopNumber": 1,
        "plannedLap": 20,
        "tyreCompound": "Soft",
        "fuelToAdd": 10.0
      }
    ]
  },
  "raceInfo": {
    "trackId": 12,
    "totalLaps": 65,
    "totalKm": 213.87,
    "fuelTankCapacity": 90.0
  },
  "conditions": {
    "temperature": 35,
    "humidity": 45,
    "rainProbability": 40
  },
  "baselineId": 7
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "overallStatus": "CRITICAL",
    "checks": [
      {
        "rule": "FUEL_CHECK",
        "status": "WARNING",
        "level": "WARNING",
        "message": "Combustible justo",
        "detail": "Necesitas 28.5L, llevas 30L. Margen de solo 1.5L",
        "suggestion": "Sube el fuel a mínimo 33L por seguridad",
        "data": {
          "required": 28.5,
          "available": 30.0,
          "margin": 1.5
        }
      },
      {
        "rule": "TYRE_CHECK",
        "status": "CRITICAL",
        "level": "CRITICAL",
        "message": "¡Te vas a quedar sin goma en la vuelta 10!",
        "detail": "Extra Soft no soporta 35°C (máximo: 22°C)",
        "suggestion": "Para 35°C usa: Medium o Hard",
        "data": {
          "compound": "Extra Soft",
          "trackTemp": 35,
          "maxTemp": 22,
          "excess": 13,
          "recommendedCompounds": ["Medium", "Hard"]
        }
      },
      {
        "rule": "WEATHER_CHECK",
        "status": "WARNING",
        "level": "WARNING",
        "message": "Posible lluvia sin plan B",
        "detail": "40% de probabilidad de lluvia",
        "suggestion": "Configura al menos 1 pit con Rain tyres",
        "data": {
          "rainProbability": 40,
          "hasRainPit": false
        }
      },
      {
        "rule": "BASELINE_CHECK",
        "status": "WARNING",
        "level": "WARNING",
        "message": "Setup desviado de la Base de Oro",
        "detail": "Desviación promedio: 8.2%",
        "suggestion": "Revisa FrontWing (-7), RearWing (-5) y Suspension (+4)",
        "data": {
          "deviationPercent": 8.2,
          "deviations": {
            "frontWing": { "baseline": 52, "current": 45, "percent": 13.5 },
            "rearWing": { "baseline": 38, "current": 33, "percent": 13.2 },
            "engine": { "baseline": 700, "current": 680, "percent": 2.9 },
            "brakes": { "baseline": 345, "current": 320, "percent": 7.2 },
            "gear": { "baseline": 155, "current": 140, "percent": 9.7 },
            "suspension": { "baseline": 68, "current": 72, "percent": 5.9 }
          }
        }
      }
    ],
    "canSave": false,
    "blockReason": "1 error crítico debe ser corregido"
  }
}
```

---

## 📝 Registros (Entries)

### `POST /entries`

Crea un nuevo registro de carrera.

**Request:**

```json
{
  "studentId": 1,
  "raceId": 5,
  "rawPracticeText": "...(texto crudo)...",
  "setup": {
    "frontWing": 52,
    "rearWing": 38,
    "engine": 700,
    "brakes": 345,
    "gear": 155,
    "suspension": 68,
    "source": "PARSED"
  },
  "strategy": {
    "tyreCompound": "Medium",
    "fuelLoad": 45.0,
    "wetSetup": false,
    "pitStops": [
      {
        "stopNumber": 1,
        "plannedLap": 22,
        "tyreCompound": "Hard",
        "fuelToAdd": 15.0
      }
    ]
  }
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "entryId": 42,
    "validationStatus": "VALID",
    "createdAt": "2026-02-06T10:30:00Z"
  }
}
```

---

### `GET /entries/{id}`

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "id": 42,
    "student": {
      "id": 1,
      "username": "Pedro"
    },
    "race": {
      "id": 5,
      "trackName": "Monaco",
      "raceNumber": 5,
      "seasonNumber": 98
    },
    "setup": { "frontWing": 52, "rearWing": 38, "engine": 700, "brakes": 345, "gear": 155, "suspension": 68 },
    "strategy": { "tyreCompound": "Medium", "fuelLoad": 45.0 },
    "validationStatus": "VALID",
    "validationAlerts": [],
    "reviewedByMentor": false,
    "mentorNotes": null,
    "createdAt": "2026-02-06T10:30:00Z"
  }
}
```

---

## 📊 Dashboard

### `GET /dashboard/race/{raceId}`

Vista comparativa de todos los alumnos para una carrera.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "race": {
      "id": 5,
      "track": "Monaco",
      "raceNumber": 5,
      "date": "2026-02-10",
      "conditions": {
        "temperature": 28,
        "humidity": 45,
        "rainProbability": 10
      }
    },
    "baseline": {
      "frontWing": 52, "rearWing": 38, "engine": 700, "brakes": 345,
      "gear": 155, "suspension": 68,
      "recommendedCompound": "Medium"
    },
    "entries": [
      {
        "student": { "id": 1, "username": "Pedro" },
        "setup": { "frontWing": 50, "rearWing": 36, "engine": 710, "brakes": 340, "gear": 150, "suspension": 65 },
        "strategy": { "tyreCompound": "Medium", "fuelLoad": 44.0 },
        "deviationPercent": 3.2,
        "status": "VALID",
        "reviewed": true
      },
      {
        "student": { "id": 3, "username": "Carlos" },
        "setup": { "frontWing": 35, "rearWing": 22, "engine": 800, "brakes": 280, "gear": 120, "suspension": 90 },
        "strategy": { "tyreCompound": "Extra Soft", "fuelLoad": 30.0 },
        "deviationPercent": 22.4,
        "status": "CRITICAL",
        "reviewed": false
      }
    ],
    "stats": {
      "totalStudents": 5,
      "submitted": 4,
      "pending": 1,
      "criticalAlerts": 1,
      "avgDeviation": 9.8
    }
  }
}
```

---

### `GET /dashboard/student/{id}?seasonId=98`

Evolución individual de un alumno.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "student": {
      "id": 3,
      "username": "Carlos",
      "joinDate": "2025-09-01"
    },
    "season": {
      "id": 98,
      "seasonNumber": 98
    },
    "raceHistory": [
      {
        "raceNumber": 1,
        "track": "Monaco",
        "deviationPercent": 28.0,
        "startPosition": 15,
        "finishPosition": 12,
        "positionsGained": 3,
        "alertsCritical": 2,
        "alertsWarning": 1,
        "alertsCorrected": 1
      },
      {
        "raceNumber": 2,
        "track": "Silverstone",
        "deviationPercent": 18.0,
        "startPosition": 12,
        "finishPosition": 10,
        "positionsGained": 2,
        "alertsCritical": 1,
        "alertsWarning": 0,
        "alertsCorrected": 1
      }
    ],
    "trends": {
      "deviationTrend": "IMPROVING",
      "avgDeviation": 15.3,
      "avgPositionsGained": 2.0,
      "correctionRate": 0.75,
      "totalRaces": 5
    }
  }
}
```

---

## ⭐ Base de Oro

### `POST /baselines`

**Request:**

```json
{
  "trackId": 12,
  "seasonId": 98,
  "mentorId": 1,
  "frontWing": 52,
  "rearWing": 38,
  "engine": 700,
  "brakes": 345,
  "gear": 155,
  "suspension": 68,
  "recommendedCompound": "Medium",
  "notes": "Setup validado en práctica. Cuidado con el understeer en la chicane."
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "baselineId": 7,
    "createdAt": "2026-02-06T09:00:00Z"
  }
}
```

---

### `PUT /entries/{id}/review`

Mentor revisa un registro.

**Request:**

```json
{
  "mentorId": 1,
  "notes": "FrontWing muy bajo para Monaco. Hablar con Carlos sobre downforce en circuitos urbanos.",
  "approved": false
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "entryId": 42,
    "reviewedByMentor": true,
    "reviewedAt": "2026-02-06T11:00:00Z"
  }
}
```

---

## 🔗 Calculadora — Endpoints de Integración 🆕

### `POST /calculator/sync-baseline`

Auto-genera la Base de Oro desde el último cálculo de la calculadora.

**Request:**

```json
{
  "mentorId": 1,
  "trackId": 12,
  "seasonId": 98
}
```

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "baselineId": 7,
    "source": "CALCULATOR",
    "calcSessionId": 234,
    "setup": {
      "frontWing": 52,
      "rearWing": 38,
      "engine": 700,
      "brakes": 345,
      "gear": 155,
      "suspension": 68
    },
    "message": "Base de Oro actualizada desde calculadora",
    "syncedAt": "2026-02-06T10:00:00Z"
  }
}
```

**Response `404` (sin datos de calculadora):**

```json
{
  "success": false,
  "error": {
    "code": "CALC_NO_DATA",
    "message": "No hay cálculo disponible para Monaco en S98"
  }
}
```

---

### `GET /calculator/compare/{entryId}`

Compara el setup del alumno contra lo que calculó la herramienta.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "entryId": 42,
    "student": "Carlos",
    "track": "Monaco",
    "calculated": {
      "frontWing": 52, "rearWing": 38, "engine": 700, "brakes": 345,
      "gear": 155, "suspension": 68,
      "sessionId": 234
    },
    "actual": {
      "frontWing": 35, "rearWing": 22, "engine": 800, "brakes": 280,
      "gear": 120, "suspension": 90
    },
    "deviations": {
      "frontWing": { "diff": -17, "percent": 32.7 },
      "rearWing": { "diff": -16, "percent": 42.1 },
      "engine": { "diff": 100, "percent": 14.3 },
      "brakes": { "diff": -65, "percent": 18.8 },
      "gear": { "diff": -35, "percent": 22.6 },
      "suspension": { "diff": 22, "percent": 32.4 }
    },
    "avgDeviation": 24.2,
    "status": "HIGH_DEVIATION",
    "insight": "El alumno se desvía significativamente de la calculadora"
  }
}
```

---

### `POST /calculator/link`

Vincula manualmente un setup calculado con un entry del alumno.

**Request:**

```json
{
  "entryId": 42,
  "calcSessionId": 234
}
```

**Response `201 Created`:**

```json
{
  "success": true,
  "data": {
    "linkId": 15,
    "deviationPercent": 24.2,
    "linkedAt": "2026-02-06T10:30:00Z"
  }
}
```

---

### `GET /calculator/track-sync`

Estado de sincronización de pistas entre la calculadora y el Reviewer.

**Response `200 OK`:**

```json
{
  "success": true,
  "data": {
    "synced": 12,
    "pending": 5,
    "tracks": [
      { "track": "Monaco", "calcTrackId": 45, "status": "OK", "lastSync": "2026-02-06" },
      { "track": "Silverstone", "calcTrackId": 22, "status": "OK", "lastSync": "2026-02-05" },
      { "track": "Interlagos", "calcTrackId": null, "status": "PENDING", "lastSync": null }
    ]
  }
}
```

---

## 🔐 Códigos de Error

| Código HTTP | Código Error | Descripción |
|-------------|-------------|-------------|
| `400` | `BAD_REQUEST` | Parámetros inválidos |
| `404` | `NOT_FOUND` | Recurso no encontrado |
| `409` | `DUPLICATE_ENTRY` | Ya existe registro para ese alumno/carrera |
| `422` | `PARSE_FAILED` | No se pudo parsear el texto |
| `422` | `VALIDATION_BLOCKED` | Errores críticos sin resolver |
| `404` | `CALC_NO_DATA` | Sin datos de calculadora para esa pista/temporada |
| `409` | `CALC_ALREADY_LINKED` | Entry ya vinculado con un cálculo |
| `500` | `INTERNAL_ERROR` | Error interno del servidor |

**Formato de error estándar:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_BLOCKED",
    "message": "No se puede guardar: 1 error crítico pendiente",
    "details": ["TYRE_CHECK: Extra Soft a 35°C"]
  }
}
```

---

[→ Siguiente: Wireframes UI](06-wireframes-ui.md)
