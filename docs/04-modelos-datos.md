# 🗄️ 04 — Modelos de Datos

[← Volver al Índice](README.md) · [← Registro Histórico](03-registro-historico.md)

---

## Resumen

Esquema de base de datos SQL Server para almacenar toda la información del módulo. Diseñado para ser **agnóstico al lenguaje** de la aplicación host.

---

## 📐 Diagrama Entidad-Relación

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐
│   Students  │     │    Tracks    │     │    Seasons       │
│─────────────│     │──────────────│     │──────────────────│
│ Id (PK)     │     │ Id (PK)      │     │ Id (PK)          │
│ Username    │     │ Name         │     │ Number           │
│ GproId      │     │ Country      │     │ StartDate        │
│ JoinDate    │     │ LengthKm     │     │ EndDate          │
│ MentorId(FK)│     │ TotalLaps    │     │ IsActive         │
│ IsActive    │     │ FuelTankCap  │     └──────────────────┘
└──────┬──────┘     │ Category     │
       │            └──────┬───────┘
       │                   │
       │    ┌──────────────▼────────────────┐
       │    │          Races                │
       │    │───────────────────────────────│
       │    │ Id (PK)                       │
       └───►│ SeasonId (FK)                 │
            │ TrackId (FK)                  │
            │ RaceNumber                    │
            │ RaceDate                      │
            └──────────────┬────────────────┘
                           │
            ┌──────────────▼────────────────┐
            │      StudentRaceEntries       │
            │───────────────────────────────│
            │ Id (PK)                       │
            │ StudentId (FK)                │
            │ RaceId (FK)                   │
            │ CreatedAt                     │
            │ UpdatedAt                     │
            │ ReviewedByMentor              │
            │ MentorNotes                   │
            └──┬────────────┬───────────┬───┘
               │            │           │
    ┌──────────▼──┐  ┌──────▼────┐  ┌───▼──────────┐
    │   Setups    │  │ Strategies│  │  RaceResults │
    │─────────────│  │───────────│  │──────────────│
    │ Id (PK)     │  │ Id (PK)   │  │ Id (PK)      │
    │ EntryId(FK) │  │ EntryId   │  │ EntryId(FK)  │
    │ Wings       │  │ Compound  │  │ QualifyPos   │
    │ Engine      │  │ FuelLoad  │  │ StartPos     │
    │ Brakes      │  │ WetSetup  │  │ FinishPos    │
    │ Gear        │  └───────┬───┘  │ BestLap      │
    │ Suspension  │          │      │ IsDNF        │
    │ Source      │   ┌──────▼────┐ │ DNFReason    │
    └─────────────┘   │ PitStops  │ └──────────────┘
                      │──────────-│
                      │ Id (PK)   │
                      │ StratId   │
                      │ LapNumber │
                      │ Compound  │
                      │ FuelAdd   │
                      └───────────┘
