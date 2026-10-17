# Gotchas de Garmin Connect

Catálogo de trampas descubiertas subiendo workouts a Garmin Connect a través de [garmin_mcp](https://github.com/Taxuspt/garmin_mcp). Casi ninguna da error: Garmin acepta el workout y el fallo aparece en el reloj, a mitad de sesión.

Cada entrada tiene el **síntoma** (lo que ves), la **regla** (lo que hay que hacer) y **cuándo se confirmó**. Las pruebas en reloj se hicieron en un **Garmin EPIX PRO**; otros modelos pueden comportarse distinto. Las que se verificaron leyendo la respuesta de la API lo indican.

Si encuentras una nueva o una de estas deja de ser cierta, abre un PR con el mismo formato.

---

## 1. `recovery` no hace cuenta atrás; `rest` sí

**Síntoma:** el descanso entre series aparece en el reloj, pero sin cuenta atrás. No sabes cuándo empezar la siguiente serie.

**Regla:** en fuerza, el descanso entre series es siempre `stepTypeId: 5` / `stepTypeKey: "rest"`. Nunca `stepTypeId: 4` / `"recovery"`.

```json
"stepType": { "stepTypeId": 5, "stepTypeKey": "rest" }
```

**Confirmado:** 2026-07-13, EPIX PRO. Un workout entero subido con `recovery` en todos los descansos.

---

## 2. `PULL_UP`, no `PULLUP`

**Síntoma:** el paso de dominadas sale sin ejercicio en la app y en el reloj. Garmin no da error: simplemente descarta el campo.

**Regla:** el nombre es `PULL_UP` (categoría `PULL_UP`). En general, **cualquier `exerciseName` que no exista en la librería de Garmin se descarta en silencio**. No adivines nombres: búscalos en [`references/exercises.md`](skills/garmin-workout-creator/references/exercises.md).

Otros que caen igual: `PALLOF` no existe (es `CORE` / `CABLE_CORE_PRESS`).

**Confirmado:** 2026-07-13, EPIX PRO.

---

## 3. `category` y `exerciseName`: los nombres exactos de los campos

**Síntoma:** el workout sube sin errores pero ningún paso tiene ejercicio.

**Regla:** los campos se llaman exactamente `category` y `exerciseName`. Las variantes `exerciseCategory` o `exercise_name` se ignoran en la subida. Los valores van en MAYÚSCULAS con guiones bajos (`GOBLET_SQUAT`).

```json
"category": "SQUAT",
"exerciseName": "GOBLET_SQUAT"
```

**Confirmado:** 2026-07-13.

---

## 4. `stepOrder` es global, no se reinicia dentro de un grupo

**Síntoma:** no está aislado. La regla sale de comparar los workouts que funcionaban con los que no; si lo reinicias dentro de un grupo, estás fuera del formato que Garmin genera y el comportamiento deja de ser predecible.

**Regla:** `stepOrder` numera **todos** los pasos del segmento de forma consecutiva, incluidos los que van dentro de un grupo de repetición. El grupo tiene su propio `stepOrder` y sus hijos siguen la cuenta.

```
stepOrder 1  — warmup                  (childStepId: null)
stepOrder 2  — RepeatGroupDTO          (childStepId: 1)
  stepOrder 3  — interval / ejercicio A (childStepId: 1)
  stepOrder 4  — rest                   (childStepId: 1)
stepOrder 5  — RepeatGroupDTO          (childStepId: 2)
  stepOrder 6  — interval / ejercicio B (childStepId: 2)
  stepOrder 7  — rest                   (childStepId: 2)
stepOrder 8  — cooldown                (childStepId: null)
```

**Confirmado:** forma parte de la estructura validada el 2026-07-13/14.

---

## 5. `childStepId` lo comparten el grupo y todos sus hijos

**Síntoma:** grupos de repetición que no se comportan como un bloque. En el caso donde se detectó, faltaba `childStepId` a la vez que otros fallos, así que el síntoma exacto no está aislado.

**Regla:**

- Pasos sueltos (fuera de grupos): `childStepId: null`.
- Cada `RepeatGroupDTO` lleva un entero creciente: 1 el primero, 2 el segundo…
- Todos los pasos dentro de un grupo llevan **el mismo** `childStepId` que su grupo.
- En el grupo, `endConditionValue` y `numberOfIterations` deben tener el mismo valor.

**Confirmado:** 2026-07-14, rehaciendo con esta estructura un workout que estaba roto; quedó correcto.

---

## 6. En `strength_training`, los pasos por tiempo no hacen cuenta atrás

**Síntoma:** un ejercicio de 45 segundos aparece como cronómetro de serie que cuenta hacia arriba. No sabes cuándo has terminado.

**Regla:** en `strength_training` el reloj espera series **por repeticiones**. Un paso `interval` con `endCondition: time` se pinta sin cuenta atrás. Es la misma trampa que `recovery` vs `rest`, un nivel más arriba.

**Si la sesión se gobierna por tiempo (movilidad, circuitos, estiramientos), no la subas como `strength_training`.** En `yoga` (id 7) sí hay cuenta atrás. `mobility` (11) y `pilates` (8) están sin probar en reloj.

| | `yoga` (7) | `strength_training` (5) |
|---|---|---|
| Cuenta atrás en paso por tiempo | ✅ sí | ❌ no |
| Animaciones del ejercicio | ❌ no | ✅ sí, pero no durante el entrenamiento |
| Nombre en pantalla | tu `description` | el nombre del ejercicio en la librería de Garmin |
| Paso sin `exerciseName` | tu `description` | `Go` / `paso N` genérico |

**Confirmado:** 2026-08-29, EPIX PRO, subiendo la misma sesión en los dos deportes.

---

## 7. "Go" / "paso N": en fuerza no controlas el texto de la pantalla

**Síntoma:** pasos que en el reloj solo dicen "Go" o "paso 3", aunque les pusiste una descripción clara.

**Regla:** en `strength_training` el reloj muestra **el nombre del ejercicio de la librería de Garmin**, no tu texto. Elegir el ejercicio *es* elegir el nombre. Cualquier paso sin `exerciseName` sale como "Go" / "paso N".

- No metas repeticiones, tempo ni indicaciones en el nombre: no se ven. Van en `description` (se ven en la app, no en la pantalla de fuerza).
- Antes de dejar un paso sin ejercicio, busca en `WARM_UP` (estiramientos y movilidad: `CAT_CAMEL`, `THORACIC_ROTATION`, `STRETCH_*`) y en `SUSPENSION` (la categoría de TRX). Resuelven casi todo.
- Nunca metas un ejercicio "parecido pero no es": ensucia el registro.

**Relacionado:** `stepName` no sirve. Existe en el formato, pero Garmin lo vacía al subir: subes `"stepName": "LO QUE SEA"` y al releer vuelve `None`, tanto en `strength_training` como en `yoga`.

**Confirmado:** 2026-08-29, EPIX PRO. Lo de `stepName`, verificado además contra el JSON crudo de la API.

---

## 8. La tabla de `sportTypeId` hay que enumerarla a mano

**Síntoma:** pides un workout de fuerza y en el reloj aparece como natación. O pides pilates y te llega HIIT.

**Regla:** los ids no son los que uno supondría y no hay una lista oficial a mano. Esta se sacó subiendo un workout por cada id y leyendo qué deporte devolvía Garmin:

| sportTypeId | sportTypeKey | Ojo |
|---|---|---|
| 1 | running | |
| 2 | cycling | |
| 3 | other | |
| 4 | swimming | ⚠️ **no** es fuerza |
| 5 | strength_training | el de gimnasio |
| 6 | cardio_training | |
| 7 | yoga | conserva `category`/`exerciseName` |
| 8 | pilates | |
| 9 | hiit | ⚠️ **no** es pilates |
| 10 | — | HTTP 400 |
| 11 | mobility | existe; sin probar en reloj |
| 12 | walking | |
| 13 | rucking | ⚠️ **no** es fuerza |
| 14+ | — | `null` |

La librería de ejercicios es una sola para todos los deportes: un workout de yoga conserva `category`/`exerciseName` igual que uno de fuerza (confirmado 2026-08-25).

**Confirmado:** enumerada el 2026-08-29 contra la API.

---

## 9. Garmin corrige el `sportTypeKey` en silencio

**Síntoma:** mandas `{"sportTypeId": 9, "sportTypeKey": "pilates"}`, la subida va bien y el workout queda guardado como `hiit`.

**Regla:** Garmin manda el **id** y reescribe la clave para que cuadre, sin avisar. Fíate siempre del id y **relee el workout después de subirlo** para confirmar el deporte.

**Confirmado:** 2026-08-29, contra la API.

---

## Bonus: el peso objetivo sí se puede poner

No es una trampa sino algo que no está documentado: añade `weightValue` y `weightUnit` a un paso y Garmin lo conserva. El reloj muestra la carga objetivo y la propone al registrar la serie.

```json
"weightValue": 20,
"weightUnit": { "unitId": 8, "unitKey": "kilogram", "factor": 1000 }
```

Omítelo en peso corporal y en gomas. Decide una convención para mancuernas (peso por mancuerna o total de las dos), apúntala y no la cambies.

**Confirmado:** 2026-08-25.
