# 🧬 Guías de Bioinformática — Servidor DEG01

### Biomolecules Laboratory · Facultad de Ciencias de la Salud, UPC

> Esta es una serie de guías estandarizadas para el uso de herramientas bioinformáticas en el servidor DEG01. Cada guía cubre un programa de forma independiente.

| N° | Programa | Estado |
|---|---|---|
| **01** | **eggNOG-mapper v2.1.13** | ✅ Esta guía |
| **02** | **HGTector v2.0b** | ✅ Disponible |
| 03 | *(próximamente)* | — |

---

# Guía 01 — eggNOG-mapper v2.1.13

Cubre el flujo de trabajo completo para la **anotación funcional de genomas y metagenomas**, incluyendo los modos Diamond, HMMER y MMseqs2.

---

## Índice — Guía 01

| Sección | Contenido |
|---|---|
| §1 | Inicio y Activación |
| §2-A | Anotación de Genomas — Modo Diamond |
| §2-B | Anotación de Genomas — Modo HMMER |
| §3-A | Anotación de Metagenomas — Diamond + Prodigal |
| §3-B | Anotación de Metagenomas — MMseqs2 |
| §4 | Parámetros Críticos Explicados |
| §5 | Análisis de Resultados |

---

## 1. Inicio y Activación

Antes de ejecutar cualquier comando, asegúrate de activar el entorno de Conda:

```bash
conda activate eggnog-mapper
```

> **Nota:** La variable de entorno `EGGNOG_DATA_DIR` ya está configurada dentro del ambiente para apuntar a la base de datos centralizada en `/db/eggnog_data`.

---

## 2. Anotación de Genomas y Proteomas

Para este flujo, la entrada es un archivo **FASTA de proteínas** (`.faa`) ya predichas. Aplica a genomas bacterianos completos o conjuntos proteicos de cultivos puros.

### A. Modo Diamond (Recomendado — Alta Velocidad y Precisión)

Este comando combina la velocidad de Diamond con la máxima sensibilidad permitida. Es el ideal para genomas bacterianos completos.

```bash
emapper.py -m diamond \
    -i /ruta/al/archivo/archivo_proteinas.faa \
    -o emapper_genome_bacteria_diamond \
    --itype proteins \
    --sensmode more-sensitive \
    --target_orthologs all \
    --go_evidence all \
    --pfam_realign realign \
    --report_orthologs \
    --dbmem \
    --cpu 30 \
    --tax_scope Bacteria \
    --temp_dir /data/tmp/
```

### B. Modo HMMER (Máxima Sensibilidad)

Utiliza este modo si necesitas identificar funciones en proteínas muy divergentes o si Diamond no reporta hits suficientes en enzimas clave.

```bash
emapper.py -m hmmer \
    -d 2 \
    -i /ruta/al/archivo/archivo_proteinas.faa \
    -o emapper_genome_bacteria_hmmer \
    --itype proteins \
    --report_orthologs \
    --cpu 30
```

---

## 3. Anotación de Metagenomas

Para metagenomas, la estrategia cambia porque usualmente se parte de **contigs (ADN)** y no de proteínas ya predichas. En este caso, eggNOG-mapper realiza la **predicción de genes** antes de la anotación funcional. Este flujo es ideal para muestras de suelo de Caral o fermentos complejos (Tocosh/Masato).

### A. Modo Diamond + Prodigal (Recomendado para Metagenomas)

El parámetro `--genepred prodigal` activa Prodigal en modo `meta`, identificando los marcos de lectura abiertos (ORFs) en los contigs antes de anotarlos. Ideal para comunidades bacterianas mixtas.

```bash
emapper.py -m diamond \
    -i tu_ensamblaje_metagenomico.fasta \
    -o emapper_metagenoma \
    --itype metagenome \
    --genepred prodigal \
    --sensmode more-sensitive \
    --target_orthologs all \
    --report_orthologs \
    --dbmem \
    --cpu 30 \
    --temp_dir /data/tmp/
```

