# 🛡️ 02 — Validador de Estrategia

[← Volver al Índice](README.md) · [← Parser de Prácticas](01-parser-practicas.md)

---

## Resumen

El Validador de Estrategia es un **motor de reglas anti-error** que revisa la configuración del novato antes de que la aplique en carrera. Detecta errores comunes que cuestan posiciones (o DNFs).

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Setup del   │────►│   Rule Engine    │────►│  Resultado   │
│  Novato      │     │   (4 checks)     │     │  ✅ / 🔴     │
└──────────────┘     └──────────────────┘     └──────────────┘
```

---

## 🏗️ Arquitectura del Validador

```
                    ┌─────────────────────────┐
                    │    STRATEGY INPUT        │
                    │  (Setup + Strategy +     │
                    │   Race Conditions)       │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     RULE ENGINE          │
                    │  (Orquestador)           │
                    └──┬─────────┬──────────┬──────────┬─┘
                       │         │          │          │
              ┌────────▼──┐ ┌───▼────┐ ┌───▼───────┐ ┌▼──────────┐
              │   FUEL    │ │  TYRE  │ │  WEATHER  │ │ CALC      │
              │  CHECK    │ │ CHECK  │ │  CHECK    │ │ CHECK 🆕  │
              └────────┬──┘ └───┬────┘ └───┬───────┘ └┬──────────┘
                       │        │          │          │
                    ┌──▼────────▼──────────▼──────────▼──┐
                    │   VALIDATION RESULT                 │
                    │   Alertas + Sugerencias             │
                    └─────────────────────────────────────┘
```

---

## 🔴 Regla 1: Fuel Check (Combustible)

### Lógica

$$\text{Combustible Necesario} = \text{Consumo por km} \times \text{km totales de carrera}$$

$$\text{Si } \text{Combustible Necesario} > \text{Capacidad del Tanque} \Rightarrow \text{🔴 ALERTA ROJA}$$

### Tabla de Consumo por Compuesto

| Compuesto | Consumo Base (L/km) | Factor Clima Caliente | Factor Lluvia |
|-----------|---------------------|-----------------------|---------------|
| Extra Soft | 0.65 | ×1.10 | ×1.05 |
| Soft | 0.60 | ×1.08 | ×1.03 |
| Medium | 0.55 | ×1.05 | ×1.00 |
| Hard | 0.50 | ×1.03 | ×0.98 |
| Rain | 0.70 | ×1.00 | ×0.95 |

### Pseudocódigo

```
FUNCTION ValidateFuel(strategy, raceInfo) → ValidationResult

    consumoBase = GetConsumoByCompound(strategy.TyreCompound)
    
    // Aplicar factores
    IF raceInfo.Temperature > 30
        consumoBase *= FACTOR_CALIENTE[strategy.TyreCompound]
    IF raceInfo.RainProbability > 50
        consumoBase *= FACTOR_LLUVIA[strategy.TyreCompound]
    
    combustibleNecesario = consumoBase * raceInfo.TotalKm
    
    IF combustibleNecesario > strategy.FuelLoad
        RETURN Error(
            level: "CRITICAL",
            message: "🔴 ¡Te vas a quedar sin combustible!",
            detail: "Necesitas {combustibleNecesario}L pero llevas {strategy.FuelLoad}L",
            suggestion: "Sube el fuel a mínimo {ceil(combustibleNecesario + margen)}L"
        )
    
    IF combustibleNecesario > strategy.FuelLoad * 0.90
        RETURN Warning(
            level: "WARNING",
            message: "🟡 Combustible justo",
            detail: "Margen de solo {diferencia}L",
            suggestion: "Considera subir {sugerencia}L por seguridad"
        )
    
    RETURN OK("✅ Combustible OK — Margen de {margen}L")

