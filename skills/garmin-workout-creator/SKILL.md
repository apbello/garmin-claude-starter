---
name: garmin-workout-creator
description: Create and upload structured strength or running workouts to Garmin Connect via the Garmin MCP tool. Use this skill whenever the user asks to create, build, upload, or save a workout to Garmin, or asks to schedule a workout on their Garmin calendar. Also use when the user wants to modify or fix an existing uploaded workout. Covers strength training, running intervals, tempo runs, and any other Garmin-supported sport.
---

# Garmin Workout Creator

> Base original de Rodrigo Alvarez; trampas y correcciones de Alvaro Perez Bello.

A skill for creating well-structured workouts and uploading them to Garmin Connect via the `garmin:upload_workout` MCP tool.

---

## Step 1 — Build the workout JSON

Always use the exact JSON structure below. Do not guess field names — every field here is required unless marked optional.

### Top-level structure

```json
{
  "workoutName": "My Workout",
  "description": "Optional workout description",
  "sportType": { "sportTypeId": 5, "sportTypeKey": "strength_training" },
  "workoutSegments": [ ... ]
}
```

**Sport type IDs — valores confirmados:**
| sportTypeId | sportTypeKey | Notas |
|---|---|---|
| 1 | running | ✅ |
| 2 | cycling | ✅ |
| 4 | swimming | ⚠️ Cuidado: NO es strength. Crear con 4 = nado en piscina |
| 5 | strength_training | ✅ Confirmado 2026-07-13 |
| 6 | cardio_training | ✅ |
| 7 | yoga | ✅ Confirmado 2026-08-25. **Sí acepta y conserva `category`/`exerciseName`** |
| 8 | pilates | ✅ |
| 9 | hiit | ⚠️ Cuidado: NO es pilates |
| 11 | **mobility** | ✅ Existe. El sport propio de una sesión de movilidad |
| 12 | walking | ✅ |
| 13 | rucking | ⚠️ Cuidado: NOT strength |
| 3 | other | ✅ |

Lista **enumerada empíricamente el 2026-08-29** subiendo un workout por id y leyendo qué
`sportTypeKey` devuelve Garmin. El id 10 da HTTP 400; del 14 en adelante, `null`. Garmin **corrige
el `sportTypeKey` en silencio** a partir del id: si mandas `{9, "pilates"}` te lo guarda como
`hiit`. Fía siempre del id, y **relee el workout tras subirlo** para confirmar el sport.

> ✅ **Animaciones y nombres de paso (2026-08-29, confirmado en un EPIX PRO).** El reloj **no**
> pinta el mismo campo en todos los sports, y esto decide el diseño de la sesión:
>
> | | `yoga` (7) | `strength_training` (5) |
> |---|---|---|
> | Animaciones | ❌ no | ✅ sí, pero **no durante el entrenamiento** |
> | Nombre en pantalla | tu `description` | **el nombre del ejercicio de la librería de Garmin** |
> | Paso sin `exerciseName` | tu `description` | **`Go / paso N`** genérico |
> | **Cuenta atrás en paso por tiempo** | ✅ sí | ❌ **no** |
>
> ⚠️ **Lo más importante de la tabla es la última fila.** En `strength_training` el reloj espera
> series por repeticiones: un paso `interval` con `endCondition: time` se pinta como cronómetro de
> serie, **sin cuenta atrás**. Para una sesión gobernada por tiempo eso es descalificatorio.
> Es la misma trampa que `recovery` (4) vs `rest` (5), un nivel más arriba.
> **Regla: si la sesión se gobierna por tiempo, no la subas como `strength_training`.**
>
> Consecuencia dura: en `strength_training` **pierdes todo control sobre el texto de la pantalla**.
> Elegir el ejercicio *es* elegir el nombre, y cualquier paso sin ejercicio sale como "Go paso N".
> No metas reps ni tempo dentro del nombre en un workout de fuerza — no se ven.
>
> `mobility` (11) y `pilates` (8) están sin probar en reloj; son los candidatos a dar animaciones
> **y** un registro de disciplina limpio.

> ❌ **`stepName` no sirve: Garmin lo vacía al subir.** Verificado el 2026-08-29 contra el JSON
> crudo de la API (`connectapi.garmin.com/workout-service/workout/{id}`), no solo con el lector
> curado del MCP: subes `"stepName": "NOMBRE UNO"` y al releer vuelve `None`. Pasa en
> `strength_training` **y** en `yoga`. El campo existe en el DTO pero no se persiste. El nombre
> vive en `description` — y solo se ve en los sports que pintan `description`.

