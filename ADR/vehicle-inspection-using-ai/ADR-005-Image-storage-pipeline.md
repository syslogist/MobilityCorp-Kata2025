# ADR 005: Image Storage, Processing Pipeline, and Data Residency

**Status:** Proposed  
**Date:** 2025-10-22  
**Deciders:** Architecture Team, Data Privacy Officer  
**Related Issue/Story:** AI-powered vehicle return inspection system

## Context and Problem Statement

The inspection system captures 3 images per vehicle (interior, front, rear) and must:

1. **Store images securely** with EU data residency compliance
2. **Process images** (resize, crop, quality checks) before AI inference
3. **Manage image lifecycle** (retention, deletion for GDPR compliance, archival)
4. **Ensure fast retrieval** for dashboard and reports
5. **Audit image access** for compliance and dispute resolution

**Key Requirements:**
- EU data residency (GDPR: images cannot leave EU)
- High availability (99.9%+)
- Cost-efficient storage (potentially 10K+ images/month)
- Fast access (<100ms retrieval for dashboard)
- Compliance: images linked to specific inspections and user accounts
- Image retention: keep for 1-2 years for dispute resolution

**Question:** Which storage solution (AWS S3, Azure Blob, on-premises NAS) and how to structure the image processing pipeline?

**Constraints:**
- Budget constraints; storage costs add up quickly
- EU infrastructure only (AWS not suitable due to data sovereignty)
- Team comfort with self-hosted or EU cloud services

## Decision Factors

- **Data Residency:** AWS S3 US default region violates GDPR; Azure Blob EU region compliant
- **Cost:** On-premises NAS requires capex; cloud storage requires opex
- **Performance:** Cloud CDN faster than self-hosted; but edge cases with Kubernetes services
- **Complexity:** Cloud storage simpler to operate; on-premises requires backup/disaster recovery
- **Scalability:** Cloud storage infinitely scalable; on-premises requires planning
- **Integration:** Native Kubernetes support for cloud storage via operators
- **Retention:** Cloud storage fine-grained lifecycle policies; on-premises requires custom scripts

## Alternatives

### Option A: AWS S3 (EU Region: eu-central-1)
- **Pros:**
  - Industry standard; reliable and performant
  - Lifecycle policies for automatic archival/deletion
  - S3 Replication for multi-region disaster recovery
  - CloudFront CDN for fast image retrieval
  - Cost-effective at scale ($0.023/GB/month in EU region)
- **Cons:**
  - **Critical Data Residency Issue:** AWS US-based company; GDPR concerns despite EU region
  - Vendor lock-in to AWS ecosystem
  - Transfer costs for data egress
  - Complex IAM policies and bucket security configurations
  - Requires monitoring and alerts for compliance

### Option B: Azure Blob Storage (EU West: Netherlands or Germany)
- **Pros:**
  - EU-based infrastructure with strong data residency guarantees
  - Integrated with Kubernetes via Azure Disk/Blob CSI driver
  - Managed backup and geo-redundancy options
  - Fine-grained access controls and audit logging
  - Compliance certifications (ISO, SOC 2)
  - Predictable cost model
- **Cons:**
  - Slightly higher cost than S3 (~$0.018/GB/month)
  - Requires Azure subscription and potentially other Azure services
  - Less mature ecosystem than AWS S3 (fewer third-party tools)

### Option C: On-Premises NAS (Network Attached Storage)
- **Pros:**
  - Complete data control; images never leave premises
  - No cloud vendor dependency
  - Lower operational costs over 5+ years vs. cloud
  - Full compliance with EU data residency
  - Direct integration with Kubernetes via NFS/iSCSI PersistentVolumes
- **Cons:**
  - High initial capital expense (€50K-100K for enterprise NAS)
  - Requires DBA/storage admin expertise
  - Backup and disaster recovery responsibility
  - Limited scalability; adds storage incrementally
  - Power, cooling, physical space requirements
  - No built-in global redundancy

### Option D: EU-Based Alternative Cloud (OVH, Hetzner, Scaleway)
- **Pros:**
  - Strong EU data residency commitment
  - Competitive pricing (sometimes cheaper than AWS/Azure)
  - Strong privacy policies
  - Good Kubernetes integration
