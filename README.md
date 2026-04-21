# 🧬 Guías de Bioinformática — Servidor DEG01

### Biomolecules Laboratory · Facultad de Ciencias de la Salud, UPC

> Esta es una serie de guías estandarizadas para el uso de herramientas bioinformáticas en el servidor DEG01. Cada guía cubre un programa de forma independiente.

| N° | Programa | Estado |
|---|---|---|
| **01** | **eggNOG-mapper v2.1.13** | ✅ Esta guía |
| 02 | *(próximamente)* | — |
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


