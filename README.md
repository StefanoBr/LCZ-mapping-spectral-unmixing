# LCZ-mapping-spectral-unmixing
Geoinformatics Porject from Politecnico of Milan - Using Sentinel-2 for LCZ mapping and material fraction computation through spectral unmixing

This repository contains two distinct but interlinked codes. One is dedicated to the Model training and validation to produce LCZ maps while the other focus on model training and validation to produce class fraction maps with spectral unmixing.

## CODENAME LCZ is structured as follows:
### Training

#### Set-Up
The classification workflow begins by selecting a target Sentinel-2 acquisition date and retrieving the corresponding imagery in Level-2A format. Specifically, data from tiles T32TMR and T32TNR are processed at 20 m resolution, and bands of interest are selected. At this point, the two tiles are separately cloud masked through the use of the SCL band. Each tile is converted from JP2 to GeoTIFF format, and the two tiles are merged into a single mosaic. To restrict the analysis to the area of interest, the mosaic is clipped using a predefined bounding box derived from a reference image.
#### Training Data Preparation
Vector geometries of LCZ training samples are imported and rasterized to match the spatial resolution and extent of the Sentinel-2 mosaic. Each vector polygon carries a class label according to the standard LCZ scheme. A visual inspection of the training areas is carried out to assess spatial coverage and balance among classes. Additionally, the total area per class is calculated to highlight potential class imbalance issues.
#### Feature Construction and Masking
The selected Sentinel-2 bands are read into memory and preprocessed. A binary mask is created to identify valid pixels (non-zero reflectance values), ensuring compatibility with machine learning input requirements. The spectral features are then combined with UCP layers such as Sky View Factor (SVF), tree canopy height, impervious surface fraction, and building height and building surface fraction. All feature layers are aligned and reshaped into a two-dimensional feature matrix.
#### Model Training
The labeled dataset is split into training and validation subsets. A supervised Random Forest classifier is trained using a grid search to tune hyperparameters, including the number of trees, feature selection strategy, and split criterion. Once trained, the classifier is used to predict LCZ classes for all valid pixels in the Sentinel-2 image.
The predicted class map is reshaped and exported as a single-band GeoTIFF. A median filter is applied post-classification to smooth the output and reduce speckle effects.

### Validation and Testing

Before initiating validation, the classified LCZ map is loaded alongside a rasterized version of an independent set of reference testing points. The testing set follows the same LCZ nomenclature and spatial resolution as the classified output.
#### Validation
The classified image is visually inspected using custom color maps and LCZ-specific legends. A confusion matrix is computed using all valid pixels from the testing set. Evaluation metrics include overall accuracy, per-class precision, recall, and F1-score.
#### Testing
To quantitatively assess the classifier's generalization ability, the predicted LCZ labels are compared against the ground truth for each pixel in the testing set. Agreement metrics are computed and visualized using interactive confusion matrices. Per-class performance is also summarized in a tabular report, including detailed statistics on support, class-wise accuracy, and error patterns.
This use case illustrates a robust pipeline for LCZ classification from Sentinel-2 imagery using supervised machine learning. The integration of ancillary layers and the use of preprocessing steps such as median filtering contribute to enhanced spatial coherence and improved classification accuracy. The workflow is designed to be scalable and modular, allowing for extension to other areas of interests and dates.

## CODENAME SPECTRAL UNMIXING is structured as follows:

### Training

#### Set Up
The classification workflow begins with the selection of a target acquisition date and the loading of the Sentinel-2 spectral bands. To account for clouds presence, a cloud masking is performed using the SCL band to mask cloudy pixels. Specifically, pixels identified as clouds, cloud shadows, or cirrus are flagged and excluded from further analysis by assigning them a designated no-data value. Each band is processed individually and then saved as a masked GeoTIFF. Once masking is complete, all processed bands are stacked into a three-dimensional array, forming the core data structure for spectral analysis.
#### Endmember Extraction
Annotated training points are provided in a GeoPackage, containing both geometric coordinates and metadata, including endmember labels and class identifiers. After reprojecting the point geometries to align with the CRS of the Sentinel-2 data, the spectral values for each point are extracted from the image cube. This results in a dataset containing pure spectrums for each land cover class. These data serve as the basis for synthetic mixture generation.
#### Synthetic Training Dataset
To simulate sub-pixel conditions, synthetic datasets are constructed by randomly combining pure spectra from different classes. For each class, pairs of spectra one representing the target class and the other a background class are linearly mixed using a randomly sampled fraction. The resulting mixed spectra, together with their known target fractions, form a training set for supervised regression. This strategy enables robust model development without requiring true sub-pixel reference data. In this use case, two thousand samples per class were generated.
#### Model Training and Inference
A separate Random Forest regressor is trained for each class using the respective synthetic dataset. Each model is configured with 100 decision trees and includes out-of-bag scoring to estimate generalization performance. Once training is complete, the stacked Sentinel-2 data are reshaped into a 2D array, filtering out masked pixels. The trained models are then applied to predict the fraction of each class for every valid pixel. Predictions are constrained to the [0, 1] interval and reshaped into 2D raster layers corresponding to each class. These layers are stacked and normalized such that the sum of fractions per pixel equals one, enforcing a physically meaningful constraint.
The final output is saved as a multi-band GeoTIFF. Metadata such as, CRS, and pixel dimensions are inherited from a reference raster to ensure spatial consistency. Each band in the output represents the estimated abundance of a single class, encoded as continuous float values.

### Validation and Testing

#### Set Up
Before initiating validation and testing, the relevant libraries and input data, namely the code-derived fraction map, the plugin-generated reference map, and the GeoPackage of testing points are loaded into memory. A check on the maps alignment is also performed.
#### Validation
Side-by-side maps and difference rasters are produced to visualize agreement. Per-class error metrics are computed, including mean absolute error, correlation coefficients, and value range differences. Particular attention is paid to pixels that are confidently classified defined as those where the top fraction exceeds a defined threshold while the second-highest fraction remains below another threshold allowing for a more conservative error assessment.
#### Testing
The second evaluation path is point-based. A separate set of validation points is loaded and sampled across both the plugin-derived and code-derived fraction rasters. For each point, the class prediction is inferred from the highest fractional value. If the confidence thresholds are met, the prediction is considered valid. These predictions are compared to ground truth labels to compute confusion matrices and classification accuracy metrics. Confidence rates, overall accuracy, and per-class precision are reported, offering a detailed picture of model performance.
