# 🔍 01 — Parser de Prácticas

[← Volver al Índice](README.md)

---

## Resumen

El Parser de Prácticas permite al novato **pegar el texto crudo** de la página de resultados de práctica de GPRO y extraer automáticamente todos los valores relevantes del setup y el feedback del piloto.

```
┌─────────────────────┐       ┌──────────────┐       ┌──────────────┐
│  Texto crudo GPRO   │──────►│    Parser    │──────►│ Setup Object │
│  (pegado por novato)│       │    Engine    │       │  (validado)  │
└─────────────────────┘       └──────────────┘       └──────────────┘
```

---

## 📥 Entrada: Texto Crudo de GPRO

El novato copia y pega algo similar a esto desde la página de prácticas:

```
Practice results

Lap  Time       Mistake  Net time
1    1:22.456   0.120s   1:22.576

Setup used:
Wings: 45
Engine: 680
Brakes: 320
Gear: 140
Suspension: 72

Tyres used: Extra Soft
Fuel load: 35 lts

Weather: 28°C, Humidity 45%, Clouds: 10%

Driver feedback:
"The car understeers in slow corners. I need more front wing. 
The brakes feel too soft. The engine is running too hot."
```

> ⚠️ **Nota:** El formato puede variar ligeramente entre temporadas de GPRO. El parser debe ser tolerante a variaciones.

---

## 🧠 Lógica de Extracción

### Campos a Extraer

| Campo | Regex Pattern | Ejemplo Match |
|-------|--------------|---------------|
| **Wings** | `Wings?:\s*(\d+)` | `Wings: 45` → `45` |
| **Engine** | `Engine:\s*(\d+)` | `Engine: 680` → `680` |
| **Brakes** | `Brakes?:\s*(\d+)` | `Brakes: 320` → `320` |
| **Gear** | `Gear:\s*(\d+)` | `Gear: 140` → `140` |
| **Suspension** | `Suspension:\s*(\d+)` | `Suspension: 72` → `72` |
| **Tyres** | `Tyres?\s*(?:used)?:\s*(.+)` | `Tyres used: Extra Soft` → `Extra Soft` |
| **Fuel** | `Fuel\s*(?:load)?:\s*(\d+)` | `Fuel load: 35` → `35` |
| **Temperature** | `(\d+)\s*°?\s*C` | `28°C` → `28` |
| **Humidity** | `Humidity\s*:?\s*(\d+)\s*%` | `Humidity 45%` → `45` |
| **Lap Time** | `(\d+:\d{2}\.\d{3})` | `1:22.456` → `1:22.456` |
| **Driver Feedback** | `(?:feedback|comment)[:\s]*"?(.+?)"?$` | Texto del piloto |

### Feedback del Piloto — Tabla de Interpretación

El feedback del piloto se traduce a **acciones concretas sobre el setup**:

| Frase del Piloto | Componente | Acción |
|-------------------|-----------|--------|
| *"I need more wing"* | Wings | ⬆️ Subir 1-3 puntos |
| *"Too much wing"* / *"The car is too slow on straights"* | Wings | ⬇️ Bajar 1-3 puntos |
| *"The car understeers"* | Wings | ⬆️ Subir front wing |
| *"The car oversteers"* | Wings | ⬇️ Bajar front wing |
| *"Brakes feel too soft"* | Brakes | ⬆️ Subir brakes |
| *"Brakes are locking up"* | Brakes | ⬇️ Bajar brakes |
| *"Engine is too hot"* / *"running hot"* | Engine | ⬇️ Bajar engine |
| *"Engine is too cool"* | Engine | ⬆️ Subir engine |
| *"The car is perfect"* | Todos | ✅ Sin cambios |
| *"Gearbox feels wrong"* | Gear | 🔄 Ajustar gear |
| *"Suspension too stiff"* | Suspension | ⬇️ Bajar suspension |
| *"Suspension too soft"* | Suspension | ⬆️ Subir suspension |

---

## ⚙️ Pseudocódigo del Parser

```
FUNCTION ParsePracticeText(rawText: string) → PracticeResult

    result = new PracticeResult()

    // 1. Extraer Setup
    result.Wings      = ExtractNumber(rawText, "Wings?:\s*(\d+)")
    result.Engine     = ExtractNumber(rawText, "Engine:\s*(\d+)")
    result.Brakes     = ExtractNumber(rawText, "Brakes?:\s*(\d+)")
    result.Gear       = ExtractNumber(rawText, "Gear:\s*(\d+)")
    result.Suspension = ExtractNumber(rawText, "Suspension:\s*(\d+)")
    
    // 2. Extraer Estrategia
    result.TyreCompound = ExtractString(rawText, "Tyres?\s*(?:used)?:\s*(.+)")
    result.FuelLoad     = ExtractNumber(rawText, "Fuel\s*(?:load)?:\s*(\d+)")
    
    // 3. Extraer Condiciones
    result.Temperature = ExtractNumber(rawText, "(\d+)\s*°?\s*C")
    result.Humidity    = ExtractNumber(rawText, "Humidity\s*:?\s*(\d+)")
    
    // 4. Extraer Tiempos de Vuelta
    result.LapTimes = ExtractAllMatches(rawText, "(\d+:\d{2}\.\d{3})")
    
    // 5. Extraer y Analizar Feedback
    feedbackText = ExtractFeedbackBlock(rawText)
    result.Feedback = AnalyzeFeedback(feedbackText)
    
    // 6. Validar completitud
    IF result.HasMissingFields()
        result.Warnings.Add("Campos faltantes: " + result.GetMissingFields())
    
    RETURN result

END FUNCTION
```

