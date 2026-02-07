# 🔍 01 — Parser de Prácticas

[← Volver al Índice](README.md)

---

## Resumen

El Parser de Prácticas permite al novato ingresar datos de dos formas:

1. **Pegar texto crudo** de la página web de prácticas de GPRO
2. **📸 Subir una captura de pantalla** desde la app móvil de GPRO

En ambos casos, el sistema extrae automáticamente todos los valores del setup y el feedback del piloto.

```
┌─────────────────────┐
│  Texto crudo GPRO   │──┐
│  (pegado por novato)│  │     ┌──────────────┐       ┌──────────────┐
└─────────────────────┘  ├────►│    Parser    │──────►│ Setup Object │
┌─────────────────────┐  │     │    Engine    │       │  (validado)  │
│  📸 Captura de la   │──┘     └──────────────┘       └──────────────┘
│  app móvil GPRO     │             ▲
└─────────────────────┘             │
                              ┌─────────┐
                              │   OCR   │
                              │  Engine │
                              └─────────┘
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

## 📸 Entrada Alternativa: Captura desde App Móvil GPRO

Muchos novatos usan la **app de celular de GPRO** en lugar del navegador web. Desde la app pueden tomar una captura de pantalla de sus resultados de práctica y subirla directamente.

### Flujo de la Captura

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  📱 App GPRO │────►│  📸 Captura  │────►│   OCR Engine  │────►│ Texto crudo  │
│  (celular)   │     │  de pantalla │     │  (extracción) │     │  (interno)   │
└──────────────┘     └──────────────┘     └───────────────┘     └──────┬───────┘
                                                                       │
                                                                ┌──────▼───────┐
                                                                │ Parser Regex │
                                                                │ (igual que   │
                                                                │  texto)      │
                                                                └──────────────┘
```

### Formatos Aceptados

| Formato | Extensión | Tamaño Máximo |
|---------|-----------|---------------|
| PNG | `.png` | 10 MB |
| JPEG | `.jpg`, `.jpeg` | 10 MB |
| WebP | `.webp` | 10 MB |

### Motor OCR — Lógica de Extracción

```csharp
public async Task<PracticeResult> ParseScreenshotAsync(IFormFile imageFile)
{
    // 1. Validar imagen
    var validExtensions = new[] { ".png", ".jpg", ".jpeg", ".webp" };
    var extension = Path.GetExtension(imageFile.FileName).ToLower();

    if (!validExtensions.Contains(extension))
        throw new InvalidImageException("Formato no soportado. Usa PNG, JPG o WebP.");

    if (imageFile.Length > MaxSizeBytes)
        throw new InvalidImageException("La imagen es muy pesada. Máximo 10 MB.");

    // 2. Pre-procesamiento de imagen
    using var stream = imageFile.OpenReadStream();
    var processedImage = _imagePreprocessor.Process(stream, new PreprocessOptions
    {
        ConvertToGrayscale = true,
        AdjustContrast     = true,
        CropStatusBar      = true,   // Recortar barra de estado del celular
        CropNavBar         = true,   // Recortar barra de navegación inferior
        Deskew             = true    // Enderezar si está rotada
    });

    // 3. OCR — Extraer texto de la imagen
    var ocrResult = await _ocrEngine.ExtractTextAsync(processedImage);

    // 4. Evaluar confianza del OCR
    if (ocrResult.Confidence < 0.70m)
    {
        throw new OcrLowConfidenceException(
            message: "No se pudo leer la captura con suficiente confianza",
            suggestion: "Intenta con una captura más nítida o pega el texto manualmente",
            confidence: ocrResult.Confidence
        );
    }

    // 5. Pasar al parser de texto normal
    var result = ParsePracticeText(ocrResult.Text);
    result.Source = SetupSource.Screenshot;
    result.OcrConfidence = ocrResult.Confidence;

    // 6. Agregar warnings si confianza es media
    if (ocrResult.Confidence < 0.85m)
        result.Warnings.Add($"⚠️ Confianza OCR media ({ocrResult.Confidence:P0}). Verifica los valores.");

    return result;
}
```

### Pre-procesamiento de Imagen

Las capturas de la app móvil pueden tener elementos que dificultan la lectura:

| Problema | Solución |
|----------|----------|
| Barra de estado del celular (hora, batería) | Recorte automático de zona superior |
| Barra de navegación inferior | Recorte automático de zona inferior |
| Modo oscuro de la app | Inversión de colores antes del OCR |
| Captura borrosa o de baja resolución | Warning + sugerencia de repetir captura |
| Texto parcialmente cortado | Parseo parcial + warning de campos faltantes |
| Notificaciones superpuestas | Detección de overlay y recorte |

### Stack OCR Recomendado

| Opción | Ventaja | Consideración |
|--------|---------|---------------|
| **Tesseract OCR** (local) | Gratis, sin dependencia externa | Requiere entrenamiento para fuentes GPRO |
| **Azure Computer Vision** | Alta precisión, API simple | Costo por llamada |
| **Google Cloud Vision** | Excelente en texto estructurado | Costo por llamada |
| **AWS Textract** | Bueno para tablas y formularios | Costo por llamada |