- **Cons:**
  - Smaller ecosystem and fewer tools
  - Less mature products than AWS/Azure
  - Support quality may be lower
  - Smaller global footprint if business expands beyond EU

### Option E: Hybrid Approach (On-Premises Hot + Cloud Archive)
- **Pros:**
  - Recent images on fast NAS (< 3 months: hot tier)
  - Older images archived to cloud or cold storage (cheaper)
  - Balances performance and compliance
  - Reduces on-premises storage costs
- **Cons:**
  - Operational complexity (managing two tiers, data migration)
  - Compliance overhead (track where images reside)
  - Slower retrieval for archived images

## Decision Outcome

**We will adopt Azure Blob Storage EU West (Netherlands) as primary with on-premises NAS backup (Option B + Element of Option E):**

### Image Storage Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Edge Devices (Rental Zones)                   │
│        (Cameras + Jetson Orin / Coral Dev Board)                 │
└────────────────────┬────────────────────────────────────────────┘
                     │ (Upload over HTTPS)
┌────────────────────▼────────────────────────────────────────────┐
│           Kubernetes Cluster (EU Region)                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Image Upload Service (Ingress Load Balancer)            │   │
│  │  - Validates image format & metadata                    │   │
│  │  - Tags with inspection_id, VIN, timestamp              │   │
│  └──────────────┬───────────────────────────────────────────┘   │
│                 │                                                 │
│  ┌──────────────▼───────────────────────────────────────────┐   │
│  │ Image Processing Pipeline (Kubernetes Jobs)             │   │
│  │  - Resize to 640×640 px                                 │   │
│  │  - Crop background using OpenCV                         │   │
│  │  - Compress (JPEG quality 85)                           │   │
│  │  - Generate thumbnail (200×200 px)                      │   │
│  │  - Calculate image hash for dedup                       │   │
│  └──────────────┬───────────────────────────────────────────┘   │
│                 │                                                 │
│  ┌──────────────▼───────────────────────────────────────────┐   │
│  │ Store to: Azure Blob Storage (EU West - Hot Tier)       │   │
│  │ Store to: PostgreSQL metadata + image URIs              │   │
│  │ Store to: NAS Backup (async replication)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Configuration

**Azure Blob Storage (Primary):**
- **Account:** creation-images (east-eu or west-eu region)
- **Containers:**
  - `raw-uploads/` – Original images from edge devices
  - `processed/` – Resized & cropped images for AI inference
  - `thumbnails/` – Small previews for dashboard
- **Lifecycle Management:**
  ```yaml
  # Move to cool tier after 30 days (save cost)
  rules:
    - name: archive-old-images
      condition:
        daysAfterModificationGreaterThan: 30
      action:
        tierToCool: true
    - name: delete-after-2years
      condition:
        daysAfterModificationGreaterThan: 730
      action:
        delete: true
  ```
- **Access Control:**
  - Managed Identity for Kubernetes pods (no hardcoded keys)
  - Role-Based Access Control (RBAC): inspectors read-only, service writers
  - SAS tokens for temporary access (inspection results to users)
- **Monitoring:**
  - Audit logging enabled (who accessed which image, when)
  - Prometheus metrics: upload latency, storage cost, quota usage
- **Cost:** ~€1,500/month for 10K images (1GB avg), 2-year retention

