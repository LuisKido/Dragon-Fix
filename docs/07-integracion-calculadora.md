# 🔗 07 — Integración con Calculadora de Setups

[← Volver al Índice](README.md) · [← Wireframes UI](06-wireframes-ui.md)

---

## Resumen

La aplicación **Dragon App** ya cuenta con una **calculadora de setups** que va calculando configuraciones óptimas para cada pista. Este documento describe cómo el módulo **Fix Academy** se integra con esa calculadora existente, cruzando datos para potenciar ambas herramientas.

```
┌──────────────────┐                    ┌──────────────────┐
│   CALCULADORA    │                    │  FIX ACADEMY     │
│   (ya existe)    │◄──── CRUCE ───────►│  (módulo nuevo)  │
│                  │      DE DATOS      │                  │
│  • Calcula setup │                    │  • Parsea texto  │
│  • BD propia     │                    │  • Valida reglas │
│  • Histórico     │                    │  • Compara alum. │
└────────┬─────────┘                    └────────┬─────────┘
         │                                       │
         └──────────────┬────────────────────────┘
                        │
               ┌────────▼────────┐
               │   SQL Server    │
               │   (BD Unificada │
               │   o Cruzada)    │
               └─────────────────┘
```

---

## 🎯 ¿Por qué cruzar los datos?

| Sin integración ❌ | Con integración ✅ |
|---------------------|---------------------|
| Calculadora y Fix Academy trabajan aislados | Los setups calculados alimentan directamente al validador |
| El mentor carga la Base de Oro manualmente | La Base de Oro se auto-genera desde la calculadora |
| El novato tiene que copiar datos entre sistemas | Un solo flujo: calculadora → parser → validación |
| No se sabe si el setup del novato es "suyo" o "calculado" | Se registra la fuente: `CALCULATOR` / `PARSED` / `MANUAL` |
| Sin comparativa entre lo calculado y lo real | Puedes comparar: setup calculado vs setup usado vs resultado |

---

## 🏗️ Estrategias de Integración

### Opción A: Base de Datos Compartida (Recomendada)

Si ambos módulos viven en la misma app, comparten el mismo SQL Server:

```
┌─────────────────────────────────────────────────┐
│                SQL Server                        │
│                                                  │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │  Tablas del       │  │  Tablas de         │ │
│  │  Calculadora     │  │  Fix Academy        │ │
│  │  ────────────    │  │  ────────────────    │ │
│  │  CalcSessions    │  │  StudentRaceEntries  │ │
│  │  CalcSetups      │◄─┤  Setups              │ │
│  │  CalcParameters  │  │  SetupBaselines      │ │
│  │  TrackProfiles   │──►  Strategies           │ │
│  │                  │  │  ...                  │ │
│  └──────────────────┘  └──────────────────────┘ │
│            │                      │              │
│            └──────────┬───────────┘              │
│                       │                          │
│              ┌────────▼────────┐                 │
│              │  Vistas (Views) │                 │
│              │  de cruce       │                 │
│              └─────────────────┘                 │
└──────────────────────────────────────────────────┘
```

### Opción B: API Bridge (Si son apps separadas)

Si la calculadora vive en otra app o servicio:

```
┌───────────────┐     ┌─────────────┐     ┌────────────────┐
│  Calculadora  │────►│  API Bridge │◄────│ Fix Academy    │
│  (App A)      │◄────│  (Servicio) │────►│ (App B)        │
└───────────────┘     └─────────────┘     └────────────────┘
```

---

## 📊 Mapeo de Datos: Calculadora → Fix Academy

### Tabla de Equivalencias

La calculadora ya tiene datos que Fix Academy necesita. Este es el mapeo:

| Dato de la Calculadora | Uso en Fix Academy | Relación |
|------------------------|--------------------|----------|
| **Setup calculado por pista** | Base de Oro automática | `CalcSetups` → `SetupBaselines` |
| **Parámetros de pista** (longitud, vueltas, tanque) | Validación de fuel | `TrackProfiles` → `Tracks` |
| **Historial de cálculos** del alumno | Comparar calculado vs usado | `CalcSessions` → `StudentRaceEntries` |
| **Consumo estimado** | Fuel Check más preciso | `CalcParameters` → `FuelValidator` |
| **Temperatura óptima por compuesto** | Tyre Check | `CalcParameters` → `TyreValidator` |
| **Setups previos exitosos** | Sugerencias automáticas | `CalcSetups` (histórico) → `ComparisonEngine` |

