# PSIB2026Q2 · TPI · Grupo 12

**Detección de leucemia linfoblástica aguda (LLA) en imágenes de frotis de sangre periférica**

16.63 Procesamiento de Señales e Imágenes Biomédicas · ITBA · 2026 Q2

Integrantes: Victoria Lautersztein (62437), María Reggio (62303), Rosario Ballester (62648)

---

## Descripción

Algoritmo **clásico e interpretable** (sin redes neuronales ni modelos de aprendizaje automático) que:

1. corrige la iluminación de cada imagen,
2. segmenta los núcleos de los leucocitos,
3. extrae descriptores de forma, color y textura de esos núcleos, y
4. clasifica la imagen como **patológica (LLA)** o **no patológica** mediante 5 reglas de umbral y votación mayoritaria.

**Resultado final (conjunto de prueba independiente, 977 imágenes):**

| Sensibilidad | Especificidad | Exactitud | Precisión positiva |
|:---:|:---:|:---:|:---:|
| 0,937 | 0,781 | 0,913 | 0,959 |

Métricas por imagen, no por paciente; no constituyen una validación clínica (ver [Limitaciones](#limitaciones)).

## Pipeline

```
 imagen RGB (224×224)
        │
        ▼
 ┌──────────────────────────┐
 │ Módulo 1 · Preprocesam.  │  corrección de iluminación en V (cierre morfológico)
 └──────────────────────────┘
        │  preprocesar()
        ▼
 ┌──────────────────────────┐
 │ Módulo 2 · Segmentación  │  k-means (k = 3) en (a*, b*) sobre toda la imagen
 └──────────────────────────┘
        │  máscara de núcleos
        ▼
 ┌──────────────────────────┐
 │ Módulo 3 · Descriptores  │  28 descriptores: forma, color, textura (GLCM/LBP)
 └──────────────────────────┘
        │  modulo3_descriptores.csv
        ▼
 ┌──────────────────────────┐
 │ Módulo 4 · Clasificación │  5 reglas de umbral (ROC/Youden) + votación mayoritaria
 └──────────────────────────┘
        │  modulo4_reglas_finales.json
        ▼
  patológica / no patológica
```

## Datos

*Acute Lymphoblastic Leukemia (ALL) Image Dataset* (Aria et al., 2021, Kaggle, doi:10.34740/kaggle/dsv/2175623): **3.256 imágenes JPG de 224 × 224 px** de 89 pacientes, con su segmentación de referencia.

| Subtipo | Clase del proyecto | Imágenes |
|---|---|---:|
| Benign | no patológica | 504 (15,5 %) |
| Early Pre-B | patológica | 985 |
| Pre-B | patológica | 963 |
| Pro-B | patológica | 804 |

Los datos **no se incluyen en el repositorio**. Los notebooks 01–03 los descargan automáticamente con `kagglehub` (si pide credenciales, configurar el `kaggle.json` de una cuenta de Kaggle).

---

## Módulos

### Módulo 1 · Contextualización y preprocesamiento
`notebooks/01_contextualizacion_preprocesamiento.ipynb`

**Qué hace**
- **Caracteriza el dataset:** resolución y formato, ejemplos por subtipo, histogramas RGB/HSV por clase, uniformidad de la iluminación (campo de iluminación del canal V: coeficiente de variación y cociente centro/borde) y variabilidad de la tinción (medias de H, S, a\*, b\* por subtipo).
- **Compara tres correcciones de iluminación**, todas aplicadas **sólo sobre V** (H y S quedan intactos para no alterar el color de la tinción):

  | Método | Qué hace |
  |---|---|
  | `gauss` | estima el campo con un gaussiano de escala amplia y normaliza V / campo |
  | `cierre` | estima el campo con un cierre morfológico (borra las células, deja el fondo) y normaliza V / campo |
  | `clahe` | ecualización adaptativa de histograma con límite de contraste |

- Las evalúa con tres métricas (dispersión del fondo, contraste núcleo–fondo en gris y detalle nuclear) sobre una muestra estratificada de 300 imágenes y sobre las imágenes con iluminación menos uniforme, y hace un **análisis de sensibilidad** de los parámetros de cada método.

**Hallazgos clave**
- Desbalance marcado: 15,5 % benignas vs. 84,5 % patológicas.
- Muchas imágenes tienen una marca de agua o barra de escala en la esquina inferior derecha.
- **El núcleo no se distingue en V** (V = máx(R, G, B), y el violeta del núcleo tiene el azul alto): se separa por **saturación y tono**. Esto orienta la segmentación del Módulo 2.
- Pro-B está teñido notablemente más pálido que los demás subtipos.

**Decisión:** corrección por **cierre morfológico** (`preprocesar(rgb, 'cierre')`): reduce la dispersión del fondo sin perder contraste ni detalle nuclear.

**Salidas:** `modulo1_descriptores_globales.csv`, `modulo1_barrido_parametros.csv`.

---

### Módulo 2 · Segmentación de núcleos
`notebooks/02_segmentacion.ipynb`

**Qué hace**
- **Secciones 1–4 (segmentación dentro de la referencia del dataset):** usa la segmentación de referencia como región de interés y compara dos métodos:
  1. **Otsu sobre S** (saturación) + apertura morfológica + descarte de componentes chicas.
  2. ***k*-means (k = 2) en el plano (a\*, b\*)** de CIE Lab; el cluster de mayor croma es el núcleo.

  Mide concordancia entre ambos (Dice), área relativa del núcleo y tiempo de cómputo. Concuerdan muy bien en Benign, Early y Pre (Dice ≈ 0,95–0,98), pero **mal en Pro-B** (Dice 0,63 ± 0,43).

- **Sección 5 (segmentación sin la referencia):** la referencia del dataset **falla en Pro-B** (vacía en el 18 % de las imágenes) y además no va a estar disponible en casos nuevos. Se prueban dos métodos sobre la imagen completa, con relleno de huecos y descarte de la esquina con marca de agua:
  1. **Otsu sobre S global**: sobre-segmenta, marca también glóbulos rojos.
  2. ***k*-means global (k = 3)** en (a\*, b\*): núcleo / glóbulos rojos + citoplasma / fondo.

  | | Benign | Early | Pre | Pro |
  |---|---|---|---|---|
  | Dice vs. con referencia — Otsu S global | 0,69 | 0,84 | 0,74 | 0,22 |
  | Dice vs. con referencia — *k*-means global | **0,72** | **0,91** | **0,82** | 0,29 |
  | % imágenes sin núcleo — con referencia | 0 | 0 | 0 | 30 % |
  | % imágenes sin núcleo — *k*-means global | 0 | 0 | 0 | **0 %** |

**Decisión:** ***k*-means global (k = 3) en (a\*, b\*)** (`segmentar(rgb, metodo='kmeans_global')`). Separa los núcleos de los eritrocitos, no depende de la referencia y recupera los núcleos de Pro-B.

**Salidas:** `modulo2_comparacion_segmentacion.csv`.

---

### Módulo 3 · Extracción de descriptores
`notebooks/03_descriptores.ipynb`

**Qué hace**
1. **Diagnóstico de la referencia del dataset:** en Pro-B ocupa una mediana de 1,1 % de la imagen (vs. 8–21 % en el resto) y está vacía en 146 imágenes. Justifica la segmentación sin referencia del Módulo 2.
2. **Intento de segmentar la célula completa** (núcleo + citoplasma) para medir la relación núcleo/citoplasma, haciendo crecer cada núcleo sobre los píxeles teñidos dentro de un anillo proporcional a su radio. **No cumple el criterio de aceptación fijado de antemano** (Dice ≥ 0,70 y Spearman ≥ 0,6): la referencia del dataset marca prácticamente sólo el núcleo, así que no hay contra qué validar el citoplasma. **Se descarta** la relación núcleo/citoplasma y se usan `n_nucleos` y `frac_nuclear` en su lugar.
3. **Calcula 28 descriptores por imagen** (forma: mediana por núcleo, excluyendo los que tocan el borde):

   | Grupo | Descriptores |
   |---|---|
   | **Forma** (7) | `area`, `perimetro`, `circularidad`, `excentricidad`, `solidez`, `n_nucleos`, `frac_nuclear` |
   | **Color** (9) | `H_media`, `H_desvio` (estadística circular), `S_media`, `S_desvio`, `a_media`, `a_desvio`, `b_media`, `b_desvio`, `S_rel` (saturación del núcleo / saturación del resto teñido de la misma imagen) |
   | **Textura GLCM** (8) | `glcm_contraste`, `glcm_homogeneidad`, `glcm_energia`, `glcm_correlacion`, cada uno a distancia *d* = 1 y *d* = 3 (32 niveles, 4 direcciones promediadas, gris normalizado por los percentiles del propio núcleo) |
   | **Textura LBP** (4) | `lbp_entropia`, `lbp_plano`, `lbp_borde`, `lbp_no_uniforme` |

4. **Experimento de compresión JPEG:** recomprime una muestra estratificada (25 imágenes por subtipo) con calidades 95, 75, 50, 25 y 10 y pasa cada versión por el pipeline completo. Mide tasa de compresión, PSNR, SSIM, Dice de la segmentación y cambio relativo de los descriptores.
5. **Control de integridad** de la tabla final (una fila por imagen, sin duplicados, sin faltantes ni infinitos) antes de exportarla.

La semilla de *k*-means está fija (42), así que el notebook es reproducible.

**Hallazgos clave**
- **Área:** núcleos más grandes en las patológicas (~430 vs. ~350 px²).
- **Circularidad y solidez:** al revés de lo esperado (menores en las benignas), probablemente por la presencia de otros leucocitos de núcleo lobulado.
- **Las medias de color reflejan el lote de tinción (Pro-B), no la biología**, así que no se proponen para clasificar. Los **desvíos de color** y `S_rel` sí diferencian las clases: las benignas tienen núcleos más heterogéneos, coherente con cromatina condensada en grumos.
- **GLCM:** las benignas tienen más contraste y mucha menos correlación entre vecinos (d = 3: ≈ 0,18 vs. ≈ 0,35).
- **LBP (R = 1) prácticamente no distingue** entre clases.
- **JPEG:** forma y color se mantienen estables hasta calidad ~50–75 (cambios < 5 %); la textura GLCM es lo más afectado. Recomendación: no recomprimir (o calidad ≥ 95) si se usan descriptores de textura.

**Salida:** `modulo3_descriptores.csv` (3.256 filas × 28 descriptores + `archivo`, `subtipo`, `clase`). **Es la entrada del Módulo 4.**

---

### Módulo 4 · Clasificación y evaluación
`notebooks/04_clasificacion_evaluacion.ipynb`

**Qué hace**
1. Carga `modulo3_descriptores.csv` (en Colab lo pide para subir si no está) y lo divide **70 % ajuste (2.279) / 30 % prueba (977)**, estratificado por clase. **La prueba no se usa para nada hasta la evaluación final.**
2. **Análisis estadístico en ajuste:** prueba de Mann–Whitney bilateral para cada descriptor, corrección por FDR de Benjamini–Hochberg (α = 0,05) y AUC univariado. 22 de los 28 descriptores resultan significativos.
3. **Selección de descriptores:** ordena por AUC y descarta los redundantes (|Spearman| por encima de un límite).
4. **Reglas de umbral:** para cada descriptor elegido, una regla *patológica si valor ≥ umbral* (o ≤, según la dirección), con umbral por **índice de Youden** o por **sensibilidad objetivo** (0,90 / 0,95). La imagen es patológica si la mayoría de las reglas lo indica.
5. **Barrido con validación cruzada (3 particiones, sólo en ajuste)** sobre número de reglas (3, 5, 7), límite de redundancia (0,80, 0,90) y criterio de umbral. Se elige, entre las configuraciones con sensibilidad CV ≥ 0,90, la de mayor especificidad.
6. **Reajuste final** de las reglas con todo el conjunto de ajuste y **evaluación única sobre prueba**.

**Clasificador final** (5 reglas, umbral por Youden, límite de Spearman 0,80). Una imagen es **patológica si se cumplen al menos 3 de 5**:

| Descriptor | Regla → patológica | Interpretación |
|---|---|---|
| `b_desvio` | ≤ 9,28 | color del núcleo más homogéneo |
| `glcm_correlacion_d3` | ≥ 0,25 | gris que varía de forma gradual (cromatina fina) |
| `S_rel` | ≤ 2,76 | núcleo menos saturado respecto del resto de la imagen |
| `H_desvio` | ≤ 2,88 | tono del núcleo más homogéneo |
| `glcm_energia_d3` | ≤ 0,095 | textura menos ordenada |

**Desempeño en prueba:**

| | Predicho no patológica | Predicho patológica |
|---|---:|---:|
| **Real no patológica** | 118 (VN) | 33 (FP) |
| **Real patológica** | 52 (FN) | 774 (VP) |

Sensibilidad 0,937 · especificidad 0,781 · exactitud 0,913 · precisión positiva 0,959.

**Salidas:** `modulo4_particiones.csv`, `modulo4_estadistica_ajuste.csv`, `modulo4_barrido_ajuste.csv`, `modulo4_predicciones_prueba.csv` y **`modulo4_reglas_finales.json`** (descriptores, signos y umbrales del clasificador; para una imagen nueva basta con calcular los mismos descriptores y aplicar estas reglas, sin reentrenar).

---

## Estructura del repositorio

```
notebooks/
  01_contextualizacion_preprocesamiento.ipynb   Módulo 1: dataset y corrección de iluminación
  02_segmentacion.ipynb                         Módulo 2: segmentación de núcleos (k-means global en a*, b*)
  03_descriptores.ipynb                         Módulo 3: descriptores de forma, color y textura + compresión JPEG
  04_clasificacion_evaluacion.ipynb             Módulo 4: reglas de umbral, votación y evaluación
requirements.txt                                Dependencias
```

Los CSV y JSON generados no se suben al repositorio (ver `.gitignore`).

## Instalación y ejecución

Los notebooks se corren **en orden**. Cada uno copia las funciones que necesita de los anteriores, así que se puede ejecutar solo; la única dependencia entre notebooks es que el **04 necesita `modulo3_descriptores.csv`**, generado por el 03.

**Opción 1: Google Colab (recomendada).** Abrir el notebook en Colab y ejecutar todas las celdas (Entorno de ejecución → Ejecutar todas). Las dependencias ya vienen instaladas. Para el Módulo 4, descargar `modulo3_descriptores.csv` del Colab del Módulo 3 y subirlo cuando el notebook lo pida.

**Opción 2: local**, con Python 3.10 o superior:

```bash
pip install -r requirements.txt
jupyter notebook notebooks/
```

El Módulo 3 recorre las 3.256 imágenes varias veces y tarda unos minutos.

## Limitaciones

- **División por imagen, no por paciente:** el dataset no trae identificador de paciente, así que imágenes de un mismo paciente pueden quedar en ajuste y en prueba.
- **Desbalance de clases** (15,5 % benignas): la especificidad se estima con sólo 151 imágenes benignas de prueba.
- **Sin máscara de referencia del núcleo:** la segmentación se valida por concordancia entre métodos e inspección visual, no contra una verdad anotada.
- **Efecto de la tinción:** Pro-B está teñido más pálido; se mitigó usando descriptores relativos y desvíos en lugar de medias absolutas de color, pero no se puede descartar del todo.
- **Relación núcleo/citoplasma no medida:** no se pudo validar la segmentación del citoplasma (Módulo 3, sección 2).
- Las métricas son indicadores internos del trabajo, **no una estimación de rendimiento clínico**.

## Referencias

- Aria M, Ghaderzadeh M, Bashash D, et al. *Acute Lymphoblastic Leukemia (ALL) image dataset*. Kaggle, 2021. doi:10.34740/kaggle/dsv/2175623.
- Mann HB, Whitney DR. On a test of whether one of two random variables is stochastically larger than the other. *Ann Math Stat*. 1947;18(1):50–60.
- Benjamini Y, Hochberg Y. Controlling the false discovery rate. *J R Stat Soc B*. 1995;57(1):289–300.
- Youden WJ. Index for rating diagnostic tests. *Cancer*. 1950;3(1):32–35.
