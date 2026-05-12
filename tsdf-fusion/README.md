This README provides instructions for 3D semantic reconstruction and evaluation using TSDF (Truncated Signed Distance Function) fusion. Here's a breakdown:





# TSDF Fusion for 3D Semantic Reconstruction

>This folder provides tools to reconstruct 3D scenes from depth images using **TSDF (Truncated Signed Distance Function) fusion**, and evaluate the quality of semantic/label predictions.

## Directory Structure

```
tsdf-fusion/
├── fusion.py              # Base TSDF fusion module (no color integration)
├── fusion2.py             # TSDF fusion with RGB color integration
├── fusion3.py             # TSDF fusion variant for 15-dim semantic features
├── dim3_recon.py          # 3D reconstruction from depth + semantic class colors
├── dim3_recon_gt.py       # Ground truth 3D reconstruction
├── dim15_recon.py         # 15-dimensional semantic reconstruction (LangSlam)
├── save_semantic_colors_gt.py  # Convert semantic labels to colorized images
├── 3d_evaluation_and_visualize_langslam_dim15.py  # LangSlam evaluation
├── 3d_evaluation_and_visualize_langsplat.py       # LangSplat evaluation
├── emd.py                 # Earth Mover's Distance module wrapper
└── PytorchEMD/            # CUDA implementation of EMD
    └── setup.py           # Compilation script for EMD module
```

---

## Pipeline Dependencies

There are **two separate evaluation pipelines** depending on whether you're evaluating LangSplat or LangSlam results.

### Pipeline A: LangSplat Evaluation

```
┌─────────────────────────────────────────────────────────────┐
│ 1. save_semantic_colors_gt.py                               │
│    Input:  semantic_class/*.png (raw labels)                │
│    Output: semantic_color/*.png + color_code.npy            │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. dim3_recon.py                                           │
│    Input:  depth/*, traj_w_c.txt, semantic_color/*          │
│    Output: GT_semantic_pc.ply, GT_semantic_mesh.ply         │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 3d_evaluation_and_visualize_langsplat.py                │
│    Input:  GT_semantic_pc.ply, semantic_pc.ply (from main   │
│            langsplat pipeline), autoencoder checkpoint      │
│    Output: per-class meshes in 3d_mesh/                     │
└─────────────────────────────────────────────────────────────┘
```

### Pipeline B: LangSlam Evaluation

