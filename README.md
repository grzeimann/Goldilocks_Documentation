# Goldilocks_Documentation
**Habitable Planet Finder (HPF) Automatic Reduction Software**

*Goldilocks* is a an automatic pipeline for the HPF instrument at the Hobby Eberly Telescope (HET).  The pipeline reduces the previous night's data the following morning and provides users with reduced spectra to either adjust their program/targets or quickly turn observations into a scientific publication.  Although the primary goal of the software is to quickly reduce the HPF data products, we still aim to provide the highest quality reductions possible for both spectral and radial velocity analysis.  To that aim, please do not hesitate to contact the author below and provide feedback or make requests.

## Table of Contents

>[Working on TACC](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Working-on-TACC)

>>[Signing up for an account](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Signing-up-for-an-account)

>[Data Products](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Data-Products)

>[Documentation](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Documentation)

>>[Reduction Steps](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Reduction-Steps)

>>>[Slope Image](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Slope-Image)

>>>[Spectral Extraction](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Spectral-Extraction)

>>>[Goldilocks vs HPF Team Reduction](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Goldilocks-vs-HPF-Team-Reduction)

>>>[Radial Velocity Corrections](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Radial-Velocity-Corrections)

>>>[Examples](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Examples)

>[Telluric Correction](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Telluric-Correction)

