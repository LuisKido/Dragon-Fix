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

### Código

```csharp
public ValidationResult ValidateFuel(Strategy strategy, RaceInfo raceInfo)
{
    var consumoBase = GetConsumoByCompound(strategy.TyreCompound);

    // Aplicar factores
    if (raceInfo.Temperature > 30)
        consumoBase *= FactorCaliente[strategy.TyreCompound];
    if (raceInfo.RainProbability > 50)
        consumoBase *= FactorLluvia[strategy.TyreCompound];

    var combustibleNecesario = consumoBase * raceInfo.TotalKm;

    if (combustibleNecesario > strategy.FuelLoad)
    {
        var deficit = combustibleNecesario - strategy.FuelLoad;
        return ValidationResult.Error(
            level: Severity.Critical,
            message: "🔴 ¡Te vas a quedar sin combustible!",
            detail: $"Necesitas {combustibleNecesario:F1}L pero llevas {strategy.FuelLoad}L",
            suggestion: $"Sube el fuel a mínimo {Math.Ceiling(combustibleNecesario + MargenSeguridad)}L"
        );
    }

    if (combustibleNecesario > strategy.FuelLoad * 0.90m)
    {
        var diferencia = strategy.FuelLoad - combustibleNecesario;
        return ValidationResult.Warning(
            level: Severity.Warning,
            message: "🟡 Combustible justo",
            detail: $"Margen de solo {diferencia:F1}L",
            suggestion: $"Considera subir {Math.Ceiling(MargenSeguridad - diferencia)}L por seguridad"
        );
    }

    var margen = strategy.FuelLoad - combustibleNecesario;
    return ValidationResult.Ok($"✅ Combustible OK — Margen de {margen:F1}L");
}
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

### Código

```csharp
public ValidationResult ValidateTyres(Strategy strategy, WeatherConditions conditions)
{
    var rango = GetOptimalRange(strategy.TyreCompound);
    var temp = conditions.Temperature;

    if (temp > rango.Max)
    {
        var deficit = temp - rango.Max;

        if (deficit > 10)
        {
            return ValidationResult.Error(
                level: Severity.Critical,
                message: "🔴 ¡Te vas a quedar sin goma en la vuelta 10!",
                detail: $"{strategy.TyreCompound} no soporta {temp}°C (máx: {rango.Max}°C)",
                suggestion: $"Cambia a {RecommendCompound(temp)}"
            );
        }

        return ValidationResult.Warning(
            level: Severity.Warning,
            message: "🟡 Neumáticos en riesgo",
            detail: $"Temperatura {deficit}°C por encima del óptimo",
            suggestion: $"Considera usar {RecommendCompound(temp)}"
        );
    }

    if (temp < rango.Min)
    {
        return ValidationResult.Warning(
            level: Severity.Warning,
            message: "🟡 Neumáticos fríos — poca adherencia",
            detail: $"{strategy.TyreCompound} necesita mínimo {rango.Min}°C",
            suggestion: $"Baja a {RecommendCompound(temp)} para mejor grip"
        );
    }

    return ValidationResult.Ok("✅ Neumáticos OK — En rango óptimo");
}
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

```csharp
// Si hay probabilidad de lluvia y no hay pit stop con Rain tyres:
if (forecast.RainProbability > 0 && !strategy.PitStops.Any(p => p.TyreCompound == "Rain"))
    // → ALERTA

// Si la probabilidad es alta y el compuesto principal no es Rain:
if (forecast.RainProbability > 60 && strategy.TyreCompound != "Rain")
    // → ALERTA CRÍTICA
```

### Código

