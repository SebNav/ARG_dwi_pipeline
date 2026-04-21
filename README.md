# Pipeline de Procesamiento de Imágenes de Difusión y Tractografía

## Descripción General

Este pipeline realiza el procesamiento completo de imágenes de resonancia magnética de difusión (DWI), desde el preprocesamiento de las imágenes crudas hasta la segmentación de fascículos de sustancia blanca y la extracción de métricas de difusión por fascículo.

El pipeline está implementado en `Pipeline.sh` y utiliza una serie de herramientas externas junto con scripts Python auxiliares.

---

## Dependencias

| Herramienta | Uso |
|---|---|
| **MRtrix3** | Conversión de formatos, denoise, corrección de distorsiones, tractografía, registro de tractos |
| **FSL** | Corrección de corrientes de Eddy (eddy/topup), registro de imágenes (flirt/fnirt) |
| **DSI Studio** | Reconstrucción GQI, exportación de métricas, tractografía deterministia/probabilística |
| **Python 3** | Segmentación de fascículos, conversión de formatos, extracción de métricas |
| **jq** | Manipulación de archivos JSON (metadatos BIDS) |

### Scripts Python requeridos

- `fibras_cambio_formato.py` — Conversión entre formatos de tractografía (`.tck`, `.trk`, `.trk.gz`, `.bundles`)
- `Segmentation_main_process.py` — Segmentación de fascículos de sustancia blanca superficial (SWM) y profunda (DWM)
- `compute_bundle_metrics.py` — Extracción de métricas de difusión por fascículo

---

## Estructura de Datos de Entrada

El pipeline espera la siguiente estructura de directorios compatible con BIDS para los datos DWI. La imagen T1w puede encontrarse en cualquier ubicación (se especifica como ruta absoluta):

```
WORKING_DIR/
└── sub-XX/
    └── ses-Y/
        └── dwi/
            ├── <DWI_PA_NAME>.nii        # Volumen DWI dirección PA (Posterior→Anterior)
            ├── <DWI_PA_NAME>.bvec
            ├── <DWI_PA_NAME>.bval
            ├── <DWI_PA_NAME>.json
            ├── <DWI_AP_NAME>.nii        # Volumen DWI dirección AP (Anterior→Posterior)
            ├── <DWI_AP_NAME>.bvec
            ├── <DWI_AP_NAME>.bval
            └── <DWI_AP_NAME>.json
```

> La imagen T1w se especifica como ruta absoluta con `--t1w`. Puede provenir de cualquier fuente: carpeta `anat/` local, salida de FreeSurfer, etc. Debe tener extracción de cráneo aplicada.

---

## Uso

```bash
bash Pipeline.sh \
    -w /ruta/al/directorio/sujetos \
    -s sub-23/ses-1 \
    --dwi_pa sub-23_ses-1_dwi-1 \
    --dwi_ap sub-23_ses-1_dwi-2 \
    --t1w /ruta/absoluta/T1w_brain.nii.gz
```

### Argumentos

| Argumento | Requerido | Descripción |
|---|---|---|
| `-w`, `--working_dir` | Sí | Directorio raíz que contiene los sujetos |
| `-s`, `--subject` | Sí | Ruta relativa del sujeto/sesión (e.g. `sub-23/ses-1`) |
| `--dwi_pa` | Sí | Nombre del archivo DWI en dirección PA (sin extensión) |
| `--dwi_ap` | Sí | Nombre del archivo DWI en dirección AP (sin extensión) |
| `--t1w` | Sí | Ruta absoluta a la imagen T1w (`.nii` o `.nii.gz`, con cráneo extraído) |
| `--mni` | No | Ruta a la plantilla MNI (por defecto: plantilla incluida en el pipeline) |
| `--trt` | No | `TotalReadoutTime` en segundos (por defecto: calculado automáticamente desde el JSON) |
| `--tract_count` | No | Número de streamlines a generar en la tractografía de cerebro completo (por defecto: `2000000`) |
| `--cleanup` | No | Elimina archivos intermedios al finalizar el pipeline |

