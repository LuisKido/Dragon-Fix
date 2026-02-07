# 🐉 Dragon App — Fix Academy

<p align="center">
  <em>Módulo de estrategia, validación y seguimiento de setups para todos los alumnos de GPRO</em>
</p>

---

[![Module](https://img.shields.io/badge/module-Fix%20Academy-blue)]()
[![Status](https://img.shields.io/badge/status-En%20Desarrollo-yellow)]()
[![GPRO](https://img.shields.io/badge/GPRO-Compatible-green)]()

## 📋 Tabla de Contenidos

| # | Documento | Descripción |
|---|-----------|-------------|
| 1 | [Parser de Prácticas](01-parser-practicas.md) | Extracción automática de datos desde texto crudo de GPRO |
| 2 | [Validador de Estrategia](02-validador-estrategia.md) | Motor de reglas anti-error para estrategias de carrera |
| 3 | [Registro Histórico](03-registro-historico.md) | Sistema de seguimiento y comparativas entre alumnos |
| 4 | [Modelos de Datos](04-modelos-datos.md) | Estructuras de datos y esquema de base de datos |
| 5 | [Contratos API](05-contratos-api.md) | Endpoints y contratos de integración |
| 6 | [Wireframes UI](06-wireframes-ui.md) | Diseño de interfaces del módulo |
| 7 | [Integración Calculadora](07-integracion-calculadora.md) | 🆕 Cruce con la calculadora de setups existente |

---

## 🎯 Objetivo del Módulo

Proveer a los mentores de una academia GPRO un sistema integrado para llevar la estrategia de todos los alumnos e ir validando sus setups:

```
┌─────────────────────────────────────────────────────┐
│                  FLUJO PRINCIPAL                     │
│                                                     │
│  Novato pega texto  ──►  Parser extrae datos        │
│         │                      │                    │
│         ▼                      ▼                    │
│  Validador revisa   ◄──  Setup parseado             │
│  estrategia                    │                    │
│         │                      │                    │
│         ▼                      ▼                    │
│  Alertas al novato       Se guarda en BD            │
│         │                      │                    │
│         ▼                      ▼                    │
│  Mentor revisa en   ◄──  Dashboard comparativo      │
│  su panel                                           │
└─────────────────────────────────────────────────────┘
```

## 🧩 Los 3 Pilares

### 1. 🔍 Parser de Prácticas
> *"La joya de la corona"*

El novato pega el texto crudo de la página de prácticas de GPRO y el sistema extrae automáticamente todos los valores del setup y el feedback del piloto.

**Problema que resuelve:** Copiar dato por dato es tedioso y propenso a errores de transcripción.

### 2. 🛡️ Validador de Estrategia
> *"El Antierror"*

Motor de reglas que valida la estrategia antes de que el novato la aplique:

| Regla | Ejemplo |
|-------|---------|
| **Fuel Check** | Si $consumo \times km > tanque$ → 🔴 Alerta |
| **Tyre Check** | Extra Soft + 35°C → 🔴 "¡Sin goma en vuelta 10!" |
| **Weather Check** | Lluvia > 0% sin Rain tyres → 🟡 Alerta |

### 3. 📊 Registro Histórico
> *"La memoria de la academia"*

Cada carrera de cada alumno se registra. El mentor puede comparar setups, ver evolución y definir una **"Base de Oro"** contra la cual medir desviaciones.

### 4. 🔗 Integración con Calculadora
> *"El puente inteligente"* — 🆕

La app ya tiene una calculadora de setups con su propia BD. Este módulo **cruza datos** entre ambos sistemas:

| Cruce | Beneficio |
|-------|----------|
| **Auto-Baseline** | La Base de Oro se genera desde la calculadora, no manual |
| **Fuel preciso** | Consumo real de la calculadora, no tabla estática |
| **Correlación** | ¿El novato que sigue la calculadora mejora? Los datos lo prueban |
| **Fuente rastreable** | Cada setup indica si es `CALCULATOR`, `PARSED` o `MANUAL` |

---

## 🏗️ Arquitectura de Alto Nivel

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│   Frontend   │────►│   API Module     │────►│  SQL Server  │
│  (Dragon App)│◄────│  Fix Academy     │◄────│   Database   │
└──────────────┘     └──────────────────┘     └──────────────┘
                            │                        ▲
                     ┌──────┼──────┐                 │
                     │      │      │          ┌──────┴───────┐
               ┌─────▼───┐ ┌▼─────────┐      │ Calculadora  │
               │  Parser │ │ Validator │      │ de Setups    │
               │  Engine │ │  Engine   │      │ (BD cruzada) │
               └─────────┘ └──────────┘      └──────────────┘
```

> **Nota:** Este módulo se integra a **Dragon App** que ya incluye una **calculadora de setups**. Los datos de la calculadora se cruzan con **Fix Academy** para auto-generar la Base de Oro, mejorar la precisión del Fuel Check y correlacionar desviaciones con resultados.

## 🔧 Stack Recomendado (Agnóstico)

| Componente | Opción Recomendada | Alternativas |
|------------|-------------------|--------------|
| Parser Engine | Regex + Mapeo de campos | NLP básico |
| Validator Engine | Motor de reglas | Drools, .NET Rules |
| Base de datos | SQL Server | PostgreSQL, MySQL |
| API | REST JSON | GraphQL |
| Frontend | Componente integrable | Web Component |

## 📁 Estructura del Módulo

```
setup-reviewer/
├── parser/
│   ├── PracticeParser        # Extrae datos del texto crudo
│   ├── FeedbackInterpreter   # Interpreta feedback del piloto
│   └── SetupMapper           # Mapea a modelo de datos
├── validator/
│   ├── FuelValidator         # Validación de combustible
│   ├── TyreValidator         # Validación de neumáticos
│   ├── WeatherValidator      # Validación meteorológica
│   └── RuleEngine            # Orquestador de reglas
├── registry/
│   ├── RaceRepository        # CRUD de carreras
│   ├── SetupBaseline         # Base de Oro del mentor
│   └── ComparisonEngine      # Motor de comparativas
├── calculator-bridge/
│   ├── CalcSyncService       # Sincronización con la calculadora
│   ├── CalcSetupLinker        # Vincula setup calculado ↔ entry
│   └── CalcBaselineImporter   # Auto-genera Base de Oro
└── api/
    ├── ParserController      # Endpoints de parseo
    ├── ValidatorController   # Endpoints de validación
    └── DashboardController   # Endpoints del dashboard
```

---

## 🚀 Quick Start para Integración

```
1. Revisar Contratos API    →  docs/05-contratos-api.md
2. Crear tablas en BD       →  docs/04-modelos-datos.md
3. Conectar Calculadora     →  docs/07-integracion-calculadora.md  🆕
4. Implementar Parser       →  docs/01-parser-practicas.md
5. Implementar Validador    →  docs/02-validador-estrategia.md
6. Conectar Dashboard       →  docs/03-registro-historico.md
```

---

<p align="center">
  <strong>Fix Academy</strong> · Módulo de Dragon App
  <br/>
  <sub>Documentación Técnica v1.1 · Febrero 2026 · Incluye integración con Calculadora</sub>
</p>