END FUNCTION
```

### Ejemplo Visual

```
Caso: Novato pone 30L para una carrera de 55km con Extra Soft

  Cálculo:  0.65 L/km × 55 km = 35.75 L necesarios
  Tanque:   30 L
  
  ┌─────────────────────────────────────────────────┐
  │  🔴 ALERTA CRÍTICA — COMBUSTIBLE                │
  │                                                  │
  │  Necesitas: 35.75 L                             │
  │  Llevas:    30.00 L                             │
  │  Déficit:   -5.75 L                             │
  │                                                  │
  │  ⛽ Te quedarás sin fuel en ~km 46              │
  │                                                  │
  │  💡 Sube el fuel a mínimo 38L (con margen)      │
  └─────────────────────────────────────────────────┘
```

---

## 🔴 Regla 2: Tyre Check (Neumáticos)

### Lógica

Cada compuesto tiene un rango óptimo de temperatura. Usar el compuesto equivocado = degradación masiva.

### Tabla de Rangos Óptimos

| Compuesto | Temp. Mínima | Temp. Óptima | Temp. Máxima | Degradación Fuera de Rango |
|-----------|-------------|-------------|-------------|---------------------------|
| **Extra Soft** | 5°C | 10-20°C | 22°C | MUY ALTA |
| **Soft** | 12°C | 18-26°C | 30°C | ALTA |
| **Medium** | 20°C | 25-33°C | 38°C | MEDIA |
| **Hard** | 28°C | 32-42°C | 48°C | BAJA |
| **Rain** | 5°C | 10-30°C | 35°C | VARIABLE |

### Pseudocódigo

```
FUNCTION ValidateTyres(strategy, conditions) → ValidationResult

    rango = GetOptimalRange(strategy.TyreCompound)
    temp  = conditions.Temperature
    
    IF temp > rango.Max
        deficit = temp - rango.Max
        
        IF deficit > 10
            RETURN Error(
                level: "CRITICAL",
                message: "🔴 ¡Te vas a quedar sin goma en la vuelta 10!",
                detail: "{strategy.TyreCompound} no soporta {temp}°C (máx: {rango.Max}°C)",
                suggestion: "Cambia a {RecommendCompound(temp)}"
            )
        ELSE
            RETURN Warning(
                level: "WARNING",
                message: "🟡 Neumáticos en riesgo",
                detail: "Temperatura {deficit}°C por encima del óptimo",
                suggestion: "Considera usar {RecommendCompound(temp)}"
            )
    
    IF temp < rango.Min
        RETURN Warning(
            level: "WARNING",
            message: "🟡 Neumáticos fríos — poca adherencia",
            detail: "{strategy.TyreCompound} necesita mínimo {rango.Min}°C",
            suggestion: "Baja a {RecommendCompound(temp)} para mejor grip"
        )
    
    RETURN OK("✅ Neumáticos OK — En rango óptimo")

END FUNCTION
```

### Ejemplo Visual

```
Caso: Novato elige Extra Soft con 35°C

  Extra Soft:  Rango óptimo 10-22°C
  Pista:       35°C
  Exceso:      +13°C ❌

  ┌─────────────────────────────────────────────────┐
  │  🔴 ALERTA CRÍTICA — NEUMÁTICOS                 │
  │                                                  │
  │  Compuesto:   Extra Soft                        │
  │  Temp. Pista: 35°C                              │
  │  Máximo:      22°C                              │
  │  Exceso:      +13°C                             │
  │                                                  │
  │  🏎️ ¡Te vas a quedar sin goma en la vuelta 10!  │
  │                                                  │
  │  💡 Para 35°C usa: Medium o Hard                │
  │                                                  │
  │  Escala de degradación:                         │
  │  ████████████████████░░░░  85% desgaste extra   │
  └─────────────────────────────────────────────────┘
```

---

## 🟡 Regla 3: Weather Check (Clima)

### Lógica

```
SI probabilidad_lluvia > 0% Y no hay pitstop con Rain tyres configurado:
    → ALERTA
    
SI probabilidad_lluvia > 60% Y el compuesto principal NO es Rain:
    → ALERTA CRÍTICA