**¿Qué hace cada parámetro nuevo?**

| Parámetro | Función |
|---|---|
| `--itype metagenome` | Indica que la entrada son nucleótidos (contigs/scaffolds) de una comunidad mixta. |
| `--genepred prodigal` | Activa Prodigal en modo `meta` para predecir ORFs antes de la anotación. |
| `--sensmode more-sensitive` | Aumenta la probabilidad de encontrar hits en secuencias fragmentadas o de organismos poco conocidos. |
| `--report_orthologs` | Reporta los ortólogos semilla identificados para cada proteína anotada. |

### B. Modo MMseqs2 (Metagenomas Grandes)

Excelente alternativa cuando se trabaja con metagenomas de gran tamaño (>10 GB de contigs). Ofrece un balance óptimo entre velocidad y sensibilidad, especialmente útil si sospechas la presencia de virus o bacterias poco caracterizada.

```bash
emapper.py -m mmseqs \
    -i contigs.fasta \
    -o emapper_metagenoma_mmseqs \
    --itype metagenome \
    --genepred prodigal \
    --report_orthologs \
    --cpu 30 \
    --temp_dir /data/tmp/
```

> **Nota:** `mmseqs` no requiere `--dbmem` ya que gestiona la memoria de forma interna. Para muestras muy grandes, considera aumentar `--cpu` al máximo disponible en DEG01.

---

## 4. Parámetros Críticos Explicados

| Parámetro | Función |
|---|---|
| `--sensmode more-sensitive` | Aumenta la detección de homólogos lejanos (fósiles ortológicos). |
| `--pfam_realign realign` | Obtiene coordenadas exactas de dominios PFAM mediante `phmmer`. |
| `--dbmem` | Carga la base de datos en RAM (~44 GB). Acelera el proceso drásticamente. |
| `--tax_scope Bacteria` | (TaxID 2) Asegura que solo se anoten proteínas del linaje bacteriano. |
| `--report_orthologs` | Genera un archivo adicional (`.emapper.orthologs`) con los ortólogos semilla de cada hit. Útil para genómica comparativa. |
| `--temp_dir /data/tmp/` | Vital: redirige los miles de archivos temporales (`emappertmp_*`) fuera de la carpeta del proyecto para mantener el orden. |

---

## 5. Análisis de Resultados

El archivo principal es el `.emapper.annotations`. Las columnas clave para nuestras líneas de investigación son:

- **CAZy:** Identificación de enzimas activas en carbohidratos.
- **COG_cat:** Perfil metabólico general de la comunidad microbiana.
- **KEGG_Pathway:** Reconstrucción de rutas bioquímicas para genómica comparativa.

Cuando se usa `--report_orthologs`, se genera adicionalmente el archivo `.emapper.orthologs`, que permite trazar la relación evolutiva de cada gen anotado con su ortólogo semilla en eggNOG.

---

**Guía 01 de N** · Serie Guías de Bioinformática — Servidor DEG01  
**Responsable:** Dr. Frank Guzman Escudero  
**Laboratorio:** Biomolecules Laboratory — Facultad de Ciencias de la Salud, UPC  
**Actualizado:** Abril, 2026

---
---

# Guía 02 — HGTector v2.0b

Estandariza el flujo de trabajo para la **identificación de eventos de Transferencia Horizontal de Genes (HGT)** en el Biomolecules Laboratory. El pipeline utiliza un enfoque basado en la atipicidad de la distribución taxonómica de los mejores hits de homología, complementando los resultados de anotación funcional obtenidos con eggNOG-mapper.

---

## Índice — Guía 02

| Sección | Contenido |
|---|---|
| §1 | Configuración del Entorno |
| §2-A | Flujo de Trabajo — Paso 1: Búsqueda de Homología (Search) |
| §2-B | Flujo de Trabajo — Paso 2: Análisis Estadístico (Analyze) |
| §3 | Interpretación del HGT Score |
| §4 | Archivos de Salida Clave |
---