---

## Descripción de los Pasos

### PASO 1 — Preprocesamiento DWI

#### 1.0 — Parche de metadatos JSON
Los archivos `.json` sidecar de la adquisición DWI pueden carecer de los campos `PhaseEncodingDirection` y `TotalReadoutTime`, que son requeridos por las herramientas de corrección de distorsiones.

- Se inyecta `PhaseEncodingDirection: "j-"` al archivo PA y `"j"` al archivo AP.
- El `TotalReadoutTime` se calcula automáticamente a partir de los parámetros de adquisición Philips usando la fórmula:

```
TRT = EchoTrainLength × (AcquisitionMatrixPE - 1) / (PixelBandwidth × AcquisitionMatrixPE)
```

> Este valor puede sobreescribirse manualmente con el argumento `--trt`.

#### 1.1 — Conversión a formato MIF (MRtrix)
Los volúmenes DWI en NIfTI se convierten al formato interno de MRtrix (`.mif`), embebiendo los gradientes (`.bvec`/`.bval`) y los metadatos de codificación de fase desde el JSON.

#### 1.2 — Reducción de ruido
Los volúmenes PA y AP se concatenan y se aplica `dwidenoise` (PCA-Marchenko-Pastur) de forma conjunta para una estimación más precisa del mapa de ruido. Luego se separan de nuevo en los volúmenes individuales.

#### 1.3 — Corrección de ringing de Gibbs
Se aplica `mrdegibbs` a cada volumen por separado para reducir los artefactos de truncamiento de la señal.

#### 1.4 — Concatenación para corrección de distorsiones
Los volúmenes AP y PA corregidos se concatenan en un único archivo.

#### 1.5 — Corrección de distorsiones y corrientes de Eddy
`dwifslpreproc` ejecuta el pipeline FSL `topup` + `eddy` utilizando la información de codificación de fase almacenada en el header para corregir:
- Distorsiones por susceptibilidad magnética (topup, par AP/PA)
- Movimiento de cabeza y corrientes de Eddy

Opciones de eddy: `--repol` (reemplazo de outliers), `--cnr_maps`, `--ol_nstd=4`, `--ol_type=sw`, `--slm=linear`.

#### 1.6 — Exportación a NIfTI
El archivo MIF preprocesado se convierte de vuelta a NIfTI (`.nii.gz`) exportando también los gradientes corregidos en formato FSL (`.bvec`/`.bval`).

#### 1.7 — Limpieza de intermedios de preprocesamiento
Se eliminan todos los archivos `.mif` intermedios generados durante los pasos 1.1–1.6.

#### 1.8 — Máscara cerebral
Se calcula una máscara binaria del cerebro a partir del DWI preprocesado usando `dwi2mask`.

---

### PASO 2 — Reconstrucción y Tractografía (DSI Studio)

#### 2.1 — Creación del archivo SRC
`dsi_studio --action=src` crea el archivo fuente `.src.gz` a partir del DWI preprocesado. Los archivos `.bvec`/`.bval` se detectan automáticamente por nombre de archivo.

#### 2.2 — Reconstrucción GQI
`dsi_studio --action=rec` realiza la reconstrucción mediante **Quantitative Anisotropy (GQI)** (método 4, `param0=1.25`), generando el archivo `.fib.gz` que contiene los orientadores de fibras y los mapas métricos.

#### 2.3 — Exportación de mapas métricos
Se exportan los siguientes mapas en formato NIfTI y se organizan en la carpeta `diffusion_metrics/`:

