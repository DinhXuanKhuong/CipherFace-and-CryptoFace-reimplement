# CipherFace and CryptoFace reimplement

This repository contains our course project for **Privacy Protection and Face
Recognition**. The project studies privacy-preserving face recognition, starting
from the Blind Vision paradigm and then reimplementing / experimenting with two
recent encrypted face-recognition systems:

- **CryptoFace: End-to-End Encrypted Face Recognition** (CVPR 2025)
- **CipherFace: A Fully Homomorphic Encryption-Driven Framework for Secure
  Cloud-Based Facial Recognition** 

Beyond reproducing the original CipherFace experiments on LFW, we extend the
evaluation to the **Pins Face Recognition (PFR)** dataset and test an improved
FaceNet128d backbone with **CBAM attention**.

## Project Goals

The core question is:

> Can a face-recognition pipeline compare faces while keeping biometric features
> encrypted, and how well does that design generalize to a harder dataset?

To answer it, this project includes:

- A review of privacy-preserving face recognition from Blind Vision to FHE-based
  systems.
- A local copy / reimplementation workflow for CryptoFace training and CKKS FHE
  inference.
- A CipherFace-style encrypted matching pipeline using DeepFace embeddings and
  homomorphic distance computation.
- Experiments on both the original LFW setting and a new PFR setting.
- A FaceNet128d + CBAM improvement experiment for a lighter cleartext backbone.

## Repository Structure

```text
.
├── README.md
├── LICENSE
├── docs/
│   ├── report.pdf          # Full final report
│   └── slide-final.pdf     # Presentation slides
├── experiment/
│   ├── LFW dataset/        # CipherFace reproduction on LFW
│   └── PFR dataset/        # Extended CipherFace experiments on PFR
├── improvement/
│   └── improve_facenet128d.ipynb
└── CryptoFace-main/        # Local CryptoFace implementation copy, ignored by git
```

The `experiment/` notebooks are organized by dataset, face embedding model,
distance metric, and FHE parameter setting. Each notebook name follows:

```text
<model> _ <distance> _ <n exponent>, <q bits>, <scale exponent>.ipynb
```

For example, `FaceNet128d _ Cosine _ 14, 422, 60.ipynb` runs FaceNet128d with
cosine distance under the larger CKKS setting `n = 2^14`, `q = 422`, `g = 2^60`.

## Methods

### CryptoFace

CryptoFace performs end-to-end encrypted face recognition with fully homomorphic
encryption (FHE). Its pipeline includes PyTorch training, CKKS model conversion,
and C++ encrypted inference using Microsoft SEAL. The local `CryptoFace-main/`
folder follows the original CryptoFace structure:

- `train.py`, `config.yaml`: PyTorch training workflow
- `ckks.py`: convert trained weights to CKKS inference files
- `datasets.py`: convert evaluation data for encrypted inference
- `cnn_ckks/`: C++ CKKS inference implementation
- `models/`: patch CNN / face-recognition model code

### CipherFace

CipherFace is a lighter encrypted matching framework. Instead of running the full
neural network inside FHE, it extracts face embeddings locally and performs
encrypted distance computation in the cloud-style setting.

We evaluate:

- Models: **FaceNet128d**, **FaceNet512d**, **VGG-Face**
- Distances: **Euclidean**, **Cosine**
- FHE library: **TenSEAL / CKKS**
- Security-oriented HE settings:
  - Small: `n = 2^13`, `q = 200`, `g = 2^40`
  - Large: `n = 2^14`, `q = 422`, `g = 2^60`

### FaceNet128d + CBAM

We also test a modified FaceNet128d architecture with CBAM attention. The goal is
to reduce parameters while improving feature quality before eventually moving the
backbone into an encrypted pipeline.

## Datasets

- **LFW (Labeled Faces in the Wild)**: standard face verification benchmark used
  by the original CipherFace paper.
- **PFR (Pins Face Recognition)**: an extended dataset used by our group to stress
  test generalization. It contains 17,534 face images from 105 identities and has
  larger variation in pose, lighting, expression, and image quality.

## Main Results

### CipherFace on LFW

Our reproduction keeps recognition quality close to the expected CipherFace
behavior: encrypted computation preserves the decision quality of the original
embedding comparison.