> **No asumas que el sportType limita la librería de ejercicios.** Un workout de yoga conserva
> `category`/`exerciseName` igual que uno de fuerza (comprobado el 2026-08-25 con un workout de
> prueba de 2 pasos). Que no exista una *categoría* de yoga en `Exercises.json` no significa que
> un workout de yoga no pueda usar ejercicios: la librería es única y sirve para todos.

**Dos categorías que se pasan por alto y resuelven casi toda la movilidad:**
- **`SUSPENSION`** — es la categoría de **TRX**: `ROW`, `PUSH_UP`, `SQUAT`, `LUNGE`,
  `SINGLE_LEG_DEADLIFT`, `GLUTE_BRIDGE`, `SIDE_PLANK`, `FACE_PULL`, `PULL_UP`, `PIKE`,
  `HAMSTRING_CURL`, `Y_FLY`, `POWER_PULL`…
- **`WARM_UP`** — todos los estiramientos y la movilidad: `CAT_CAMEL`, `THORACIC_ROTATION`,
  `STRETCH_PIRIFORMIS`, `STRETCH_LUNGE_WITH_SPINAL_TWIST`, `STRETCH_LUNGING_HIP_FLEXOR`,
  `STRETCH_PECTORAL`, `STRETCH_LAT`, `STRETCH_HAMSTRING`, `STRETCH_CHILDS_POSE`…

Antes de dejar un paso como bloque de tiempo sin ejercicio, **busca en estas dos**. Solo déjalo
pelado si de verdad no hay equivalente (p. ej. respiración o báscula pélvica no existen), y nunca
metas un ejercicio "parecido pero no es" — ensucia el registro.

> Para gimnasio/fuerza por repeticiones, usa siempre `sportTypeId: 5`. Si la sesión se gobierna
> por tiempo, ver la tabla de arriba: `strength_training` no hace cuenta atrás.

---

### Step types

| stepTypeId | stepTypeKey | Use for |
|---|---|---|
| 1 | warmup | Warm-up block |
| 2 | cooldown | Cool-down block |
| 3 | interval | Work sets |
| 4 | recovery | ❌ NO usar para strength — no muestra cuenta atrás en el reloj |
| 5 | **rest** | **Rest entre series — SIEMPRE este para fuerza** |
| 6 | repeat | Repeat group wrapper |

> ⚠️ CRÍTICO: `stepTypeId: 4 / "recovery"` no muestra cuenta atrás en el reloj Garmin. Usar `stepTypeId: 5 / "rest"`. Error confirmado el 2026-07-13 en un EPIX PRO: un workout entero subido con 4 no mostraba la cuenta atrás de los descansos.

---

### End conditions

| conditionTypeId | conditionTypeKey | Use for |
|---|---|---|
| 2 | time | Timed steps (value in seconds) |
| 7 | iterations | Repeat group loop count |
| 10 | reps | Rep-based sets |

---

### Executable step (single step)

```json
{
  "type": "ExecutableStepDTO",
  "stepOrder": 1,
  "childStepId": null,
  "stepType": { "stepTypeId": 3, "stepTypeKey": "interval" },
  "description": "Notes shown in app (optional)",
  "endCondition": { "conditionTypeId": 10, "conditionTypeKey": "reps" },
  "endConditionValue": 10,
  "targetType": { "workoutTargetTypeId": 1, "workoutTargetTypeKey": "no.target" },
  "category": "SQUAT",
  "exerciseName": "GOBLET_SQUAT"
}
```

**Peso objetivo por paso (confirmado 2026-08-25):** añade `weightValue` + `weightUnit` al paso y
Garmin los conserva. El reloj muestra la carga objetivo y prellena la serie, en vez de obligar a
leer la descripción. Funciona también en workouts que no son de fuerza.

```json
"weightValue": 17.5,
"weightUnit": { "unitId": 8, "unitKey": "kilogram", "factor": 1000 }
```

Omítelo en peso corporal (dominadas, planchas) y en goma. Decide una convención para mancuernas
(**peso por mancuerna o total combinado**, p. ej. 2×10 kg → `weightValue: 20`), apúntala en tu
perfil y no la cambies.