> 💡 **Recomendación:** Usar **Tesseract** como opción por defecto (gratis) con fallback a una API cloud si la confianza es baja.

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

## ⚙️ Código del Parser

```csharp
public PracticeResult ParsePracticeText(string rawText)
{
    var result = new PracticeResult();

    // 1. Extraer Setup
    result.Setup = new SetupData
    {
        Wings      = ExtractNumber(rawText, @"Wings?:\s*(\d+)"),
        Engine     = ExtractNumber(rawText, @"Engine:\s*(\d+)"),
        Brakes     = ExtractNumber(rawText, @"Brakes?:\s*(\d+)"),
        Gear       = ExtractNumber(rawText, @"Gear:\s*(\d+)"),
        Suspension = ExtractNumber(rawText, @"Suspension:\s*(\d+)")
    };

    // 2. Extraer Estrategia
    result.Strategy = new StrategyData
    {
        TyreCompound = ExtractString(rawText, @"Tyres?\s*(?:used)?:\s*(.+)"),
        FuelLoad     = ExtractNumber(rawText, @"Fuel\s*(?:load)?:\s*(\d+)")
    };

    // 3. Extraer Condiciones
    result.Conditions = new ConditionsData
    {
        Temperature = ExtractNumber(rawText, @"(\d+)\s*°?\s*C"),
        Humidity    = ExtractNumber(rawText, @"Humidity\s*:?\s*(\d+)\s*%")
    };

    // 4. Extraer Tiempos de Vuelta
    result.LapTimes = ExtractAllMatches(rawText, @"(\d+:\d{2}\.\d{3})");

    // 5. Extraer y Analizar Feedback
    var feedbackText = ExtractFeedbackBlock(rawText);
    result.Feedback = AnalyzeFeedback(feedbackText);

    // 6. Validar completitud
    var missingFields = result.GetMissingFields();
    if (missingFields.Any())
        result.Warnings.Add($"Campos faltantes: {string.Join(", ", missingFields)}");

    return result;
}

private int? ExtractNumber(string text, string pattern)
{
    var match = Regex.Match(text, pattern, RegexOptions.IgnoreCase);
    return match.Success && int.TryParse(match.Groups[1].Value, out var value)
        ? value
        : null;
}

private string? ExtractString(string text, string pattern)
{
    var match = Regex.Match(text, pattern, RegexOptions.IgnoreCase);
    return match.Success ? match.Groups[1].Value.Trim() : null;
}

private List<string> ExtractAllMatches(string text, string pattern)
{
    return Regex.Matches(text, pattern)
        .Select(m => m.Groups[1].Value)
        .ToList();
}
```

---

## 🔄 Flujo del Parser

```
                    ┌──────────────────────┐
                    │  ¿Cómo ingresa       │
                    │   los datos?          │
                    └────┬────────────┬─────┘
                         │            │
                  Texto  │            │  📸 Captura
                         │            │
              ┌──────────▼──┐   ┌─────▼──────────┐
              │  Novato     │   │  Novato sube   │
              │  pega texto │   │  screenshot    │
              └──────────┬──┘   └─────┬──────────┘
                         │            │
                         │   ┌────────▼────────┐
                         │   │  OCR Engine     │
                         │   │  extrae texto   │
                         │   │  de la imagen   │
                         │   └────────┬────────┘
                         │            │
                         │   ┌────────▼────────┐
                         │   │ ¿Confianza      │
                         │   │  OCR > 70%?     │
                         │   └──┬──────────┬───┘
                         │      │          │
                         │ Sí   │          │ No
                         │      │          │
                         │      │   ┌──────▼─────────┐
                         │      │   │ Error: captura │
                         │      │   │ no legible,    │
                         │      │   │ pega texto     │
                         │      │   └────────────────┘
                    ┌────▼──────▼───────────┐
                    │  ¿Texto válido?       │
                    │  (tiene estructura    │
                    │   reconocible)        │
                    └────┬────────────┬─────┘
                         │            │
                    Sí   │            │  No
                         │            │
              ┌──────────▼──┐   ┌─────▼──────────┐
              │  Ejecutar   │   │  Mostrar error │
              │  Regex por  │   │  "Formato no   │
              │  cada campo │   │   reconocido"  │
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
  "source": "TEXT | SCREENSHOT",
  "ocrConfidence": null,
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
│  ── o ──────────────────────────────────────────────────    │
│                                                             │
│  📸 ¿Usas la app de celular? Sube una captura:             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │      [ 📷 Seleccionar imagen ]                      │   │
│  │                                                     │   │
│  │      PNG, JPG o WebP · Máx. 10 MB                  │   │
│  │      También puedes arrastrar aquí                  │   │
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
| 📸 Imagen borrosa / baja resolución | Error: "Captura no legible" + sugerir texto manual |
| 📸 Captura en modo oscuro | Pre-procesamiento: inversión de colores |
| 📸 Captura con notificaciones encima | Detección de overlay + recorte |
| 📸 Captura parcial (datos cortados) | Parseo parcial + warning de campos faltantes |
| 📸 Formato no soportado | Error: "Usa PNG, JPG o WebP" |
| 📸 Archivo > 10 MB | Error: "La imagen es muy pesada" |

---

[→ Siguiente: Validador de Estrategia](02-validador-estrategia.md)
