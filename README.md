# RawCalFormer

**RawCalFormer: Reconstruction-Gradient-Guided Multi-View Calibration Transformer for Low-Dose CT Restoration**

RawCalFormer is a low-dose CT restoration framework that combines **full-detector multi-view restoration (MVR)** with **reconstruction-gradient calibration (RGC)**. Multi-view information is first exploited to recover complementary structural information from detector-domain observations, while reconstruction-gradient cues are subsequently used to calibrate the restored projections. The calibrated projections are then reconstructed and fused to obtain the final low-dose CT restoration result.

The framework additionally incorporates protocol-conditioned training data synthesis to support restoration under different CT acquisition settings.

Code, configuration files, and pretrained weights have not yet been released. The workflow below illustrates the overall framework and planned interface.

![RawCalFormer framework](figs/framework.png)

## 1. Environment Setup

Reference environment:

- Linux
- Python 3.9
- PyTorch 2.5.1
- CUDA 12.1
- CTorch 1.0

Create the environment:

```bash
conda create -n rawcalformer python=3.9 -y
conda activate rawcalformer
python -m pip install torch==2.5.1 --index-url https://download.pytorch.org/whl/cu121
python -m pip install -r requirements.txt
```

Obtain the CT simulation and reconstruction backend from [AIAI-CTorch](https://github.com/JHU-AIAI-Shared/AIAI-CTorch) and install it using a compatible CUDA Toolkit and C++ compiler:

```bash
python -m pip install --no-build-isolation /path/to/AIAI-CTorch
```

## 2. Data Preparation

RawCalFormer supports protocol-specific simulation and restoration settings. Select the corresponding configuration file from `configs/`, for example:

```text
siemens.yaml
ge.yaml
philips.yaml
united_imaging.yaml
```

Before running the pipeline, verify the acquisition parameters and specify the input and output paths.

Input CT images are expected to be **3D NPY volumes in Hounsfield units (HU)** with dimensions ordered as:

```text
[z, y, x]
```

Voxel spacing, orientation, and grid dimensions should be verified before simulation. Each configuration preset defines the corresponding reference acquisition and reconstruction protocol.

Edit the data and runtime fields in the selected configuration while retaining the remaining protocol-specific parameters:

```yaml
data:
  root: /path/to/ndct
  cases:
    - {case_id: case_001, patient_id: patient_001, ndct: case_001.npy, split: test}

run:
  output: ../runs/siemens
  device: cuda:0
  doses: [0.10]                   # 10% dose
  projection_cache: clean
```

Paths specified by `ndct` are relative to `data.root`, whereas other relative paths are resolved with respect to the YAML configuration file.

For training and evaluation, assign cases to `train`, `val`, and `test` subsets while ensuring that all scans from the same patient remain within the same split.

## 3. Low-Dose CT Simulation

Run the protocol-specific simulation using:

```bash
python simulate.py --config configs/siemens.yaml
```

The simulation pipeline performs the following operations:

1. Forward projection of the reference NDCT volume.
2. Protocol-conditioned low-dose noise simulation.
3. Reconstruction of the corresponding LDCT volume.
4. Generation of projection and slice-mapping metadata required by RawCalFormer.

An example simulated LDCT volume is saved as:

```text
runs/siemens/ldct/dose10/case_001.npy
```

Projection caching is controlled through:

```yaml
projection_cache: clean
```

Available modes include:

- `clean`: cache clean projections for subsequent training or testing.
- `none`: save only reconstructed LDCT volumes and associated metadata.
- `noisy`: cache the generated noisy projections.
- `both`: retain both clean and noisy projections.

Projection data are stored under:

```text
run.output/projections
```

A custom projection directory can be specified through `run.projection_dir` when required.

The simulator records the noise realization associated with each LDCT volume. During testing, the corresponding noisy projections can therefore be regenerated from the cached clean projections within the same validated numerical environment.

When exact projection inputs must be preserved across different computational environments, caching the noisy projections is recommended.

## 4. RawCalFormer Training

RawCalFormer contains two major restoration stages:

1. **Multi-View Restoration (MVR)** for recovering complementary structural information from full-detector multi-view observations.
2. **Reconstruction-Gradient Calibration (RGC)** for refining the restored projections using reconstruction-domain gradient supervision.

Assign the training cases to the `train` split and perform simulation with clean projection caching before model training.

Run the two training stages sequentially:

```bash
python fd.py train --config configs/siemens.yaml
python pf.py train --config configs/siemens.yaml
```

The first stage trains the multi-view restoration component. The second stage loads the corresponding restoration checkpoint, constructs reconstruction-gradient supervision targets, and trains the calibration component.

Model checkpoints are stored under:

```text
run.output/checkpoints
```

The original NDCT volumes should be retained because they are required for the construction of reference reconstruction and gradient-supervision targets.

Training iterations, batch size, learning rate, dose settings, and other optimization parameters are specified in the corresponding YAML configuration.

To reproduce protocol-specific and cross-protocol experiments, use the same case partitions and acquisition settings as defined for the corresponding experiment.

## 5. Testing and Inference

After preparing the required clean or noisy projection cache, run:

```bash
python test.py --config configs/siemens.yaml
```

The testing pipeline loads the paired RawCalFormer checkpoints associated with the current experiment and performs:

1. Multi-view restoration.
2. Reconstruction-gradient calibration.
3. Matched reconstruction.
4. Slice fusion.
5. Quantitative evaluation.

When pretrained weights are provided, they can be used in place of locally trained checkpoints.

By default, testing restores the selected cases and reports the following image-quality metrics:

- PSNR
- SSIM
- MAE

For reconstruction only, use:

```bash
python test.py --config configs/siemens.yaml --mode infer
```

To evaluate previously generated restoration results, use:

```bash
python test.py --config configs/siemens.yaml --mode evaluate
```

Reference NDCT images are required only for quantitative evaluation. Inference itself uses the projection inputs and the corresponding acquisition and reconstruction metadata.

## 6. Output Structure

A typical output directory is organized as follows:

```text
runs/siemens/
├── ldct/
│   └── dose10/
│       └── case_001.npy
├── projections/
│   └── clean/
│       └── case_001/
├── teacher/
├── checkpoints/
│   ├── fd.pt
│   └── pf.pt
└── rawcalformer/
    └── dose10/
        ├── case_001.npy
        └── metrics/
```

Here, `dose10` represents a 10% dose setting, and restored volumes are named according to their case IDs.

The output slices are automatically aligned with the corresponding NDCT reference volume, retaining only the valid reconstruction range. Noise settings, reconstruction parameters, and slice mappings are recorded as metadata for subsequent restoration and evaluation.

Evaluation excludes only exterior air regions. SSIM is computed directly in Hounsfield units, and CSV files store the corresponding raw metric values.

## 7. Repository Status

The current repository provides the framework overview and planned usage interface for RawCalFormer.

The following resources will be released upon completion of code organization:

- Source code
- Configuration files
- Pretrained model weights
- Training and inference scripts
- Reproduction instructions

Please refer to this repository for future updates.
