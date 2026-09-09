# AI Content Generation Pipeline — Validation + QA

Caso demostrativo de automatización con **n8n, OpenAI API y Google Sheets**. El flujo recibe briefs de contenido, valida los datos antes de llamar al modelo, genera una salida estructurada, aplica controles de calidad y registra el resultado de cada item.

El objetivo del proyecto no es demostrar "prompting" aislado, sino un patrón más general: **validar → procesar → controlar → registrar**.

> Este repositorio es un proyecto de portfolio. Los resultados y métricas documentados corresponden a una corrida de prueba acotada, no a una afirmación de rendimiento productivo a escala.

---

## Problema

Un flujo de generación automática puede fallar antes, durante o después de la llamada al modelo:

- briefs incompletos o con valores inválidos;
- llamadas innecesarias a la API por entradas defectuosas;
- respuestas que no respetan estructura o longitud;
- pérdida de contexto entre nodos del workflow;
- falta de registro para entender por qué un item falló;
- mezcla entre contenido aprobado y contenido que requiere revisión.

---

## Solución

El workflow separa el proceso en etapas explícitas:

```text
Google Sheets / briefs
        ↓
Validación de entrada
   ├─ inválido → log + estado de error
   └─ válido
        ↓
Loop de procesamiento
        ↓
Generación con OpenAI
        ↓
QA determinístico
   ├─ fail → log + revisión
   └─ pass → output aprobado
        ↓
Actualización de estado + logs
```

La salida aprobada queda separada del registro de eventos y del estado del input.

---

## Arquitectura de datos

El caso utiliza tres hojas:

- **Input** — briefs y estado de procesamiento.
- **Output** — contenido que pasó los controles de QA.
- **Logs** — eventos de validación, QA y ejecución.

Esto permite revisar la historia del proceso sin mezclar resultados finales con errores intermedios.

---

## Validación previa

Antes de llamar al modelo se controlan campos requeridos y valores admitidos. En la demo se validan, entre otros:

- `id`;
- `topic`;
- `audience`;
- `language` (`EN` / `ES`);
- `length` (`short` / `medium`).

Las entradas inválidas se registran y no avanzan a generación.

---

## QA de la salida

Después de la generación se controlan condiciones estructurales como:

- JSON esperado;
- coincidencia de metadata con el input;
- longitud del título;
- rango de palabras del cuerpo;
- estructura de párrafos.

Sólo el contenido que pasa estas reglas llega a la hoja de output.

---

## Resultado de la corrida documentada

Prueba con **11 briefs**:

| Resultado | Cantidad |
|---|---:|
| Briefs recibidos | 11 |
| Errores de validación de input | 3 |
| Items enviados a generación | 8 |
| QA pass | 7 |
| QA fail | 1 |

Sobre los 8 items generados, 7 pasaron las reglas de QA de esa corrida (**88%**). Esta cifra describe únicamente ese set de prueba.

---

## Decisiones y aprendizajes

### Validar antes de consumir una API

Los errores estructurales del input se detectan antes de una llamada externa. El patrón reduce trabajo innecesario y simplifica el diagnóstico.

### Código cuando el workflow visual deja de ser claro

La primera versión distribuía reglas entre varios nodos IF. Al crecer la cantidad de validaciones, se consolidaron en nodos de código para centralizar lógica y facilitar mantenimiento.

### No asumir que un nodo conserva todo el contexto

Durante la implementación, algunos nodos de Google Sheets devolvían sólo los datos escritos y no el payload original. La solución fue referenciar explícitamente el nodo que conserva la información necesaria.

### Procesamiento secuencial deliberado

La demo utiliza batch size 1 para priorizar aislamiento de errores y debugging simple. No se presenta como una estrategia óptima de throughput para cargas grandes.

### Logging como parte del flujo

Los estados de validación y QA no quedan sólo en la ejecución de n8n: se escriben como eventos consultables.

---

## Estructura

```text
.
├── workflows/
│   └── content-generation-workflow.json
├── docs/
├── README.md
└── LICENSE
```

---

## Cómo probarlo

Requisitos:

- una instancia de n8n;
- Google Sheets y sus credenciales;
- credencial de OpenAI API.

Pasos generales:

1. importar `workflows/content-generation-workflow.json`;
2. crear las hojas de input, output y logs según la documentación;
3. configurar credenciales dentro de n8n;
4. cargar briefs de prueba;
5. ejecutar el workflow;
6. revisar estados, output y logs.

Las credenciales deben configurarse en n8n; no se almacenan en el repositorio.

---

## Qué demuestra este proyecto

- Diseño de workflows con etapas y responsabilidades explícitas.
- Validación antes de integrar servicios externos.
- Controles de calidad determinísticos sobre salida de un LLM.
- Registro de errores y resultados para debugging.
- Integración entre n8n, Google Sheets y una API de IA.
- Criterio para decidir cuándo una regla visual conviene migrarla a código.

---

## Alcance y límites

Este caso no pretende demostrar una plataforma de contenido lista para producción a gran escala. No incluye, entre otras cosas, observabilidad centralizada, colas distribuidas, pruebas de carga ni una estrategia completa de retry/idempotencia para múltiples workers.

Su propósito es mostrar de forma revisable el patrón de validación, generación, QA y logging.

---

## Contacto

**Mauro Montenegro**  
[LinkedIn](https://www.linkedin.com/in/mauro-montenegro-data/) · [GitHub](https://github.com/mauro-montenegro-data)
