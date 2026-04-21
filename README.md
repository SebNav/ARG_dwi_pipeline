# Pipeline de Tractografía y Métricas de Difusión

## Descripción General

Este pipeline realiza el procesamiento completo de imágenes de resonancia magnética de difusión (DWI): desde el preprocesamiento hasta la segmentación de fascículos de sustancia blanca y la extracción de métricas de difusión por fascículo.

Todo el software necesario (FSL, MRtrix3, DSI Studio, Python) está incluido en la imagen Docker. No se requiere instalación adicional.

---

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) instalado
- (Opcional, pero **muy recomendado**) GPU NVIDIA con [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

> **¿Por qué GPU?** El paso de corrección de distorsiones y movimiento (`eddy`) es el más lento del pipeline. Con GPU (`eddy_cuda`) este paso puede completarse en **5–10 minutos**; sin GPU (`eddy_cpu`) puede tomar entre **30 y 90 minutos** por sujeto. Si tu equipo tiene una tarjeta NVIDIA, se recomienda instalar CUDA y el nvidia-container-toolkit antes de usar el pipeline.


## Carga de la Imagen

Descarga la imagen Docker desde el drive compartido. Desde el directorio donde se encuentra el archivo descargado `dwi_pipeline.tar.gz`:

```bash
docker load -i dwi_pipeline.tar.gz
```

> La imagen incluye FSL (~5 GB), MRtrix3 y DSI Studio. La carga puede tardar unos minutos.

---

## Estructura de Datos de Entrada

El pipeline espera que los datos DWI estén organizados en formato BIDS. La imagen T1w puede estar en cualquier ubicación:

```
/ruta/a/mi/carpeta/de/sujetos/
└── sub-XX/
    └── ses-Y/
        └── dwi/
            ├── <nombre_PA>.nii        # DWI dirección PA (Posterior→Anterior)
            ├── <nombre_PA>.bvec
            ├── <nombre_PA>.bval
            ├── <nombre_PA>.json
            ├── <nombre_AP>.nii        # DWI dirección AP (Anterior→Posterior)
            ├── <nombre_AP>.bvec
            ├── <nombre_AP>.bval
            └── <nombre_AP>.json
```

> La imagen T1w debe ser la imagen preprocesada para un mejor registro al espacio de difusion. Puede provenir de cualquier fuente: carpeta `anat/`, FreeSurfer, etc.

---

## Sobre los volúmenes de Docker (`-v`)

Docker es un contenedor aislado: por defecto no puede ver los archivos de tu computador. La opción `-v` conecta una carpeta de tu sistema con una carpeta dentro del contenedor:

```
-v /ruta/en/tu/computador:/ruta/dentro/del/contenedor
```

Por ejemplo:

```
-v /media/data/Subjects:/data
```

Hace que tu carpeta `/media/data/Subjects` sea accesible dentro del contenedor como `/data`. Cualquier archivo que el pipeline guarde en `/data` aparecerá directamente en `/media/data/Subjects` en tu computador.

Cuando uses el pipeline, reemplaza `/ruta/a/mi/carpeta/de/sujetos` con la ruta real en tu sistema. Por ejemplo, si tus sujetos están en `/home/usuario/MRI/Subjects`, el comando sería:

```bash
-v /home/usuario/MRI/Subjects:/data
```

Si además necesitas montar la carpeta de FreeSurfer, agrega un segundo `-v`:

```bash
-v /home/usuario/MRI/Subjects:/data \
-v /home/usuario/freesurfer:/freesurfer
```

Dentro del contenedor, la T1w de FreeSurfer estaría accesible como `/freesurfer/sub-XX/mri/T1w_brain.nii.gz`.

---

## Uso

### Ejecución básica

```bash
docker run --rm \
    -v /ruta/a/mi/carpeta/de/sujetos:/data \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /data/sub-13/ses-1/anat/T1w_brain.nii.gz
```

### Con GPU (recomendado — acelera la corrección de Eddy)

```bash
docker run --rm --gpus all \
    -v /ruta/a/mi/carpeta/de/sujetos:/data \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /data/sub-13/ses-1/anat/T1w_brain.nii.gz
```

### Con T1w de FreeSurfer (montar directorio adicional)

```bash
docker run --rm --gpus all \
    -v /ruta/a/mi/carpeta/de/sujetos:/data \
    -v /ruta/a/freesurfer:/freesurfer \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /freesurfer/sub-13/mri/T1w_brain.nii.gz
```

---

## Argumentos del Pipeline

| Argumento | Requerido | Descripción |
|---|---|---|
| `-s`, `--subject` | Sí | Ruta relativa del sujeto/sesión dentro de `/data` (e.g. `sub-13/ses-1`) |
| `--dwi_pa` | Sí | Nombre del archivo DWI en dirección PA (sin extensión) |
| `--dwi_ap` | Sí | Nombre del archivo DWI en dirección AP (sin extensión) |
| `--t1w` | Sí | Ruta absoluta dentro del contenedor a la imagen T1w (`.nii` o `.nii.gz`) |
| `--mni` | No | Ruta a una plantilla MNI alternativa (por defecto: incluida en la imagen) |
| `--trt` | No | `TotalReadoutTime` en segundos (por defecto: calculado desde el JSON) |
| `--tract_count` | No | Número de streamlines para la tractografía (por defecto: `2000000`) |
| `--cleanup` | No | Elimina archivos intermedios al finalizar, conservando solo los resultados finales |

> **Nota:** el directorio raíz `/data` está configurado automáticamente — no es necesario pasar `-w`.

---

## Salidas

Los resultados se guardan en `/ruta/a/mi/carpeta/de/sujetos/sub-XX/ses-Y/Tractography/`:

```
Tractography/
├── DWI_AP_PA_preprocessed.nii.gz       # DWI preprocesado
├── DWI_AP_PA_preprocessed.bvec/.bval   # Gradientes corregidos
├── DWI_AP_PA_preprocessed.fib.gz       # Reconstrucción GQI (DSI Studio)
├── DWI_brain_mask.nii.gz               # Máscara cerebral
├── wholebrain_tractography.tck         # Tractografía de cerebro completo
├── T1w_in_DWI.nii.gz                   # T1w en espacio DWI
├── T1w_in_MNI.nii.gz                   # T1w en espacio MNI
├── diffusion_metrics/
│   ├── FA.nii.gz                       # Anisotropía Fraccional — GQI (DSI Studio)
│   ├── FA_dti.nii.gz                   # Anisotropía Fraccional — tensor DTI (MRtrix3)
│   ├── QA.nii.gz                       # Quantitative Anisotropy (GQI, DSI Studio)
│   ├── QIR.nii.gz                      # Quantitative Isotropic Ratio (GQI, DSI Studio)
│   ├── RD.nii.gz                       # Difusividad Radial — GQI (DSI Studio)
│   ├── RD_dti.nii.gz                   # Difusividad Radial — tensor DTI (MRtrix3)
│   ├── AD.nii.gz                       # Difusividad Axial — tensor DTI (MRtrix3)
│   ├── MD.nii.gz                       # Difusividad Media — tensor DTI (MRtrix3)
│   ├── ISO.nii.gz                      # Componente Isotrópica (GQI, DSI Studio)
│   └── RDI.nii.gz                      # Restricted Diffusion Imaging (GQI, DSI Studio)
├── bundles/
│   ├── results_folder_DWM_DWI_space/   # Fascículos DWM en espacio DWI (.tck)
│   └── results_folder_SWM_e_DWI_space/ # Fascículos SWM estables en espacio DWI (.tck)
└── outputs/                            ← carpeta de resultados principales
    ├── bundle_metrics_DWM.csv          # Métricas por fascículo DWM
    ├── bundle_metrics_SWM_e.csv        # Métricas por fascículo SWM
    ├── connectivity.tt.gz.HCP-MMP.connectivity.mat     # Matriz de conectividad 360×360 (formato MATLAB)
    └── connectivity.tt.gz.HCP-MMP.network_measures.txt # Métricas de red por región (texto plano)
```

> **¿Por qué hay dos FA y dos RD?**
> DSI Studio genera métricas a partir de una reconstrucción GQI (*Generalized Q-sampling Imaging*), un modelo más avanzado que el tensor DTI clásico. Sin embargo, DSI Studio **no exporta AD ni MD** ya que su reconstrucción no calcula los eigenvalores del tensor completo.
> Para obtener AD y MD se utilizó MRtrix3 (`dwi2tensor` + `tensor2metric`), que ajusta el tensor DTI estándar directamente sobre el DWI preprocesado. Este proceso también produce sus propias versiones de FA y RD (`FA_dti`, `RD_dti`), que pueden diferir ligeramente de las de DSI Studio.
>
> | Archivo | Origen | Cuándo usarlo |
> |---|---|---|
> | `FA.nii.gz` | DSI Studio (GQI) | Análisis basados en GQI; consistente con QA y QIR |
> | `FA_dti.nii.gz` | MRtrix3 (tensor DTI) | Comparación con estudios DTI clásicos |
> | `RD.nii.gz` | DSI Studio (GQI) | Análisis basados en GQI |
> | `RD_dti.nii.gz` | MRtrix3 (tensor DTI) | Usar junto con AD y MD para análisis DTI completo |
> | `AD.nii.gz` | MRtrix3 (tensor DTI) | Difusividad axial (λ₁) |
> | `MD.nii.gz` | MRtrix3 (tensor DTI) | Difusividad media ((λ₁+λ₂+λ₃)/3) |

> **Archivos de conectividad global (HCP-MMP atlas)**
> La conectividad estructural se calcula sobre la tractografía de cerebro completo usando el atlas HCP-MMP (360 regiones). DSI Studio genera dos archivos:
>
> | Archivo | Descripción |
> |---|---|
> | `connectivity.tt.gz.HCP-MMP.connectivity.mat` | Matriz 360×360 con el número de streamlines entre cada par de regiones. Formato MATLAB, legible en Python con `scipy.io.loadmat()` |
> | `connectivity.tt.gz.HCP-MMP.network_measures.txt` | Métricas de teoría de grafos por región: grado nodal, coeficiente de clustering, eficiencia local, longitud de camino, etc. Formato texto plano, abrir directamente en Excel o cualquier editor |

### Columnas de los CSV de métricas

| Columna | Descripción |
|---|---|
| `bundle` | Nombre del fascículo |
| `metrics` | Métrica (AD, FA, ISO, MD, QA, QIR, RD, RDI) |
| `n_streamlines` | Número de streamlines del fascículo |
| `vol_mm3` | Volumen en mm³ (vóxeles únicos × volumen de vóxel) |
| `mean` | Media de las medias por streamline |
| `std` | Desviación estándar entre streamlines |
| `median` | Mediana de las medias por streamline |
| `IQR` | Rango intercuartílico entre streamlines |

---

## Ejemplo Completo

```bash
docker run --rm --gpus all \
    -v /media/data/Subjects:/data \
    -v /media/data/freesurfer:/freesurfer \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /freesurfer/sub-13/mri/T1w_brain.nii.gz \
    --tract_count 5000000 \
    --cleanup
```

### Procesamiento de todos los sujetos en lote

El siguiente script itera sobre todos los sujetos y sesiones encontrados en la carpeta de sujetos y lanza el pipeline para cada uno:

```bash
#!/bin/bash
SUBJECTS_DIR=/ruta/a/mi/carpeta/de/sujetos
FREESURFER_DIR=/ruta/a/freesurfer   # opcional, ajustar según corresponda
DWI_PA_SUFFIX=dir-PA_dwi            # sufijo del archivo PA (sin sub-XX_ses-Y_)
DWI_AP_SUFFIX=dir-AP_dwi            # sufijo del archivo AP (sin sub-XX_ses-Y_)

for sub_dir in "${SUBJECTS_DIR}"/sub-*/ses-*/; do
    # Extraer sub-XX/ses-Y desde la ruta completa
    SUBJECT=$(echo "${sub_dir}" | grep -oP 'sub-[^/]+/ses-[^/]+')
    SUB=$(echo "${SUBJECT}" | cut -d'/' -f1)   # sub-XX
    SES=$(echo "${SUBJECT}" | cut -d'/' -f2)   # ses-Y
    PREFIX="${SUB}_${SES}"

    echo "=============================="
    echo "Procesando: ${SUBJECT}"
    echo "=============================="

    docker run --rm --gpus all \
        -v "${SUBJECTS_DIR}:/data" \
        -v "${FREESURFER_DIR}:/freesurfer" \
        dwi_pipeline \
        -s "${SUBJECT}" \
        --dwi_pa "${PREFIX}_${DWI_PA_SUFFIX}" \
        --dwi_ap "${PREFIX}_${DWI_AP_SUFFIX}" \
        --t1w "/freesurfer/${SUB}/mri/T1w_brain.nii.gz" \
        --cleanup
done
```

> Los sujetos se procesan **de forma secuencial**.

---

## Solución de Problemas

| Problema | Causa | Solución |
|---|---|---|
| Archivos con ícono de candado | Docker configurado para correr como root en ese sistema | Agregar `-u $(id -u):$(id -g) -w /tmp` al comando |
| `eddy_cuda` falla → usa CPU | Sin GPU o sin `--gpus all` | Agregar `--gpus all` si hay GPU NVIDIA disponible |
| `dsi_studio: command not found` | PATH incorrecto en la imagen | Recargar la imagen con `docker load` |

---

## Autor

Sebastián Navarrete — sebastian.navarrete@biomedica.udec.cl
