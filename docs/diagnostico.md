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
