# MICA — Motor de Similitud Académica

Actor *serverless* desarrollado en Python sobre la plataforma **Apify** para automatizar la extracción, estructuración y evaluación semántica de planes de estudio universitarios. Forma parte del proyecto Capstone **MICA** (*Samsung Innovation Campus*).

---

## Arquitectura y Modos de Operación

El Actor implementa una arquitectura desacoplada en dos modos ejecutables mediante el parámetro `operation`:

1. **`analytics` (Ingesta ETL y Preprocesamiento Curricular):**
   * Descarga programas sintéticos desde directorios con PDFs individuales (`directorio_pdfs`) o desde ordenanzas curriculares consolidadas (`pdf_maestro`).
   * Elimina ruido institucional, segmenta fichas por asignatura, deduplica contenidos mínimos y normaliza el texto base para modelos Transformer (`texto_nlp_base`).
   * Publica el dataset resultante en el *Default Dataset* de la ejecución y persiste el JSON estructurado por carrera en el Key-Value Store `mica-planes-estudio`.

2. **`inference` (Motor Híbrido de Equivalencias en Tiempo Real):**
   * Consume directamente el índice vectorial precalculado con `intfloat/multilingual-e5-base` (768 dimensiones, normalizado L_2) alojado en el Key-Value Store (`utn_planes_maestro.parquet` y `utn_embeddings_e5.npy`).
   * Calcula la matriz de similitud coseno mediante producto punto directo y aplica reglas normativas de carga horaria para emitir un pre-dictamen en menos de 2 segundos sin recargar modelos pesados en memoria.

```text
[PDFs Oficiales] ──(operation: "analytics")──► [Apify KVS: JSONs por carrera]
                                                          │
                                            (Indexación E5-base 768D)
                                                          ▼
[Google Sheets] ◄──► [Make] ──(operation: "inference")──► [Apify KVS: .parquet + .npy]
```

---

## Requisitos en el Key-Value Store (`mica-planes-estudio`)

Para ejecutar el modo `inference`, el Key-Value Store nombrado **`mica-planes-estudio`** debe contener los artefactos generados en el pipeline de indexación:

| Archivo en KVS | Tipo | Descripción |
| :--- | :--- | :--- |
| `utn_planes_maestro.parquet` | Binario (Parquet) | Tabla consolidada de asignaturas (`163 x 20`) con metadatos, carga horaria y `carrera_slug`. |
| `utn_embeddings_e5.npy` | Binario (NumPy) | Matriz `float32` de embeddings normalizados L_2 (`163 x 768`) generados con `intfloat/multilingual-e5-base`. |
| `{UNIVERSIDAD}_{CARRERA_SLUG}.json` | JSON | Archivos individuales por plan de estudio generados por el modo `analytics`. |

---

## Parámetros de Entrada (`Input Schema`)

### Parámetros Generales
* **`operation`** *(string, requerido)*: Selecciona el flujo de ejecución:
  * `"analytics"`: Ejecuta el pipeline ETL sobre documentos PDF.
  * `"inference"`: Ejecuta la comparación curricular entre carrera de origen y destino.

### Parámetros para `operation: "analytics"`
* **`universidad`** *(string)*: Sigla de la institución o facultad regional (ej. `"UTN-FRTDF"`).
* **`carrera_slug`** *(string)*: Identificador canónico sin espacios ni tildes (ej. `"ingenieria-electromecanica"`).
* **`plan`** *(string)*: Año o código de ordenanza del plan de estudios (ej. `"2023"`).
* **`modo_fuente`** *(string)*: `"directorio_pdfs"` (carpeta con archivos `PDF01.pdf`, `PDF02.pdf`, etc.) o `"pdf_maestro"` (único PDF con toda la ordenanza).
* **`url_fuente`** *(string)*: URL base del directorio (terminada en `/`) o enlace directo al PDF maestro.
* **`url_indice`** *(string, opcional)*: URL de una página HTML índice para descubrimiento automático de enlaces `.pdf`.

