# Diagnóstico del Pipeline de Integración Continua

## 1.1 Diagnóstico de Defectos

**Defecto 1: Instalación de dependencias sin archivo de bloqueo (lockfile)**
* **Qué está mal:** Se ejecuta `pip install -r requirements.txt` resolviendo versiones de forma dinámica en lugar de forzar la instalación estricta desde un archivo de bloqueo (como `requirements.lock`).
* **Archivo y líneas:** `.github/workflows/pipeline.yml` (Líneas 20–22 y 49–51).
* **Consecuencia:** Se pierde la **garantía de reproducibilidad e inmutabilidad** del entorno de construcción; actualizaciones no controladas en paquetes externos pueden alterar el comportamiento del sistema sin previo aviso.

**Defecto 2: Ausencia de caché de dependencias**
* **Qué está mal:** La acción `actions/setup-python@v5` omite la propiedad de almacenamiento en caché (`cache: 'pip'`).
* **Archivo y líneas:** `.github/workflows/pipeline.yml` (Líneas 14–17 y 43–46).
* **Consecuencia:** Se pierde la **garantía de eficiencia y retroalimentación rápida** en CI; el runner redescarga e instala la totalidad de paquetes desde internet en cada job y ejecución.

**Defecto 3: Ausencia de detención por Quality Gate**
* **Qué está mal:** El paso de SonarQube analiza el código pero no ejecuta una acción que espere el resultado y detenga el pipeline si la evaluación falla.
* **Archivo y líneas:** `.github/workflows/pipeline.yml` (Líneas 27–35).
* **Consecuencia:** Se pierde la **garantía de inspección de calidad e integridad**; el pipeline finaliza con éxito (*en verde*) aunque el código contenga vulnerabilidades, deudas técnicas o cobertura insuficiente.

**Defecto 4: Publicación genérica sin restricciones ni dependencias de validación**
* **Qué está mal:** El artefacto no utiliza el prefijo `despachos-`, el job `publicar` no exige la aprobación previa (`needs: validar`) y carece de restricción por la rama principal (`if: github.ref == 'refs/heads/main'`).
* **Archivo y líneas:** `.github/workflows/pipeline.yml` (Líneas 41 y 58).
* **Consecuencia:** Se pierde la **garantía de gobierno, trazabilidad y control de entregables**; se distribuyen artefactos no verificados, creados desde ramas de desarrollo secundarias y con un nombrado que viola el estándar del sistema.

---

## 1.2 Defecto Explicativo de la Duración

* **Defecto responsable:** **Defecto 2 (Ausencia de caché de dependencias)**.
* **Sustento:** La redescarga limpia e instalación de paquetes Python (ejecutada por duplicado en los dos jobs aislados) es la causa directa del tiempo consumido. En las ejecuciones registradas en `docs/linea-base.md`, los tiempos fueron **1m 09s**, **59s** y **1m 15s** (con un promedio de **1m 07s / 67.6 segundos**). La mayor parte de esta duración corresponde a la latencia de red y descarga repetida de binarios en cada ejecución.

---

## 1.3 Vínculo con el Caso Transversal

* **Defecto relacionado:** **Defecto 2 (Ausencia de caché de dependencias) / Defecto 4 (Publicación sin validación)**.
* **Sustento VSM:** Este defecto ataca directamente el desperdicio (*Lead Time* excesivo / retrabajo).

---

## 1.4 Métrica DORA

* **Métricas alcanzables sin despliegue a producción:** *Lead Time for Changes (LTC)* y *Change Failure Rate (CFR)*.
* **Métrica elegida:** **Lead Time for Changes (LTC)**.
* **Justificación:** Al optimizar la velocidad del pipeline mediante la caché de dependencias y evitar reconstrucciones redundantes, se reduce drásticamente el tiempo transcurrido desde que el desarrollador hace `commit` hasta que la validación confirma que el artefacto es apto para despliegue.

---

## 1.5 Declaración del Proxy (Métrica Concreta)

* **Proxy declarado:** **Duración total de ejecución del workflow `pipeline.yml` en GitHub Actions (medido en segundos)**.
* **Valor actual (Línea Base):** **67.6 segundos en promedio** (rango entre 59s y 75s).
* **Meta esperada:** Reducir la duración promedio del pipeline por debajo de los **40 segundos** tras implementar la caché, manteniendo las validaciones de calidad activas.

## 4.1 Medición posterior

Después de la intervención, la ejecución del pull request #2 tardó **99 segundos**
(1m39s). La línea base era de **67.6 segundos** en promedio, calculada a partir
de las tres ejecuciones registradas en `docs/linea-base.md`.

El tiempo aumentó en **31.4 segundos**, equivalente a aproximadamente **46.4%**
respecto de la línea base. La ejecución posterior terminó con fallo en el paso
**Analisis de calidad**, porque el Quality Gate de SonarQube detectó cobertura
insuficiente sobre código nuevo. Esto confirma que el pipeline sí detiene la
ejecución cuando el análisis de calidad falla.

Ejecución posterior:
https://github.com/DivanoPerezNina/INF384-lab2-20221481/actions/runs/34539888011

## 4.2 Justificación de la versión

La versión declarada es **1.3.0**, registrada tanto en `VERSION` como en
`pyproject.toml`. El valor fue establecido por el commit `ae2b389` (`parte 2
correccion 1`). El historial que sustenta el estado entregado incluye:

* `a31a0bf`: diagnóstico y trabajo de la Parte 1.
* `77d2d36` y `ae2b389`: implementación y correcciones de la Parte 2.
* `d1c1665`: incorporación de `prioridad_pedido` para la Parte 3.
* `884e9a7`: ejecución del pipeline también en pull requests dirigidos a `main`.

## 4.3 Lo que no se resolvió

El pipeline todavía no publica artefactos cuando se ejecuta sobre un pull request:
el job `publicar` está condicionado a la rama `main`. Para resolver esta
limitación habría que definir una política de publicación para PR, por ejemplo
generar artefactos temporales para revisión sin publicarlos como una versión
oficial, y agregar controles para evitar que esos artefactos se confundan con
los de una entrega aprobada.

## 4.4 Declaración de uso de IA generativa

Se utilizaron herramientas de IA generativa como apoyo para completar las Partes
1, 2 y 3 de este laboratorio. En particular, se utilizó GitHub Copilot para
entender el código existente, proponer y revisar cambios en el pipeline,
interpretar resultados de pruebas y cobertura, y redactar parte de la
documentación. La decisión final sobre los cambios, la ejecución de las
pruebas, la revisión del historial y la verificación del pull request fueron
realizadas por el estudiante.

Los prompts utilizados incluyeron solicitudes equivalentes a:

* "Analiza el pipeline de GitHub Actions, identifica sus defectos y propone
	correcciones justificadas."
* "Implementa la funcionalidad de la Parte 3 manteniendo el estilo del
	proyecto y verifica la cobertura de pruebas."
* "Ayúdame a documentar la línea base, la medición posterior, la versión y las
	limitaciones del pipeline."
