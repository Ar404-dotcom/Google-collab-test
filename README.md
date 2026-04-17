# StegoImageDS Malware Detector

This project is built specifically for the Kaggle `Stego-Images-Dataset`:
[https://www.kaggle.com/datasets/marcozuppelli/stegoimagesdataset/data](https://www.kaggle.com/datasets/marcozuppelli/stegoimagesdataset/data)

It is shaped around the dataset described in the associated 2022 paper by Cassavia et al.:

- 512x512 icons
- 8,000 clean images
- 5 stego payload families: `html`, `javascript`, `powershell`, `ethereum`, `urlip`
- extra `stego_b64` and `stego_zip` sets for robustness testing

## Why this model is stronger than a plain CNN

The pipeline combines multiple signals that matter for this exact dataset:

1. `RGB branch`
   Learns visible priors and coarse artifacts from the icon.
2. `SRM/high-pass residual branch`
   Highlights subtle noise patterns caused by bit manipulation.
3. `Bit-plane branch`
   Reads the lowest `k` bit-planes directly, which is highly relevant for LSB steganography.
4. `Payload-byte branch`
   Reconstructs bytes from the first LSB stream and uses a byte CNN to learn dataset-specific payload signatures.
5. `Statistical feature branch`
   Adds handcrafted steganalysis and payload-style features:
   - LSB occupancy
   - transition ratios
   - chi-square pair statistics
   - entropy
   - RS-like block features
   - printable/base64/hex ratios
   - HTML/JS/PowerShell/URL/Ethereum token features
6. `Multi-task learning`
   The model jointly predicts:
   - binary stego detection
   - payload family classification
   - optional variant (`plain`, `b64`, `zip`)

## Project Layout

```text
configs/stego_multibranch.yaml   Training config
requirements.txt                 Python dependencies
src/stegomalware/dataset.py      Dataset indexing + caching
src/stegomalware/features.py     LSB extraction + stego features
src/stegomalware/model.py        Multi-branch neural architecture
src/stegomalware/losses.py       Focal / multitask losses
src/stegomalware/engine.py       Training and evaluation loops
src/stegomalware/train.py        Main training entrypoint
src/stegomalware/evaluate.py     Evaluation script
src/stegomalware/predict.py      Single image inference
```

## Install

```bash
pip install -r requirements.txt
```

## Train

```bash
python -m src.stegomalware.train --config configs/stego_multibranch.yaml
```

## Google Colab Workflow

Use Google Drive for the output directory so checkpoints survive session disconnects:

```python
from google.colab import drive
drive.mount("/content/drive")
```

Install and train:

```bash
pip install -r requirements.txt
python -m src.stegomalware.train \
  --config configs/stego_multibranch.yaml \
  --output-dir /content/drive/MyDrive/stego_runs/run1
```

Important checkpoint files:

- `.../checkpoints/latest.pt`: resume from the most recent completed epoch
- `.../checkpoints/best.pt`: best validation model so far
- `.../checkpoints/epoch_XXX.pt`: periodic snapshots you can upload into another account
- `.../checkpoints/interrupted.pt`: written if training is manually interrupted

Resume on another Colab account after uploading or remounting Drive:

```bash
python -m src.stegomalware.train \
  --config configs/stego_multibranch.yaml \
  --output-dir /content/drive/MyDrive/stego_runs/run1 \
  --resume /content/drive/MyDrive/stego_runs/run1/checkpoints/latest.pt
```

The current config is already tuned for Colab-style GPUs:

- smaller micro-batch with `accumulation_steps` to preserve effective batch size
- mixed precision enabled
- channels-last tensors on CUDA
- persistent checkpoints every epoch
- low worker count to avoid DataLoader instability in small Colab runtimes

## Evaluate

```bash
python -m src.stegomalware.evaluate \
  --config configs/stego_multibranch.yaml \
  --checkpoint outputs/stego_multibranch/best.pt
```

## Predict One Image

```bash
python -m src.stegomalware.predict \
  --config configs/stego_multibranch.yaml \
  --checkpoint outputs/stego_multibranch/best.pt \
  --image path/to/image.png
```

## Important Notes

- Do not apply JPEG compression, blur, color jitter, or arbitrary resizing before bit extraction.
- Horizontal and vertical flips are safe enough for augmentation; heavy photometric augmentation is not.
- For high accuracy, keep `stego_b64` and `stego_zip` as evaluation-only at first, then optionally fine-tune later for robustness.
- If you want the strongest final score, train an ensemble of 3 to 5 seeds and average predictions at inference.

## Sources Used

- Kaggle dataset page: [Stego-Images-Dataset](https://www.kaggle.com/datasets/marcozuppelli/stegoimagesdataset/data)
- Paper:
  [Detection of Steganographic Threats Targeting Digital Images in Heterogeneous Ecosystems Through Machine Learning](https://jowua.com/wp-content/uploads/2022/12/jowua-v13n3-4.pdf)

From the paper, the key dataset/model details used here are:

- 512x512 icons
- 8,000 clean images and 36,000 stego images overall
- payload classes: HTML, JavaScript, PowerShell, Ethereum wallets, URL/IP
- extra Base64 and ZIP-obfuscated evaluation sets
- the published baseline processes extracted `k`-LSB data with dense subnet stacks

This repository extends that idea into a richer steganalysis + payload-aware architecture.
# Google-collab-malware-test