---

## 🗄️ Tablas de Enlace (Nuevas)

### `CalcSetupLink` — Vincula setup calculado con el entry del alumno

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Registro del alumno |
| `CalcSessionId` | `INT` | FK → CalcSessions* | Sesión de la calculadora |
| `CalcWings` | `INT` | NOT NULL | Wings que calculó la herramienta |
| `CalcEngine` | `INT` | NOT NULL | Engine calculado |
| `CalcBrakes` | `INT` | NOT NULL | Brakes calculado |
| `CalcGear` | `INT` | NOT NULL | Gear calculado |
| `CalcSuspension` | `INT` | NOT NULL | Suspension calculado |
| `DeviationPercent` | `DECIMAL(5,2)` | NULL | Desviación del alumno vs calculado |
| `LinkedAt` | `DATETIME2` | NOT NULL | Fecha de vinculación |

> *`CalcSessions` = tabla existente de la calculadora (no se modifica).

### `CalcTrackSync` — Sincroniza datos de pista entre ambos sistemas

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `ReviewerTrackId` | `INT` | FK → Tracks | Pista en Fix Academy |
| `CalcTrackId` | `INT` | NOT NULL | ID de la pista en la calculadora |
| `LastSyncAt` | `DATETIME2` | NOT NULL | Última sincronización |
| `SyncStatus` | `NVARCHAR(20)` | NOT NULL | OK / STALE / ERROR |

---

## 🔄 Flujos de Integración

### Flujo 1: Base de Oro Auto-generada desde Calculadora

```
┌─────────────────┐
│  Mentor usa la  │
│  calculadora    │
│  para la pista  │
└────────┬────────┘
         │
┌────────▼────────┐     ┌──────────────────────────┐
│  Calculadora    │────►│  Fix Academy importa       │
│  genera setup   │     │  como Base de Oro         │
│  óptimo         │     │  automáticamente          │
└─────────────────┘     └──────────────────────────┘

Trigger: Al guardar un cálculo, si el mentor tiene flag "auto-baseline":
  → INSERT/UPDATE en SetupBaselines con Source = 'CALCULATOR'
```

### Flujo 2: Novato compara su setup vs el cálculo

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│ Novato pega  │────►│  Parser      │────►│  Cruce con       │
│ su práctica  │     │  extrae      │     │  Calculadora     │
└──────────────┘     │  setup       │     └────────┬─────────┘
                     └──────────────┘              │
                                          ┌────────▼─────────┐
                                          │  Comparativa:    │
                                          │                  │
                                          │  Calculado: W:52 │
                                          │  Parseado:  W:45 │
                                          │  Diff:      -7   │
                                          │  Desvío:  13.5%  │
                                          └──────────────────┘
```

### Flujo 3: Fuel Check potenciado con datos de calculadora

```
ANTES (sin calculadora):
  consumo = tabla estática por compuesto

AHORA (con calculadora):
  consumo = CalcParameters.FuelConsumption  ← dato real de la calculadora
            para esa pista + compuesto + condiciones

  → Validación mucho más precisa
```

---

## 🔧 Pseudocódigo de Integración

### Auto-Baseline desde Calculadora

```
FUNCTION SyncBaselineFromCalculator(mentorId, trackId, seasonId)

    // 1. Buscar el último cálculo del mentor para esa pista
    calcSetup = CalcDB.GetLatestSetup(mentorId, trackId, seasonId)
    
    IF calcSetup IS NULL
        RETURN "No hay cálculo disponible para esta pista"
    
    // 2. Crear o actualizar la Base de Oro
    baseline = SetupBaselines.FindOrCreate(trackId, seasonId, mentorId)
    
    baseline.Wings      = calcSetup.Wings
    baseline.Engine     = calcSetup.Engine
    baseline.Brakes     = calcSetup.Brakes
    baseline.Gear       = calcSetup.Gear
    baseline.Suspension = calcSetup.Suspension
    baseline.Source     = "CALCULATOR"
    baseline.CalcSessionId = calcSetup.SessionId
    baseline.Notes      = "Auto-importado desde calculadora — " + NOW()
    
    baseline.Save()
    
    RETURN "Base de Oro actualizada desde calculadora ✅"

