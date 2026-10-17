# AGENTS.md

Instrucciones para los asistentes de IA (y para las personas) que trabajen en este repo. Léelo completo antes de escribir código.

## El proyecto en tres líneas

Asistente de finanzas personales. Clasifica movimientos bancarios (ML clásico), pronostica el gasto del mes (regresión), arma un plan de ahorro óptimo (programación lineal) y lo explica todo por chat (agente LLM con *tool use*). Es el proyecto final de IA (SI3003) de EAFIT. La especificación completa está en [`README.md`](README.md) y los contratos de datos en [`docs/contratos.md`](docs/contratos.md).

## Reglas de oro (no negociables)

1. **El LLM nunca calcula.** Toda cifra que vea el usuario sale de código Python (herramientas, modelo u optimizador). El LLM solo elige la herramienta y explica el resultado. Si una respuesta necesita un número que ninguna herramienta devuelve, se crea la herramienta, no se le pide al LLM.
2. **El agente solo lee.** Ninguna herramienta del agente escribe, actualiza ni borra en la base de datos. Guardar un plan lo hace el usuario con un botón de la interfaz.
3. **El `usuario_id` nunca es un parámetro de una herramienta.** La identidad sale del JWT de la sesión, no de lo que diga el LLM.
4. **El backend consulta Supabase con el JWT del usuario, nunca con `service_role`.** La llave `service_role` solo se usa en scripts de carga de datos que corren en local y no se exponen.
5. **RLS en todas las tablas**, con la política `auth.uid() = usuario_id`. Una migración sin RLS no se aprueba.
6. **Los montos son enteros en pesos colombianos (COP).** Nunca `float` para plata. Los egresos son negativos.
7. **Nada de secretos en el repo.** Las llaves van en `.env` (ignorado por git). Si agregas una variable, agrégala también, sin valor, a `.env.example`.
8. **Datos sintéticos solamente.** No se suben ni se mandan al proveedor del LLM extractos bancarios reales.

## Idioma

- Todo en español: código (nombres de variables, funciones y tablas), comentarios, commits, PRs y documentación.
- Excepciones: palabras reservadas, librerías y términos técnicos sin traducción natural (`token`, `endpoint`, `tool use`).
- Los identificadores van sin tildes ni `ñ`: `categoria`, `pronostico`, `anio`.

## Stack

| Capa | Tecnología |
|---|---|
| Frontend | Next.js + TypeScript + Tailwind, gráficos con Recharts |
| Backend | FastAPI + Pydantic, Python 3.12 |
| ML y optimización | scikit-learn, scipy (`linprog`) |
| Base de datos y login | Supabase (Auth + Postgres + RLS) |
| LLM | Proveedor configurable por variables de entorno, con el cliente de OpenAI (NVIDIA o Groq; se decide con una prueba) |
| Dependencias de Python | `uv`, con `pyproject.toml` y `uv.lock` versionado |
| Tests | pytest |

No se agregan librerías ni frameworks fuera de esta tabla sin discutirlo en el PR. En particular, **el agente no usa LangChain ni frameworks similares**: el bucle se escribe a mano para poder explicarlo.

Las dependencias se fijan con versión exacta. Con `uv`, el `uv.lock` se sube al repo para que todo el equipo tenga las mismas versiones.

## Estructura

```
frontend/              Next.js
backend/app/           FastAPI: main.py, agente.py, herramientas/, esquemas.py
backend/tests/         pytest
ml/                    generador de datos, entrenamiento y evaluación de modelos
supabase/migrations/   tablas + RLS
evals/                 casos de evaluación del agente (JSONL)
docs/                  contratos y decisiones (docs/decisiones/ para los ADR)
```

Las carpetas se crean cuando llega su fase, no antes.

## Flujo de trabajo en equipo

**Ramas cortas.** Una rama por tarea pequeña, que vive pocos días:

```
feature/<componente>-<tema>     feature/optimizador-modelo-lp
fix/<componente>-<tema>         fix/agente-error-429
docs/<tema>                     docs/contratos
```

- Nunca se hace push directo a `main`.
- Todo entra por PR a `main`, con **Squash and merge**, y al menos **una aprobación** de otra persona del equipo.
- Antes de abrir el PR: actualizar la rama con `main` y correr los tests.

**Commits:** [Conventional Commits](https://www.conventionalcommits.org/es/) con el componente como scope.

```
feat(optimizador): detectar metas imposibles
test(agente): casos de selección de herramienta
fix(clasificador): normalizar mayúsculas en la descripción
docs(contratos): definir la salida de pronosticar_gasto
```

Scopes válidos: `datos`, `clasificador`, `pronostico`, `optimizador`, `agente`, `api`, `db`, `frontend`, `evals`, `docs`, `infra`.

## Tests

- Toda herramienta del agente tiene tests con casos conocidos, incluido el caso de error.
- El optimizador tiene tests de: plan factible, meta imposible, y que el plan respete mínimos y sumas.
- Toda migración nueva tiene una prueba de que un usuario no puede leer filas de otro.
- No se aprueba un PR con tests rotos.

## Cómo responder a errores del LLM

- Las herramientas no lanzan excepciones al LLM. Devuelven `{"error": "mensaje claro"}` y el agente se lo explica al usuario.
- El cliente del LLM maneja el error 429: lee `retry-after`, espera y reintenta (máximo 3 veces).
- El bucle del agente tiene un tope de pasos (por ejemplo, 5 llamadas a herramientas por pregunta) para no gastar el cupo gratis en un ciclo infinito.
