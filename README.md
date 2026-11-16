# RetryLint

RetryLint 0.1.0 es una herramienta de línea de comandos, escrita en Kotlin, que detecta riesgos de reintentos entre servicios que no se ven al revisar la configuración de un solo servicio. Combina una topología explícita de llamadas síncronas (`retrylint.yml`) con un subconjunto soportado de la configuración Resilience4j (Retry y TimeLimiter) de cada servicio. El análisis es estático y determinista: no ejecuta los servicios.

| Regla | Riesgo que reporta |
|---|---|
| RL001 | Amplificación multiplicativa de reintentos a lo largo de una ruta de llamadas |
| RL002 | Un timeout entrante menor que la ventana de reintentos de la llamada siguiente |
| RL003 | Reintentos que pueden repetir una operación no idempotente o de idempotencia desconocida |

El alcance es deliberadamente acotado: analiza grafos acíclicos (DAG) estáticos y síncronos. No modela tráfico real, probabilidades, colas, circuit breakers ni todo el comportamiento de Resilience4j. El detalle está en el [contrato del núcleo v0.1.0](docs/CORE_CONTRACT_V0.1.md) y en las [limitaciones](docs/LIMITATIONS.md).

> El README está en español. El resto de la documentación (`docs/`, `release/`, `testbed/README.md`) está en inglés.

## Requisitos

- Java 21 (`JAVA_HOME` debe apuntar a una instalación de Java 21).
- Git, usado por las evaluaciones para comprobar que el núcleo del analizador no cambió.
- El wrapper de Gradle incluido en el repositorio; no hace falta instalar Gradle.
- Solo para el experimento de la semana 8: PowerShell y Docker Desktop con Docker Compose.

## Estructura del repositorio

```text
RetryLint/
├── README.md                    Este documento
├── build.gradle.kts             Configuración común de Gradle (grupo y versión 0.1.0)
├── settings.gradle.kts          Módulos: retrylint-cli y testbed-service
├── gradlew, gradlew.bat         Wrapper de Gradle (Linux/macOS y Windows)
├── gradle/                      Wrapper y catálogo de versiones (libs.versions.toml)
├── .github/workflows/ci.yml     Integración continua: tests, build y cobertura con Java 21
│
├── retrylint-cli/               El analizador (producto principal)
│   ├── build.gradle.kts         Dependencias, JaCoCo y tareas de evaluación
│   └── src/
│       ├── main/kotlin/retrylint/
│       │   ├── cli/             Comandos validate, analyze y version; códigos de salida
│       │   ├── input/           Lectura y límites del manifiesto retrylint.yml
│       │   ├── graph/           Validación de la topología (referencias, ciclos)
│       │   ├── config/          Resolución de Resilience4j (herencia baseConfig, duraciones)
│       │   ├── rules/           Reglas RL001, RL002 y RL003
│       │   ├── analysis/        Orquestación del análisis
│       │   ├── model/           Hallazgos y configuración resuelta
│       │   └── report/          Salida en texto y JSON
│       └── test/
│           ├── kotlin/retrylint/         Tests unitarios, de contrato y de evaluación
│           └── resources/fixtures/       30 proyectos de prueba y expected-results.json
│
├── testbed-service/             Servicio Spring Boot usado como sistema real de cuatro servicios
│   ├── Dockerfile
│   └── src/                     Servicio, contadores de invocaciones y tests de contrato con Spring
│
├── testbed/                     Experimento en tiempo de ejecución (semana 8)
│   ├── compose.yml              checkout -> orders -> payments -> bank
│   ├── configs/scenarios/       Escenarios 1, 3 y 9 (el 27 usa el fixture complete-example)
│   ├── scripts/                 start, stop, reset, run-scenario, verify-week8 (PowerShell)
│   └── README.md
│
├── evaluation/                  Evidencia registrada
│   ├── week8-results.json       Resultados del testbed
│   ├── week8-result-template.json
│   ├── week9/                   Benchmark de mutación: 10 semillas, 30 mutantes y resultados
│   ├── week10/                  Escalabilidad: resultados y resumen
│   ├── coverage/retrylint-cli/  Reporte JaCoCo archivado (HTML, XML, CSV)
│   └── test-results/            Resultados JUnit XML de ambos módulos
│
├── docs/                        Arquitectura, algoritmos, evaluación, limitaciones,
│   │                            reproducibilidad, índice de evidencia y guía de defensa
│   ├── figures/                 Figuras SVG generadas desde la evidencia
│   └── scripts/generate-figures.ps1
│
├── release/                     Registro de la entrega
│   ├── RELEASE_NOTES_v0.1.1.md
│   ├── FINAL_VERIFICATION.md
│   ├── OFFLINE_BACKUP.md
│   ├── SHA256SUMS.txt           Suma SHA-256 de la distribución del CLI
│   └── demo/                    Ejemplo completo y salidas esperadas para la demostración
│
└── scripts/finalize-delivery.ps1   Ejecuta todas las verificaciones de la entrega (Windows)
```