**Calentamiento y vuelta a la calma:** no los dejes en un `WALKOUT` / `STRETCH_CHILDS_POSE`
genérico si la descripción dice otra cosa. Móntalos como varios pasos `warmup`/`cooldown` con
ejercicios reales — `THORACIC_ROTATION`, `SHOULDER_CIRCLES`, `GROINERS`,
`BANDED_EXERCISES`/`PULL_APART`, y los `STRETCH_*` específicos de lo que se ha entrenado.

- `stepOrder` must be sequential across the entire segment (not restarting inside repeat groups)
- `childStepId` is `null` for top-level steps; inside a repeat group it matches the repeat group's `childStepId` value
- `category` and `exerciseName` must use values from the Garmin exercise database (see reference file)
- `description` is the notes field — use it to show exercise cues (e.g. "10 reps, 3 sec descent")
- **Field names confirmados** (2026-07-13): el campo correcto es `category` (snake_case) + `exerciseName` (camelCase). Variantes `exerciseCategory` o `exercise_name` NO funcionan en upload.

---

### Repeat group (3 sets of an exercise)

```json
{
  "type": "RepeatGroupDTO",
  "stepOrder": 2,
  "childStepId": 1,
  "stepType": { "stepTypeId": 6, "stepTypeKey": "repeat" },
  "endCondition": { "conditionTypeId": 7, "conditionTypeKey": "iterations" },
  "endConditionValue": 3,
  "numberOfIterations": 3,
  "skipLastRestStep": false,
  "smartRepeat": false,
  "workoutSteps": [
    {
      "type": "ExecutableStepDTO",
      "stepOrder": 3,
      "childStepId": 1,
      "stepType": { "stepTypeId": 3, "stepTypeKey": "interval" },
      "description": "Goblet Squat — 10 reps, 3 sec descent",
      "endCondition": { "conditionTypeId": 10, "conditionTypeKey": "reps" },
      "endConditionValue": 10,
      "targetType": { "workoutTargetTypeId": 1, "workoutTargetTypeKey": "no.target" },
      "category": "SQUAT",
      "exerciseName": "GOBLET_SQUAT"
    },
    {
      "type": "ExecutableStepDTO",
      "stepOrder": 4,
      "childStepId": 1,
      "stepType": { "stepTypeId": 5, "stepTypeKey": "rest" },
      "description": "Rest 3 min",
      "endCondition": { "conditionTypeId": 2, "conditionTypeKey": "time" },
      "endConditionValue": 180,
      "targetType": { "workoutTargetTypeId": 1, "workoutTargetTypeKey": "no.target" },
      "category": null,
      "exerciseName": null
    }
  ]
}
```

**Key rules for repeat groups:**
- `childStepId` on the RepeatGroupDTO = an incrementing integer (1 for first group, 2 for second, etc.)
- All child steps inside the group share the same `childStepId` as the group
- `stepOrder` is global — it never resets inside a group
- `endConditionValue` and `numberOfIterations` must match

---

### Step ordering example (3 exercises × 3 sets)

```
stepOrder 1  — warmup (childStepId: null)
stepOrder 2  — RepeatGroupDTO (childStepId: 1)
  stepOrder 3  — interval / Exercise A (childStepId: 1)
  stepOrder 4  — rest (childStepId: 1)
stepOrder 5  — RepeatGroupDTO (childStepId: 2)
  stepOrder 6  — interval / Exercise B (childStepId: 2)
  stepOrder 7  — rest (childStepId: 2)
stepOrder 8  — RepeatGroupDTO (childStepId: 3)
  stepOrder 9  — interval / Exercise C (childStepId: 3)
  stepOrder 10 — rest (childStepId: 3)
stepOrder 11 — cooldown (childStepId: null)
```

---

### Exercise names

Exercise `category` and `exerciseName` values **must exactly match** the Garmin exercise database. See `references/exercises.md` for the full list.

**Format rules:**
- `category`: UPPER_CASE (e.g. `SQUAT`, `ROW`, `BENCH_PRESS`)
- `exerciseName`: UPPER_SNAKE_CASE (e.g. `GOBLET_SQUAT`, `DUMBBELL_ROW`, `ROMANIAN_DEADLIFT`)

**Common exercises quick reference:**

