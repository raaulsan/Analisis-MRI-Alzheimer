# 🧠 Cuantificación Longitudinal del Alzheimer mediante Resonancia Magnética (OASIS-2)

![Python](https://img.shields.io/badge/Python-3.8%2B-blue)
![SimpleITK](https://img.shields.io/badge/SimpleITK-Image%20Registration-brightgreen)
![MedSAM](https://img.shields.io/badge/MedSAM-Zero--Shot%20Segmentation-ff69b4)
![Environment](https://img.shields.io/badge/Environment-Kaggle-20BEFF)

## 📌 Resumen del Proyecto
Este proyecto desarrolla un **Pipeline Médico End-to-End** automatizado para la cuantificación de la atrofia cerebral (un biomarcador clave en la enfermedad de Alzheimer). Utilizando resonancias magnéticas (MRI) longitudinales del dataset [OASIS-2](https://www.oasis-brains.org/), el sistema alinea, estandariza y segmenta automáticamente las estructuras cerebrales de un mismo paciente a lo largo de diferentes años para medir la dilatación ventricular.

## 👥 El Equipo
Este proyecto ha sido desarrollado colaborativamente por:
* **Álvaro Espejo Martínez** - [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/espejomartinezalvaro-spec)
* **Javier Hernández Rosique** - [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/javierhernandezrosique)   
* **Raúl Sánchez Ibáñez** - [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/raaulsan)

---
## 🏗️ Arquitectura del Pipeline

El pipeline se divide en **6 etapas encadenadas**, diseñadas para procesar 
cada par longitudinal (V1/V2) de forma completamente automatizada.

### 1. 📐 Preprocesamiento Robusto (P01/P02)
Las imágenes T1 de OASIS-2 presentan alta variabilidad de contraste entre 
sujetos y sesiones. Una normalización Min-Max estándar es sensible a 
artefactos de adquisición, por lo que implementamos una **normalización 
robusta por percentiles**:
- Recorte del 1% inferior y superior de la distribución de intensidades 3D
- Mapeo lineal del rango residual `[p1, p99]` → `[0, 1]`
- Produce histogramas comparables entre sujetos, condición necesaria para 
  que los umbrales de segmentación sean transferibles entre visitas

### 2. 📍 Registro Intra-Sujeto Rígido (P03)
Para que las diferencias de volumen entre visitas reflejen cambios reales 
y no artefactos de posicionamiento, alineamos V2 al espacio de V1 mediante 
una **transformación rígida de 6 DOF** (3 traslaciones + 3 rotaciones):
- **Métrica:** Información Mutua de Mattes (robusta frente a diferencias 
  de contraste entre sesiones)
- **Optimizador:** Regular Step Gradient Descent (200 iteraciones)
- **Inicialización:** `CenteredTransformInitializer` por momentos, para 
  evitar mínimos locales
- **Resultado:** NCC +104%, MSE −52% tras registro sobre el paciente de 
  referencia

> Comparamos experimentalmente tres métodos (rígido, afín y deformable 
> Demons). El rígido se selecciona por ofrecer la mejor relación 
> calidad/coste: Demons mejora las métricas fotométricas un 4% en NCC pero 
> triplica el tiempo de cómputo y puede absorber la atrofia real en el campo 
> de deformación, sesgando las tasas a la baja.

### 3. 🧩 Segmentación Ventricular (P04)
Segmentación automática de los ventrículos laterales explotando el alto 
contraste del CSF en T1, con dos **correcciones de consistencia longitudinal** 
propias:

**Algoritmo base (3 pasos):**
1. Máscara cerebral por umbral + componente conexo + cierre morfológico
2. Umbral CSF = percentil 12 dentro de la máscara cerebral
3. Filtrado por tamaño mínimo (≥500 vóxeles) y distancia al centro cerebral

**Correcciones longitudinales (aportación propia):**
- *Histogram matching* de V2 sobre V1 (1024 niveles, 7 puntos de ajuste)
- *Prior espacial*: búsqueda en V2 restringida a dilatación de 8 vóxeles 
  sobre la máscara de V1

**Validación cruzada con MedSAM:** el prompt de bounding box se genera 
automáticamente a partir de la segmentación clásica, sin anotación manual, 
y se pasa al `SamImageProcessor` de HuggingFace.

### 4. 🗺️ Registro Inter-Sujeto al Atlas MNI (Atlas)
Para comparar volúmenes entre pacientes, llevamos cada cerebro al espacio 
estándar **MNI-ICBM152** mediante una transformación afín de 12 DOF:
- Inversión de la transformación → propagación de la máscara hipocampal 
  bilateral del atlas **Harvard-Oxford** (etiquetas 9 y 19) al espacio nativo
- Como V2 ya está alineada a V1, el ROI hipocampal es idéntico en ambas 
  visitas: cualquier reducción de tejido refleja atrofia real
- Cobertura: 50/50 pacientes con resultado válido

### 5. 📊 Análisis Longitudinal
Cálculo de la **tasa de atrofia anual** para cada paciente:

$$\tau = \frac{V_2 - V_1}{V_1 \cdot \Delta t} \times 100  [\%/\text{año}]$$

Aplicado sobre dos biomarcadores independientes:
- **Ventrículos laterales** (segmentación clásica, 76/77 pacientes)
- **Hipocampo bilateral** (propagación atlas MNI, 50 pacientes)

### 6. ✅ Validación Clínica
Correlación de los biomarcadores de imagen con las escalas clínicas del CSV:
- Correlación de Pearson y Spearman con **CDR** y **MMSE**
- Test de **Mann-Whitney U** entre grupos Demented / Nondemented
- Análisis de *converters* (Nondemented en V1 → Demented en V2)
---

## 🚀 Tecnologías Utilizadas
* **Lenguaje:** Python 3
* **Procesamiento de Imagen Médica:** `SimpleITK`, `nibabel`, `scikit-image`, `scipy.ndimage`
* **Deep Learning / Foundation Models:** `MedSAM` (basado en Vision Transformers)
* **Manipulación Numérica y Visualización:** `NumPy`, `Matplotlib`

## ☁️ Entorno de Desarrollo (Kaggle)
Todo el flujo de trabajo, desde el preprocesamiento hasta la inferencia de la red neuronal, ha sido diseñado para ejecutarse en el ecosistema **Kaggle**. 
* El cuaderno (`.ipynb`) incluido en este repositorio contiene el código fuente completo.
* Requiere acceso al dataset público OASIS-2 montado en el entorno.
* La inferencia de la arquitectura Vision Transformer (MedSAM) depende de aceleración por hardware (GPU T4x2 / P100) para procesar los volúmenes 3D longitudinales en tiempos viables.

---
*Agradecimientos al proyecto [OASIS Brains](https://www.oasis-brains.org/) por hacer públicos estos datos de incalculable valor para la investigación clínica y académica.*