```
┌─────────────────────────────────────────────────────────────┐
│ 1. dim15_recon.py                                           │
│    Input:  depth/*, traj_w_c.txt, LangSlam/*.npy           │
│    Output: semantic_pc.ply, semantic_mesh.ply               │
└─────────────────────────────────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 3d_evaluation_and_visualize_langslam_dim15.py           │
│    Input:  semantic_pc.ply, GT_semantic_pc.ply,             │
│            autoencoder checkpoint, online checkpoint         │
│    Output: per-class meshes in 3d_mesh/                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Path Requirements by Script

### 1. `save_semantic_colors_gt.py` *(Pipeline A - Step 1)*

Converts raw semantic class labels (integer IDs) into colorized images for ground truth visualization.

| Variable | Description | Example |
|----------|-------------|---------|
| `sem_path` | Path to folder containing raw semantic class PNG images (`semantic_class_*.png`) | `/data/replica_v2/room_0/imap/00/semantic_class` |
| `sem_save` | Output directory for colorized semantic images | `/data/replica_v2/room_0/imap/00/semantic_color` |

**Input:** Raw semantic label images (`semantic_class_*.png`)
**Output:** Colorized images (`semantic_class_*.png`) + `color_code.npy`

---

### 2. `dim3_recon.py` *(Pipeline A - Step 2)*

Reconstructs a 3D mesh from depth images and semantic class colors. Uses **fusion2**.

| Variable | Description | Example |
|----------|-------------|---------|
| `prefix` | Replica v2 dataset root containing `depth/` folder and `traj_w_c.txt` | `/data/replica_v2/room_0/imap/00` |
| `color_prefix` | Path to colorized semantic class images (from Step 1) | `/data/replica_v2/room_0/imap/00/semantic_color` |
| `save_path` | Output directory for reconstructed mesh/point cloud | `/output/room_0/results` |

**Input:** Depth images (`depth/depth_*.png`), camera poses (`traj_w_c.txt`), semantic colors
**Output:** `GT_semantic_mesh.ply`, `GT_semantic_mesh_color.ply`, `GT_semantic_pc.ply`

---

### 3. `3d_evaluation_and_visualize_langsplat.py` *(Pipeline A - Step 3)*

Evaluates LangSplat predictions using EMD and Chamfer distance.

| Variable | Description | Example |
|----------|-------------|---------|
| `path` | Replica v2 `render_config.yaml` | `/datasets/replica_v2/room_2/imap/00/render_config.yaml` |
| `load_path` | LangSplat results directory | `/langsplat_results/room2` |
| `pc_save_path` | Output directory for per-class meshes | `{load_path}/3d_mesh` |
| `ae_ckpt_path` | Autoencoder checkpoint (`.pth`) | `/langsplat_results/room2/ae_ckpt/best_ckpt.pth` |
| `color_mat` | GT color code numpy array (from Step 1) | `/datasets/replica_v2/room_2/imap/00/color_code.npy` |
| `gt_pcd` | Ground truth point cloud (from Step 2) | `{load_path}/GT_semantic_pc.ply` |

**Requires:** `GT_semantic_pc.ply` from `dim3_recon.py` + `semantic_pc.ply` (from main langsplat pipeline)
**Output:** Per-class meshes in `pc_save_path/`

---

### 4. `dim15_recon.py` *(Pipeline B - Step 1)*

15-dimensional semantic reconstruction for LangSlam results. Uses **fusion3**.

| Variable | Description | Example |
|----------|-------------|---------|
| `prefix` | Replica v2 dataset root with `depth/` and `traj_w_c.txt` | `/datasets/Replica2/vmap/room_0_small/imap/00` |
| `color_prefix` | Path to LangSlam 15-dim semantic predictions (`.npy` files) | `/results/room_0/2025-03-24/psnr/before_opt/lang` |

**Input:** Depth images, LangSlam semantic feature files (`{i}.npy`)
**Output:** `semantic_mesh.ply`, `semantic_mesh_color.ply`, `semantic_pc.ply` (saved relative to `color_prefix/..`)

---

### 5. `3d_evaluation_and_visualize_langslam_dim15.py` *(Pipeline B - Step 2)*

Evaluates LangSlam 15-dim predictions using EMD and Chamfer distance.

| Variable | Description | Example |
|----------|-------------|---------|
| `path` | Replica v2 `render_config.yaml` for class definitions | `/datasets/Replica2/vmap/room_0/imap/00/render_config.yaml` |
| `load_path` | LangSlam results directory | `/results/room_0_small/2025-03-24/psnr/before_opt` |
| `pc_save_path` | Output directory for per-class meshes | `{load_path}/3d_mesh` |
| `ae_ckpt_path` | Autoencoder checkpoint (`.ckpt`) | `/training_weights/ae_149_he.ckpt` |
| `online_ckpt` | Online encoder checkpoint (`.pth`) | `{load_path}/online_15_room0.pth` |
| `color_mat` | GT color code numpy array | `/datasets/Replica2/vmap/room_0/imap/00/color_code.npy` |
| `gt_pcd` | Ground truth point cloud (`.ply`) | `{load_path}/GT_semantic_pc.ply` |

**Requires:** `semantic_pc.ply` from `dim15_recon.py` (Step 1) + `GT_semantic_pc.ply` (from Pipeline A)
**Output:** Per-class meshes in `pc_save_path/`

---

### Additional Script: `dim3_recon_gt.py`

An alternative ground truth reconstruction script. Uses **fusion** (no color integration variant).

| Variable | Description | Example |
|----------|-------------|---------|
| `prefix` | Replica v2 dataset root with `depth/` and `traj_w_c.txt` | `/datasets/Replica2/vmap/room_0/imap/00` |
| `color_prefix` | Path to ground truth semantic color images | `/datasets/Replica2/vmap/room_0/imap/00/semantic_color` |
| `save_path` | Output directory for GT mesh/point cloud | `/results/room_0/before_opt` |

**Input:** Depth images, GT semantic colors, camera poses
**Output:** `GT_semantic_mesh.ply`, `GT_semantic_pc.ply`

---

## Data Directory Structure

The pipeline expects the following structure for Replica v2 data:

```
replica_v2/
└── {scene_name}/
    └── imap/
        └── 00/
            ├── depth/
            │   ├── depth_0.png
            │   ├── depth_1.png
            │   └── ...          # 16-bit PNG, depth in millimeters
            ├── semantic_class/
            │   ├── semantic_class_0.png   # Raw semantic labels (integer IDs)
            │   └── ...
            ├── semantic_color/           # Colorized semantic (auto-generated by Step 1)
            │   ├── semantic_class_0.png
            │   └── color_code.npy        # Color mapping array
            ├── traj_w_c.txt              # Camera poses (N×4×4 matrices)
            └── render_config.yaml        # Scene class definitions
```

---

## Prerequisites

### Data Requirements
- **Replica v2 dataset** downloaded from imap
  - Depth images are stored in **millimeters (mm)**
  - If using a different version of Replica or another dataset, verify the depth scale in your data before proceeding
- **LangSlam or LangSplat results** (pre-computed 3D predictions from the main pipeline)

### Software Requirements
- Python 3.8+
- PyTorch
- NumPy
- OpenCV (for image processing)
- A C++ compiler (for building the PytorchEMD module)

---

## Quick Start

> **Important:** Before running any script, open the file and update all path variables to match your local directory structure. Each script contains comments indicating which paths need to be modified.

### LangSplat Evaluation Pipeline

```bash
# Step 1: Colorize semantic labels
python3 save_semantic_colors_gt.py

# Step 2: Reconstruct ground truth
python3 dim3_recon.py

# Step 3: Evaluate
cd PytorchEMD && python3 setup.py install && cd ..
# Copy emd_cuda*.so to tsdf-fusion/ directory
python3 3d_evaluation_and_visualize_langsplat.py
```

### LangSlam Evaluation Pipeline

```bash
# Step 1: Reconstruct LangSlam predictions
python3 dim15_recon.py

# Step 2: Evaluate
cd PytorchEMD && python3 setup.py install && cd ..
# Copy emd_cuda*.so to tsdf-fusion/ directory
python3 3d_evaluation_and_visualize_langslam_dim15.py
```

---

## What is TSDF Fusion?

**TSDF** (Truncated Signed Distance Function) is a volumetric representation used for 3D reconstruction. Each voxel in a volume stores:

- A **signed distance** to the nearest surface (negative = inside, positive = outside)
- The distance is **truncated** to a small band around surfaces for efficiency

By integrating multiple depth images from different viewpoints, TSDF fusion produces a clean 3D reconstruction of the scene geometry.
