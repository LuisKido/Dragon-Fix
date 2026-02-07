# 📊 03 — Registro Histórico y Comparativa

[← Volver al Índice](README.md) · [← Validador de Estrategia](02-validador-estrategia.md)

---

## Resumen

El sistema guarda **cada carrera de cada alumno** en SQL Server, permitiendo al mentor:
- Ver evolución individual de cada novato
- Comparar setups entre todos los alumnos para una misma carrera
- Definir una **"Base de Oro"** (setup del mentor) contra la cual medir desviaciones

```
┌──────────┐    ┌──────────┐    ┌──────────────────┐
│ Carrera 1│    │ Carrera 2│    │  Carrera N...    │
│ Setup    │    │ Setup    │    │  Setup           │
│ Result   │───►│ Result   │───►│  Result          │
└──────────┘    └──────────┘    └────────┬─────────┘
                                         │
                              ┌──────────▼──────────┐
                              │  Dashboard Mentor   │
                              │  Comparativas +     │
                              │  Tendencias         │
                              └─────────────────────┘
```

---

## 📁 ¿Qué se Guarda por Carrera?

| Categoría | Campos |
|-----------|--------|
| **Identificación** | Alumno, Temporada, Carrera #, Pista |
| **Setup** | FrontWing, RearWing, Engine, Brakes, Gear, Suspension |
| **Estrategia** | Compuesto, Fuel, Pit stops (vuelta + compuesto) |
| **Condiciones** | Temperatura, Humedad, Lluvia (%) |
| **Resultados** | Posición salida, Posición final, Mejor vuelta, DNF |
| **Validación** | Alertas que tuvo, Si las corrigió |
| **Feedback** | Feedback parseado del piloto |
| **Metadata** | Fecha registro, Revisado por mentor (sí/no), Notas mentor |

---

## 👨‍🏫 Dashboard del Mentor

### Vista 1: Comparativa de Carrera Actual