**On-Premises NAS Backup:**
- **Hardware:** Synology DS1821+ or equivalent (8-bay, 64GB RAM)
- **Configuration:** RAID 6 (tolerates 2 disk failures)
- **Replication:** Async nightly snapshot to Blob Storage via rsync over VPN
- **Purpose:** 
  - Compliance backup (prove we didn't lose data)
  - Disaster recovery (if cloud access disrupted)
  - Local fast access for dispute investigations
- **Capacity:** 80TB (enough for 2 years of images at 5K vehicles, 10K/month inspections)
- **Cost:** ~€8K capex + €2K/year maintenance

### Image Processing Pipeline

**Kubernetes Job (triggered on each upload):**

```python
# image-processor.py (runs in Kubernetes pod)
import cv2
import hashlib
from azure.storage.blob import BlobClient

def process_inspection_images(inspection_id, raw_image_uris):
    """
    Process raw images: resize, crop, compress, and upload to Azure.
    """
    for image_uri in raw_image_uris:
        # Download raw image
        raw_image = download_from_blob(image_uri)
        
        # Quality checks
        if not is_sharp(raw_image):
            flag_for_manual_review(inspection_id, "blurry_image")
            continue
        
        # Preprocessing
        processed = preprocess_image(raw_image)
        # - Resize to 640×640 (YOLO input size)
        # - Auto-orient based on EXIF
        # - Normalize lighting
        
        # Crop (remove background)
        cropped = crop_background_opencv(processed)
        
        # Compress
        _, compressed = cv2.imencode('.jpg', cropped, [cv2.IMWRITE_JPEG_QUALITY, 85])
        
        # Generate thumbnail
        thumbnail = cv2.resize(processed, (200, 200))
        _, thumb_compressed = cv2.imencode('.jpg', thumbnail, [cv2.IMWRITE_JPEG_QUALITY, 75])
        
        # Calculate hash (for dedup)
        image_hash = hashlib.sha256(compressed).hexdigest()
        
        # Upload to Azure Blob
        blob_name_processed = f"processed/{inspection_id}/{image_uri.split('/')[-1]}"
        upload_to_blob(blob_name_processed, compressed)
        
        blob_name_thumb = f"thumbnails/{inspection_id}/{image_uri.split('/')[-1]}"
        upload_to_blob(blob_name_thumb, thumb_compressed)
        
        # Record in PostgreSQL
        db.execute("""
            INSERT INTO image_metadata (inspection_id, image_hash, blob_uri, status)
            VALUES (%s, %s, %s, 'processed')
        """, (inspection_id, image_hash, blob_name_processed))
    
    # Trigger AI inference
    trigger_yolov8_inference(inspection_id)

def is_sharp(image):
    """Check image sharpness using Laplacian variance."""
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    laplacian_var = cv2.Laplacian(gray, cv2.CV_64F).var()
    return laplacian_var > 100  # Threshold for "sharp enough"

def crop_background_opencv(image):
    """Remove background using contour detection (car-centric)."""
    # Simplified approach: find largest contour (the car) and crop to bounding rect
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    _, thresh = cv2.threshold(gray, 100, 255, cv2.THRESH_BINARY)
    contours, _ = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    if contours:
        largest = max(contours, key=cv2.contourArea)
        x, y, w, h = cv2.boundingRect(largest)
        return image[max(0, y-20):min(image.shape[0], y+h+20), 
                     max(0, x-20):min(image.shape[1], x+w+20)]
    return image
```

### GDPR Compliance Measures

- **Data Retention:** Lifecycle policy automatically deletes images after 2 years
- **Right to Erasure:** API endpoint to delete all images for a specific inspection/user
- **Audit Logging:** All access logged to Azure activity log and PostgreSQL audit table
- **Data Portability:** Export images for specific inspection in standardized format
- **Purpose Limitation:** Images only used for damage assessment; not sold or shared

### Rationale

1. **Azure Blob EU West:** Strong GDPR guarantees, managed infrastructure, integrated Kubernetes support
2. **On-Premises NAS:** Compliance backup, disaster recovery, local fast access for disputes
3. **Hybrid approach:** Balances cost (hot tier in Azure), performance (NAS for local access), compliance
4. **Image processing pipeline:** Standardizes image preparation before AI (improves YOLO accuracy)
5. **Lifecycle policies:** Automatic cleanup meets GDPR data minimization principle

## Cost Estimation (Annual)

- Azure Blob Storage (1TB/month avg): €1,500/month = €18,000/year
- NAS backup (depreciation + maintenance): €2,000/year
- **Total:** ~€20,000/year

## Links / References

- ADR-002: Database Selection (image URIs stored in PostgreSQL)
- ADR-003: Damage Detection Model (processed images fed to YOLOv8)
- ADR-004: Kubernetes Deployment (Job and Pod configuration)
- External: Azure Blob Storage documentation, OpenCV image processing, GDPR compliance checklist
- Tools: Azure CLI for management, Kubernetes CSI drivers for pod access
