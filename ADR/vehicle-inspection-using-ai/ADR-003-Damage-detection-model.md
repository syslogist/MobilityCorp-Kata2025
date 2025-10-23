# ADR 003: Damage Detection AI Model - Pre-trained vs. Custom Training

**Status:** Proposed  
**Date:** 2025-10-22  
**Deciders:** ML Lead, Architecture Team  
**Related Issue/Story:** AI-powered vehicle return inspection system

## Context and Problem Statement

The core AI functionality must accurately detect vehicle damage (scratches, dents, bumper damage) from photos captured during return inspections. The system requires:

1. **Object detection** for damage localization (YOLO family, Faster R-CNN)
2. **Image segmentation** for internal damage assessment (tears, stains, broken trim)
3. **Damage classification** (type: scratch, dent, tear; severity: minor, moderate, severe)

Key question: Should the system use existing pre-trained models (YOLOv8, Detectron2) potentially fine-tuned on automotive damage, or invest in training a custom model from scratch?

**Constraints:**
- Real-time inference required (< 500ms per vehicle inspection, 3 images per inspection)
- 5,000 vehicles across European zones; need consistent damage detection
- Budget constraints for model training infrastructure
- Need to detect: scratches (>5cm), dents (visible), major cosmetic damage
- Cannot rely on perfect accuracy; human review for edge cases

## Decision Factors

- **Accuracy:** Pre-trained models on ImageNet/COCO datasets may not generalize to vehicle damage (domain shift problem)
- **Data Availability:** Do we have labeled automotive damage dataset (expensive to create)?
- **Time to Market:** Pre-trained models deployable in weeks; custom training takes months
- **Infrastructure Cost:** Pre-trained requires only inference hardware (GPU); training requires GPU cluster
- **Maintenance:** Pre-trained models stable; custom models require ongoing retraining as damage patterns evolve
- **Regulatory:** Insurance/rental industry may prefer explainable, validated models over black boxes
- **Performance:** Custom models can optimize for specific damage types (rental fleet context)

## Alternatives

### Option A: Use Pre-trained YOLOv8 Object Detection Model
- **Pros:**
  - Available immediately; excellent pre-training on general objects
  - Fast inference (~50-100ms per image on GPU)
  - Well-documented and extensively used in production
  - Can be deployed in Kubernetes containers easily
  - Community-driven updates and improvements
  - Lower inference GPU requirements (runs on edge devices)
- **Cons:**
  - General object detection not optimized for vehicle damage
  - May struggle with small scratches or hard-to-see damage
  - Requires heavy post-processing (grouping detections, filtering false positives)
  - Cannot easily explain why damage was detected (black box)

### Option B: Fine-tune Pre-trained Model on Labeled Automotive Damage Dataset
- **Pros:**
  - Combines benefits: transfer learning speeds training, custom data improves accuracy
  - Better generalization to rental vehicle context
  - Moderate effort: fewer labeled images needed than training from scratch
  - Can reach 85-90% accuracy with 500-1000 labeled images
  - Retains explainability through attention maps
- **Cons:**
  - Requires creating labeled dataset (500-1000 damage photos + annotations)
  - Annotation cost (~$2-5 per image, or 2-4 weeks engineer time)
  - Still GPU resources for training (~24-48 hours on V100)
  - Model staleness; needs periodic retraining as new damage types emerge

### Option C: Train Custom Deep Learning Model from Scratch
- **Pros:**
  - Optimized specifically for rental vehicle damage patterns
  - Can achieve 92-95% accuracy with sufficient training data
  - Full control over model architecture and inference speed
  - Can optimize for false negative vs. false positive trade-off
- **Cons:**
  - Requires 5,000-10,000 labeled automotive damage images (expensive: $20K-30K)
  - Requires ML expertise and GPU infrastructure for months
  - High maintenance burden and model drift over time
  - Not justified for rental fleet scale (overkill)

### Option D: Hybrid + Third-Party AI Service (Tractable, Inspektlabs, Ravin.AI)
- **Pros:**
  - Proven accuracy on automotive damage (95%+)
  - Legal protection (service provider liability for false assessments)
  - No model maintenance burden
  - Can focus on integration rather than ML pipeline
  - Turnkey solution ready for deployment
- **Cons:**
  - Per-inspection cost (~$0.50-2.00 per image analysis)
  - Vendor dependency; service can change pricing or features
  - Images sent to third-party (privacy concerns)
  - Limited customization for rental-specific damage types
  - May be overkill for rental fleet (designed for insurance claims)

### Option E: Lightweight Open-Source Model + Human-in-the-Loop System
- **Pros:**
  - Use smaller YOLOv5/v8 nano model for initial detection (fast, runs on edge)
  - Flag ambiguous cases for human review (inspector or AI team)
  - Improves accuracy over time through active learning
  - Low infrastructure cost initially
  - Data stays in-house
- **Cons:**
  - Requires human review queue management
  - Slower customer experience for edge cases
  - Still need to label images for retraining loop

## Decision Outcome

**We will adopt Option B: Fine-tune YOLOv8 on automotive damage dataset with human validation:**

1. **Model Selection:** YOLOv8 Medium (40M parameters) pre-trained on COCO
2. **Fine-tuning Approach:**
   - Collect 500-800 labeled images of actual rental vehicle damage (scratches, dents, damage from fleet)
   - Annotate using CVAT or similar tool (bounding boxes for damage regions)
   - Fine-tune YOLOv8 for 50-100 epochs on automotive damage dataset
   - Target metrics: 85%+ precision, 80%+ recall for damage detection
3. **Inference Pipeline:**
   - YOLOv8 runs on Kubernetes GPU pods (inference latency: 100-150ms per image)
   - Confidence threshold set to 0.6 for detection; below 0.7 flagged for human review
   - Output: bounding boxes + confidence scores + damage location
4. **Fallback:** Ambiguous cases (confidence 0.6-0.7) reviewed by rental inspector or escalated to GPT-4o vision API
5. **Rationale:**
   - Balances accuracy, cost, and time to market
   - Transfer learning leverages YOLOv8's pre-training (weeks to deployment, not months)
   - Fine-tuning on real rental fleet data improves domain-specific accuracy
   - Human review loop catches edge cases and improves model via active learning
   - Low infrastructure cost compared to training from scratch
   - Maintainable by DevOps team without PhD-level ML expertise

## Implementation Timeline

- **Week 1:** Collect and annotate 500 damage images
- **Week 2:** Fine-tune YOLOv8 and validate accuracy
- **Week 3:** Integrate into Kubernetes inference pipeline
- **Week 4:** Deploy beta to one rental zone with human review

## Future Iterations

- If accuracy plateaus below 80%, escalate to custom training or third-party service
- Periodically retrain (quarterly) as new damage types emerge
- Track false positives/negatives in production; use for active learning

## Links / References

- ADR-001: LLM Model Selection (LLM processes YOLOv8 outputs into natural language)
- ADR-004: Kubernetes Deployment (GPU pod configuration for inference)
- ADR-005: Data Pipeline Architecture (image preprocessing before YOLOv8)
- External: YOLOv8 documentation, transfer learning best practices, CVAT annotation tool
- Dataset sources: Open Images, automotive damage datasets on Kaggle