```

---

## 📋 Tablas Detalladas

### 🏫 `Students` — Alumnos de la Academia

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador único |
| `Username` | `NVARCHAR(50)` | NOT NULL | Nombre de usuario en la academia |
| `GproUsername` | `NVARCHAR(50)` | NULL | Nombre de usuario en GPRO |
| `GproId` | `NVARCHAR(20)` | NULL | ID numérico en GPRO |
| `Email` | `NVARCHAR(100)` | NULL | Correo electrónico |
| `MentorId` | `INT` | FK → Mentors | Mentor asignado |
| `GroupName` | `NVARCHAR(50)` | NULL | Grupo dentro de la academia |
| `JoinDate` | `DATETIME2` | NOT NULL | Fecha de ingreso |
| `IsActive` | `BIT` | NOT NULL | ¿Sigue activo? |

---

### 🏁 `Tracks` — Circuitos

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador único |
| `Name` | `NVARCHAR(100)` | NOT NULL | Nombre del circuito |
| `Country` | `NVARCHAR(50)` | NOT NULL | País |
| `LengthKm` | `DECIMAL(5,3)` | NOT NULL | Longitud en km |
| `TotalLaps` | `INT` | NOT NULL | Vueltas de carrera |
| `FuelTankCapacity` | `DECIMAL(5,1)` | NOT NULL | Capacidad del tanque (litros) |
| `Category` | `NVARCHAR(20)` | NOT NULL | Rookie/Amateur/Pro/Master |
| `AvgTempMin` | `INT` | NULL | Temperatura mínima promedio |
| `AvgTempMax` | `INT` | NULL | Temperatura máxima promedio |

---

### 📅 `Seasons` — Temporadas

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `SeasonNumber` | `INT` | NOT NULL | Número de temporada GPRO |
| `StartDate` | `DATE` | NOT NULL | Fecha de inicio |
| `EndDate` | `DATE` | NULL | Fecha de fin |
| `IsActive` | `BIT` | NOT NULL | ¿Temporada activa? |

---

### 🏎️ `Races` — Carreras

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `SeasonId` | `INT` | FK → Seasons | Temporada |
| `TrackId` | `INT` | FK → Tracks | Circuito |
| `RaceNumber` | `INT` | NOT NULL | # de carrera en la temporada |
| `RaceDate` | `DATE` | NOT NULL | Fecha de la carrera |
| `WeatherTemp` | `INT` | NULL | Temperatura real (°C) |
| `WeatherHumidity` | `INT` | NULL | Humedad real (%) |
| `WeatherRainProb` | `INT` | NULL | Probabilidad de lluvia (%) |
| `DidRain` | `BIT` | NULL | ¿Llovió realmente? |

---

### 📝 `StudentRaceEntries` — Registro por Alumno por Carrera

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `StudentId` | `INT` | FK → Students | Alumno |
| `RaceId` | `INT` | FK → Races | Carrera |
| `RawPracticeText` | `NVARCHAR(MAX)` | NULL | Texto crudo pegado |
| `ParseConfidence` | `DECIMAL(3,2)` | NULL | Confianza del parseo (0-1) |
| `ValidationStatus` | `NVARCHAR(20)` | NOT NULL | VALID / WARNING / CRITICAL |
| `ValidationAlerts` | `NVARCHAR(MAX)` | NULL | JSON con alertas |
| `ReviewedByMentor` | `BIT` | NOT NULL | ¿Revisado? |
| `MentorNotes` | `NVARCHAR(MAX)` | NULL | Notas del mentor |
| `CreatedAt` | `DATETIME2` | NOT NULL | Fecha de creación |
| `UpdatedAt` | `DATETIME2` | NOT NULL | Última actualización |

> **Unique constraint:** `(StudentId, RaceId)` — Un alumno, un registro por carrera.

---

### 🔧 `Setups` — Configuración del Coche

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Entrada relacionada |
| `Wings` | `INT` | NOT NULL | Valor de alerón |
| `Engine` | `INT` | NOT NULL | Valor de motor |
| `Brakes` | `INT` | NOT NULL | Valor de frenos |
| `Gear` | `INT` | NOT NULL | Valor de caja |
| `Suspension` | `INT` | NOT NULL | Valor de suspensión |
| `Source` | `NVARCHAR(20)` | NOT NULL | PARSED / MANUAL / CALCULATOR |

---

### ⛽ `Strategies` — Estrategia de Carrera

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Entrada |
| `TyreCompound` | `NVARCHAR(20)` | NOT NULL | ExtraSoft/Soft/Medium/Hard/Rain |
| `FuelLoad` | `DECIMAL(5,1)` | NOT NULL | Combustible (litros) |
| `RiskLevel` | `INT` | NULL | Nivel de riesgo (0-100) |
| `WetSetup` | `BIT` | NOT NULL | ¿Tiene setup de lluvia? |

---

### 🔄 `PitStops` — Paradas en Pits

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `StrategyId` | `INT` | FK → Strategies | Estrategia |
| `StopNumber` | `INT` | NOT NULL | Número de parada (1, 2, 3...) |
| `PlannedLap` | `INT` | NOT NULL | Vuelta planificada |
| `TyreCompound` | `NVARCHAR(20)` | NOT NULL | Compuesto para poner |
| `FuelToAdd` | `DECIMAL(5,1)` | NULL | Combustible a agregar |

---

### 🏆 `RaceResults` — Resultados

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Entrada |
| `QualifyPosition` | `INT` | NULL | Posición en clasificación |
| `StartPosition` | `INT` | NULL | Posición de salida |
| `FinishPosition` | `INT` | NULL | Posición de llegada |
| `BestLapTime` | `NVARCHAR(15)` | NULL | Mejor vuelta (mm:ss.SSS) |
| `TotalTime` | `NVARCHAR(15)` | NULL | Tiempo total |
| `PositionsGained` | `INT` | NULL | Posiciones ganadas/perdidas |
| `IsDNF` | `BIT` | NOT NULL | ¿Abandonó? |
| `DNFReason` | `NVARCHAR(100)` | NULL | Razón del abandono |

---

### ⭐ `SetupBaselines` — Base de Oro del Mentor

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `TrackId` | `INT` | FK → Tracks | Circuito |
| `SeasonId` | `INT` | FK → Seasons | Temporada |
| `MentorId` | `INT` | FK → Mentors | Mentor |
| `Wings` | `INT` | NOT NULL | Wings de referencia |
| `Engine` | `INT` | NOT NULL | Engine de referencia |
| `Brakes` | `INT` | NOT NULL | Brakes de referencia |
| `Gear` | `INT` | NOT NULL | Gear de referencia |
| `Suspension` | `INT` | NOT NULL | Suspension de referencia |
| `RecommendedCompound` | `NVARCHAR(20)` | NULL | Compuesto recomendado |
| `Source` | `NVARCHAR(20)` | NOT NULL | MANUAL / CALCULATOR |
| `CalcSessionId` | `INT` | NULL | FK → CalcSessions (si fuente es calculadora) |
| `Notes` | `NVARCHAR(MAX)` | NULL | Notas del mentor |
| `CreatedAt` | `DATETIME2` | NOT NULL | Fecha de creación |

> **Unique constraint:** `(TrackId, SeasonId, MentorId)`

---

### 💬 `DriverFeedback` — Feedback Parseado del Piloto

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Entrada |
| `RawFeedback` | `NVARCHAR(MAX)` | NOT NULL | Texto original |
| `Component` | `NVARCHAR(20)` | NOT NULL | Wings/Engine/Brakes/Gear/Suspension |
| `Action` | `NVARCHAR(10)` | NOT NULL | INCREASE / DECREASE / OK |
| `Priority` | `NVARCHAR(10)` | NOT NULL | HIGH / MEDIUM / LOW |
| `Reason` | `NVARCHAR(200)` | NULL | Razón interpretada |

---

### 🔗 `CalcSetupLink` — Cruce Setup Calculado ↔ Entry del Alumno 🆕

> Vincula el setup que la calculadora generó con el registro real del alumno.

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `EntryId` | `INT` | FK → StudentRaceEntries | Registro del alumno |
| `CalcSessionId` | `INT` | NOT NULL | ID de la sesión en la calculadora |
| `CalcWings` | `INT` | NOT NULL | Wings que calculó la herramienta |
| `CalcEngine` | `INT` | NOT NULL | Engine calculado |
| `CalcBrakes` | `INT` | NOT NULL | Brakes calculado |
| `CalcGear` | `INT` | NOT NULL | Gear calculado |
| `CalcSuspension` | `INT` | NOT NULL | Suspension calculado |
| `DeviationPercent` | `DECIMAL(5,2)` | NULL | Desviación promedio del alumno vs calculado |
| `LinkedAt` | `DATETIME2` | NOT NULL | Fecha de vinculación |

---

### 🔗 `CalcTrackSync` — Sincronización de Pistas con Calculadora 🆕

> Mapea los IDs de pistas entre la calculadora existente y el Reviewer.

| Columna | Tipo | Null | Descripción |
|---------|------|------|-------------|
| `Id` | `INT` | PK, Identity | Identificador |
| `ReviewerTrackId` | `INT` | FK → Tracks | Pista en el Reviewer |
| `CalcTrackId` | `INT` | NOT NULL | ID de la pista en la calculadora |
| `LastSyncAt` | `DATETIME2` | NOT NULL | Última sincronización |
| `SyncStatus` | `NVARCHAR(20)` | NOT NULL | OK / STALE / ERROR |

---

## 🗃️ Script SQL de Creación

```sql
-- =============================================
-- Dragon App — Fix Academy
-- Database Schema v1.1 (con integración Calculadora)
-- =============================================