```

### Pseudocódigo

```
FUNCTION ValidateWeather(strategy, forecast) → ValidationResult

    results = []
    
    // Check 1: Lluvia sin pits de lluvia
    IF forecast.RainProbability > 0
        hasPitConLluvia = strategy.PitStops.Any(p => p.TyreCompound == "Rain")
        
        IF NOT hasPitConLluvia
            IF forecast.RainProbability > 60
                results.Add(Error(
                    level: "CRITICAL",
                    message: "🔴 ¡Lluvia muy probable y sin Rain tyres!",
                    detail: "{forecast.RainProbability}% de probabilidad de lluvia",
                    suggestion: "Configura al menos 1 pit stop con Rain tyres"
                ))
            ELSE
                results.Add(Warning(
                    level: "WARNING",
                    message: "🟡 Posible lluvia — sin plan B",
                    detail: "{forecast.RainProbability}% de probabilidad",
                    suggestion: "Considera tener un pit con Rain por precaución"
                ))
    
    // Check 2: Rain tyres sin lluvia
    IF forecast.RainProbability == 0
        usaRain = strategy.TyreCompound == "Rain" 
                  OR strategy.PitStops.Any(p => p.TyreCompound == "Rain")
        
        IF usaRain
            results.Add(Warning(
                level: "WARNING",
                message: "🟡 Rain tyres sin lluvia prevista",
                detail: "Los Rain son más lentos en seco",
                suggestion: "Cambia a un compuesto de seco"
            ))
    
    IF results.IsEmpty()
        RETURN OK("✅ Estrategia climática OK")
    
    RETURN results

END FUNCTION
```

### Ejemplo Visual

```
Caso: 40% de lluvia, sin Rain tyres en pit stops

  ┌─────────────────────────────────────────────────┐
  │  🟡 ALERTA — CLIMA                              │
  │                                                  │
  │  🌧️ Probabilidad de lluvia: 40%                 │
  │                                                  │
  │  Pit Stops configurados:                        │
  │    Pit 1 (vuelta 15): Soft    ← Sin Rain ⚠️    │
  │    Pit 2 (vuelta 35): Medium  ← Sin Rain ⚠️    │
  │                                                  │
  │  💡 Configura al menos 1 pit con Rain tyres     │
  │     como plan de contingencia                    │
  └─────────────────────────────────────────────────┘
```

---

## 🔗 Regla 4: Calculator Check (Cruce con Calculadora) 🆕

### Lógica

Si la calculadora tiene datos para esta pista/temporada, comparar automáticamente el setup del novato contra lo que la calculadora calculó como óptimo.

$$\text{Desviación} = \frac{1}{N} \sum_{i=1}^{N} \left| \frac{\text{Setup}_{alumno,i} - \text{Setup}_{calc,i}}{\text{Setup}_{calc,i}} \right| \times 100$$

### Pseudocódigo

```
FUNCTION ValidateAgainstCalculator(setup, raceInfo) → ValidationResult

    // 1. Buscar setup calculado para esta pista
    calcSetup = CalcDB.GetLatestSetup(raceInfo.TrackId, raceInfo.SeasonId)
    
    IF calcSetup IS NULL
        RETURN Info(
            level: "INFO",
            message: "ℹ️ Sin datos de calculadora para esta pista",
            suggestion: "Usa la calculadora para generar un setup de referencia"
        )
    
    // 2. Calcular desviación por componente
    deviations = {
        wings:      |setup.Wings - calcSetup.Wings| / calcSetup.Wings * 100,
        engine:     |setup.Engine - calcSetup.Engine| / calcSetup.Engine * 100,
        brakes:     |setup.Brakes - calcSetup.Brakes| / calcSetup.Brakes * 100,
        gear:       |setup.Gear - calcSetup.Gear| / calcSetup.Gear * 100,
        suspension: |setup.Suspension - calcSetup.Suspension| / calcSetup.Suspension * 100
    }
    
    avgDeviation = Average(deviations)
    
    // 3. Evaluar
    IF avgDeviation > 20
        RETURN Warning(
            level: "WARNING",
            message: "🟡 Setup muy desviado de la calculadora",
            detail: "Desviación promedio: {avgDeviation}%",
            suggestion: "Revisa los valores — la calculadora sugiere W:{calcSetup.Wings}...",
            deviations: deviations
        )
    
    IF avgDeviation > 10
        RETURN Info(
            level: "INFO",
            message: "ℹ️ Diferencias moderadas con la calculadora",
            detail: "Desviación: {avgDeviation}%",
            deviations: deviations
        )
    
    RETURN OK("✅ Setup alineado con la calculadora (desvío: {avgDeviation}%)")