Las carpetas `build/`, `.gradle/` y `release/artifacts/` se generan localmente y no se versionan.

## Compilar y probar

Todos los comandos se ejecutan desde la raíz del repositorio. En Windows use `.\gradlew.bat`; en Linux o macOS use `./gradlew`.

```bash
./gradlew clean test     # ejecuta los 116 tests
./gradlew build          # tests, cobertura mínima de ramas (80 %) y distribución
```

`build` falla si la cobertura de ramas de los paquetes `config`, `graph` y `rules` baja del 80 %. El reporte queda en `retrylint-cli/build/reports/jacoco/test/html/index.html`.

## Ejecutar el analizador

### Desde Gradle

Validar el manifiesto y las rutas de configuración, sin ejecutar las reglas:

```bash
./gradlew -q :retrylint-cli:run --args="validate retrylint-cli/src/test/resources/fixtures/complete-example/retrylint.yml"
```

Salida esperada: `VALID: checkout-demo (4 services, 4 operations, 3 calls)`.

Analizar el ejemplo completo:

```bash
./gradlew -q :retrylint-cli:run --args="analyze retrylint-cli/src/test/resources/fixtures/complete-example/retrylint.yml --format text --fail-on never"
```

Las tres capas de reintento del ejemplo permiten tres intentos cada una, así que RL001 reporta `3 x 3 x 3 = 27`. Es la amplificación posible según la configuración, no una afirmación de que ocurrieron 27 operaciones duplicadas.

### Desde la distribución empaquetada

```bash
./gradlew :retrylint-cli:distZip
```

Genera `retrylint-cli/build/distributions/retrylint-cli-0.1.0.zip`. Tras descomprimirlo:

```bash
bin/retrylint-cli version                          # Windows: bin\retrylint-cli.bat
bin/retrylint-cli validate ruta/a/retrylint.yml
bin/retrylint-cli analyze ruta/a/retrylint.yml --format json --output reporte.json
```

### Comandos y opciones

| Comando | Qué hace |
|---|---|
| `validate <retrylint.yml>` | Valida el manifiesto, la topología y la configuración de cada servicio |
| `analyze <retrylint.yml>` | Ejecuta RL001, RL002 y RL003 y emite el reporte |
| `version` | Imprime `RetryLint 0.1.0` |

| Opción de `analyze` | Efecto |
|---|---|
| `--format text\|json` | Formato del reporte; por defecto `text` |
| `--output <archivo>` | Escribe el reporte en un archivo UTF-8 en lugar de la salida estándar |
| `--fail-on warning\|error\|never` | Umbral que produce código de salida 1; si se omite, se usa `analysis.failOn` del manifiesto |
| `--verbose` | Añade diagnósticos deterministas al texto |
| `--no-color` | Se acepta por compatibilidad; el reporte nunca usa colores |