```csharp
public List<ValidationResult> ValidateWeather(Strategy strategy, WeatherForecast forecast)
{
    var results = new List<ValidationResult>();

    // Check 1: Lluvia sin pits de lluvia
    if (forecast.RainProbability > 0)
    {
        var hasPitConLluvia = strategy.PitStops.Any(p => p.TyreCompound == "Rain");

        if (!hasPitConLluvia)
        {
            if (forecast.RainProbability > 60)
            {
                results.Add(ValidationResult.Error(
                    level: Severity.Critical,
                    message: "🔴 ¡Lluvia muy probable y sin Rain tyres!",
                    detail: $"{forecast.RainProbability}% de probabilidad de lluvia",
                    suggestion: "Configura al menos 1 pit stop con Rain tyres"
                ));
            }
            else
            {
                results.Add(ValidationResult.Warning(
                    level: Severity.Warning,
                    message: "🟡 Posible lluvia — sin plan B",
                    detail: $"{forecast.RainProbability}% de probabilidad",
                    suggestion: "Considera tener un pit con Rain por precaución"
                ));
            }
        }
    }

    // Check 2: Rain tyres sin lluvia
    if (forecast.RainProbability == 0)
    {
        var usaRain = strategy.TyreCompound == "Rain"
                      || strategy.PitStops.Any(p => p.TyreCompound == "Rain");

        if (usaRain)
        {
            results.Add(ValidationResult.Warning(
                level: Severity.Warning,
                message: "🟡 Rain tyres sin lluvia prevista",
                detail: "Los Rain son más lentos en seco",
                suggestion: "Cambia a un compuesto de seco"
            ));
        }
    }

    if (!results.Any())
        results.Add(ValidationResult.Ok("✅ Estrategia climática OK"));

    return results;
}
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

$$\text{Desviación} = \frac{1}{6} \sum_{i=1}^{6} \left| \frac{\text{Setup}_{alumno,i} - \text{Setup}_{calc,i}}{\text{Setup}_{calc,i}} \right| \times 100$$

### Código

```csharp
public ValidationResult ValidateAgainstCalculator(SetupData setup, RaceInfo raceInfo)
{
    // 1. Buscar setup calculado para esta pista
    var calcSetup = _calcDb.GetLatestSetup(raceInfo.TrackId, raceInfo.SeasonId);

    if (calcSetup is null)
    {
        return ValidationResult.Info(
            level: Severity.Info,
            message: "ℹ️ Sin datos de calculadora para esta pista",
            suggestion: "Usa la calculadora para generar un setup de referencia"
        );
    }

    // 2. Calcular desviación por componente (6 de GPRO)
    var deviations = new Dictionary<string, decimal>
    {
        ["FrontWing"]  = CalcDeviation(setup.FrontWing, calcSetup.FrontWing),
        ["RearWing"]   = CalcDeviation(setup.RearWing, calcSetup.RearWing),
        ["Engine"]     = CalcDeviation(setup.Engine, calcSetup.Engine),
        ["Brakes"]     = CalcDeviation(setup.Brakes, calcSetup.Brakes),
        ["Gear"]       = CalcDeviation(setup.Gear, calcSetup.Gear),
        ["Suspension"] = CalcDeviation(setup.Suspension, calcSetup.Suspension)
    };

    var avgDeviation = deviations.Values.Average();

    // 3. Evaluar
    if (avgDeviation > 20)
    {
        return ValidationResult.Warning(
            level: Severity.Warning,
            message: "🟡 Setup muy desviado de la calculadora",
            detail: $"Desviación promedio: {avgDeviation:F1}%",
            suggestion: $"Revisa los valores — la calculadora sugiere FW:{calcSetup.FrontWing} RW:{calcSetup.RearWing}...",
            deviations: deviations
        );
    }

    if (avgDeviation > 10)
    {
        return ValidationResult.Info(
            level: Severity.Info,
            message: "ℹ️ Diferencias moderadas con la calculadora",
            detail: $"Desviación: {avgDeviation:F1}%",
            deviations: deviations
        );
    }

    return ValidationResult.Ok($"✅ Setup alineado con la calculadora (desvío: {avgDeviation:F1}%)");
}

private static decimal CalcDeviation(int actual, int expected)
    => expected == 0 ? 0 : Math.Abs(actual - expected) / (decimal)expected * 100;
```

### Ejemplo Visual

```
Caso: Novato pone Front Wing: 35, Calculadora dice Front Wing: 52

  ┌─────────────────────────────────────────────────┐
  │  🟡 AVISO — CRUCE CON CALCULADORA               │
  │                                                  │
  │  🔗 La calculadora tiene datos para Monaco S98  │
  │                                                  │
  │  Componente  │ Calculado │  Tuyo  │ Desvío       │
  │  ────────────┼───────────┼────────┼──────────   │
  │  Front Wing  │    52     │   35   │ 🔴 32.7%    │
  │  Rear Wing   │    40     │   42   │ ✅  5.0%    │
  │  Engine      │   700     │  800   │ 🔴 14.3%    │
  │  Brakes      │   345     │  280   │ 🔴 18.8%    │
  │  Gear        │   155     │  120   │ 🔴 22.6%    │
  │  Suspension  │    68     │   90   │ 🔴 32.4%    │
  │  ────────────┼───────────┼────────┼──────────   │
  │  Promedio    │           │        │ 🔴 20.9%    │
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
│  │  → Sugerencia: Revisa Front Wing y Gear                │  │
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
