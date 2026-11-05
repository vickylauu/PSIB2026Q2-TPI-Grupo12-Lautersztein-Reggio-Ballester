# PSIB2026Q2 · TPI · Grupo 12

**Detección de leucemia linfoblástica aguda en imágenes de frotis de sangre periférica**

16.63 Procesamiento de Señales e Imágenes Biomédicas · ITBA · 2026 Q2

Integrantes: Victoria Lautersztein (62437), María Reggio (62303), Rosario Ballester (62648)

## Descripción

Algoritmo clásico e interpretable para segmentar los núcleos de los linfocitos en imágenes de frotis de sangre periférica y clasificar cada imagen como patológica (LLA) o no patológica, mediante reglas de decisión sobre descriptores de forma, color y textura.

## Datos

*Acute Lymphoblastic Leukemia (ALL) Image Dataset* (Aria et al., 2021, Kaggle, doi:10.34740/kaggle/dsv/2175623): 3.256 imágenes de 224 × 224 px de 89 pacientes, en cuatro clases (Benign, Early Pre-B, Pre-B, Pro-B), con su segmentación de referencia.

Los datos **no se incluyen en el repositorio**. El primer notebook los descarga automáticamente con `kagglehub` (si pide credenciales, configurar el `kaggle.json` de una cuenta de Kaggle).

## Estructura

```
notebooks/
  01_contextualizacion_preprocesamiento.ipynb  Módulo 1: caracterización del dataset y corrección de iluminación
  02_segmentacion.ipynb                        Módulo 2: segmentación de núcleos (HSV y k-means en Lab; sección 5: k-means global sin referencia)
  03_descriptores.ipynb                        Módulo 3: descriptores de forma, color y textura (GLCM, LBP) + experimento de compresión JPEG
  04_clasificacion_evaluacion.ipynb            Módulo 4: 
requirements.txt                               Dependencias
```

## Instalación y ejecución

**Opción 1: Google Colab (recomendada).** Abrir el notebook en Colab y ejecutar todas las celdas en orden (Entorno de ejecución → Ejecutar todas). Las dependencias ya vienen instaladas.

**Opción 2: local**, con Python 3.10 o superior (los notebooks se corren en orden; el 03 genera `modulo3_descriptores.csv`, que usa el Módulo 4):

```bash
pip install -r requirements.txt
jupyter notebook notebooks/01_contextualizacion_preprocesamiento.ipynb
```

## Estado

| Módulo | Contenido | Estado |
|---|---|---|
| 1 | Contextualización y preprocesamiento (corrección de iluminación) | Hecho |
| 2 | Segmentación de núcleos: k-means global en (a*, b*), independiente de la segmentación de referencia (que falla en Pro-B) | Hecho |
| 3 | Extracción de descriptores (forma, color, textura GLCM/LBP) y efecto de la compresión JPEG | Hecho |
| 4 | Clasificación y evaluación | Hecho |
