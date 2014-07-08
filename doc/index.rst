.. zap documentation master file, created by
   sphinx-quickstart on Mon Nov 25 09:46:49 2013.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to ZAP's documentation!
===============================

.. toctree::
   :maxdepth: 2

ZAP (the Zurich Atmosphere Purge) is a high precision sky subtraction tool which can be used as complete sky subtraction solution, or as an enhancement to previously sky-subtracted data.  **Currently, the best results come from applying ZAP to a datacube with no initial sky subtraction.** The method uses PCA to isolate the residual features and remove them from the observed datacube.

Examples
========

In its most hands-off form, ZAP can take an input fits datacube, operate on it, and output a final fits datacube.::

  from mpdaf_user import zap
  zap.process('INPUT.fits', 'OUTPUT.fits')

Care should be taken, however, since this case assumes a sparse field. 

There are a number of options that can be passed to the code which we tabulate here, and will describe in several diffferent use cases. While every option is available in each type of processing, I outine several example cases below.

It is important to note that many of these optioons come into play when building a basis set that will go into doing sky subtraction calculation.

========  ========  ==============================================================================
Keyword   Default   Function
========  ========  ==============================================================================
clean     True      Interpolates over NaN values in the datacube to allow processing on all 
                    spectra for the input SVD. The NaN values are replaced in the final datacube.
		    Any spaxel that includes a NaN pixel will be rejected from the SVD 
		    calculation, so this step is used to maximixe the number of contributors, but
		    the NaNs remain in the final datacube.

zlevel    'median'  This option is used to define the method for determining the zeroth order 
                    subtraction of the sky. This is used to remove any systematic sky feature 
		    that exists over the whole field. Options are 'median', 'sigclip', and 'none'.
		    The 'none' option should only be applied when ZAP is applied to enhance a 
		    previous sky subtraction.

q         0         Quartile selection, this option will remove the top n quartiles of the 
                    dataset to calculate of the zlsky. This option works well set to 1 with the 
		    'sigclip' option.

cftype    'median'  The type of filtering that is applied to remove the continuum.  'weight' 
                    refers to the weighted median, which uses the calculated zlevel sky as the
		    weight. 'median' refers to a rolling median filter with a nested small 
		    uniform filter. The 'weight' option provides a better result, but is much
		    slower (an additional 10 minutes on a single exposure) and should only be 
		    run in the complete sky subtraction case.

cfwidth   300       Size of the filterbox used to remove the continuum features in order to
                    sterilize the basis set used to calculate the eigenbasis.

optimize  True      A flag used to call the optimization method. This automatically determines the
                    number of eigenspectra/eigenvalues to use per segment. 

nevals    []        Number of eigenspectra/eigenvaules used per spectral segment. If this
                    is used, the pevals is ignored. Provide either a single value that will be 
		    used for all of the segments, or a list of 9 values that will be used for 
		    each of the segments.

pevals    []        Percentage of the caclulated eigenspectra/eigenvaules used per spectral
                    segment. Provide either a single value that will be used for all of the 
		    segments, or a list of 9 values that will be used for each of the segments.

extSVD    ''        An optional parameter that allows the input of a externally calculated 
                    eigenbasis as well as a zlevel

mask      ''        (only used with the SVDoutput method) A 2D fits image to exclude regions that
                    may contaminate the zlevel or eigenspectra.

========  ========  ==============================================================================

The code can handle datacubes trimmed in wavelength space. Since the code uses the correlation of segments of the emission line spectrum, it is best to trim the cube at specific wavelengths. The cube can include any connected subset of these segments. (for example 6400 - 8200 Angstroms) ::

  [4600, 5400]
  [5400, 5850]
  [5850, 6400]
  [6400, 6700]
  [6700, 7150]
  [7150, 7700]
  [7700, 8200]
  [8200, 8700]
  [8700, 9900]


Sparse Field Case
-----------------

As noted above, this case can be handled simply with the observed datacube itself, using: ::

  zap.process('INPUT.fits', 'OUTPUT.fits')

It is possible to change the zlevel calculation with: ::

  zap.process('INPUT.fits', 'OUTPUT.fits', zlevel='sigclip')


Partially Filled Case
---------------------

There are two approaches to this case, which both use an algorithm to limit the effect of the large object on the determination of the sky. The first approach is similar to the sparse case, except to include quartile rejection for the zlevel calculation. ::

  zap.process('INPUT.fits', 'OUTPUT.fits', q=1, zlevel='sigclip')

This option removes the top quatile per spectral channel of the data before determining the contribution to the zlevel. This option used along with the sigma clipping can identify the background sky level, without including signal from the objects. This is the best option for a field that has many objects.

