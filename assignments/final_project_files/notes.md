This folder will contain our presentation, and any additional files or data needed for our final project.
Below are notes on our work.

# Clay Foundation Model - Project Notes

https://clay-foundation.github.io/model/  
v1.5

## What It Is

Open source foundation model trained on 70M satellite image chips. 
Takes imagery + location + timestamp.
Outputs 1024-dimensional embeddings. 
Trained via self-supervised Masked Autoencoder (MAE).

## Architecture

- Vision Transformer (ViT), 632M params total
- Encoder outputs 1024-dim embeddings
- Input chip size: **256x256 pixels**
- Accepts any bands/wavelengths via metadata config
- Position encoding captures lat/lon, timestamp, and ground sampling distance (GSD)

## Relevant Sensors for This Project

- NAIP: 4 bands (RGB + NIR), 0.6–1m res
- Sentinel-2: 10 bands, 10m res

**We will use both NAIP and Sentinel-2.**
NAIP at 0.6m provides spatial resolution to detect block-level signals — roof condition, vegetation overgrowth, lot maintenance. Sentinel-2 contributes richer spectral information (10 bands including red-edge and SWIR) that may better capture surface material changes.
Clay's architecture means we can generate embeddings from each independently and compare their utility.

## Spatial Unit

At NAIP 0.6m resolution, a 256x256 chip covers ~150m x 150m — roughly a city block. **Clay operates at the block level, not the parcel level.** Rather than centering chips on individual parcels, we tile Detroit into a regular 256x256 grid. Each tile gets one embedding per time period. This is an appropriate planning unit — the DLBA needs to know which blocks to prioritize for inspection, not which exact parcel.

## Installation + Usage

```bash
pip install git+https://github.com/Clay-foundation/model.git
wget https://huggingface.co/made-with-clay/Clay/resolve/main/v1.5/clay-v1.5.ckpt
```

```python
import yaml, torch
from claymodel.module import ClayMAEModule

model = ClayMAEModule.load_from_checkpoint("clay-v1.5.ckpt")
model.eval()

with open("configs/metadata.yaml", "r") as f:
    metadata = yaml.safe_load(f)

sensor = "naip"

# Wavelengths (μm → nm)
wavelengths = torch.tensor([[metadata[sensor]["bands"]["wavelength"][b] * 1000
                              for b in metadata[sensor]["band_order"]]], dtype=torch.float32)

# Normalize
means = torch.tensor([metadata[sensor]["bands"]["mean"][b]
                      for b in metadata[sensor]["band_order"]]).view(1, -1, 1, 1)
stds  = torch.tensor([metadata[sensor]["bands"]["std"][b]
                      for b in metadata[sensor]["band_order"]]).view(1, -1, 1, 1)

chips = (raw_chips - means) / stds  # shape: (batch, 4, 256, 256)
timestamps = torch.tensor([[0, 0, lat, lon]], dtype=torch.float32)

with torch.no_grad():
    embeddings = model.encoder(chips, timestamps, wavelengths)
# embeddings.shape → (batch, 1024)
```

## Project Workflow

1. Pull NAIP imagery for Detroit, two epochs (~2017 and ~2023)
2. Tile into 256x256 chips on a regular grid
3. Run Clay encoder → 1024-dim embedding per chip per epoch
4. Compute embedding difference (epoch 2 − epoch 1) per chip
5. Cluster change vectors → interpret against DLBA demolition + permit records
6. Compare cluster quality against AlphaEarth embeddings

## Limitations/Considerations

- At most 6 time steps per location during training — not a constraint for our 2-epoch approach
- Block-level spatial unit may miss fine-grained parcel variation

---

# AlphaEarth Foundations — Project Notes

## What It Is

Geospatial embedding model by Google DeepMind. Trained on annual time-series of multi-sensor Earth observation data.
Outputs **64-dimensional embeddings per pixel**, precomputed and available directly in Google Earth Engine. No model to run.

Key difference from Clay: embeddings are already computed at 10m resolution globally. You load them like a dataset. 
Also pixel-level rather than chip-level — finer spatial granularity for parcel or block analysis.

## Output

- **64-band image** per year — each pixel is a 64-dim embedding vector
- Each band (A00–A63) is one axis of the embedding space — use all 64 together
- Embedding vectors are unit-normalized (Euclidean length = 1)
- **Resolution:** 10m | **Coverage:** 2017–2025, annual, global land surface

## Training Data Sources

Fuses multiple sensors into a single unified representation:
- Sentinel-2 (optical, 10m)
- Landsat 8/9 (optical, 30m)
- Sentinel-1 SAR (radar, 10m)
- Elevation models, climate data, and more

Embeddings are robust to clouds, scan lines, and missing data because of multi-source fusion.


## Project Workflow

1. Load AlphaEarth embeddings for Detroit for two years (~2017 and ~2023)
2. Sample/aggregate embedding vectors within spatial units (pixels, blocks, or parcels)
3. Compute difference vector (2023 − 2017) per unit
4. Cluster change vectors → interpret against DLBA demolition + permit records
5. Compare cluster quality against Clay embeddings

## Key Differences vs. Clay

| | AlphaEarth | Clay |
|---|---|---|
| Embeddings | Precomputed, load from GEE | Must run model on imagery |
| Dimensions | 64 | 1024 |
| Spatial unit | Per pixel (10m) | Per chip (~150m x 150m on NAIP) |
| Access | GEE image collection | pip + HuggingFace weights |
| Input sensors | Multi-sensor fused | Single sensor per run |

## Limitations/Considerations

- 10m resolution: ~1–4 pixels per Detroit parcel — may need to aggregate to block level
- Annual summaries: no control over acquisition season
- 64 dimensions vs. Clay's 1024 — less details but sufficient for clustering
- Raw COG access outside GEE requires de-quantization and use of provided `.vrt` files