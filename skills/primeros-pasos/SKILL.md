---
name: primeros-pasos
description: Onboarding del sistema de entrenamiento con Garmin. Comprueba que la conexión con Garmin funciona, sube un workout de prueba y lo verifica, prepara el perfil, el plan y el log en la carpeta de trabajo y propone la primera sesión. Úsala cuando el usuario diga "empecemos", "primeros pasos", "acabo de instalarlo", "configura esto", "¿funciona?", o cuando la carpeta de trabajo tenga las plantillas sin rellenar.
---

# Primeros pasos

Lleva al usuario desde "acabo de instalarlo" hasta "tengo mi primera sesión en el reloj". El objetivo es arrancar rápido, no tenerlo todo perfecto: el contexto bueno sale de usar el sistema, no de escribirlo el primer día.

Ve paso a paso, di en una línea qué estás haciendo y no avances si un paso falla.

## 1. Comprobar la conexión con Garmin

Llama a una herramienta de lectura del MCP de Garmin, por ejemplo `get_devices` o `get_full_name`.

- **Funciona:** confirma con qué cuenta y qué reloj (sin repetir datos personales que no hagan falta) y sigue.
- **No aparecen herramientas de Garmin:** el MCP no está configurado. Indica que ejecute `claude mcp list`, y si `garmin` no aparece, el comando `claude mcp add` del README. Después hay que reiniciar Claude Code.
- **Error de autenticación:** casi siempre son tokens caducados (duran ~6 meses) o que nunca se generaron. Indica que ejecute en una terminal:
  ```bash
  uvx --python 3.12 --from git+https://github.com/Taxuspt/garmin_mcp garmin-mcp-auth --force-reauth
  ```
  Nunca pidas la contraseña de Garmin en la conversación. La autenticación la hace el usuario en su terminal.

## 2. Subir un workout de prueba y verificarlo

Si en la carpeta de trabajo está `fuerza-full-body.json`, súbelo tal cual con la skill `garmin-workout-creator`. Si no está, monta uno pequeño con esa skill (dos o tres ejercicios en grupos de repetición con descanso `rest`).

Después de subirlo:

1. **Reléelo** desde Garmin y comprueba que el deporte es `strength_training`, que cada ejercicio conserva `category` y `exerciseName` y que los descansos son `rest`. Si algo se ha perdido, dilo: es justo el tipo de fallo que documenta `GOTCHAS.md`.
2. Ofrece programarlo para hoy o mañana.
3. Pide al usuario que sincronice el reloj, entre en el entrenamiento y mire tres cosas: que los ejercicios tienen nombre (no "Go"), que los descansos hacen cuenta atrás y que propone el peso en las series que lo llevan.

Si alguna de las tres falla en su reloj, es información valiosa: apúntala con `persistir-aprendizaje` (síntoma, modelo de reloj, fecha) antes de seguir.

Cuando haya terminado la prueba, pregunta si quiere conservar el workout de ejemplo o borrarlo.

## 3. Preparar la carpeta de trabajo

Comprueba que en la carpeta actual están `perfil.md`, `plan.md` y `log.md`. Si falta alguno, créalo con la misma estructura que las plantillas del repo.

Para el perfil, empieza por lo que hace falta para montar una primera sesión: objetivo, días disponibles, minutos por sesión y material. Pregúntalo de forma natural, en un solo mensaje. Si el usuario quiere contar más (ejercicios que no puede hacer, preferencias, cómo apunta las cargas), escríbelo también en su sección. Si no, no insistas: el resto se irá llenando con el uso.

Recuérdale, en una línea, que es normal que el perfil se quede corto al principio.

Si la carpeta no está bajo git, ofrece hacer `git init`: así `persistir-aprendizaje` puede confirmar con un commit cada regla que guarde. Si la carpeta está dentro de un clon de este repo público, avisa de que no conviene guardar ahí sus datos personales y sugiere una carpeta aparte.

## 4. Proponer la primera sesión

Con lo que haya en el perfil, propone una primera sesión sencilla y fácil de ejecutar: pocos ejercicios, cargas conservadoras y un formato por repeticiones. Enséñasela antes de subirla, ajústala con lo que diga el usuario y súbela con `garmin-workout-creator`.

Escribe en `plan.md` la sesión tal como quedó y, en "Decisiones", una línea con la fecha y el porqué de las elecciones principales.

## 5. Cerrar

Termina con un resumen corto:

- Qué funciona ya (conexión, workout de prueba, primera sesión subida y para cuándo).
- Qué queda en el aire, si algo falló.
- Qué hacer ahora: ir a entrenar y apuntar en `log.md` todo lo que falle o sorprenda, aunque sea en bruto. La próxima conversación empieza por ahí.