### Parámetros para `operation: "inference"`
* **`universidad_origen`** *(string)*: Regional de origen del estudiante (ej. `"UTN-FRTDF"`).
* **`carrera_origen_slug`** *(string)*: Slug de la carrera origen (ej. `"ingenieria-electromecanica"`).
* **`universidad_destino`** *(string)*: Regional de destino a evaluar (ej. `"UTN-FRSN"`, `"UTN-FRBB"`).
* **`carrera_destino_slug`** *(string)*: Slug de la carrera destino (`"ingenieria-mecanica"`, `"ingenieria-electrica"`, `"ingenieria-civil"`).
* **`materias_aprobadas`** *(array[integer], opcional)*: Lista con los números de orden (`nro_orden`) aprobados por el estudiante (ej. `[1, 2, 3, 9, 13, 22, 41]`). Si se envía vacío `[]`, evalúa la totalidad de asignaturas del plan origen.
* **`umbral_directa`** *(number, default: `0.92`)*: Similitud coseno mínima para sugerir equivalencia directa.
* **`umbral_parcial`** *(number, default: `0.87`)*: Similitud coseno mínima para derivar a revisión o coloquio.
* **`ratio_horas_min`** *(number, default: `0.75`)*: Relación mínima horas_origen / horas_destino exigida para equivalencia directa.

---

## Ejemplos de Ejecución (JSON Input)

### 1. Ejemplo de Inferencia (Pase Inter-Regional)
```json
{
  "operation": "inference",
  "universidad_origen": "UTN-FRTDF",
  "carrera_origen_slug": "ingenieria-electromecanica",
  "universidad_destino": "UTN-FRSN",
  "carrera_destino_slug": "ingenieria-mecanica",
  "materias_aprobadas": [1, 2, 3, 9, 13, 22, 41],
  "umbral_directa": 0.92,
  "umbral_parcial": 0.87,
  "ratio_horas_min": 0.75
}
```

### 2. Ejemplo de Ingesta ETL (`pdf_maestro`)
```json
{
  "operation": "analytics",
  "universidad": "UTN-FRSN",
  "carrera_slug": "ingenieria-mecanica",
  "plan": "2023",
  "modo_fuente": "pdf_maestro",
  "url_fuente": "https://ejemplo.edu.ar/ordenanza_mecanica_2023.pdf"
}
```

---

## Lógica de Dictamen y Esquema de Salida

El motor aplica un criterio híbrido (*Human-in-the-Loop*) que combina afinidad semántica y carga horaria reglamentaria:

* **`EQUIVALENCIA_DIRECTA`**: similitud 0.92 y ratio_horas 0.75.
* **`REVISION_COLOQUIO`**: similitud 0.92 con ratio_horas < 0.75, o bien 0.87 similitud < 0.92.
* **`NO_EQUIVALENTE`**: similitud < 0.87.

Cada ítem devuelto en el *Dataset* de salida tiene la siguiente estructura:

```json
{
  "universidad_origen": "UTN-FRTDF",
  "carrera_origen": "INGENIERÍA ELECTROMECÁNICA",
  "nro_orden_origen": 22,
  "materia_origen": "Termodinámica Técnica",
  "hs_reloj_origen": 96,
  "universidad_destino": "UTN-FRSN",
  "carrera_destino": "INGENIERÍA MECÁNICA",
  "nro_orden_destino": 17,
  "materia_destino": "Termodinámica",
  "hs_reloj_destino": 120,
  "ratio_horas": 0.8,
  "similitud_coseno": 0.96,
  "porcentaje_afinidad": 96.0,
  "dictamen_sugerido": "EQUIVALENCIA_DIRECTA",
  "observacion": "Contenidos y carga horaria compatibles",
  "segunda_opcion": "Física II (92.9%)"
}
```

---

## Integración con Make y Google Sheets

Para automatizar solicitudes desde una hoja de cálculo:

1. **Google Sheets (`Watch New Rows`):** Monitorea nuevas filas en la pestaña `Solicitudes` (`Estudiante`, `Universidad_Origen`, `Carrera_Origen`, `Universidad_Destino`, `Carrera_Destino`, `Materias_Aprobadas`).
2. **Apify (`Run an Actor`):** Ejecuta este Actor con **Run synchronously = Yes** pasando las celdas mapeadas al JSON de entrada.
3. **Apify (`Get Dataset Items`):** Recupera la lista de dictámenes usando el `defaultDatasetId` generado en el paso anterior.
4. **Google Sheets (`Add a Row`):** Inserta cada asignatura evaluada en la pestaña `Dictamen`.

---

## Estructura del Proyecto y Despliegue Local

```text
.
├── .actor/
│   ├── actor.json
│   └── input_schema.json
├── my_actor/
│   ├── __init__.py
│   └── main.py
├── Dockerfile
├── requirements.txt
└── README.md
```

Para probar localmente con Apify CLI:

```bash
pip install -r requirements.txt
apify run -p
```