---

## 🔄 Flujo del Parser

```
                    ┌──────────────────────┐
                    │  Novato pega texto   │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  ¿Texto válido?      │
                    │  (tiene estructura   │
                    │   reconocible)       │
                    └────┬────────────┬────┘
                         │            │
                    Sí   │            │  No
                         │            │
              ┌──────────▼──┐   ┌─────▼──────────┐
              │  Ejecutar   │   │  Mostrar error  │
              │  Regex por  │   │  "Formato no    │
              │  cada campo │   │   reconocido"   │
              └──────────┬──┘   └────────────────┘
                         │
              ┌──────────▼──────────┐
              │  ¿Todos los campos  │
              │   encontrados?      │
              └────┬───────────┬────┘
                   │           │
              Sí   │           │  No
                   │           │
          ┌────────▼──┐   ┌────▼──────────┐
          │  Parseo   │   │  Parseo con   │
          │  completo │   │  warnings de  │
          │  ✅       │   │  campos       │
          └────────┬──┘   │  faltantes ⚠️ │
                   │      └────┬──────────┘
                   │           │
              ┌────▼───────────▼────┐
              │  Analizar Feedback  │
              │  del piloto         │
              └────────┬────────────┘
                       │
              ┌────────▼────────────┐
              │  Retornar           │
              │  PracticeResult     │
              │  + Sugerencias      │
              └─────────────────────┘
```

---

## 📦 Objeto de Salida

```json
{
  "setup": {
    "wings": 45,
    "engine": 680,
    "brakes": 320,
    "gear": 140,
    "suspension": 72
  },
  "strategy": {
    "tyreCompound": "Extra Soft",
    "fuelLoad": 35
  },
  "conditions": {
    "temperature": 28,
    "humidity": 45,
    "clouds": 10
  },
  "performance": {
    "bestLap": "1:22.456",
    "netTime": "1:22.576",
    "mistake": 0.120
  },
  "feedback": {
    "raw": "The car understeers in slow corners...",
    "suggestions": [
      {
        "component": "Wings",
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
  "warnings": [],
  "parseConfidence": 0.95,
  "calculatorComparison": {
    "available": true,
    "calcSetup": { "wings": 52, "engine": 700, "brakes": 345, "gear": 155, "suspension": 68 },
    "avgDeviation": 24.2,
    "note": "🔗 Datos cruzados automáticamente con la calculadora"
  }
}
```

---

## 🎨 Interfaz del Parser

```
┌─────────────────────────────────────────────────────────────┐
│  🔍 Parser de Prácticas                              [?]   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  📋 Pega aquí el texto de tus prácticas de GPRO:           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │  Practice results                                   │   │
│  │  Lap  Time       Mistake  Net time                  │   │
│  │  1    1:22.456   0.120s   1:22.576                  │   │
│  │  ...                                                │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  [ 🔍 Parsear Datos ]  [ 🗑️ Limpiar ]                      │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  ✅ Datos Extraídos:                                       │
│                                                             │
│  Setup                    │  Estrategia                     │
│  ────────────────────     │  ────────────────               │
│  Wings:      45           │  Neumáticos: Extra Soft         │
│  Engine:    680           │  Combustible: 35 lts            │
│  Brakes:   320           │                                  │
│  Gear:     140           │  Condiciones                     │
│  Suspension: 72          │  ────────────────                │
│                           │  Temp: 28°C                     │
│                           │  Humedad: 45%                   │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  💡 Sugerencias del Feedback:                              │
│                                                             │
│  🔴 ALTA   Wings ⬆️  — Substeering detectado               │
│  🟡 MEDIA  Brakes ⬆️ — Frenos muy blandos                  │
│                                                             │
│  [ 💾 Guardar Setup ]  [ ➡️ Enviar a Validador ]            │
│  [ 🔗 Comparar con Calculadora ]                            │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚠️ Casos Borde

| Caso | Manejo |
|------|--------|
| Texto vacío | Error: "Pega el texto de tus prácticas" |
| Texto sin estructura GPRO | Error: "No se reconoce el formato" |
| Solo algunos campos | Warning + parseo parcial |
| Feedback en español | Soporte bilingüe de keywords |
| Múltiples laps | Extraer todos, usar mejor tiempo |
| Valores fuera de rango (ej: Wings: 999) | Warning: "Valor inusual" |

---

[→ Siguiente: Validador de Estrategia](02-validador-estrategia.md)
