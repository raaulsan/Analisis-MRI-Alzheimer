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

Nuestro enfoque aborda y soluciona varios de los desafíos clásicos en el procesamiento de imágenes médicas (*Medical Image Computing*), dividiendo el flujo de trabajo en 4 fases críticas:

### 1. Preprocesamiento y Normalización Robusta
Los datos crudos de MRI presentan alta variabilidad de contraste. Implementamos una normalización robusta basada en el análisis de histogramas 3D.
* **Filtro de Fondo:** Eliminación del ruido del aire (< 10% del percentil).
* **Clipping de Percentiles (1-99%):** Recorte de artefactos extremos.
* **Escalado Min-Max:** Estandarización de la densidad de tejido al rango `[0, 1]`.

### 2. Registro Intra-Sujeto (Alineación Longitudinal)
Para poder comparar la visita del Año 1 con la del Año 2, es imperativo alinear ambas cabezas milimétricamente en el mismo espacio físico.
* **Header Stripping:** Solucionamos problemas de metadatos corruptos (comunes en el formato Analyze 7.5 antiguo) extrayendo las matrices puras con `nibabel` e inicializando geometrías neutras.
* **SimpleITK Registration:** Utilizamos *Mattes Mutual Information* y descenso de gradiente, inicializando la transformación mediante el centro de masa real (*MOMENTS*).

### 3. El Desafío del "Skull Stripping" y la Decisión de Diseño
Experimentamos con métodos clásicos (Multi-Otsu, erosiones morfológicas asimétricas y rechazo estadístico espacial mediante Z-Scores 3D) para extraer el cerebro. 
Documentamos un límite fundamental: la similitud de intensidades y conectividad física entre la cara/cuello y el córtex provoca que los métodos matemáticos globales sean propensos a generar artefactos anatómicos. Esto nos llevó a **pivotar hacia Foundation Models**.

### 4. Segmentación "Zero-Shot" con MedSAM
Para superar los límites de la visión clásica, integramos **MedSAM** sin necesidad de reentrenamiento (*Zero-Shot Inference*), utilizando una estrategia de *Doble Prompting Automático*:
* **Fase Macro (Cerebro Completo):** Utilizamos máscaras de Otsu recicladas para calcular un *Bounding Box* matemático, permitiendo a MedSAM aislar el cerebro e ignorar el cráneo.
* **Fase Micro (Ventrículos):** Calculamos el centroide de la masa cerebral y lo inyectamos como un *Point Prompt*. MedSAM segmenta los ventrículos con precisión, permitiéndonos calcular su volumen exacto y la atrofia longitudinal de los pacientes.

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