The second option is to use a mask to generate the intial SVD calculation, as described below.


Pre-determined zlevel and eigenspectra
--------------------------------------

Another option is to use a mask to isolate a sky within an exposure to pre-determine the zlevel and eigenspectra, which is then passed back into zap. This approach will allow the inclusion of a mask file, which is a 2d fits image matching the spatial dimensions of the input datacube. The values in the mask image will be 0 in the masked regions (such a where an extended object is) and 1 in the unmasked regions. Set this parameter with ``mask='maskfilename.fits'`` ::

  zap.SVDoutput('INPUT.fits', svdfn='Masked_ZAP_SVD.fits', mask='maskfilename.fits') # create SVD file
  zap.process('INPUT.fits', 'OUTPUT.fits', extSVD='ZAP_SVD.fits')

Saturated Field Case and Time Series Sky
----------------------------------------

This approach also can address the saturated field case and is robust in the case of strong emission lines, in this case the input is an offset sky observation. ::

  zap.SVDoutput('Offset_Field_CUBE.fits', svdfn='Offset_ZAP_SVD.fits')
  zap.process('INPUT.fits', 'OUTPUT.fits', extSVD='ZAP_SVD.fits')

The integration time of this frame does not need to be the same as the object exposure, but rather just a 2-3 minute exposure. A series sky datacubes can also be used to identify time varying components of the sky.  For example if one particular set of sky lines is present at the beginning of the exposure and absent at the end of the exposure. In this case it is best to take a sky frame before and after the science frame. This is most helpful in the saturated field case, since the offset frames are taken at a different time than the science frame. To use this option, simply provide a list of offset file names. ::

  OffsetCubeList = ['OffsetFieldCUBE1.fits','OffsetFieldCUBE2.fits', 'OffsetFieldCUBE3.fits']
  zap.SVDoutput(OffsetCubeList, svdfn='Offset_ZAP_SVD.fits')
  zap.process('INPUT.fits', 'OUTPUT.fits', extSVD='ZAP_SVD.fits')

Top Level Functions
===================

Aside from the main "full process", I have also included two functions that can be run outside of the entire zap process to facilitate some investigations.

**nan removal**

This function replaces the nan valued pixels with an average of the adjacent valid pixels. It can be called as below::

  zap.nancleanfits('INPUT.fits', outfn='NANCLEAN_CUBE.fits', rejectratio=0.25, boxsz=1)

"rejectratio" defines a cutoff for the ratio of pixels in a spaxel before the spaxel is avoided completely.

"boxsz" defines the number of pixels that defines the box around the offending nan pixel. With boxsz set to 1 the function looks for the nearest 26 neighbors which is a 3x3 box. 

This step is only an intermediary step in the full ZAP process as a way to create a clean input into the SVD calculation, but this function allows you to run it as a standalone step. 

**continuum removal**

This function applies a filter on the datacube that removes most continuum features. This function allows for the enhancement of emission line characteristics. The filtering method has been enhanced by multiprocessing to produce a rapid result. It can be called as below::

  zap.contsubfits(musecubefits, contsubfn='CONTSUB_CUBE.fits', cfilter=100):

It applies a nested set of filters, one uniform of width 3 pixels and one median with a width defined by cfilter. This filter differs from the new enhanced method in the ZAP process.




Interactive mode
================

ZAP can also  be used interactively from within ipython using pyfits. ::

  from mpdaf_user import zap
  zclass = zap.interactive('INPUT.fits')

The run method operates on the datacube, and retains all of the data and methods necessary to
process a final data cube in a python class named zclass. You can elect to investigate the data product via the zclass, and even reprocess the cube with a different number of eigenspectra per region.  A workflow may go as follows: ::

  from mpdaf_user import zap
  from matplotlib import pyplot as plt
  
  zobj = zap.interactive('INPUT.fits', pevals=1) #choose 1% of modes per segment
  
  #investigate the dataproduct with pyplot
  plt.figure()
  
  #plot a spectrum extracted from the original cube
  plt.plot(zobj.cube[:,50:100,50:100].sum(axis=(1,2)), 'b', alpha=0.3)
  
  #plot a spectrum of the cleaned ZAP dataproduct
  plt.plot(zobj.cleancube[:,50:100,50:100].sum(axis=(1,2)), 'g')
  
  #choose just the major mode
  zobj.reprocess(nevals=1) 

  #plot a spectrum extracted from the original cube
  plt.plot(zobj.cube[:,50:100,50:100].sum(axis=(1,2)), 'b', alpha=0.3)
  
  #plot a spectrum of the cleaned ZAP dataproduct
  plt.plot(zobj.cleancube[:,50:100,50:100].sum(axis=(1,2))), 'g')

  #choose some number of modes by hand
  zobj.reprocess(nevals=[2,5,2,4,6,7,9,8,5]) 

  #plot a spectrum
  plt.plot(zobj.cleancube[:,50:100,50:100].sum(axis=(1,2))), 'k')

  #Use the optimization algorithm to identify the best number of modes per segment
  zobj.optimize()

  #compare to the previous versions
  plt.plot(zobj.cleancube[:,50:100,50:100].sum(axis=(1,2))), 'r')  

  #identify a pixel in the dispersion axis that shows a residual feature in the original
  plt.figure()
  plt.matshow(zobj.cube[2903,:,:])

  #compare this to the zap dataproduct
  plt.figure()
  plt.matshow(zobj.cleancube[2903,:,:])

  #write the processed cube
  zobj.writecube('DATACUBE_ZAP.fits')

  #or merge the zap datacube into to whole inout datacube, replacing the data extension
  zobj.writefits(outcubefits='DATACUBE_FINAL_ZAP.fits')