| Archivo | Métrica |
|---|---|
| `QA.nii.gz` | Quantitative Anisotropy |
| `QIR.nii.gz` | Quantitative Isotropic Ratio |
| `FA.nii.gz` | Anisotropía Fraccional (DTI) |
| `AD.nii.gz` | Difusividad Axial — λ₁ (DTI) |
| `RD.nii.gz` | Difusividad Radial — (λ₂+λ₃)/2 (DTI) |
| `MD.nii.gz` | Difusividad Media — (λ₁+λ₂+λ₃)/3 (DTI) |
| `ISO.nii.gz` | Componente Isotrópica (GQI) |
| `RDI.nii.gz` | Restricted Diffusion Imaging |

#### 2.4 — Tractografía de cerebro completo
Se genera una tractografía probabilística de cerebro completo usando los orientadores GQI. Parámetros:

| Parámetro | Valor por defecto |
|---|---|
| Número de streamlines | 2 000 000 (configurable con `--tract_count`) |
| Umbral FA | 0.03 |
| Paso | 0.5 mm |
| Ángulo máximo de giro | 45° |
| Longitud mínima | 15 mm |
| Longitud máxima | 300 mm |
| Suavizado | 0.1 |

La tractografía se guarda inicialmente en formato `.trk.gz` y luego se convierte a `.tck` (MRtrix) mediante `fibras_cambio_formato.py`.

> Para estudios de SWM (sustancia blanca superficial) donde los fascículos cortos estén subrepresentados, se recomienda aumentar el conteo a 5 000 000 con `--tract_count 5000000`.

---

### PASO 3 — Registro de Imágenes

#### 3.1 — Registro T1w → espacio DWI (6 DOF)
La imagen T1w se registra al espacio DWI en tres pasos para preservar la resolución nativa de la T1w:

1. **FLIRT (6 DOF, normmi)**: calcula la matriz de transformación rígida usando el b0 medio (sin cráneo) como referencia.
2. **transformconvert**: convierte la matriz FSL al formato de transformación de MRtrix.
3. **mrtransform** (sin `-template`): aplica la transformación manteniendo la resolución nativa de la T1w con interpolación cúbica.

> Se usa el b0 medio como referencia en lugar del volumen DWI completo porque tiene la misma modalidad y geometría que la T1w sin la contaminación de la difusión.

#### 3.2 — Registro T1w → espacio MNI (no lineal) y transformación de tractos

1. **FLIRT (12 DOF)**: registro afín T1w → MNI como inicialización.
2. **FNIRT**: registro no lineal T1w → MNI generando el campo de deformación `warps_T1w2MNI`.
3. **applywarp**: aplica el warp para generar `T1w_in_MNI.nii.gz`.
4. **invwarp**: invierte el warp para obtener la transformación MNI → DWI.
5. **Construcción del warp para MRtrix**: usando `warpinit` + `applywarp` se construye un campo de deformación compatible con `tcktransform` (grilla DWI con coordenadas MNI como valores).
6. **tcktransform**: aplica el warp a la tractografía completa para obtener `wholebrain_tractography_MNI.tck` en espacio MNI.
7. **warpinvert**: almacena el warp inverso (MNI → DWI) para uso futuro.

---

### PASO 4 — Segmentación de Fascículos

#### 4.1 — Remuestreo de streamlines
La tractografía en espacio MNI se remuestrea a **21 puntos por streamline** con `tckresample`. Este paso es requerido por el algoritmo de segmentación `main_index`.

#### 4.2 — Segmentación (`Segmentation_main_process.py`)
Se ejecuta el algoritmo de segmentación sobre la tractografía en espacio MNI. Los tipos de segmentación disponibles son:

| Tipo | Atlas | Descripción |
|---|---|---|
| `DWM` | `atlas_faisceaux/` | Sustancia blanca profunda (Deep White Matter) |
| `SWM` | `AtlasRo/` | Sustancia blanca superficial (Shallow White Matter) |
| `SWM_e` | `AtlasRo/` (estables) | Fascículos SWM estables únicamente |