CREATE TABLE Seasons (
    Id              INT IDENTITY(1,1) PRIMARY KEY,
    SeasonNumber    INT NOT NULL,
    StartDate       DATE NOT NULL,
    EndDate         DATE NULL,
    IsActive        BIT NOT NULL DEFAULT 0
);

CREATE TABLE Tracks (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    Name                NVARCHAR(100) NOT NULL,
    Country             NVARCHAR(50) NOT NULL,
    LengthKm           DECIMAL(5,3) NOT NULL,
    TotalLaps           INT NOT NULL,
    FuelTankCapacity    DECIMAL(5,1) NOT NULL,
    Category            NVARCHAR(20) NOT NULL,
    AvgTempMin          INT NULL,
    AvgTempMax          INT NULL
);

CREATE TABLE Students (
    Id              INT IDENTITY(1,1) PRIMARY KEY,
    Username        NVARCHAR(50) NOT NULL,
    GproUsername     NVARCHAR(50) NULL,
    GproId          NVARCHAR(20) NULL,
    Email           NVARCHAR(100) NULL,
    MentorId        INT NULL,
    GroupName       NVARCHAR(50) NULL,
    JoinDate        DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    IsActive        BIT NOT NULL DEFAULT 1
);

CREATE TABLE Races (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    SeasonId            INT NOT NULL REFERENCES Seasons(Id),
    TrackId             INT NOT NULL REFERENCES Tracks(Id),
    RaceNumber          INT NOT NULL,
    RaceDate            DATE NOT NULL,
    WeatherTemp         INT NULL,
    WeatherHumidity     INT NULL,
    WeatherRainProb     INT NULL,
    DidRain             BIT NULL,
    UNIQUE (SeasonId, RaceNumber)
);