ZCLASS
======
.. autoclass:: zap.zclass
   :members:


Algorithm description
=============================
ZAP is designed to remove the residual sky emission features after an initial Sky subtraction. To make this correction, several steps must be performed. Many of these are constructed to create a set of input spectra for singular value decomposition. By performing these prior steps we can better isolate the eigenspectra and eigenvalues that reconstruct the sky residuals without influencing the Astronomical objects. Below is a description of each of the processing steps.

The key point in this approach is that the preliminary steps are use to remove as many easily determined non-sky features from the spectra going into the SVD calculation. The sky features are then reconstructed and removed from the original datacube.


NaN cleaning
++++++++++++
Set by keyword "clean = True"

NaN (Not a Number) pixels interfere with the SVD (Singular Value Decomposition) by being unbounded. Since the eigenspectra can not reconstruct these values, the pixel must be either replaced with a float or the spaxel for which it is a member removed from the calculation. By Performing the NaN cleaning step, we choose the first of these options.

This algorithm identifies NaN pixels in the datacube and replaces them by interpolating with the 26 (3x3x3 - 1) nearest neighbor pixels in the 3D datacube. Any NaN values in this set of pixels is ingored and the mean of the finite values replaces the central pixel. ZAP retains the position of these NaN values so that the pixels can be converted back into NaN values in the final dataproduct.

The user has the option to exlude this step (run_clean = False), but should be advised that the presence of a single NaN pixel will exclude an entire spaxel from the entire calculation leaving it uncorrected.


Zero Level Subtraction
++++++++++++++++++++++
Remove systematic offset in each spectral channel.

Set by keyword "zlevel = True"

This processing step is performed to remove residual sky features that are consistent over the entire field. The "zero level" is determined from the median per spectral channel, producing a spectrum of the residual. This spectrum can be accessed in the interactive mode from the produced instance of the zclass (Described below)

The user should be cautious in cases where a spectral feature, such as an emission line, covers the entire field. In these cases, the median calculation will erroneously remove this feature.  Several approaches are in development for handling this case.

Continuum Filtering
+++++++++++++++++++
Remove the continuum from each spaxel via filtering.

Filter size set by keyword "cfilt = 100"

The continuum filter is a process that removes astrophysical continuum features through the use of a combination of two sprectral filters. A uniform filter that is rougly on the scale of the spectral resolution (3 pixels) smooths any extreme variations on this small scale. The next filter is a large scale (100 pixels) median filter, which has the property of tracing a multitude of continuum shapes, including sharp edges in the spectrum. The result of these filters is then subtracted from the spectra leaving only object emission lines and sky residuals in the spectra.

Spectral Segmentation
+++++++++++++++++++++
This algorithm operates by segmenting the data into regions of the spectrum with strongly correlated features. This segmentation benefits the calculation in two ways. First, the correlated features are isolated from each other, making the dominant modes more pronounced, and therefore easier to choose when the algorithm reconstructs the residuals.  Second, this data segmentation allows us to implement multiprocessing on the calculations, which greatly reduces the calculation time.

Normalization
+++++++++++++
Normalize the spectra to be inserted into the SVD calculation.

The method of normalization has a strong effect on any SVD calculation. In this approach, we normalize to the variance for each spectrum within the operating spectral segments.


Singular Value Decomposition
++++++++++++++++++++++++++++
Calulate the eigenspectra and eigenvalues that characterize the residual emission line features.

Using the previous steps, we have produced a set of spectra that are prepared for the Singular Value Decomposition, which cacluated the eigenspectra and the eigenvalues that create the entire set of input spectra. To use these eigenspectra effectively, the eigenmodes must be chosen to identify only the contributions by the sky residuals.

Optimization
++++++++++++