END FUNCTION
```

### Comparar Setup del Novato vs Calculadora

```
FUNCTION CompareWithCalculator(entryId) → ComparisonResult

    entry = StudentRaceEntries.Get(entryId)
    setup = Setups.GetByEntry(entryId)
    
    // 1. Obtener el setup calculado para esta pista/temporada
    race = Races.Get(entry.RaceId)
    calcSetup = CalcDB.GetLatestSetup(
        trackId: race.TrackId, 
        seasonId: race.SeasonId
    )
    
    IF calcSetup IS NULL
        RETURN { available: false, message: "Sin datos de calculadora" }
    
    // 2. Calcular desviaciones
    deviations = {
        wings:      CalcDeviation(setup.Wings, calcSetup.Wings),
        engine:     CalcDeviation(setup.Engine, calcSetup.Engine),
        brakes:     CalcDeviation(setup.Brakes, calcSetup.Brakes),
        gear:       CalcDeviation(setup.Gear, calcSetup.Gear),
        suspension: CalcDeviation(setup.Suspension, calcSetup.Suspension)
    }
    
    avgDeviation = Average(deviations.Values)
    
    // 3. Registrar el cruce
    CalcSetupLink.Create({
        EntryId:           entryId,
        CalcSessionId:     calcSetup.SessionId,
        CalcWings:         calcSetup.Wings,
        CalcEngine:        calcSetup.Engine,
        CalcBrakes:        calcSetup.Brakes,
        CalcGear:          calcSetup.Gear,
        CalcSuspension:    calcSetup.Suspension,
        DeviationPercent:  avgDeviation
    })
    
    RETURN {
        available: true,
        calculated: calcSetup,
        actual: setup,
        deviations: deviations,
        avgDeviation: avgDeviation,
        status: avgDeviation > 10 ? "HIGH_DEVIATION" : "OK"
    }

END FUNCTION
```

---

## 📐 Consulta SQL: Setup Calculado vs Usado vs Resultado

```sql
-- Triple comparativa: ¿El novato usó lo que calculó? ¿Le fue bien?
SELECT 
    s.Username,
    -- Setup que usó
    su.Wings AS UsedWings, su.Engine AS UsedEngine, 
    su.Brakes AS UsedBrakes, su.Gear AS UsedGear, su.Suspension AS UsedSusp,
    -- Setup calculado
    cl.CalcWings, cl.CalcEngine, cl.CalcBrakes, cl.CalcGear, cl.CalcSuspension,
    -- Desviación
    cl.DeviationPercent,
    -- Resultado
    rr.StartPosition, rr.FinishPosition, rr.PositionsGained,
    -- ¿Seguir la calculadora le ayudó?
    CASE 
        WHEN cl.DeviationPercent < 5 AND rr.PositionsGained > 0 
            THEN '✅ Siguió cálculo + ganó posiciones'
        WHEN cl.DeviationPercent > 15 AND rr.PositionsGained < 0 
            THEN '🔴 Ignoró cálculo + perdió posiciones'
        WHEN cl.DeviationPercent > 15 AND rr.PositionsGained > 0 
            THEN '🟡 Ignoró cálculo pero le fue bien'
        ELSE '📊 Neutral'
    END AS Insight
