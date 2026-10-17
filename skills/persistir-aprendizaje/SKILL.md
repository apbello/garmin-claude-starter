---
name: persistir-aprendizaje
description: Convierte cada fallo, corrección o preferencia nueva en una regla escrita en el fichero correcto, para que la próxima sesión ya la sepa. Úsala cuando el usuario diga "acuérdate de", "apúntalo", "a partir de ahora", "esto ya lo habíamos visto", "otra vez lo mismo", cuando señale un error que no debería repetirse, o cuando en el gimnasio algo salga distinto de lo esperado (un ejercicio sin nombre en el reloj, un descanso sin cuenta atrás, una máquina que no existe, una carga que no cuadra).
---

# Persistir aprendizaje

Esta skill es la pieza que hace que el sistema mejore en vez de descubrir lo mismo tres veces. Sin ella, cada conversación empieza de cero y los fallos se repiten. Con ella, cada fallo se escribe una vez, en un sitio concreto, y la siguiente sesión lo lee.

La regla de oro: **un aprendizaje que solo está en la conversación no existe.** Si no está en un fichero, se ha perdido.

## Dónde se escribe cada cosa

Cada aprendizaje va a **un solo sitio**, el más específico que aplique:

| Qué has aprendido | Dónde va | Ejemplo |
|---|---|---|
| Una trampa técnica de Garmin (formato, campos, comportamiento del reloj) | `GOTCHAS.md` y, si cambia cómo se construye un workout, la `SKILL.md` de `garmin-workout-creator` | "En `strength_training` los pasos por tiempo no hacen cuenta atrás" |
| Algo sobre ti: material, horarios, ejercicios que no puedes o no quieres hacer, convenciones | `perfil.md` | "Las mancuernas se apuntan en peso total combinado" |
| Una decisión sobre el entrenamiento: progresión, cambio de ejercicio, ajuste de volumen | `plan.md` | "Sube 2,5 kg en sentadilla cuando completes las 3 series sin fallo" |
| Lo que pasó en una sesión concreta | `log.md` | "Martes: la polea alta estaba ocupada, hice el jalón con goma" |
| Una regla sobre cómo debe trabajar Claude contigo | `perfil.md`, sección de preferencias | "Antes de subir un workout, enséñame la lista de ejercicios" |

Si una nota del log se repite dos veces, ya no es un evento: es una regla. Súbela a `perfil.md`, `plan.md` o `GOTCHAS.md`.

## Quién la escribe

**Claude la escribe; el usuario la aprueba.** El usuario no tiene que redactar nada: basta con que diga qué ha pasado. Claude:

1. Formula la regla en una línea, en imperativo y con el porqué si no es obvio.
2. Decide el fichero según la tabla de arriba.
3. Comprueba si ya existe una regla parecida. Si existe, la actualiza en vez de añadir otra. Si la nueva contradice a una antigua, sustituye la antigua y lo dice.
4. Escribe la regla con fecha (`AAAA-MM-DD`). Para trampas de Garmin, añade también el modelo de reloj y el síntoma observado.

Si el usuario no está presente (por ejemplo, Claude detecta un fallo al releer un workout recién subido), escribe la regla igualmente y lo dice en el siguiente mensaje.

## Cómo se confirma que quedó guardada

No basta con decir "apuntado". Antes de responder:

1. **Relee el fichero** después de escribir y comprueba que la línea está ahí, en la sección correcta.
2. Si el directorio está bajo git, **haz commit** con un mensaje que diga qué regla se añadió (`Añade regla: rest en vez de recovery en fuerza`).
3. **Responde con tres cosas:** qué regla se escribió (literal), en qué fichero, y qué va a cambiar la próxima vez.

Ejemplo de respuesta:

> Guardado en `GOTCHAS.md`: "En fuerza, el descanso es `stepTypeId: 5` (`rest`), nunca `4` (`recovery`). Síntoma: sin cuenta atrás. 2026-07-13." La próxima vez que monte un workout de fuerza, los descansos irán con `rest`.

## Qué no hacer

- No guardes la conversación entera: guarda la regla.
- No crees un fichero nuevo por cada aprendizaje. Hay pocos sitios y cada cosa tiene el suyo.
- No apuntes hipótesis como si fueran hechos. Si algo no está confirmado, escríbelo como "sin confirmar" y di qué prueba lo resolvería.
- No esperes al domingo para escribir algo que se descubrió hoy. El repaso semanal sirve para consolidar, no para recordar.

## Repaso semanal

Una vez a la semana (veinte minutos bastan), con el usuario:

1. Lee el `log.md` de la semana.
2. Busca patrones: lo que se repitió, lo que falló más de una vez, lo que se cambió sobre la marcha.
3. Sube a `perfil.md`, `plan.md` o `GOTCHAS.md` lo que se haya convertido en regla.
4. Ajusta el plan de la semana siguiente y sube los workouts.