END FUNCTION
```

### Ejemplo Visual

```
Caso: Novato pone Wings: 35, Calculadora dice Wings: 52

  ┌─────────────────────────────────────────────────┐
  │  🟡 AVISO — CRUCE CON CALCULADORA               │
  │                                                  │
  │  🔗 La calculadora tiene datos para Monaco S98  │
  │                                                  │
  │  Componente │ Calculado │  Tuyo  │ Desvío        │
  │  ───────────┼───────────┼────────┼──────────     │
  │  Wings      │    52     │   35   │ 🔴 32.7%     │
  │  Engine     │   700     │  800   │ 🔴 14.3%     │
  │  Brakes     │   345     │  280   │ 🔴 18.8%     │
  │  Gear       │   155     │  120   │ 🔴 22.6%     │
  │  Suspension │    68     │   90   │ 🔴 32.4%     │
  │  ───────────┼───────────┼────────┼──────────     │
  │  Promedio   │           │        │ 🔴 24.2%     │
  │                                                  │
  │  💡 Tu setup se desvía mucho de lo calculado.   │
  │     Consulta con tu mentor antes de aplicarlo.   │
  └─────────────────────────────────────────────────┘
```

> **Nota:** Esta regla es informativa (`WARNING`), no bloquea el registro. El novato puede tener razones válidas para desviarse de la calculadora.

---

## 🎯 Resumen de Severidades

| Icono | Nivel | Acción | ¿Bloquea registro? |
|-------|-------|--------|---------------------|
| 🔴 | CRITICAL | El novato DEBE corregir | ✅ Sí — No puede guardar |
| 🟡 | WARNING | Recomendación fuerte | ❌ No — Puede guardar con confirmación |
| ✅ | OK | Todo correcto | ❌ No — Guardar libre |

---

## 🎨 Interfaz del Validador

```
┌─────────────────────────────────────────────────────────────┐
│  🛡️ Validador de Estrategia                          [?]   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Carrera: GP de Mónaco — 65 vueltas — 213.87 km            │
│  Alumno: @NovatorX                                          │
│                                                             │
│  ┌─── Resultados de Validación ─────────────────────────┐  │
│  │                                                       │  │
│  │  ⛽ Combustible                                       │  │
│  │  ✅ OK — Llevas 45L, necesitas 38.2L (margen: 6.8L) │  │
│  │                                                       │  │
│  │  🏎️ Neumáticos                                       │  │
│  │  🔴 CRÍTICO — Extra Soft a 35°C                      │  │
│  │  ¡Te vas a quedar sin goma en la vuelta 10!          │  │
│  │  → Sugerencia: Usa Medium o Hard                     │  │
│  │                                                       │  │
│  │  🌧️ Clima                                            │  │
│  │  🟡 WARNING — 40% lluvia sin Rain en pits            │  │
│  │  → Sugerencia: Agrega Rain en Pit 2                  │  │
│  │                                                       │  │
│  │  🔗 Calculadora                                       │  │
│  │  🟡 WARNING — Desviación 24.2% vs setup calculado    │  │
│  │  → Sugerencia: Revisa Wings y Gear                   │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Estado: 🔴 NO VÁLIDO — 1 error crítico                    │
│                                                             │
│  [ 🔄 Revalidar ]  [ ❌ Corregir Errores ]                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 📐 Reglas Extendibles

El motor de reglas está diseñado para crecer. Futuras reglas posibles:

| Regla | Descripción |
|-------|-------------|
| **Pit Timing** | Validar que los pit stops estén bien distribuidos |
| **Risk Level** | Comparar agresividad del setup vs nivel del piloto |
| **Track History** | Si en la misma pista siempre falla algo, alertar |
| ~~**Baseline Check**~~ | ~~Comparar contra la "Base de Oro" del mentor~~ → ✅ Implementado como Regla 4 |

---

[→ Siguiente: Registro Histórico](03-registro-historico.md)