El mentor ve a **todos sus alumnos** para la carrera que viene, comparados lado a lado:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  📊 Dashboard del Mentor — GP Mónaco (Carrera #5)              [🔄]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BASE DE ORO (Tu setup):  FW:52  RW:38  E:700  B:345  G:155  S:68     │
│  ──────────────────────────────────────────────────────────             │
│                                                                         │
│  ┌─────────┬──────┬──────┬──────┬──────┬──────┬──────┬────────┬────────┐│
│  │ Alumno  │ FWng │ RWng │ Eng  │ Brk  │ Gear │ Susp │ Status │ Desv.  ││
│  ├─────────┼──────┼──────┼──────┼──────┼──────┼──────┼────────┼────────┤│
│  │ @Pedro  │  50  │  36  │ 710  │ 340  │ 150  │  65  │  ✅   │  3.2%  ││
│  │ @María  │  48  │  40  │ 695  │ 350  │ 160  │  70  │  🟡   │  6.8%  ││
│  │ @Carlos │  35  │  22  │ 800  │ 280  │ 120  │  90  │  🔴   │ 22.4%  ││
│  │ @Ana    │  55  │  42  │ 690  │ 330  │ 148  │  72  │  ✅   │  4.1%  ││
│  │ @Luis   │  --  │  --  │  --  │  --  │  --  │  --  │  ⏳   │  ---   ││
│  └─────────┴──────┴──────┴──────┴──────┴──────┴──────┴────────┴────────┘│
│                                                                         │
│  🔴 Carlos se desvía un 22.4% — Hacer click para ver detalle          │
│                                                                         │
│  [ 📩 Enviar feedback ] [ 📊 Exportar ] [ 📝 Notas ]                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Vista 2: Evolución Individual del Alumno

```
┌─────────────────────────────────────────────────────────────────────────┐
│  📈 Evolución de @Carlos — Temporada 98                        [🔄]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Desviación vs Base de Oro por carrera:                                │
│                                                                         │
│  30% │  ■                                                              │
│  25% │  ■                                      ■                       │
│  20% │  ■    ■                                  ■                       │
│  15% │  ■    ■                                                         │
│  10% │  ■    ■    ■              ■                                      │
│   5% │  ■    ■    ■    ■    ■   ■                                      │
│   0% │──■────■────■────■────■───■────■──────────────                   │
│      │  C1   C2   C3   C4   C5  C6   C7                               │
│                                                                         │
│  Tendencia: 📉 Mejorando (promedio bajando)                            │
│                                                                         │
│  ┌── Detalle por Carrera ────────────────────────────────────────────┐ │
│  │  C1: Mónaco    │ P15 → P12 │ ⬆️3  │ Desvío 28% │ 2 alertas 🔴  │ │
│  │  C2: Silverst. │ P12 → P10 │ ⬆️2  │ Desvío 18% │ 1 alerta 🟡   │ │
│  │  C3: Spa       │ P10 → P8  │ ⬆️2  │ Desvío 12% │ 0 alertas ✅  │ │
│  │  C4: Monza     │ P8  → P7  │ ⬆️1  │ Desvío 8%  │ 0 alertas ✅  │ │
│  │  C5: Suzuka    │ P7  → P6  │ ⬆️1  │ Desvío 5%  │ 0 alertas ✅  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  [ 💬 Comentar ] [ 📊 Exportar PDF ] [ 🔍 Comparar con otro alumno ]  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ⭐ Base de Oro (Setup Baseline)

### Concepto

La **Base de Oro** es el setup de referencia del mentor para cada pista. Puede venir de:

1. **Setup propio del mentor** de la temporada actual
2. **Mejor setup histórico** de temporadas anteriores
3. **Setup manual** ingresado por el mentor
4. 🆕 **Auto-importado desde la Calculadora** — Se sincroniza automáticamente cuando el mentor guarda un cálculo en la calculadora existente de la app. Ver [Integración Calculadora](07-integracion-calculadora.md).

### Cálculo de Desviación

$$\text{Desviación} = \frac{1}{6} \sum_{i=1}^{6} \left| \frac{\text{Valor}_{alumno,i} - \text{Valor}_{base,i}}{\text{Valor}_{base,i}} \right| \times 100$$

Donde los 6 componentes del setup son: FrontWing, RearWing, Engine, Brakes, Gear, Suspension.

### Ejemplo de Cálculo

| Componente | Base de Oro | Alumno | Diferencia | % Desviación |
|-----------|-------------|--------|------------|-------------|
| FrontWing | 52 | 35 | -17 | 32.7% |
| RearWing | 38 | 22 | -16 | 42.1% |
| Engine | 700 | 800 | +100 | 14.3% |
| Brakes | 345 | 280 | -65 | 18.8% |
| Gear | 155 | 120 | -35 | 22.6% |
| Suspension | 68 | 90 | +22 | 32.4% |
| | | | **Promedio** | **27.2%** 🔴 |

### Umbrales de Alerta

| Desviación | Estado | Significado |
|-----------|--------|-------------|
| 0 - 5% | ✅ Verde | Setup alineado |
| 5 - 10% | 🟡 Amarillo | Diferencias menores, revisar |
| > 10% | 🔴 Rojo | Desviación significativa, requiere corrección |

---

## 🔄 Flujo Completo del Registro

```
┌──────────────┐
│ Novato pega  │
│ texto de     │
│ práctica     │
└──────┬───────┘
       │
┌──────▼───────┐     ┌──────────────┐
│   Parser     │────►│ Setup Object │
│   extrae     │     │ parseado     │
└──────────────┘     └──────┬───────┘
                            │
                   ┌────────▼────────┐
                   │   Validador     │
                   │   revisa reglas │
                   └──┬──────────┬──┘
                      │          │
                 ✅ OK │    🔴 Error
                      │          │
           ┌──────────▼──┐  ┌───▼────────────┐
           │  Guardar en │  │ Novato corrige │
           │  SQL Server │  │ y revalida     │
           └──────────┬──┘  └────────────────┘
                      │
           ┌──────────▼──────────┐
           │  Aparece en el      │
           │  Dashboard del      │
           │  Mentor             │
           └──────────┬──────────┘
                      │
           ┌──────────▼──────────┐
           │  Mentor revisa,     │
           │  compara vs Base    │
           │  de Oro, comenta    │
           └─────────────────────┘
```

---

## 📊 Métricas del Dashboard

### Por Alumno
| Métrica | Descripción |
|---------|-------------|
| Desviación promedio | Promedio de desviación vs Base de Oro en las últimas N carreras |
| Tendencia | ¿Mejorando o empeorando? (pendiente de la curva) |
| Alertas totales | # de alertas rojas + amarillas acumuladas |
| Tasa de corrección | % de alertas que el alumno corrigió antes de la carrera |
| Posiciones ganadas | Diferencia entre posición de salida y llegada |

### Por Carrera (General)
| Métrica | Descripción |
|---------|-------------|
| Alumnos registrados | % de alumnos que subieron su setup |
| Desviación media | Promedio de desviación de todos los alumnos |
| Alertas activas | # de alertas sin resolver |
| Mejor alumno | Menor desviación vs Base de Oro |

---

## 📝 Notas del Mentor

El mentor puede agregar notas privadas por alumno por carrera:

```json
{
  "studentId": 3,
  "raceId": 5,
  "mentorNotes": "Carlos sigue poniendo el wing muy bajo. 
                  Hablar con él sobre la importancia del downforce 
                  en circuitos urbanos. Mejoró en engine.",
  "reviewed": true,
  "reviewedAt": "2026-02-06T10:30:00Z"
}
```

---

## 🔮 Futuras Mejoras

| Mejora | Descripción |
|--------|-------------|
| ~~**Auto-Baseline**~~ | ~~Generar Base de Oro automáticamente~~ → ✅ Implementado via [Calculadora](07-integracion-calculadora.md) |
| **Predictor** | Predecir posición estimada basándose en setup + historial |
| **Notificaciones** | Alertar al mentor cuando un alumno sube un setup con 🔴 |
| **Exportar PDF** | Informe semanal de la academia en PDF |
| **Ranking** | Tabla de posiciones interna de la academia |
| 🆕 **Correlación Calc↔Resultado** | Medir si seguir la calculadora mejora los resultados. Ver [doc 07](07-integracion-calculadora.md) |

---

[→ Siguiente: Modelos de Datos](04-modelos-datos.md)
