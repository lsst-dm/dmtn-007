..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

..
    ## Dipole measurement and classification
    ----------------------------------------

    -  `Additional random dipole characterization thoughts <README.md>`__ in
       no particular order.

    -  `Summary of current implementation (``ip_diffim``
       ``dipoleMeasurement``) <#summary-of-current-implementation-ip_diffim>`__
    -  `Evaluation of dipole fitting
       accuracy <#evaluation-of-dipole-fitting-accuracy>`__
    -  `Putative issues with the ``dipoleMeasurement`` PSF fitting
       algorithm <#putative-issues-with-the-ip_diffim-psf-fitting-algorithm>`__
    -  `Generic dipole fitting
       complications <#generic-dipole-fitting-complications>`__
    -  `Possible solutions and tests <#possible-solutions-and-tests>`__

Abstract
========

In image subtraction analysis (also called "difference image
analysis", or DIA), an observation image is subtracted from a template
image in order to identify transients from either image. To perform
such analysis, the observation must be astrometrically aligned to the
template image, and the PSFs of the two images are matched. Arising
primarily due to slight astrometric alignment or PSF matching errors
between the two images, flux "dipoles" are a common artifact often
observed in image differences (and even if these issues were perfectly
corrected, unless the two images were obtained at similar zenith
distances, differential chromatic refraction [DCR] would still lead to
such artifacts). These dipoles will lead to false detections of
transients unless correctly identified and eliminated. Moreover,
accurate measurement of the flux dipoles can enable evaluation of
methods for mitigating the processes which caused them.

Summary of existing LSST stack implementation (``ip_diffim``, ``dipoleMeasurement``)
=========================================================

The current dipole measurement task in the LSST stack is intialized
with ``SourceDetection`` performed on the image difference, in both
positive and negative modes to identify significant pos. and
neg. sources. These pos. and neg. source catalogs are merged to
identify candidate dipoles with overlapping pos./neg. footprints. The
measurement task then performs two separate measurements on these
dipole candidates (i.e., those footprints with two sources):

1. A "naive" dipole measurement which computes a 3x3 weighted moment
   around the nominal centroids of each peak in the `Source`
   `Footprint`. It estimates the pos./neg. fluxes correspondingly as
   sums of the pos./neg. pixel values within the merged footprint.
2. Measurements resulting from a weighted least-squares joint (dual)
   PSF model fit to the negative and positive lobes
   simultaneously. This fit simultaneously solves for the negative and
   positive lobe centroids and fluxes using non-linear least squares
   minimization. The existing method is implementated in `C++` and
   uses the `minuit2` `C++` library with standard parameters
   (tolerances) for minimization of six parameters: two centroids and
   two fluxes.

The two measurements are performed independently and do not appear
inform each other; in other words the centroids and fluxes from the
naive dipole measurement are not used to initialize the starting
parameters in the least-squares minimization.

Evaluation of dipole fitting accuracy, robustness
=====================================

We implemented a faux dipole generation routine with a double Gaussian
PSF (default sigma 2.0 pixels; see `notebooks
<https://github.com/lsst-dm/dmtn-007/tree/master/_notebooks>`__ for
additional parameters) and realistic non-background-limited (Poisson)
noise. We then implemented a separate 2-D dipole fitting function in
"pure" python (we used the ``lmfit`` package, which is a wrapper
around ``scipy.optimize.leastsq()`` that allows for parameter
constraints and additional useful tools for evaluating model fit
accuracy and quality). The dipole (and the function which is
minimized) is generated using a `Psf` as implemented in the LSST
stack.

Interesting findings include:

1. Surprisingly, the "pure python" optimization is roughly as fast as
   the existing ``dipoleMeasurement`` implementation. Further
   investigation and timings of the existing implementation showed
   that it performed a complete optimization of a single dipole in
   $\sim 20$ ms. This runtime could potentially be sped up by a
   factor of $\sim 2$ by improving parameter boxing and
   initialization.
2. Using similar constraints (i.e., none), the "pure python"
   optimization results in model fits with similar characteristics to
   those of the ``dipoleMeasurement`` code. For example, a plot of
   recovered x-coordinate of dipoles of fixed flux with gradually
   increasing separations is shown below:

 |Figure 1|

Note that in all figures, including this one, "New" refers to the "pure
python" dipole fitting routine, and "Old" refers to the fitting in the
existing ``dipoleMeasurement`` code. These are for (default) PSFs with
$\sigma=2.0$. All units are pixels.

**For all figures, click on the image to see a larger version for easier
reading of axis values.**

A primary result of comparisons of both dipole fitting routines showed
that if unconstrained, they would have difficulty finding accurate
fluxes (and separations) at separations smaller than ~1 sigma. This is
best shown below, in a plot of fitted dipole fluxes as a function of
dipole separation for a number of realizations per separation (and
input flux of 3000).

 |Figure 2|

Here, `pos` -itive and `neg` -ative lobe flux estimates are shown
side-by-side in blue and yellow respectively, on the same (positive)
flux axis.

Below we investigate this issue and find that it arises from the extreme
covariance between the dipole separation and flux parameters, which
exacerbates the optimization at low signal-to-noise.

Additional comparisons may be found in the `IPython notebooks
<https://github.com/lsst-dm/dmtn-007/tree/master/_notebooks>`__.

Putative issues with the ``dipoleMeasurement`` PSF fitting algorithm
====================================================================

The PSF fitting is slow. It takes ~20ms per dipole for most
measurements on my fast Macbook Pro (longer times, especially for
closely-separated dipoles).

Why is it slow? Thoughts on possible reasons (they will need to be
evaluated further if deemed important):

1. ``PsfDipoleFlux::chi2()`` computes the PSF *image* (pos. and neg.) to
   compute the model, rather than using something like
   ``afwMath.DoubleGaussianFunction2D()``. Or if that is not possible
   (may need to use a pixelated input PSF) then potentially speed up the
   computation of the dipole model image (right now it uses multiple
   vectorized ``afw::Image`` function calls).
2. It spends a lot of time floating around near the minimum and perhaps
   can be cut off more quickly (note this may be exacerbated by (1.)).
3. The starting parameters (derived from the input source footprints)
   could be made more accurate. At least it appears that the starting
   flux values are initialized from the peak pixel value in the
   footprint, rather than (an estimate of) the source flux.
4. ``chi2`` is computed over the entire footprint bounding box
   (confirm this?) rather than within just the footprint itself or
   just the inner 2,3,4, or 5 sigma of the PSF.
5. Some calculations are computed each time during minimization (in
   ``chi2`` function) that can be moved outside (not sure if these
   calc's are really expensive though).
6. There are no constraints on the parameters (e.g. ``fluxPos`` > 0;
   ``fluxNeg`` < 0; possibly ``fluxPos`` = ``fluxNeg``; centroid
   locations from pixel coordinates of max./min. signal, etc.). Fixing
   this is also likely to increase fitting accuracy (see below).

Note: It seems that the dipole fit is a lot faster for dipoles of
greater separation than for those that are closer (apparently, the
optimization [via ``minuit2``] takes longer to converge).

Generic dipole fitting complications
====================================

There is a degeneracy in dipole fitting between closely-separated
dipoles from bright sources and widely-separated dipoles from faint
sources. This is further explored using 1-d simulated dipoles in `this
notebook <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/8a_1d_dipole_fitting_and_contours.ipynb>`__.

Here is an example:

 |Figure 3|

There are many such examples, and this strong covariance between
amplitude (or flux) and dipole separation is most easily shown by
plotting error contours from a least-squares fit to simulated 1-d
dipole data:

 |Figure 4|

Here are the error contours, where the blue dot indicates the input
parameters (used to generate the data), the yellow dot shows the
starting parameters for the minimization and the green dot indicates the
least-squares parameters:

 |Figure 5|

Possible solutions and tests
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This degeneracy is a big problem if we are going to fit dipole
parameters using the subtracted data alone. Three possible solutions
are:

1. Use starting parameters and parameter bounds based on measurements
   from the pre-subtracted images (obs. and template) to constrain the
   dipole fit.
2. Include the pre-subtracted image data in the fit to constrain the
   minimization.
3. A combination of (1.) and (2.).

It is noted that these solutions may not help in all cases of dipoles
on top of bright backgrounds (or backgrounds with large gradients),
such as cases of a faint dipole superimposed on a bright-ish
background galaxy. But these cases will be rare, and I believe we can
adjust the weighting of the pre-subtracted image data (i.e., in [2]
above) to compensate (see below). An alternative that we will
investigate below is including in the fit parameters for a linear
gradient in the pre-subtracted images as well.

As an example, I performed a fit to the same data as shown above, but
included the "pre-subtracted" data as two additional planes. In this
example, I chose to down-weight the pre-subtracted data points to 1/20th
(5%) of the subtracted data points for the least-squares fit. The
resulting contours are shown below:

 |Figure 6|
 
Unsurprisingly, including the original data serves to significantly
constrain the fit and reduce the degeneracy.

I believe that this is a possible way forward in the dipole
characterization task in ``dipoleMeasurement``. The primary drawback
is if the source falls on a bright background or a background with a
steep gradient - which is why we do the DIA for in the first place -
then the pre-subtraction data may provide an inaccurate measure of the
original source. This will also require passing the two
pre-subtraction planes (and their variance planes) to the dipole
characterization task, and thus a potential slow-down of 3x.

*Recommendation:* Test the dipole fitting including using the additional
(pre-subtraction) data planes, including simulating bright and
steep-gradient backgrounds. Investigate the tolerance of very low
weighting (5 to 10%) on the pre-subtraction planes in order to ensure
that we are "mostly" fitting on the imDiff plane.

This same degeneracy is seen in simulated 2-d dipoles, as shown in
`this notebook
<https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/7c_plot_dipole_fit_error_contours.ipynb>`__.
First, a brief overview. Here is a simulated 2-d dipole and the
footprints for positive and negative detected sources in the image:

 |Figure 7|

and here are the least-squares model fit and residuals:

 |Figure 8|

A contour plot of confidence interval contours shows a similar
degeneracy as that described above, here between dipole flux and
x-coordinate of the positive dipole lobe (below, left). This is also
seen in the covariance between x- and y-coordinate of the positive
lobe centroid, which points generally toward the dipole centroid
(below, right):

 |Figure 9| |Figure 10|

These contours look surprisingly similar for fits to closely-separated
and widely-separated dipoles of (otherwise) similar parameterization
(see the `notebook
<https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/7c_plot_dipole_fit_error_contours.ipynb>`__
for more).

After updating the dipole fit code to include the pre-subtraction images
(again with 5% weighting), as shown in `this
notebook <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/8b_2d_dipole_fitting_with_new_constraints.ipynb>`__,
the fits once again improves.

The new (constrained) result, fitting to the same simulated dipole data
(note the difference in axis limits):

 |Figure 11| |Figure 12|

Adding the constraining data to the fit unsurprisingly improves the
flux fits for a variety of dipole separations (the figure below may be
compared with the similar one shown `above <#figure2>`__, generated
without any constraint).

 |Figure 13|

`DipoleMeasurementTask` refactored as `DipoleFitTask`: implementation details
====================================


Additional recommendations and tests
====================================

1. Complete refactoring of existing ``dipoleMeasurement`` code.
2. Investigate the robustness of the updated (constrained) fitting
   method, including variable backgrounds (with large gradients) that
   are removed in the image difference but bright and noisy in the
   template/science images.
3. Investigate adding the constraints to the existing
   ``dipoleMeasurement`` code, including parameter windowing. This
   will require refactoring of ``diffIm`` code to pass pre-subtraction
   images/heavy footprints to ``dipoleMeasurement``.
4. Investigate `iminuit
   <http://nbviewer.jupyter.org/github/iminuit/iminuit/blob/master/tutorial/tutorial.ipynb>`__
   package - possibly more robust and/or more efficient minimization?

Appendix I. IPython notebooks
=================

All figures and methods investigated for this report were generated
using IPython notebooks. The relevant notebooks may be found `in this
repo
<https://github.com/lsst-dm/dmtn-007/tree/master/_notebooks/>`__. Much
of the code in these notebooks is exploratory; below are the
highlights (i.e., the ones from which the figures of this report were
extracted):

* `Final, versions of direct, benchmarked comparisons
  <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/7b_compare_new_and_old_dipole_fitting.ipynb>`__
  between new "pure python" dipole fitting routines and existing
  ``ip_diffim`` codes on sample dipoles with realistic noise. This
  notebook does not include the "constrained" optimizations but does
  include bounding boxes on parameters during optimization.

* `Demonstration of constructing dipole fit error profiles
  <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/7c_plot_dipole_fit_error_contours.ipynb>`__,
  revealing covariance between dipole source flux and separation.

* `Tests using simplified 1-d dipoles
  <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/8a_1d_dipole_fitting_and_contours.ipynb>`__,
  including demonstrations of flux/separation covariance and
  integration of pre-subtraction data to alleviate the degeneracy.

* `Update the 2-D dipole fits to include the ability to constrain fit
  parameters using pre-subtraction data
  <https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/8b_2d_dipole_fitting_with_new_constraints.ipynb>`__,
  including error contours.

Appendix II. Additional random dipole characterization thoughts
====================================

An informal list of ideas, thoughts and questions (in no particular
order) are located separately, `here
<https://github.com/lsst-dm/dmtn-007/blob/master/_notebooks/README.md>`__.


.. |Figure 1| image:: /_static/figure_01.png
              :width: 100 %
.. |Figure 2| image:: /_static/figure_02.png
              :width: 100 %
.. |Figure 3| image:: /_static/figure_03.png
              :width: 60 %
.. |Figure 4| image:: /_static/figure_04.png
              :width: 60 %
.. |Figure 5| image:: /_static/figure_05.png
              :width: 60 %
.. |Figure 6| image:: /_static/figure_06.png
              :width: 60 %
.. |Figure 7| image:: /_static/figure_07.png
              :width: 70 %
.. |Figure 8| image:: /_static/figure_08.png
              :width: 50 %
.. |Figure 9| image:: /_static/figure_09.png
              :width: 45 %
.. |Figure 10| image:: /_static/figure_10.png
              :width: 45 %
.. |Figure 11| image:: /_static/figure_11.png
              :width: 45 %
.. |Figure 12| image:: /_static/figure_12.png
              :width: 45 %
.. |Figure 13| image:: /_static/figure_13.png
              :width: 100 %
