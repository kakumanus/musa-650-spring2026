# Final Project Proposal - MUSA 6500

**Joey Cahill, Sujan Kakumanu, Angel Rutherford**  
MUSA 6500 - Spring 2026

---

## Problem Definition & Use Case

Cities, particularly those in economically distressed regions such as the U.S. Rust Belt, face challenges in monitoring neighborhood change at scale. The City of Detroit provides a clear example. In 2014, the city partnered with multiple organizations to conduct a comprehensive parcel-level survey of nearly 380,000 properties. This effort relied heavily on crowdsourced photo collection, and produced a detailed snapshot of property conditions, which informed demolition strategies targeting vacant and/or blighted structures. While effective, this process required substantial time, labor, and financial resources, and its reliance on periodic resurveying limits its ability to track ongoing neighborhood change in a timely manner.

This project proposes a parcel-level change classification approach using satellite imagery to monitor changes in the built environment over time. Rather than replacing detailed parcel-level surveys, this approach aims to provide a consistent, repeatable method for identifying areas of potential change across the entire city, enabling more efficient targeting of limited public resources.

The primary users of this project will be the City of Detroit and Detroit Land Bank Authority (DLBA) officials. These stakeholders are responsible for prioritizing property inspections, allocating limited reinvestment resources, and determining appropriate interventions across neighborhoods experiencing varying degrees of stability, decline, and recovery.

To support them, the primary output of this project is a spatial change detection layer comparing two time periods, categorizing areas of the city into three groups: stable areas with minimal observable change, declining areas exhibiting increasing signs of distress, and improving areas exhibiting increasing signs of reinvestment. These categories are derived from changes in observable physical characteristics such as vegetation, surface materials, and building patterns, and do not directly measure vacancy or property condition. Instead, they serve as proxies for underlying neighborhood dynamics.

---

## Technical Justification

This project focuses on neighborhood change by detecting shifts in the observable physical characteristics of the built environment over time. Rather than directly identifying vacancy or property conditions, the model will be designed to learn patterns in satellite imagery that reflect changes in surface materials, vegetation, and structures. These patterns are derived from both spectral and spatial features present in this imagery.

At a spectral level, the model captures variation across bands associated with vegetation, impervious surfaces, and roofing materials. For example, increases in vegetation intensity might reflect either unmanaged lots or intentional planting. Decreases in impervious surface signals may suggest deterioration of driveways or removal of built structures. At a spatial level, the model will capture textural and structural patterns such as whether roofs have holes, parcel delineation such as where fencing or other structures have failed and blurred property lines, or the general absence of consistent built form. Changes in these features over time such as increasing lot irregularity serve as indicators of shifts in the built environment.

A "positive" prediction in this context does not correspond to a specific condition such as vacancy or disrepair, but rather to a measurable change in these learned features between two time periods. These changes are then interpreted as potential signals of neighborhood decline, reinvestment, or stability.

Given the absence of consistent, high-quality labels for property condition or reinvestment, this project will use an unsupervised learning approach. While parcel-level data from the DLBA provides indicators such as vacancy or demolition status, these data sources wind up being temporally inconsistent and could reflect properties that have undergone significant transition already. As a result, they will not be used for model training. Instead, unsupervised methods allow the model to learn underlying structure in the imagery without imposing categories, making it possible to identify patterns of changes that may not be captured in existing datasets. After that, data from the city and DLBA will be used as an external reference for interpretation and validation of detected patterns.

Change detection is the project's primary task because the underlying planning question is temporal: the goal is not to classify the condition of a location at a single point in time (the parcel surveys achieve that), but to understand how that condition is evolving over time. Alternatives are less appropriate given this important distinction. For example, a supervised classification approach would require ground truth labels which as mentioned previously are not available. Segmentation requires clearly defined classes at the pixel level, which is not the scope that we are interested in. Regression is unsuitable, because there is no continuous target variable representing neighborhood condition. Unsupervised change detection aligns directly with both the available data and exploratory nature of this problem.

It is important to note some limitations. First, many of the signals captured by satellite imagery are ambiguous in interpretation. For example, increased vegetation may mean neglect or it could mean intentional landscaping. Vacant lots may exhibit similar signatures to pop-up parks and urban farms. Changes in surface reflectance may not indicate deterioration, rather new materials with similar properties to damaged ones. Finally, the spatial resolution of the imagery may limit the model's ability to detect fine-grained changes in building condition or structural change.

---

## Methodological Precedent

**Zou, S. & Wang, L. (2020). Individual Vacant House Detection in Very-High-Resolution Remote Sensing Images. *Annals of the American Association of Geographers*, 110(2), 449–461.**

Zou and Wang develop a parcel-level vacant house detection pipeline using multitemporal VHR aerial imagery. Training labels are derived automatically via building change detection using the Morphological Building Index (MBI) and shadow extraction, identifying parcels where a structure disappears between time periods. A random forest classifier is trained on spectral (NDVI, raw bands), textural (GLCM), and geometric (parcel shape) features, with selection via Out-of-Bag error and VIF analysis. The model is evaluated using fivefold cross-validation. The authors note that vacancy is inferred from physical change rather than occupancy status directly, meaning well-maintained vacant properties may go undetected. Additionally, the reliance on commercial VHR imagery limits the scalability and reproducibility of the approach. We borrow their strategy of deriving labels from observed physical change rather than administrative records. Our key adaptations are extending the binary classification to a three-class change typology, and substituting commercial VHR imagery with freely available NAIP and Sentinel-2 data.

**Caye Daudt, R., Le Saux, B., & Boulch, A. (2018). Fully Convolutional Siamese Networks for Change Detection. *IEEE International Conference on Image Processing (ICIP)*, Athens, Greece.**