>>[Methodology](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#methodology)

>>[Sky Subtraction And Deblazing](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#sky-subtraction-and-deblazing)

>>[Continuum Normalization](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#continuum-normalization)

>>[Principal Component Analysis](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#principal-component-analysis)

>[Author](https://github.com/grzeimann/Goldilocks_Documentation/blob/master/README.md#Author)

## Working on TACC 
The reductions are designed to be run on TACC where a copy of the raw data lives.  We will describe how to get started on TACC, where the automatic reduction products reside.

### Signing up for an account
https://portal.tacc.utexas.edu/
<p align="center">
  <img src="images/tacc_create_account.png" width="650"/>
</p>

After creating an accounting using the link above, please send Greg Zeimann <gregz@astro.as.utexas.edu> your TACC username and he will add you to the HET group.  When that step is complete, you can ssh into TACC using:
```
ssh -Y USERNAME@ls6.tacc.utexas.edu
```

## Data Products
*Goldilocks* is run each morning 11:30am.  The run time varies on cpu availability and the amount of data, but the reductions should finish by 1pm or so.  An accompanying email from the HET astronomer will alert the user that target(s) were observed the previous night and data reductions are (or will be soon) available.  The data products are in the following path:
```
/work/03946/hetdex/maverick/HPF/PROGRAM-ID
```
where PROGRAM-ID, is your program number, for example HET20-3-999.  To get all of the current reductions for your program, simply:
```
scp -r username@ls6.tacc.utexas.edu:/work/03946/hetdex/maverick/HPF/PROGRAM-ID .

OR on the destination machine:

cd PROGRAM-ID
rsync -avz username@ls6.tacc.utexas.edu:/work/03946/hetdex/maverick/HPF/PROGRAM-ID/ .
```
You merely have to use your "username" and your "PROGRAM-ID" and you can copy over your products.  There is a small complication.  The data products in your "PROGRAM-ID" folder include the products of the HPF team as well as the new automatic pipeline.  Products released from the HPF team occur on a quarterly basis (every three months or so) and include the naming scheme:
```
Slope-DATETIME_R01_OBSID.fits [SLOPE AND VARIANCE IMAGE]
Slope-DATETIME_R01_OBSID.optimal.fits [SCIENCE, SKY, AND CAL SPECTRA]
```
For information on the HPF products please see: 

https://psuastro.github.io/HPF/HPF-First-Data-Release-Documentation/

Products from *Goldilocks* are produced daily and have the naming scheme:
```
Goldilocks-DATETIME_v01_OBSID.fits [SLOPE AND VARIANCE IMAGE]
Goldilocks-DATETIME_v01_OBSID.spectra.fits  [SCIENCE, SKY, AND CAL SPECTRA]
```
The "V01" refers to the version of Goldilocks used.

```
Goldilocks-DATETIME_v01_OBSID.fits:

0. Primary Header
1. Slope image (e-/sec)
2. Error of the slope image (e-/sec)
```

The spectra fits file includes 9 extensions and a primary extension containing the header.  Note that Goldilocks' spectra store the error and not the variance, unlike the standard HPF team's reduction.
```
Goldilocks-DATETIME_v01_OBSID.spectra.fits:

0. Primary Header
1. Science fiber flux in units of (e-/sec)
2. Sky fiber flux in units of (e-/sec)
3. Calibration fiber flux in units of (e-/sec)
4. Science fiber error in units of (e-/sec)
5. Sky fiber error in units of (e-/sec)
6. Calibration fiber error in units of (e-/sec)
7. Science fiber’s wavelength per pixel (Vacuum wavelength in Angstroms)
8. Sky fiber’s wavelength per pixel (Vacuum wavelength in Angstroms)
9. Calibration fiber’s wavelength per pixel (Vacuum wavelength in Angstroms)
```

A png output is also included:
```
Goldilocks-DATETIME_v01_OBSID.spectra.png
```
which plots the sci, sky, and cal fibers in blue, orange, and green respectively.  The plots also include the median S/N for each order.  The png file is quite large to allow for sufficient zoom to peak up on any given feature.

## Documentation

### Reduction Steps
#### Slope Image
Much of the code for generating slope images is originally from the pyhxrg module in HxRGproc by Ninan et al. (in prep).  The algorithms of which are based on Ninan et al. (2018)[https://ui.adsabs.harvard.edu/abs/2018SPIE10709E..2UN/abstract].

```
Slope and Error Image Generation:

1. Subtract first frame from all frames in the data cube
2. Subtract even and odd reference pixels on the 4 pixel border
3. Subtract linear model using top and bottom reference pixels within a channel
4. Apply non-linear correction using polynomial model with input counts and output corrected counts
5. Mask values above 30,000 ADU
6. Multiply by gain
7. Calculate slope image (e-/s) which is average change in counts from frame to frame
8. Remove residual bias in each channel and then over all rows using residual in border pixels from slope image
9. Calculate error image (e-/s) using Robberto 2010 (Eq 7) and applying a square root
10. Subtract 2D 2nd order polynomial to remove scattered light
11. Divide slope and error image by flat field; this results in zeros outside of the fiber model region (+/-7 pixels) for each order (zero values are masked pixels)
```

We combine Alpha Bright exposures over many nights to make a single master frame for all three fibers.  This master frame is used to model the trace of the fibers, the fiber profile for each fiber and order as a function of column, and to remove flat fielding features.  Below is a zoom-in of the hpf_trace_image.fits frame in the calibrations folder.  The three fibers (sky, sci, and cal) are marked, as well as flat-fielding features which are removed for science frames.

<p align="center">
  <img src="images/master_alpha_bright.png" width="650"/>
</p>

#### Spectral Extraction
```
Extracted Spectra for Sky, Sci, and Cal Fibers:

1. Measure trace from master Alpha Bright image
2. Build a fiber profile as a function of column for each fiber and order using the master Alpha Bright image
3. Evaluate the profile for each fiber and order at each column and use the profile for optimal extraction
4. spectrum[fiber][order][column] = sum(profile * data * mask / variance) / sum(profile^2 * mask / variance) [Horne 1986]
```

We use the master Alpha Bright image to build a trace solution (5th order polynomial) for each fiber and order.  We then build the fiber profile along the trace, again for each fiber and order as function of column.  An example of the fiber profiles are shown below for different orders.

<p align="center">
  <img src="images/hpf_fiber_profile.png" width="1050"/>
</p>

We use Horne (1986) and the simple formula in step 4 to calculate the spectrum as function of column.  **Note that we erroneously do not rectify the spectra before we extract.**  This results in a mismatch between the expected fiber profile for the column and the data due to the data being tilted perpendicular to the trace.  This can be as large of a tilt as 6% for some orders.  The benefits of rectification before extraction are obvious but time consuming (flux-conserving algorithms can take up to an hour for each observation).  For now, Goldilocks makes this known error in extraction for speed over accuracy.  We plan to undertake the investigation to find what impact this extraction has on the accuracy and precision of rv analyses.  Programs with a focus on flux estimations will see very little impact from this extraction method.

#### Goldilocks vs HPF Team Reduction
We compare our extractions to those of the HPF team and find quite good agreement.  After subtracting off the sky fiber from the science fiber for each order, we typically agree to <1% and agree on average at the 0.5% level.  The mismatch and offset between the two reductions is likely due to differences in flat-fielding corrections, fiber profile estimates, and rectification before extraction.

<p align="center">
  <img src="images/goldilocks_vs_hpf.png" width="850"/>
</p>

#### Radial Velocity Corrections
**Note: all observations have the same wavelength solution while instrumental drifts and barycentric corrections are modeled, put in the header, and left to the users to correct the wavelength solutions.**

We model the Laser Frequency Comb (LFC) for the calibration fiber throughout the night to infer the instrumental radial velocity correction needed for a given scientific observation.  The radial velocity change throughout the night is sufficiently modeled with a linear regression and the results of which are shown in:
```
/work/03946/hetdex/maverick/HPF/CALS/{DATE}_rv.png
```

The typical correction ranges from -10 to 10 m/s.  The inferred correction is stored in the header as **LRVCORR**.  **A value of 0.0 exactly indicates a failure in the instrumental rv correction.  These will be addressed using a longer period model as discussed below to aid on failed nights** We show the instrument RV modeling for 01/07/2020 below.  Note that the ticks for the date-time line up with the ticks for the modified julian date and occur on the hour.

<p align="center">
  <img src="images/rv_example.png" width="850"/>
</p>

Longer trends in the instrumental drift can be seen as well.  Here we show an analysis of the RV shift from late September 2019 to March 2020.  Missing dates either had a failed analysis or were insufficiently covered by laser frequency comb (LFC) exposures.  There is a sawtooth shape to each day that can be seen on the zoom in.  The origin is discussed extensively in Metcalf et al. (2019), and it is related to the weight of the liquid nitrogen tank between re-fills.  Over a period of weeks you can see drifts in the instrumental RV and a strong correlation between the intercept of a given night and its neighbors.  This allows us to model nights with missing LFC data.  In this plot we simply use a smoothing spline to capture the drift over these 6 months.  *Since Goldilocks is run each morning, this long term analysis is not applied to the reductions but could be provided if requested.*

<p align="center">
  <img src="images/RV_Goldilocks_Analysis.png" width="850"/>
</p>

We also estimate the barycentric radial velocity correction similarly to the following example code which draws on astropy's modules (EarthLocation, SkyCoord, Time):
```
loc = EarthLocation.of_site('McDonald Observatory')
sc = SkyCoord(RA, Dec)
t = Time(begin+length/2.)
vcorr = sc.radial_velocity_correction(kind='barycentric', obstime=t, location=loc)
```
The barycentric radial velocity correction is stored in the header as **BRVCORR**.  We expect this correction to have some error as we don't use the exact geographical location of the HET and rather use the McDonald Observatory.  We also don't calculate the flux-weighted average time for each order but instead just use the time at the middle of the exposure.

#### Examples
Here we are showing a twilight spectrum for each of the 28 orders.  The three colors: blue, orange, and green, correspond to the three fibers: sci, sky, and cal.  This ".png" is available for each reduced source.

<p align="center">
  <img src="images/example_twi.png" width="850"/>
</p>

## Telluric Correction

### Methodology

All HPF spectra suffer from attenuation in the Earth's atmosphere producing telluric lines in the spectrum.  Often, telluric lines are corrected using a hot star observed on the same night normalized by a template A/B stellar spectrum, denoted as the empirical method.  Alternatively, radiative transfer codes can produce telluric absorption spectra for a range of input to fit a given observation, denoted as the theoretical method.  Both methods are successful to a degree, but each can have their drawbacks.  The empirical method is quick and includes the nuiances of the site location / instrument, but requires an extrapolation step that can be quite uncertain.  The theoritical method is flexible and physically motivated, but it can be slow and requires significant effort to map successfully to the instrumental setup and atmospheric conditions.  

We instead take a non-parametric approach in Goldilocks and construct a principal component basis for the theoretical models of TelFit (Gullikson et al. 2014), iteratively informing the basis using telluric standard star observations.  This can be viewed as a hybrid approach between the empirical efforts and theoritical codes.  We collected all telluric standard star (HRXXXX) observations from Jan-2021 through Sep-2021, totalling 126 observations.  We ran each star through the same processing steps described below. 

### Sky Subtraction And Deblazing

We begin our analysis with the initial products of Goldilocks which include science, sky, and calibration fiber spectra in e-/s.  The science fiber still includes night sky emission and requires deblazing.  We use the Laser Driven Light Source (LDLS) master frame in the Goldilocks calibration folder to produce a vector normalization for the sky fiber to the science fiber (the sky fiber is typically 93-97% of the throughput of the science fiber).  After normalization, we subtract the sky fiber and  use the same LDLS master frame to deblaze the science spectra.  The deblazing function is just a smooth continuum of the science fiber LDLS spectrum normalized by the median value and the science observation is divided by the deblazing function.  We show an example of the sky subtraction and deblazing steps below.  

<p align="center">
  <img src="images/skysub_deblaze_v02.png" width="850"/>
</p>

### Continuum Normalization

Although our spectra are deblazed, they are not continuum normalized.  Rather than fitting stellar templates to normalize our spectra, we choose a flexible, more localised continuum fitting effort that can ignore telluric absorption lines but also fit broad stellar features in the hot telluric standards.  We explain the process to estimate the continuum below and show an example in the following figure.

```
Continuum estimation:

1. Using a default telluric model from TelFit (50% humidity, 2.0e5 O2, and the median airmass of the HET), 
   we identify and mask wavelengths with tranmission <98.5% (where 100% is zero absorption).
2. For each order, using 100 bins for the 2040 data points, we calculate the 50th-% for the given bin ignoring masked values.  
   If all data points in a bin are masked, the bin is ignored in the next process.
3. Linearly interpolate between each bin (including extrapolation for the end points) to construct our continuum model.  
```
<p align="center">
  <img src="images/telluric_continuum_v02.png" width="850"/>
</p>

### Principal Component Analysis

After continuum normalization, we can begin fitting telluric absorption line models. Since the HET observes at a nearly fixed airmass, the range of observed telluric absorption is smaller than other observatories.  We can use this to our advantange and subtract an average telluric absorption model and only fit the residuals with a PCA basis.  This is a more ideal useage of principal component analysis as most PCA efforts require the subtraction of a mean signal before fitting.  However, we must first determine an average telluric absorption model, and we need a set of telluric standard star residuals to construct our PCA basis.  So we start with an initial fitting effort for our telluric standard library to build the median model and construct a residual PCA basis.

We could use the empirical method to create our average telluric model and residual PCA basis, but we don't want to have features in the principal components that might be related to poor continuum normalization of real stellar features.  Instead we want to create our average/residual models from theoritical telluric spectra to avoid overfitting.  So, we construct a grid of TelFit models at the HPF resolution for three parameters: airmass, O2, and H2O.  This grid is rather coarse with only three points for airmass (min, median, and maximum airmass for the HET), three points for O2 (0.5e5, 2e5, and 3.5e5), and finally 8 points for H20 (ranging from 0-100% humidity).  We don't use the grid itself for fitting, but instead we build a PCA model with 15 components from our 72 grid spectra to find the initial best fit telluric model for each of our 126 stars.  We fit order by order for this stage to construct the best initial models that we can.  After fitting each order for each star, we tabulate all of the fits and determine the average telluric absorption for all channels.  Our average is a summation of the biweight average model fits and the biweight average offset of the models from the data.  This lowers the systematics between the HPF spectra and the TelFit models.  We then construct a PCA basis from the residuals of the biweight average model and the individual model fits.  The figure below shows a small wavelength range of the average telluric model and the first 5 components of the 15 component basis.

<p align="center">
  <img src="images/telluric_pca.png" width="850"/>
</p>

Now that we have constructed our average telluric spectrum and our residual PCA basis we can again fit our telluric standard star library to gauge how well our hybrid model works.  Unlike the previous PCA fitting, we do not fit order by order but instead fit the whole spectrum at once.  We do ignore more challenging orders in which continuum estimation is difficult due to saturation from water vapor absorption.  We isolate our fits to the data in orders=[1, 2, 3, 7, 8, 9, 10, 14, 15, 16, 17, 18, 19, 20, 24, 25, 26, 27, 28], using the convention that the orders start at 1 and not 0.  After continuum normalization, we subtract our average telluric spectrum, and then we perform a least squares minimization for wavelengths with <98.5% atmospheric throughput according to our average telluric model.  The solution are 15 eigenvalues related to our standard 15 principal components.  The dot product of which is added to our average telluric model to produce a telluric estimation for all 28 orders.  We show an example fit for HR4829, order 20, observed on 20210517 and a weather reading of 21% humidity.  As you can see, the hybrid approach does a remarkable job estimating telluric absorption across the whole order.  Remember, the telluric model was not fit to this order alone but the spectrum as a whole bolstering the support for this approach.

<p align="center">
  <img src="images/telluric_model_v02.png" width="850"/>
</p>

Theoritical telluric fits tend to struggle with O2 bands.  Here we show the 28th order for HR4829 (same star as above), which has a significant O2 band in the wavelength region.  The fit looks very accurate compared to other previous efforts (e.g., Gullikson et al. 2014, Ulmer-Moll et al. 2019).

<p align="center">
  <img src="images/telluric_model_v03.png" width="850"/>
</p>

## Author
Greg Zeimann

**HET Data Scientist**

*grzeimann@gmail.com* or *gregz@astro.as.utexas.edu*


