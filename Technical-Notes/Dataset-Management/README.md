# 💾 Dataset Management - Gestion Automatisée des Données

> Système complet de gestion des datasets pour FeatherFaceV2 avec téléchargement automatique et validation

---

## 📋 Vue d'ensemble

Le système de gestion des datasets assure une pipeline robuste et automatisée pour tous les aspects des données : téléchargement, preprocessing, validation, versioning et backup. Conçu pour supporter WIDERFace comme dataset principal avec extensibilité pour datasets académiques additionnels.

---

## 🎯 Datasets Supportés

### 📊 Dataset Principal : WIDERFace
| Aspect | Détails |
|--------|---------|
| **Type** | Face Detection Benchmark |
| **Images** | ~32K images |
| **Annotations** | ~400K face bounding boxes |
| **Subsets** | Easy (51%), Medium (33%), Hard (16%) |
| **Size** | ~3.8GB (images + annotations) |
| **URL** | [WIDERFace Official](http://shuoyang1213.me/WIDERFACE/) |
| **Usage** | Training (40%), Validation (10%), Test (50%) |

#### Structure WIDERFace
```
data/widerface/
├── WIDER_train/           # Training images (12,880 images)
│   └── images/           # Organized by event folders
├── WIDER_val/            # Validation images (3,226 images)  
│   └── images/           # Organized by event folders
├── WIDER_test/           # Test images (16,097 images)
│   └── images/           # Organized by event folders
├── wider_face_split/     # Train/Val annotations
│   ├── wider_face_train_bbx_gt.txt
│   └── wider_face_val_bbx_gt.txt
└── eval_tools/           # Official evaluation scripts
    ├── ground_truth/     # Ground truth for evaluation
    └── evaluation.py     # Official evaluation protocol
```

### 📚 Datasets Académiques Additionnels
- **FDDB :** Face Detection Data Set and Benchmark (2,845 images)
- **AFW :** Annotated Faces in the Wild (205 images) 
- **PASCAL FACE :** PASCAL VOC subset with face annotations
- **Custom Academic :** Données spécifiques environnement académique

---

## 🔄 Automated Download System

### 📥 Script Principal de Téléchargement
```python
# data/scripts/download_datasets.py
import os
import requests
import zipfile
from pathlib import Path
from tqdm import tqdm

class DatasetDownloader:
    """
    Automated dataset downloader with progress tracking
    """
    def __init__(self, data_root="data/"):
        self.data_root = Path(data_root)
        self.datasets_config = {
            "widerface": {
                "urls": {
                    "train": "https://huggingface.co/datasets/wider_face/...",
                    "val": "https://huggingface.co/datasets/wider_face/...",
                    "test": "https://huggingface.co/datasets/wider_face/...",
                    "annotations": "http://shuoyang1213.me/WIDERFACE/..."
                },
                "checksums": {
                    "train": "sha256_hash_here",
                    "val": "sha256_hash_here"
                }
            }
        }
    
    def download_widerface(self, force_redownload=False):
        """Download WIDERFace dataset with verification"""
        widerface_dir = self.data_root / "widerface"
        
        if self._is_dataset_complete("widerface") and not force_redownload:
            print("✅ WIDERFace already downloaded and verified")
            return
            
        print("📥 Downloading WIDERFace dataset...")
        self._download_with_progress("widerface", "train") 
        self._download_with_progress("widerface", "val")
        self._download_with_progress("widerface", "annotations")
        
        self._extract_and_organize()
        self._verify_dataset_integrity()
        print("✅ WIDERFace download completed successfully")
```

### 🔧 Auto-Download Integration
```python
# training/train.py - Integration automatique
def setup_datasets():
    """Setup datasets avec download automatique si nécessaire"""
    downloader = DatasetDownloader()
    
    # Check si WIDERFace existe et est complet
    if not downloader.is_dataset_available("widerface"):
        print("🔄 WIDERFace not found, downloading automatically...")
        downloader.download_widerface()
    
    # Validation integrity
    if not downloader.verify_dataset("widerface"):
        print("⚠️ Dataset integrity check failed, re-downloading...")
        downloader.download_widerface(force_redownload=True)
    
    return downloader.get_dataset_paths("widerface")

# Usage dans scripts training
if __name__ == "__main__":
    dataset_paths = setup_datasets()  # Auto-download si nécessaire
    # Continue avec training...
```

---

## 🔄 Preprocessing Pipeline

### 📊 Pipeline Configuration
```yaml
# data/configs/preprocessing_config.yaml
preprocessing:
  widerface:
    # Image preprocessing
    target_size: [640, 640]
    normalize:
      mean: [0.485, 0.456, 0.406]
      std: [0.229, 0.224, 0.225]
    
    # Augmentations training
    augmentations:
      - random_flip: {prob: 0.5}
      - random_crop: {prob: 0.3, min_scale: 0.8}
      - color_jitter: {brightness: 0.2, contrast: 0.2}
      - random_rotation: {prob: 0.1, max_angle: 15}
    
    # Quality filtering
    min_face_size: 16  # pixels
    min_visibility: 0.3
    max_occlusion: 0.7
    
    # Output format
    annotation_format: "coco"  # ou "yolo", "pascal_voc"
    save_format: "jpg"
    quality: 95
```

### 🛠️ Preprocessing Engine
```python
# data/scripts/preprocess_widerface.py
import albumentations as A
from albumentations.pytorch import ToTensorV2

class WIDERFacePreprocessor:
    """
    Preprocessing pipeline pour WIDERFace avec augmentations
    """
    def __init__(self, config_path="data/configs/preprocessing_config.yaml"):
        self.config = self._load_config(config_path)
        self.train_transform = self._create_train_transforms()
        self.val_transform = self._create_val_transforms()
    
    def _create_train_transforms(self):
        """Transformations pour training avec augmentations"""
        return A.Compose([
            A.Resize(640, 640),
            A.HorizontalFlip(p=0.5),
            A.RandomBrightnessContrast(p=0.2),
            A.HueSaturationValue(p=0.1),
            A.Normalize(
                mean=[0.485, 0.456, 0.406], 
                std=[0.229, 0.224, 0.225]
            ),
            ToTensorV2()
        ], bbox_params=A.BboxParams(
            format='pascal_voc',
            label_fields=['category_ids']
        ))
    
    def process_dataset_split(self, split="train"):
        """Process un split complet du dataset"""
        print(f"🔄 Processing {split} split...")
        
        # Load annotations
        annotations = self._load_annotations(split)
        
        # Process chaque image
        processed_count = 0
        for image_path, bboxes in tqdm(annotations.items()):
            try:
                processed_data = self._process_single_image(
                    image_path, bboxes, split
                )
                if processed_data:
                    processed_count += 1
            except Exception as e:
                print(f"❌ Error processing {image_path}: {e}")
        
        print(f"✅ Processed {processed_count} images for {split}")
```

---

## 📊 Data Versioning & Backup

### 🔄 Versioning Strategy
```python
# data/scripts/data_versioning.py
import hashlib
import json
from datetime import datetime

class DatasetVersionManager:
    """
    Versioning system pour datasets avec integrity checking
    """
    def __init__(self, data_root="data/"):
        self.data_root = Path(data_root)
        self.version_file = self.data_root / "dataset_versions.json"
        self.versions = self._load_versions()
    
    def create_version_snapshot(self, dataset_name, description=""):
        """Créer snapshot version du dataset"""
        timestamp = datetime.now().isoformat()
        version_id = f"{dataset_name}_v{len(self.versions.get(dataset_name, [])) + 1}"
        
        # Calculate checksums pour tous les fichiers
        dataset_path = self.data_root / dataset_name
        file_checksums = {}
        
        for file_path in dataset_path.rglob("*"):
            if file_path.is_file():
                with open(file_path, 'rb') as f:
                    checksum = hashlib.sha256(f.read()).hexdigest()
                file_checksums[str(file_path.relative_to(dataset_path))] = checksum
        
        # Version metadata
        version_data = {
            "version_id": version_id,
            "timestamp": timestamp,
            "description": description,
            "file_count": len(file_checksums),
            "total_size": self._calculate_total_size(dataset_path),
            "checksums": file_checksums
        }
        
        # Save version
        if dataset_name not in self.versions:
            self.versions[dataset_name] = []
        self.versions[dataset_name].append(version_data)
        self._save_versions()
        
        print(f"✅ Created version {version_id} for {dataset_name}")
        return version_id
```

### 💾 Backup Strategy
```yaml
# data/configs/backup_config.yaml
backup:
  strategy: "incremental"  # full, incremental, differential
  schedule: "daily"        # daily, weekly, on_change
  retention: 
    daily: 7     # Keep 7 daily backups
    weekly: 4    # Keep 4 weekly backups  
    monthly: 6   # Keep 6 monthly backups
  
  destinations:
    - type: "local"
      path: "E:/backup/datasets/"
      enabled: true
    - type: "cloud" 
      provider: "google_drive"
      enabled: false
    - type: "network"
      path: "//server/shared/datasets/"
      enabled: false
  
  compression: true
  encryption: false
  verify_integrity: true
```

---

## 🔍 Validation & Quality Control

### ✅ Dataset Validation Pipeline
```python
# data/scripts/validate_datasets.py
class DatasetValidator:
    """
    Comprehensive dataset validation et quality control
    """
    def __init__(self):
        self.validation_rules = {
            "image_format": ["jpg", "jpeg", "png"],
            "min_image_size": (32, 32),
            "max_image_size": (4096, 4096),
            "min_face_size": 16,
            "annotation_format": "pascal_voc"
        }
    
    def validate_widerface(self, thorough=True):
        """Validation complète WIDERFace dataset"""
        print("🔍 Validating WIDERFace dataset...")
        
        validation_report = {
            "timestamp": datetime.now().isoformat(),
            "total_images": 0,
            "valid_images": 0,
            "invalid_images": [],
            "total_annotations": 0,
            "invalid_annotations": [],
            "quality_metrics": {}
        }
        
        # Validate structure
        self._validate_directory_structure()
        
        # Validate images et annotations
        for split in ["train", "val", "test"]:
            split_report = self._validate_split(split, thorough)
            validation_report.update(split_report)
        
        # Generate quality metrics
        validation_report["quality_metrics"] = self._calculate_quality_metrics()
        
        # Save validation report
        self._save_validation_report(validation_report)
        
        return validation_report["valid_images"] / validation_report["total_images"] > 0.95
    
    def _validate_single_image(self, image_path, annotations):
        """Validation d'une image avec ses annotations"""
        try:
            # Load et check image
            img = cv2.imread(str(image_path))
            if img is None:
                return False, "Cannot load image"
            
            h, w = img.shape[:2]
            if w < 32 or h < 32:
                return False, "Image too small"
            
            # Validate annotations
            for bbox in annotations:
                x, y, w_bbox, h_bbox = bbox[:4]
                if w_bbox < self.validation_rules["min_face_size"]:
                    return False, f"Face too small: {w_bbox}px"
                if x + w_bbox > w or y + h_bbox > h:
                    return False, "Bbox outside image bounds"
            
            return True, "Valid"
            
        except Exception as e:
            return False, f"Validation error: {str(e)}"
```

---

## 📊 Monitoring & Analytics

### 📈 Dataset Analytics Dashboard
```python
# data/scripts/dataset_analytics.py
def generate_dataset_analytics(dataset_name="widerface"):
    """Generate comprehensive analytics pour dataset"""
    
    analytics = {
        "overview": {
            "total_images": 0,
            "total_faces": 0,
            "splits": {"train": 0, "val": 0, "test": 0}
        },
        "face_size_distribution": {},
        "aspect_ratio_distribution": {},
        "difficulty_distribution": {},
        "quality_metrics": {}
    }
    
    # Analyze chaque split
    for split in ["train", "val", "test"]:
        split_analytics = analyze_split(split)
        analytics["overview"]["splits"][split] = split_analytics["image_count"]
        # Merge other metrics...
    
    # Generate visualizations
    create_analytics_plots(analytics)
    
    return analytics

def create_analytics_plots(analytics):
    """Create visualizations pour dataset analytics"""
    import matplotlib.pyplot as plt
    import seaborn as sns
    
    # Face size distribution
    plt.figure(figsize=(12, 8))
    plt.subplot(2, 2, 1)
    # Plot face size histogram...
    
    # Save plots
    plt.savefig("data/analytics/dataset_analytics.png", dpi=300)
```

---

## 🔗 Integration Points

### 📚 Connexions Académiques
- **[[02-Literature-Review/Face-Detection-Approaches/|Approaches]]** ↔ Dataset benchmarking standards
- **[[03-Methodology/Validation-Methodology|Validation]]** ↔ Dataset validation protocols
- **[[04-Implementation/Experimental-Setup/|Setup]]** ↔ Data preprocessing pipeline
- **[[05-Results/Performance-Analysis/|Performance]]** ↔ Dataset analytics et metrics

### 🧪 Workflow Expérimentation
1. **Auto-download :** Scripts détectent datasets manquants
2. **Preprocessing :** Pipeline automatically appliqué selon config
3. **Validation :** Quality checks avant utilisation training
4. **Versioning :** Snapshots pour reproducibilité experiments
5. **Analytics :** Insights pour optimisation architecture

---

*Dataset Management v1.0 | Auto-download: WIDERFace | Preprocessing: Configurable | Validation: Comprehensive*