### Códigos de salida

| Código | Significado |
|---|---|
| 0 | El análisis terminó y ningún hallazgo alcanzó el umbral |
| 1 | El análisis terminó y algún hallazgo alcanzó el umbral |
| 2 | Entrada, topología, configuración, opción o destino de salida inválidos |
| 3 | Fallo interno inesperado |

## Formato de entrada

El manifiesto `retrylint.yml` declara servicios, operaciones, llamadas y parámetros del análisis:

```yaml
version: 1
name: checkout-demo

services:
  - id: checkout
    config: services/checkout/application.yml
  - id: orders
    config: services/orders/application.yml

operations:
  - id: checkout.place-order
    service: checkout
    idempotency: idempotent        # idempotent | non-idempotent | unknown
  - id: orders.create-order
    service: orders
    idempotency: non-idempotent

calls:
  - id: checkout-to-orders
    from: checkout.place-order
    to: orders.create-order
    retry: orders-client           # instancia de Retry en el servicio que llama
    timeLimiter: orders-client     # instancia de TimeLimiter en el servicio que llama

analysis:
  roots:
    - checkout.place-order
  amplificationWarning: 9
  amplificationError: 27
  failOn: error                    # warning | error | never
```

Reglas de la entrada:

- Cada servicio referencia **un único** archivo de configuración YAML, con ruta relativa y dentro del directorio del manifiesto.
- No se aceptan rutas absolutas, rutas que salgan del directorio ni enlaces simbólicos.
- El grafo de llamadas debe ser acíclico; un ciclo se rechaza con código 2.
- El manifiesto y cada archivo de servicio admiten hasta 2 MiB; el resto de límites está en el contrato.

De cada archivo de servicio solo se interpretan estas claves de Resilience4j, en camelCase o kebab-case:

- `resilience4j.retry.configs` e `instances`: `maxAttempts`, `waitDuration`, `baseConfig`.
- `resilience4j.timelimiter.configs` e `instances`: `timeoutDuration`, `baseConfig`.

Los valores por omisión son los de Resilience4j: 3 intentos, 500 ms de espera y 1 s de timeout. La herencia por `baseConfig` admite hasta 8 niveles. No se soportan perfiles de Spring, placeholders, archivos `.properties` ni configuración programática. El backoff exponencial o aleatorio no se convierte en cero: se reporta como vacío de completitud en RL002.

## Evaluación y evidencia

| Evaluación | Resultado registrado | Cómo reproducirla |
|---|---|---|
| Semana 8: validación en ejecución | La predicción estática de 27x coincidió con 27 llamadas al banco en las 10 repeticiones registradas | `powershell -ExecutionPolicy Bypass -File testbed/scripts/verify-week8.ps1` |
| Semana 9: benchmark de mutación | 40 casos; TP=30, FP=0, FN=0 sobre el corpus controlado y revisado | `./gradlew :retrylint-cli:week9Benchmark` |
| Semana 10: escalabilidad | Los DAG generados de 10, 100, 500 y 1000 operaciones terminaron correctamente | `./gradlew :retrylint-cli:week10Scalability` |
| Tests y cobertura | 116 tests sin fallos; 82,5 % de cobertura de ramas en `config`, `graph` y `rules` | `./gradlew build :retrylint-cli:archiveCoverage` |

### Cómo re-ejecutar las evaluaciones

Las tres evaluaciones necesitan el **repositorio clonado con git**, no el `.zip` de la entrega: antes de correr comparan `retrylint-cli/src/main` contra el tag `v0.1.0` (excluyendo el paquete `cli`) y abortan si no pueden hacerlo o si hay diferencias.

1. Clone el repositorio y confirme que el tag existe:

   ```bash
   git clone https://github.com/camilo-zuluaga/retry-lint.git RetryLint
   cd RetryLint
   git fetch --tags
   git rev-parse "v0.1.0^{commit}"    # debe imprimir 85ffebc096eb1b7d007270638ccffcecb62605b3
   ```