El script:
1. Convierte la tractografía `.tck` a formato `.bundles` para el algoritmo de segmentación.
2. Ejecuta `main_index` que asigna cada streamline al fascículo más similar del atlas.
3. Convierte los resultados de `.bundles` a `.tck`.
4. **Retroproyección al espacio DWI** (`--tract_to_DWI`): usando los índices de segmentación (que apuntan a posiciones en la tractografía original en espacio DWI), extrae los streamlines correspondientes preservando el número original de puntos por streamline.

---

### PASO 5 — Extracción de Métricas por Fascículo (`compute_bundle_metrics.py`)

Para cada fascículo segmentado en espacio DWI, se muestrean los mapas métricos a lo largo de los streamlines y se calculan estadísticas.

**Método de muestreo**: Se calcula la media de cada streamline individualmente y luego se agregan esas medias, dando **igual peso a cada streamline** independientemente de su número de puntos. Esto es equivalente a `tcksample -stat_tck mean` y evita el sesgo hacia streamlines largos.

**Estadísticas calculadas por fascículo y métrica**:

| Columna | Descripción |
|---|---|
| `bundle` | Nombre del fascículo |
| `metrics` | Nombre de la métrica (FA, QA, AD, RD, MD, etc.) |
| `n_streamlines` | Número de streamlines del fascículo (proxy de conectividad) |
| `vol_mm3` | Volumen del fascículo en mm³ (vóxeles únicos × volumen de vóxel) |
| `mean` | Media de las medias por streamline |
| `std` | Desviación estándar entre streamlines |
| `median` | Mediana de las medias por streamline |
| `IQR` | Rango intercuartílico entre streamlines |

Los resultados se guardan en:
- `bundles/bundle_metrics_DWM.csv`
- `bundles/bundle_metrics_SWM.csv`
- `bundles/bundle_metrics_SWM_e.csv`

---

### CLEANUP — Limpieza de Archivos Intermedios

Si se ejecuta con `--cleanup`, se eliminan todos los archivos intermedios conservando únicamente:

```
Tractography/
├── DWI_brain_mask.nii.gz
├── DWI_AP_PA_preprocessed.nii.gz
├── DWI_AP_PA_preprocessed.bvec
├── DWI_AP_PA_preprocessed.bval
├── DWI_AP_PA_preprocessed.fib.gz
├── wholebrain_tractography.tck
├── diffusion_metrics/
│   ├── FA.nii.gz
│   ├── QA.nii.gz
│   └── ...
└── bundles/
    ├── results_folder_DWM_DWI_space/
    ├── results_folder_SWM_DWI_space/
    ├── results_folder_SWM_e_DWI_space/
    ├── bundle_metrics_DWM.csv
    ├── bundle_metrics_SWM.csv
    └── bundle_metrics_SWM_e.csv
```

---

## Ejemplo de Ejecución Completa

```bash
bash Pipeline.sh \
    -w /media/data/Subjects \
    -s sub-23/ses-1 \
    --dwi_pa sub-23_ses-1_dwi-1 \
    --dwi_ap sub-23_ses-1_dwi-2 \
    --t1w /media/data/freesurfer/sub-23/mri/T1w_brain.nii.gz \
    --tract_count 5000000 \
    --cleanup
```

---

## Ejecución con Docker

La imagen Docker incluye todos los programas necesarios (FSL, MRtrix3, DSI Studio) y la plantilla MNI. Solo es necesario montar el directorio de datos:

```bash
docker run --rm \
    -v /media/data/Subjects:/data \
    -v /media/data/freesurfer:/freesurfer \
    dwi-pipeline \
    -s sub-23/ses-1 \
    --dwi_pa sub-23_ses-1_dwi-1 \
    --dwi_ap sub-23_ses-1_dwi-2 \
    --t1w /freesurfer/sub-23/mri/T1w_brain.nii.gz \
    --cleanup
```

> La plantilla MNI (`MNI152_T1_3mm_brain.nii.gz`) está incluida en la imagen en `/opt/pipeline/` y se usa automáticamente si no se especifica `--mni`.

---

## Autores

Sebastián Navarrete — sebastian.navarrete@biomedica.udec.cl