| Exercise | category | exerciseName |
|---|---|---|
| Goblet Squat | SQUAT | GOBLET_SQUAT |
| Barbell Back Squat | SQUAT | BACK_SQUATS |
| Romanian Deadlift | DEADLIFT | ROMANIAN_DEADLIFT |
| Dumbbell Bench Press | BENCH_PRESS | DUMBBELL_BENCH_PRESS |
| Dumbbell Row (unilateral) | ROW | DUMBBELL_ROW |
| Dumbbell Biceps Curl | CURL | DUMBBELL_BICEPS_CURL |
| **Pull Up** | **PULL_UP** | **PULL_UP** ← ⚠️ NO usar PULLUP, no es válido |
| Plank | PLANK | PLANK |
| Dead Bug | HIP_STABILITY | DEAD_BUG |
| Jump Rope | CARDIO | JUMP_ROPE |
| Hip Flexor Stretch | WARM_UP | STRETCH_HIP_FLEXOR_AND_QUAD |
| **Barbell Hip Thrust** | **HIP_RAISE** | **BARBELL_HIP_THRUST_WITH_BENCH** |
| **Pallof press** | **CORE** | **CABLE_CORE_PRESS** ← ⚠️ `PALLOF` no existe |
| Band pull-apart | BANDED_EXERCISES | PULL_APART |
| Remo unilateral (preciso) | ROW | ONE_ARM_BENT_OVER_ROW |
| Hip thrust en suelo | HIP_RAISE | BARBELL_HIP_THRUST_ON_FLOOR |
| Búlgara con mancuernas | LUNGE | DUMBBELL_BULGARIAN_SPLIT_SQUAT |
| Farmer carry | CARRY | FARMERS_CARRY |
| Jalón al pecho | PULL_UP | LAT_PULLDOWN |
| Press inclinado mancuernas | BENCH_PRESS | INCLINE_DUMBBELL_BENCH_PRESS |
| Press hombro sentado | SHOULDER_PRESS | SEATED_DUMBBELL_SHOULDER_PRESS |
| Face pull | ROW | FACE_PULL |
| Side plank | PLANK | SIDE_PLANK |
| **Reverse Dumbbell Box Lunge** | **LUNGE** | **REVERSE_DUMBBELL_BOX_LUNGE** |
| **Dumbbell Lateral Raise** | **LATERAL_RAISE** | **DUMBBELL_LATERAL_RAISE** |
| Walkout (warm-up) | WARM_UP | WALKOUT |
| Child's Pose (cool-down) | WARM_UP | STRETCH_CHILDS_POSE |

> ⚠️ **PULLUP no existe** como exerciseName en la base de Garmin. El nombre correcto es `PULL_UP`. Si se envía un nombre inválido, el campo `exercise_name` no se guarda (confirmado 2026-07-13).

For exercises not in this table, always consult `references/exercises.md` to find the exact string. Never guess.

---

## Step 2 — Upload

```
garmin:upload_workout(workout_data = { ... })
```

On success, the response contains `workout_id`. Note it in case the user needs to delete or schedule the workout.

**Después de subir, relee el workout** (`get_workout_by_id`) y comprueba que el `sportTypeKey`, los
`exerciseName` y los `rest` siguen ahí. Garmin descarta en silencio lo que no reconoce.

---

## Step 3 — Schedule (optional)

If the user wants the workout on their calendar:

```
garmin:schedule_workout(workout_id = <id>, calendar_date = "YYYY-MM-DD")
```

---

## Common mistakes to avoid

- ❌ **`sportTypeId` incorrecto** — 4=swimming, 13=rucking. Para fuerza siempre `5`
- ❌ **`stepTypeKey: "recovery"` (stepTypeId: 4)** — no muestra cuenta atrás de descanso en el reloj. Usar siempre `"rest"` (stepTypeId: 5) para strength
- ❌ **`exerciseCategory` o `exercise_name`** en el upload — los nombres de campo correctos son `category` (snake_case) y `exerciseName` (camelCase)
- ❌ **`PULLUP`** como exerciseName — no existe en la DB de Garmin, el campo queda vacío. Usar `PULL_UP`
- ❌ Using exercise names not in the Garmin database — the field will appear empty in the app
- ❌ Resetting `stepOrder` inside repeat groups — it must be globally sequential
- ❌ Mismatching `childStepId` — group and all its children must share the same value
- ❌ Putting exercise cues in `exerciseName` — that field is a database key, put cues in `description`
- ❌ Using `endConditionValue` and `numberOfIterations` with different values in repeat groups

---

## Reference files

- `../../GOTCHAS.md` (en la raíz del repo) — catálogo de trampas con síntoma y regla

- `references/exercises.md` — Full Garmin exercise database (category + name for every exercise)

Read `references/exercises.md` whenever you need an exercise not in the quick reference table above.