Daudt et al. propose fully convolutional Siamese architectures for pixel-level change detection using co-registered image pairs, using shared-weight encoders that compute feature differences between two image streams to focus the network directly on what has changed. Networks are trained end-to-end on two open datasets: the multispectral Onera Satellite Change Detection dataset (Sentinel-2) and the RGB Air Change aerial dataset, evaluated using precision, recall, and F1. The authors note that scarcity of annotated change detection datasets limits model complexity, and that their binary changed/unchanged framework does not address semantic change detection, meaning distinguishing the type of change rather than just its presence. We adopt the Siamese shared-weight architecture as the basis for our twin CNN design, extending it to produce a three-class typology rather than a binary mask, directly pursuing the semantic change direction the authors identify as future work.

**Thompson, E.S. & de Beurs, K.M. (2018). Tracking the removal of buildings in Rust Belt cities with open-source geospatial data. *International Journal of Applied Earth Observation and Geoinformation*, 73, 471–481.**

Thompson and de Beurs monitor building removal in Detroit and Youngstown using open-source LiDAR, aerial orthoimagery, and GIS parcel data, finding that 12.9% of Detroit parcels lost a structure between 2009 and 2014 with less than 1% rebuilt. Evaluation is conducted against city demolition records and parcel surveys. The authors note dependence on temporally consistent LiDAR data limits generalizability to cities without such infrastructure. This paper validates our use of administrative demolition records as a reference dataset. We extend their approach by substituting LiDAR with multispectral imagery and expanding from binary removal detection to a three-class change typology.

---

## Data Plan

This project will use a multi-source remote sensing approach combining Landsat, Sentinel, and NAIP data in order to balance temporal depth, spatial resolution, and feature richness.

| Dataset | Source | Spatial Resolution | Temporal Resolution | Spectral Bands |
|---|---|---|---|---|
| Landsat 8 | NASA/USGS | 30m | 16-day revisit, 1984–present | Red, NIR, SWIR, Thermal, Panchromatic |
| Sentinel-2 | Copernicus | 10m | 5-day revisit, 2015–present | RGB, Red-edge, NIR, SWIR |
| NAIP | USDA | 0.6m (2018–present), 1m (2003–2017) | ~2-year refresh | RGB, NIR |
| DLBA Property Data | DLBA | Parcel-level | 2020–present | 59,303 records |
| Commercial Demolitions | City of Detroit Open Data | Parcel-level | 2019–present | 1,264 records |
| Completed Residential Demolitions | City of Detroit Open Data | Parcel-level | 2021–present | 31,500 records |

**Landsat 8** will serve as the primary sensor for long-term change detection, spanning decades with consistent band structures to examine long-term neighborhood transitions.

**Sentinel-2** will serve to detect minute spatial changes at the parcel and house level. Policy makers in Detroit need a model that can capture individually changing lots such as those experiencing demolitions and renovations.

**NAIP** will provide critical validation and interpretation of unsupervised clusters. The ability to distinguish a vacant lot from a maintained green space, for example, will be crucial in categorizing areas as declining.

**DLBA Property Data** consists of vacant lots, abandoned houses, and other foreclosed structures. It will be used in interpretation and validation of detected patterns.

**Commercial and Residential Demolition Data** will serve as reference datasets for interpreting detected changes, ensuring that a detected spectral change corresponds to the intentional removal of a structure rather than sensor noise or unrelated land cover change.

---

## Modeling Approach

### Baseline

As a baseline, this project will implement a random forest classifier using custom spectral change features from Landsat 8 and Sentinel-2 satellite imagery.

Key features will include:

- **NDVI** calculated from red and near-infrared bands (Landsat 8/9: Band 4 (Red), Band 5 (NIR); Sentinel-2: Band 4 (Red), Band 8 (NIR))
- **NDBI** derived from shortwave infrared and near-infrared bands (Landsat 8/9: Band 6 (SWIR1), Band 5 (NIR); Sentinel-2: Band 11 (SWIR), Band 8 (NIR))
- **Raw SWIR reflectance** (Landsat Band 6; Sentinel Band 11) to capture changes in surface materials such as demolition or exposed soil

These features are computed from temporally aligned pre- and post-period composites and aggregated to parcel or grid-cell units. This baseline will establish whether conventional spectral indices alone can detect neighborhood change.

This approach is limited by both data resolution and feature design. Landsat's 30-meter resolution supports consistent temporal comparison but may obscure changes at the parcel level, while Sentinel-2's 10-meter resolution improves spatial detail but does not address the reliance on predefined indices. Spectral features alone cannot fully capture spatial structure, making it difficult to distinguish between changes that are visually similar but functionally different.

### Primary Model

Because of these limitations, this project will use a twin convolutional neural network (CNN) for unsupervised change detection. Paired image patches from two time periods will be passed independently through convolutional encoders with shared weights. Landsat imagery may be incorporated to extend temporal coverage and ensure consistency across longer time horizons. The network learns latent representations from multispectral inputs (visible, NIR, and SWIR bands), and the difference between paired feature vectors forms a measure of change. These embeddings are then clustered to identify patterns corresponding to stable, declining, and improving areas.

High-resolution NAIP imagery (0.6–1m) will not be used as a primary model input but will support validation and interpretation, allowing clusters to be linked to observable conditions such as structural presence, roof quality, and lot maintenance.

---

## Evaluation Strategy

Because this project uses an unsupervised change detection approach, evaluation cannot rely exclusively on traditional supervised metrics such as accuracy against a validated and labeled dataset. Instead, model performance will be assessed through a combination of internal clustering quality, external agreement with reference datasets, and practical usefulness for planning decisions.