CREATE TABLE StudentRaceEntries (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    StudentId           INT NOT NULL REFERENCES Students(Id),
    RaceId              INT NOT NULL REFERENCES Races(Id),
    RawPracticeText     NVARCHAR(MAX) NULL,
    ParseConfidence     DECIMAL(3,2) NULL,
    ValidationStatus    NVARCHAR(20) NOT NULL DEFAULT 'PENDING',
    ValidationAlerts    NVARCHAR(MAX) NULL,
    ReviewedByMentor    BIT NOT NULL DEFAULT 0,
    MentorNotes         NVARCHAR(MAX) NULL,
    CreatedAt           DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt           DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UNIQUE (StudentId, RaceId)
);

CREATE TABLE Setups (
    Id          INT IDENTITY(1,1) PRIMARY KEY,
    EntryId     INT NOT NULL REFERENCES StudentRaceEntries(Id),
    Wings       INT NOT NULL,
    Engine      INT NOT NULL,
    Brakes      INT NOT NULL,
    Gear        INT NOT NULL,
    Suspension  INT NOT NULL,
    Source      NVARCHAR(20) NOT NULL DEFAULT 'MANUAL'  -- MANUAL / PARSED / CALCULATOR
);

CREATE TABLE Strategies (
    Id              INT IDENTITY(1,1) PRIMARY KEY,
    EntryId         INT NOT NULL REFERENCES StudentRaceEntries(Id),
    TyreCompound    NVARCHAR(20) NOT NULL,
    FuelLoad        DECIMAL(5,1) NOT NULL,
    RiskLevel       INT NULL,
    WetSetup        BIT NOT NULL DEFAULT 0
);

CREATE TABLE PitStops (
    Id              INT IDENTITY(1,1) PRIMARY KEY,
    StrategyId      INT NOT NULL REFERENCES Strategies(Id),
    StopNumber      INT NOT NULL,
    PlannedLap      INT NOT NULL,
    TyreCompound    NVARCHAR(20) NOT NULL,
    FuelToAdd       DECIMAL(5,1) NULL
);

CREATE TABLE RaceResults (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    EntryId             INT NOT NULL REFERENCES StudentRaceEntries(Id),
    QualifyPosition     INT NULL,
    StartPosition       INT NULL,
    FinishPosition      INT NULL,
    BestLapTime         NVARCHAR(15) NULL,
    TotalTime           NVARCHAR(15) NULL,
    PositionsGained     INT NULL,
    IsDNF               BIT NOT NULL DEFAULT 0,
    DNFReason           NVARCHAR(100) NULL
);

CREATE TABLE SetupBaselines (
    Id                      INT IDENTITY(1,1) PRIMARY KEY,
    TrackId                 INT NOT NULL REFERENCES Tracks(Id),
    SeasonId                INT NOT NULL REFERENCES Seasons(Id),
    MentorId                INT NOT NULL,
    Wings                   INT NOT NULL,
    Engine                  INT NOT NULL,
    Brakes                  INT NOT NULL,
    Gear                    INT NOT NULL,
    Suspension              INT NOT NULL,
    RecommendedCompound     NVARCHAR(20) NULL,
    Source                  NVARCHAR(20) NOT NULL DEFAULT 'MANUAL',  -- MANUAL / CALCULATOR
    CalcSessionId           INT NULL,  -- FK a tabla de la calculadora
    Notes                   NVARCHAR(MAX) NULL,
    CreatedAt               DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UNIQUE (TrackId, SeasonId, MentorId)
);