FROM StudentRaceEntries e
JOIN Students s ON e.StudentId = s.Id
JOIN Setups su ON su.EntryId = e.Id
LEFT JOIN CalcSetupLink cl ON cl.EntryId = e.Id
LEFT JOIN RaceResults rr ON rr.EntryId = e.Id
WHERE e.RaceId = @RaceId
ORDER BY cl.DeviationPercent ASC;
```

---

## 🎨 Interfaz: Panel de Cruce con Calculadora

```
┌─────────────────────────────────────────────────────────────────┐
│  🔗 Cruce con Calculadora                              [🔄]   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alumno: @Carlos    Pista: Monaco    Temporada: S98            │
│                                                                 │
│  ┌── Comparativa Setup ─────────────────────────────────────┐  │
│  │                                                           │  │
│  │  Componente │ Calculadora │  Usado  │  Diff  │ Desvío    │  │
│  │  ───────────┼─────────────┼─────────┼────────┼─────────  │  │
│  │  Wings      │     52      │   35    │  -17   │ 🔴 32.7%  │  │
│  │  Engine     │    700      │  800    │ +100   │ 🔴 14.3%  │  │
│  │  Brakes     │    345      │  280    │  -65   │ 🔴 18.8%  │  │
│  │  Gear       │    155      │  120    │  -35   │ 🔴 22.6%  │  │
│  │  Suspension │     68      │   90    │  +22   │ 🔴 32.4%  │  │
│  │  ───────────┼─────────────┼─────────┼────────┼─────────  │  │
│  │  PROMEDIO   │             │         │        │ 🔴 24.2%  │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌── 📊 Historial: ¿Seguir la calculadora le ayuda? ────────┐ │
│  │                                                           │  │
│  │  Carrera │ Desvío Calc │ Posiciones │ Insight              │  │
│  │  ────────┼─────────────┼────────────┼──────────────────── │  │
│  │  C1      │    28.0%    │    +3      │ 🟡 Ignoró pero OK  │  │
│  │  C2      │    18.0%    │    +2      │ 🟡 Ignoró pero OK  │  │
│  │  C3      │    12.0%    │    +2      │ 📊 Neutral          │  │
│  │  C4      │     8.0%    │    +1      │ ✅ Siguió + mejoró  │  │
│  │  C5      │     5.0%    │    +1      │ ✅ Siguió + mejoró  │  │
│  │                                                           │  │
│  │  💡 Tendencia: A medida que Carlos sigue más la          │  │
│  │     calculadora, sus resultados mejoran.                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────┐  ┌───────────────────────┐                │
│  │ 📥 Importar como│  │ 📊 Ver en Dashboard   │                │
│  │    Base de Oro   │  │    del Mentor         │                │
│  └─────────────────┘  └───────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Configuración de Sincronización

El mentor configura cómo se cruzan los datos:

```json
{
  "calculatorSync": {
    "enabled": true,
    "autoBaseline": true,
    "baselineSource": "CALCULATOR",
    "syncFrequency": "ON_CALC_SAVE",
    "deviationThreshold": 10,
    "showCalcComparison": true,
    "fuelDataSource": "CALCULATOR",
    "fallbackToStatic": true
  }
}
```

| Opción | Descripción |
|--------|-------------|
| `autoBaseline` | Auto-actualiza Base de Oro al guardar cálculo |
| `baselineSource` | De dónde viene la baseline: `CALCULATOR` / `MANUAL` / `BEST_STUDENT` |
| `syncFrequency` | Cuándo sincronizar: `ON_CALC_SAVE` / `ON_RACE_CREATE` / `MANUAL` |
| `deviationThreshold` | % de desviación que dispara alertas |
| `fuelDataSource` | Si usar consumo de la calculadora o tabla estática |
| `fallbackToStatic` | Si no hay dato de calculadora, usar tabla estática |

---

## 🔮 Valor Agregado del Cruce

| Métrica | Sin Calculadora | Con Calculadora |
|---------|----------------|-----------------|
| Precisión Fuel Check | ±15% (tabla estática) | ±3% (dato calculado) |
| Base de Oro | Manual por el mentor | Auto desde calculadora |
| Fuente del setup | No se sabe | `CALCULATOR` / `PARSED` / `MANUAL` |
| Insight de resultados | Solo posiciones | Correlación desvío↔resultado |
| Tiempo del mentor | Cargar baseline a mano | Auto-sync |

---

[← Volver al Índice](README.md)