2. Compruebe que el núcleo no cambió; el comando no debe imprimir nada:

   ```bash
   git diff v0.1.0 -- retrylint-cli/src/main ':(exclude)retrylint-cli/src/main/kotlin/retrylint/cli'
   ```

3. Ejecute la evaluación que necesite:

   | Evaluación | Comando | Requiere | Archivos que reescribe |
   |---|---|---|---|
   | Semana 8 | `powershell -ExecutionPolicy Bypass -File testbed/scripts/verify-week8.ps1` | Windows, PowerShell, Docker Desktop en ejecución y los puertos 18080 a 18083 libres | `evaluation/week8-results.json` |
   | Semana 9 | `./gradlew :retrylint-cli:week9Benchmark` | Java 21 | `evaluation/week9/results/` |
   | Semana 10 | `./gradlew :retrylint-cli:week10Scalability` | Java 21 | `evaluation/week10/results/` |

4. Revise los cambios con `git status` y `git diff evaluation/` antes de versionarlos. Si solo quería comprobar que corren, descártelos con `git checkout -- evaluation/`.

Notas:

- Los archivos de resultados son evidencia: cada ejecución los sobrescribe. Los tiempos de la semana 10 varían entre ejecuciones y entre máquinas.
- La semana 8 levanta los cuatro servicios con Docker Compose, los detiene al terminar (use `-KeepRunning` para dejarlos arriba) y falla si algún conteo no coincide con el esperado.
- El script de la semana 8 también registra los escenarios 1, 3, 9 y el caso en que el banco responde en la tercera llamada. El `week8-results.json` versionado es anterior a ese cambio y solo contiene las 10 corridas del escenario 27; hay que ejecutar el script para regenerarlo.
- Para regenerar las figuras a partir de la evidencia: `powershell -ExecutionPolicy Bypass -File docs/scripts/generate-figures.ps1`.
- `scripts/finalize-delivery.ps1` ejecuta en Windows toda la verificación de la entrega: build, cobertura, resultados de tests, distribución con su SHA-256 y el testbed.

El alcance de cada resultado está en [Evaluation](docs/EVALUATION.md) y el mapa de cada afirmación a su archivo, en el [Evidence index](docs/EVIDENCE_INDEX.md).

## Documentación

- [Arquitectura](docs/ARCHITECTURE.md)
- [Algoritmos y configuración soportada](docs/ALGORITHMS.md)
- [Contrato del núcleo v0.1.0](docs/CORE_CONTRACT_V0.1.md)
- [Evaluación y resultados](docs/EVALUATION.md)
- [Limitaciones y amenazas a la validez](docs/LIMITATIONS.md)
- [Guía de reproducibilidad](docs/REPRODUCIBILITY.md)
- [Índice de evidencia](docs/EVIDENCE_INDEX.md)
- [Guía de demostración y defensa](docs/DEFENSE_GUIDE.md)
- [Testbed de la semana 8](testbed/README.md)

## Versiones y entrega

- **Núcleo del analizador `v0.1.0`:** la implementación evaluada en las semanas 8 a 10. El comando `version` reporta `RetryLint 0.1.0`.
- **Entrega del proyecto `v0.1.1`:** documentación, evaluación, empaquetado y registro de la entrega. No cambia la semántica del análisis.
- **Cambios posteriores a `v0.1.1`:** `validate` devuelve el código 3 ante fallos internos, como exige el contrato; se añadieron el test de contrato contra Spring Boot, cinco fixtures y la evidencia archivada de cobertura y tests. Los paquetes de análisis siguen idénticos a `v0.1.0`.

Vea las [notas de la versión v0.1.1](release/RELEASE_NOTES_v0.1.1.md), el [registro de verificación final](release/FINAL_VERIFICATION.md) y la [guía de respaldo sin conexión](release/OFFLINE_BACKUP.md).
