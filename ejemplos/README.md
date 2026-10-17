# Ejemplos

## `fuerza-full-body.json`

Un workout de fuerza completo: dos pasos de calentamiento, cinco ejercicios en grupos de repetición con su descanso y un paso de vuelta a la calma. Los ejercicios y las cargas son inventados; sustitúyelos por los tuyos.

Es el que sube la skill `primeros-pasos` para comprobar que todo funciona. También puedes subirlo a mano pidiéndole a Claude Code: *"Sube a mi Garmin el workout de `fuerza-full-body.json`"*.

JSON no admite comentarios y Garmin podría rechazar campos que no conoce, así que el fichero va limpio y los comentarios están aquí. Estos son los campos que más se fallan, en el orden en que aparecen:

### `sportType`

```json
"sportType": { "sportTypeId": 5, "sportTypeKey": "strength_training" }
```

- Va **dos veces**: en la raíz y dentro de cada segmento. Las dos iguales.
- `5` es fuerza. `4` es natación y `13` es rucking, no fuerza. Garmin manda el id y reescribe la clave en silencio. → [GOTCHAS #8 y #9](../GOTCHAS.md)

### `stepOrder`

Numera **todos** los pasos del 1 al 18 de forma consecutiva, incluidos los grupos de repetición y los pasos dentro de ellos. No se reinicia dentro de un grupo. → [GOTCHAS #4](../GOTCHAS.md)

### `childStepId`

- `null` en los pasos sueltos (calentamiento y vuelta a la calma).
- `1`, `2`, `3`… en cada grupo de repetición, por orden.
- Los dos pasos dentro de cada grupo (ejercicio y descanso) llevan el mismo número que su grupo. → [GOTCHAS #5](../GOTCHAS.md)

### `endConditionValue` y `numberOfIterations` en los grupos

Las dos con el mismo valor: el número de series.

### Descanso: `rest`, no `recovery`

```json
"stepType": { "stepTypeId": 5, "stepTypeKey": "rest" }
```

Con `stepTypeId: 4` (`recovery`) el descanso no hace cuenta atrás en el reloj. → [GOTCHAS #1](../GOTCHAS.md)

### `category` y `exerciseName`

```json
"category": "LUNGE",
"exerciseName": "DUMBBELL_REVERSE_LUNGE"
```

- Esos nombres de campo exactos. `exerciseCategory` o `exercise_name` se ignoran. → [GOTCHAS #3](../GOTCHAS.md)
- Los valores tienen que existir en la librería de Garmin ([`exercises.md`](../skills/garmin-workout-creator/references/exercises.md)). Si no existen, se descartan sin error y el paso sale como "Go" en el reloj. → [GOTCHAS #2 y #7](../GOTCHAS.md)
- En los pasos de descanso, los dos van a `null`.

### `description`

Las indicaciones ("8 por pierna, rodilla trasera casi al suelo") van aquí, no en `exerciseName`. Se ven en la app; en la pantalla de fuerza del reloj se ve el nombre del ejercicio de Garmin. No uses `stepName`: Garmin lo vacía al subir. → [GOTCHAS #7](../GOTCHAS.md)

### `weightValue` y `weightUnit`

```json
"weightValue": 20,
"weightUnit": { "unitId": 8, "unitKey": "kilogram", "factor": 1000 }
```

Opcional. El reloj muestra la carga objetivo y la propone al registrar la serie. Se omite en peso corporal (flexiones, curl-up). En este ejemplo, el peso de las mancuernas es el total de las dos.

### Por qué todo va por repeticiones

Incluso el calentamiento y la vuelta a la calma. En `strength_training` un paso por tiempo no hace cuenta atrás en el reloj. Si tu sesión se gobierna por tiempo, súbela con otro deporte. → [GOTCHAS #6](../GOTCHAS.md)

### Después de subir

Relee el workout desde Garmin y comprueba que el deporte, los ejercicios y los descansos siguen como los mandaste. Garmin no avisa de lo que descarta.