## 1. Configuración del Entorno

Antes de iniciar el análisis, activa el ambiente de Conda:

```bash
conda activate hgtector
```

> **Nota:** Las variables de entorno `$HGTECTOR_DB` y `$HGTECTOR_TAX` están pre-configuradas en el servidor para apuntar a la base de datos centralizada en `/db/hgtector_db/`.

---

## 2. Flujo de Trabajo (Pipeline)

HGTector opera en dos pasos secuenciales: primero una búsqueda de homología y luego el análisis estadístico sobre esos resultados.

### A. Paso 1: Búsqueda de Homología (`search`)

Utiliza Diamond para comparar las proteínas de entrada contra la base de datos de referencia. Es el paso de mayor carga computacional.

```bash
hgtector search \
    -i /ruta/al/archivo/archivo_proteinas.faa \
    -o resultados_hgt_search \
    -d $HGTECTOR_DB \
    -t $HGTECTOR_TAX \
    -p 30
```

| Parámetro | Función |
|---|---|
| `-i` | Entrada en formato FASTA de aminoácidos (`.faa`). |
| `-d $HGTECTOR_DB` | Ruta a la base de datos Diamond pre-formateada. |
| `-t $HGTECTOR_TAX` | Directorio con los archivos de taxonomía NCBI. |
| `-p 30` | Número de hilos de procesamiento (CPUs). No confundir con `--cpu`. |

### B. Paso 2: Análisis Estadístico (`analyze`)

Procesa los resultados de la búsqueda para calcular las puntuaciones de HGT basadas en el consenso taxonómico.

```bash
hgtector analyze \
    -i resultados_hgt_search/archivo_proteinas.tsv \
    -o resultados_hgt_final \
    -t $HGTECTOR_TAX
```

> **Nota:** El archivo de entrada `-i` corresponde al `.tsv` generado en el Paso 1, ubicado dentro del directorio `resultados_hgt_search/`.

---

## 3. Interpretación del HGT Score

El HGT Score es una medida de **atipicidad taxonómica** que oscila entre 0 y 1. Un valor alto indica que el gen tiene mayor homología con grupos taxonómicos distantes que con el propio organismo.

| Score | Categoría | Interpretación |
|---|---|---|
| 0.0 – 0.5 | Típico | Genes heredados verticalmente o transferencias muy antiguas ya mimetizadas con el genoma (amelioración). |
| 0.5 – 0.7 | Moderado | Candidato de interés. El gen muestra más homología con grupos externos. Común en genes metabólicos adaptativos. |
| 0.8 – 1.0 | Fuerte | Transferencias muy recientes o genes bajo presión selectiva extrema. Alta identidad con el donante lejano. |

---

## 4. Archivos de Salida Clave

El directorio de salida del análisis (`resultados_hgt_final/`) contiene los siguientes archivos críticos:

- **`hgts/*.txt`:** La "lista de oro" con los IDs de los genes predichos como HGT, su Score y el TaxID del donante potencial.
- **`scores.tsv`:** Tabla completa con las métricas de todos los genes analizados.
- **`scatter.png`:** Gráfico de dispersión que visualiza la relación entre hits cercanos (*close*) y distales (*distal*). Los puntos alejados de la diagonal son candidatos a HGT.
- **`*.kde.png` / `*.hist.png`:** Gráficos de densidad que ayudan a validar estadísticamente si los candidatos son realmente valores atípicos (*outliers*).

---

**Guía 02 de N** · Serie Guías de Bioinformática — Servidor DEG01  
**Responsable:** Dr. Frank Guzman Escudero  
**Laboratorio:** Biomolecules Laboratory — Facultad de Ciencias de la Salud, UPC  
**Actualizado:** Abril, 2026

