# Pipeline de Tractografía y Métricas de Difusión

## Descripción General

Este pipeline realiza el procesamiento completo de imágenes de resonancia magnética de difusión (DWI): desde el preprocesamiento hasta la segmentación de fascículos de sustancia blanca y la extracción de métricas de difusión por fascículo.

Todo el software necesario (FSL, MRtrix3, DSI Studio, Python) está incluido en la imagen Docker. No se requiere instalación adicional.

---

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) instalado
- (Opcional) GPU NVIDIA con [nvidia-container-toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) para acelerar la corrección de Eddy

### Agregar tu usuario al grupo Docker (evita usar `sudo`)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Construcción de la Imagen

Desde el directorio que contiene el `Dockerfile`:

```bash
docker build -t dwi_pipeline .
```

> La imagen incluye FSL (~5 GB), MRtrix3 y DSI Studio. La construcción puede tardar 20–40 minutos dependiendo de la conexión.

---

## Estructura de Datos de Entrada

El pipeline espera que los datos estén organizados en formato BIDS. La imagen T1w puede estar en cualquier ubicación (se monta como volumen adicional):

```
/ruta/a/mis/datos/
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

> La imagen T1w debe tener el cráneo extraído. Puede provenir de cualquier fuente: carpeta `anat/`, FreeSurfer, etc.

---

## Uso

### Ejecución básica

```bash
docker run --rm \
    -u $(id -u):$(id -g) \
    -w /tmp \
    -v /ruta/a/mis/datos:/data \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /data/sub-13/ses-1/anat/T1w_brain.nii.gz
```

### Con GPU (recomendado — acelera la corrección de Eddy)

```bash
docker run --rm --gpus all \
    -e OMP_NUM_THREADS=$(nproc) \
    -u $(id -u):$(id -g) \
    -w /tmp \
    -v /ruta/a/mis/datos:/data \
    dwi_pipeline \
    -s sub-13/ses-1 \
    --dwi_pa sub-13_ses-1_dir-PA_dwi \
    --dwi_ap sub-13_ses-1_dir-AP_dwi \
    --t1w /data/sub-13/ses-1/anat/T1w_brain.nii.gz
```

### Con T1w de FreeSurfer (montar directorio adicional)

```bash
docker run --rm --gpus all \
    -e OMP_NUM_THREADS=$(nproc) \
    -u $(id -u):$(id -g) \
    -w /tmp \
    -v /ruta/a/mis/datos:/data \
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

> **Nota sobre `-w`, `--working_dir`:** el directorio raíz es `/data` y está configurado automáticamente por el contenedor — no es necesario especificarlo.

---

## Opciones de Docker relevantes

| Opción | Descripción |
|---|---|
| `--gpus all` | Habilita la GPU para `eddy_cuda` (5–10× más rápido que CPU) |
| `-e OMP_NUM_THREADS=$(nproc)` | Usa todos los núcleos disponibles para `eddy_cpu` |
| `-u $(id -u):$(id -g)` | Los archivos de salida tendrán tu usuario como propietario (sin ícono de candado) |
| `-w /tmp` | MRtrix3 escribe archivos temporales aquí (necesario al usar `-u`) |
| `-v /mis/datos:/data` | Monta tu directorio de datos dentro del contenedor |
| `--rm` | Elimina el contenedor automáticamente al terminar |

---

## Salidas

Los resultados se guardan en `/ruta/a/mis/datos/sub-XX/ses-Y/Tractography/`:

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
│   ├── FA.nii.gz                       # Anisotropía Fraccional (GQI)
│   ├── FA_dti.nii.gz                   # Anisotropía Fraccional (DTI tensor)
│   ├── QA.nii.gz                       # Quantitative Anisotropy
│   ├── QIR.nii.gz                      # Quantitative Isotropic Ratio
│   ├── RD.nii.gz                       # Difusividad Radial (GQI)
│   ├── RD_dti.nii.gz                   # Difusividad Radial (DTI tensor)
│   ├── AD.nii.gz                       # Difusividad Axial (DTI tensor)
│   ├── MD.nii.gz                       # Difusividad Media (DTI tensor)
│   ├── ISO.nii.gz                      # Componente Isotrópica
│   └── RDI.nii.gz                      # Restricted Diffusion Imaging
└── bundles/
    ├── results_folder_DWM_DWI_space/   # Fascículos DWM en espacio DWI (.tck)
    ├── results_folder_SWM_DWI_space/   # Fascículos SWM en espacio DWI (.tck)
    ├── bundle_metrics_DWM.csv          # Métricas por fascículo DWM
    └── bundle_metrics_SWM.csv          # Métricas por fascículo SWM
```

### Columnas de los CSV de métricas

| Columna | Descripción |
|---|---|
| `bundle` | Nombre del fascículo |
| `metrics` | Métrica (FA, QA, AD, RD, MD, etc.) |
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
    -e OMP_NUM_THREADS=$(nproc) \
    -u $(id -u):$(id -g) \
    -w /tmp \
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

---

## Solución de Problemas

| Problema | Causa | Solución |
|---|---|---|
| Archivos con ícono de candado | Container ejecutado como root | Usar `-u $(id -u):$(id -g)` y sin `sudo` |
| `Permission denied` en `/opt/pipeline` | MRtrix3 escribe temporales en el WORKDIR | Agregar `-w /tmp` |
| `eddy_cuda` falla → usa CPU | Sin GPU o sin `--gpus all` | Agregar `--gpus all` si hay GPU NVIDIA |
| `dsi_studio: command not found` | PATH incorrecto en la imagen | Reconstruir la imagen con `docker build` |

---

## Autores

Sebastián Navarrete — sebastian.navarrete@biomedica.udec.cl