| Model | Best / typical LFW accuracy | AUC |
| --- | ---: | ---: |
| FaceNet128d | 0.92 - 0.96 | 0.99 |
| FaceNet512d | 0.94 - 0.97 | 0.99 - 1.00 |
| VGG-Face | 0.95 | 0.99 - 1.00 |

Runtime follows the expected FHE pattern: the large CKKS setting is much slower
than the small one, while encryption and decryption remain in the millisecond
range.

### CipherFace on PFR

PFR is harder than LFW. The encrypted distance computation still works, but the
fixed thresholds tuned for LFW become too strict on noisier, more diverse data.

| Model | PFR accuracy range | AUC range | Observation |
| --- | ---: | ---: | --- |
| FaceNet128d | 0.65 - 0.88 | 0.98 | Cosine with large HE setting has very low recall |
| FaceNet512d | 0.79 - 0.91 | 0.98 - 0.99 | More stable than 128d |
| VGG-Face | 0.88 - 0.91 | 0.97 | Stronger on PFR under fixed thresholds |

The most important finding is not that FHE fails, but that **static thresholds do
not transfer well** from LFW to PFR. For example, FaceNet128d + Cosine under the
large setting reaches precision near `1.00`, but recall drops to `0.30`, meaning
the system rejects many true matches because the threshold is too conservative.

### FaceNet128d + CBAM

| Model | Params | Accuracy | Precision | Recall | F1 |
| --- | ---: | ---: | ---: | ---: | ---: |
| FaceNet128d baseline | 27.98M | 0.96 | 0.95 | 0.93 | 0.96 |
| FaceNet128d + CBAM | 23.20M | 0.97 | 0.97 | 0.96 | 0.95 |

The CBAM variant reduces parameters by about 17% while improving accuracy,
precision, and recall in our cleartext experiment.

## How to Run

### 1. Read the report

The complete methodology, tables, discussion, and references are in:

```text
docs/report.pdf
```

### 2. Run the CipherFace notebooks

Open the notebooks under:

```text
experiment/
```

Recommended Python packages:

```bash
pip install deepface tenseal numpy pandas scikit-learn matplotlib seaborn
```

The notebooks are grouped by:

- Dataset: `LFW dataset/` or `PFR dataset/`
- Model: `FaceNet128d`, `FaceNet512d`, `VGG-Face`
- Distance: `Cosine`, `Euclidean`
- HE setting: `(2^13, 200, 2^40)` or `(2^14, 422, 2^60)`

### 3. Run the FaceNet128d improvement notebook

Open:

```text
improvement/improve_facenet128d.ipynb
```

Additional packages used by the improvement notebook include:

```bash
pip install facenet-pytorch torch torchvision numpy scipy scikit-learn
```

### 4. Run CryptoFace

The `CryptoFace-main/` folder follows the original CryptoFace workflow. A minimal
setup is:

```bash
conda create -n cryptoface python=3.9
conda activate cryptoface
pip install -U scikit-learn mxnet wandb tqdm einops multipledispatch
pip install numpy==1.23.1
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
```

Training:

```bash
cd CryptoFace-main
python train.py config.yaml
```

For encrypted CKKS inference, build the C++ code under:

```bash
CryptoFace-main/cnn_ckks/
```

The original CryptoFace implementation requires GMP, NTL, OpenMP, Microsoft
SEAL, model weights, and converted evaluation datasets.

## Notes

- `CryptoFace-main/` is currently ignored by `.gitignore`, so it may exist only
  in the local workspace unless explicitly added or documented as an external
  dependency.
- Large generated dependencies such as GMP and NTL archives / build folders are
  intentionally ignored.
- Dataset folder names contain spaces, so quote notebook paths when opening them
  from a shell.

## References

1. Wei Ao and Vishnu Naresh Boddeti. **CryptoFace: End-to-End Encrypted Face
   Recognition**. CVPR 2025.
2. Sefik Ilkin Serengil and Alper Ozpinar. **CipherFace: A Fully Homomorphic
   Encryption-Driven Framework for Secure Cloud-Based Facial Recognition**. 2025.
3. Shai Avidan and Moshe Butman. **Blind Vision**. ECCV 2006.
4. Sanghyun Woo et al. **CBAM: Convolutional Block Attention Module**. ECCV 2018.

## License

This repository is released under the MIT License. See [LICENSE](LICENSE).