CREATE TABLE DriverFeedback (
    Id              INT IDENTITY(1,1) PRIMARY KEY,
    EntryId         INT NOT NULL REFERENCES StudentRaceEntries(Id),
    RawFeedback     NVARCHAR(MAX) NOT NULL,
    Component       NVARCHAR(20) NOT NULL,
    Action          NVARCHAR(10) NOT NULL,
    Priority        NVARCHAR(10) NOT NULL DEFAULT 'MEDIUM',
    Reason          NVARCHAR(200) NULL
);

-- =============================================
-- Tablas de Integración con Calculadora (v1.1)
-- =============================================

CREATE TABLE CalcSetupLink (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    EntryId             INT NOT NULL REFERENCES StudentRaceEntries(Id),
    CalcSessionId       INT NOT NULL,  -- FK lógica a la BD de la calculadora
    CalcWings           INT NOT NULL,
    CalcEngine          INT NOT NULL,
    CalcBrakes          INT NOT NULL,
    CalcGear            INT NOT NULL,
    CalcSuspension      INT NOT NULL,
    DeviationPercent    DECIMAL(5,2) NULL,
    LinkedAt            DATETIME2 NOT NULL DEFAULT GETUTCDATE()
);

CREATE TABLE CalcTrackSync (
    Id                  INT IDENTITY(1,1) PRIMARY KEY,
    ReviewerTrackId     INT NOT NULL REFERENCES Tracks(Id),
    CalcTrackId         INT NOT NULL,  -- ID en la BD de la calculadora
    LastSyncAt          DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    SyncStatus          NVARCHAR(20) NOT NULL DEFAULT 'OK',
    UNIQUE (ReviewerTrackId, CalcTrackId)
);

-- Indexes for common queries
CREATE INDEX IX_StudentRaceEntries_Student ON StudentRaceEntries(StudentId);
CREATE INDEX IX_StudentRaceEntries_Race ON StudentRaceEntries(RaceId);
CREATE INDEX IX_Races_Season ON Races(SeasonId);
CREATE INDEX IX_Races_Track ON Races(TrackId);
CREATE INDEX IX_SetupBaselines_Track ON SetupBaselines(TrackId, SeasonId);
CREATE INDEX IX_CalcSetupLink_Entry ON CalcSetupLink(EntryId);
CREATE INDEX IX_CalcTrackSync_Track ON CalcTrackSync(ReviewerTrackId);
```

---

## 📊 Consultas Útiles

### Comparativa de alumnos para una carrera

```sql
SELECT 
    s.Username,
    su.Wings, su.Engine, su.Brakes, su.Gear, su.Suspension,
    e.ValidationStatus,
    e.ReviewedByMentor
FROM StudentRaceEntries e
JOIN Students s ON e.StudentId = s.Id
JOIN Setups su ON su.EntryId = e.Id
WHERE e.RaceId = @RaceId
ORDER BY s.Username;
```

### Desviación vs Base de Oro

```sql
SELECT 
    s.Username,
    ABS(su.Wings - b.Wings) * 100.0 / b.Wings AS WingsDev,
    ABS(su.Engine - b.Engine) * 100.0 / b.Engine AS EngineDev,
    ABS(su.Brakes - b.Brakes) * 100.0 / b.Brakes AS BrakesDev,
    ABS(su.Gear - b.Gear) * 100.0 / b.Gear AS GearDev,
    ABS(su.Suspension - b.Suspension) * 100.0 / b.Suspension AS SuspDev
FROM StudentRaceEntries e
JOIN Students s ON e.StudentId = s.Id
JOIN Setups su ON su.EntryId = e.Id
JOIN Races r ON e.RaceId = r.Id
JOIN SetupBaselines b ON b.TrackId = r.TrackId AND b.SeasonId = r.SeasonId
WHERE e.RaceId = @RaceId;
```

### 🆕 Setup Calculado vs Usado vs Resultado (Cruce con Calculadora)

```sql
SELECT 
    s.Username,
    su.Wings AS UsedWings, su.Engine AS UsedEngine,
    cl.CalcWings, cl.CalcEngine,
    cl.DeviationPercent,
    rr.StartPosition, rr.FinishPosition, rr.PositionsGained,
    CASE 
        WHEN cl.DeviationPercent < 5 AND rr.PositionsGained > 0 
            THEN 'Siguió cálculo + ganó posiciones'
        WHEN cl.DeviationPercent > 15 AND rr.PositionsGained < 0 
            THEN 'Ignoró cálculo + perdió posiciones'
        ELSE 'Neutral'
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

[→ Siguiente: Contratos API](05-contratos-api.md)
