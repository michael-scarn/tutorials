Semblance, Coherence, and other Discontinuity Attributes
========================================================

Abstract
--------

Coherence/discontinuity calculations are some of the most commonly used
seismic attributes.  They measure the degree of similarity between adjacent
seismic traces. However, there are several different algorithms to calculate
a discontinuity estimate and many more terms used to refer to those algorithms. 
As a result, it is often difficult to understand the differences between
various similar discontinuity attributes.  While the exact discontinuity
algorithm that any particular proprietary software package uses may be unknown,
most are based on one of a few published methods.  Therefore, it is often
helpful to consider simplified implementations of the most common  algorithms.
In this tutorial and the accompanying [Jupyter/IPython
notebook](https://github.com/seg/tutorials), we'll explore simple Python
implementations of a few widely used discontinuity algorithms.

Introduction
------------

Discontinuity attributes are a variety of post-stack calculations that measure
the degree of local similarity or dis-similarity between seismic traces.
Discontinuities in a seismic volume, such as faults or mass transport deposits,
are apparent as regions of low similarity.  Therefore, discontinuity
attributes are most often applied to highlight faults in a seismic volume,
but are also useful to detect subtle stratigraphic features in map view.
Discontinuity attributes such as semblance-based coherence (Marfurt, et al, 1998),
eigenstructure-based coherence (Gersztenkorn and Marfurt, 1999), and many other
similar algorithms are among the most widely applied seismic attributes today.

However, the terminology used for discontinuity attributes in practice can be
very confusing.  Terms such as "coherence", "semblance", "similarity",
"discontinuity" and many others are used to refer to both the general class
of attribute and to specific algorithms.  For these
reasons, in this tutorial we'll use the term "discontinuity" to refer to the
general class of attribute. Specific algorithms will always include a reference
in addition to the name the original author used to avoid confusion.

In practice, most discontinuity attributes are calculated and applied in
proprietary seismic interpretation packages.  The exact algorithm used by a
particular package is sometimes a considered a trade secret or may be
undocumented.  For that reason, we won't directly refer to or compare
commercially available implementations in this tutorial. However, most
proprietary discontinuity attributes are similar to (or based on) published
algorithms.  Therefore, understanding widely cited published algorithms for
discontinuity is a good first step to understanding the trade-offs between
different commercial implementations.

We'll focus on demonstrating a few of the most cited published discontinuity
algorithms with short Python code snippets using the numpy, scipy, and
matplotlib Python libraries.  The data we'll be working with is a small subset
of the Penobscot 3D seismic dataset from offshore Nova Scotia, owned by the
Nova Scotia Department of Energy and distributed by dGB Earth Sciences under an
CC-BY-SA license (Nova Scotia Department of Energy, 1992) (Figure 1A). These
examples eschew practical considerations (e.g. memory usage and speed) and
focus on demonstrating the algorithm using high-level concepts.  The
accompanying [Jupyter/IPython notebook](https://github.com/seg/tutorials)
expands on these examples and gives runnable versions of these short snippets.

Early Algorithms - Cross-correlation
----------------
The earliest discontinuity algorithm was developed by Bahorich and Farmer
(1995) and used the maximum cross correlation value of three traces.  Bahorich
and Farmer coined the term "coherence" for the attribute, based on its
conceptual similarity to pre-stack methods for estimating stacking velocities (e.g. Taner and Koehler, 1969).
While this exact approach is computationally expensive and not widely used
today, it provides a good starting point to understand later algorithms.

In Bahorich and Farmer's (1995) method, each trace is correlated with a
"moving-window" subset of two neighboring traces. We can demonstrate this with
the python function shown below (Figure 1B):

```
def bahorich_coherence(data, zwin):
    ni, nj, nk = data.shape
    out = np.zeros_like(data)
    padded = np.pad(data, ((0, 0), (0, 0), (zwin//2, zwin//2)), mode='reflect')

    for i, j, k in np.ndindex(ni - 1, nj - 1, nk - 1):
        center_trace = data[i,j,:]
        center_std = center_trace.std()
        x_trace = padded[i+1, j, k:k+zwin]
        y_trace = padded[i, j+1, k:k+zwin]

        xcor = np.correlate(center_trace, x_trace)
        ycor = np.correlate(center_trace, y_trace)

        px = xcor.max() / (xcor.size * center_std * x_trace.std())
        py = ycor.max() / (ycor.size * center_std * y_trace.std())
        out[i,j,k] = np.sqrt(px * py)

    return out
```

Generalization to an Arbitrary Number of Traces
-----------------------------------------------
Bahorich and Farmer's (1995) approach was very successful, but it is
sensitive to noise because only three traces are used. Marfurt, et al (1998)
generalized Bahorich and Farmer's cross-correlation approach to an arbitrary
number of input traces, referred to by the authors as "semblance-based
coherence" (Figure 1C). As an example (Note: we'll reuse the ``moving_window``
function in future examples):

```
def moving_window(data, func, window):
    wrapped = lambda region: region.reshape(window)
    return scipy.ndimage.generic_filter(data, wrapped, window)

def marfurt_semblance(region):
    region = region.reshape(-1, region.shape[-1])
    ntraces, nsamples = region.shape

    square_of_sums = np.sum(region, axis=0)**2
    sum_of_squares = np.sum(region**2, axis=0)
    return square_of_sums.sum() / sum_of_squares.sum() / ntraces

result = moving_window(seismic_data, marfurt_semblance, (3, 3, 9))
```

Conceptually, Marfurt, et al's (1998) algorithm treats each seismic trace
within the moving window as a separate dimension and measures how close the
resulting point cloud is to a hyper-plane with a slope of 1 in all directions
and intercept of 0.  It's easiest to visualize for the case of two traces, as
shown in Figure 2C.  In that case, semblance-based coherence computes how close
the points fall to a line with a slope of 1 and intercept of 0.

Removing Amplitude Sensitivity
------------------------------

One caveat to both Marfurt, et al's (1998) and Bahorich and Farmer's (1995)
method is that they're sensitive to lateral differences in amplitude as well as
differences in phase.  While this is desirable for detecting stratigraphic
features, differences due to lateral changes in amplitude can obscure subtle
structural features.  Gersztenkorn and Marfurt (1999) proposed an
implementation that is sensitive only to lateral changes in phase of the input
waveforms, and not to changes in amplitude: "eigenstructure-based coherence" (Figure 1D, Figure 2D).

Eigenstructure-based coherence (Gersztenkorn and Marfurt, 1999) computes the
covariance matrix of the input region, similar to the previous example.
However, it uses the ratio of the largest eigenvalue of the covariance matrix
to the sum of the eigenvalues:

```
def gersztenkorn_eigenstructure(region):
    region = region.reshape(-1, region.shape[-1])

    cov = region.dot(region.T)
    vals = np.linalg.eigvalsh(cov)
    return vals.max() / vals.sum()

result = moving_window(seismic_data, gersztenkorn_eigenstructure, (3, 3, 9))
```

Conceptually, this is similar to treating each seismic trace within the moving
window as a separate dimension and calculating how well the resulting point
cloud is fit by a plane.  To contrast with Marfurt, et al's (1998)
method, for the case of two traces, this measures the scatter about the
best-fit line instead of a line with a slope of 1, as shown in Figure 2D.

Further Enhancements
--------------------

A drawback to Marfurt, et al's (1998) and Gersztenkorn and Marfurt's (1999)
approaches is that dipping reflectors will have a uniformly higher
discontinuity (lower coherence/similarity) than non-dipping reflectors. In
other words, these attributes don't distinguish between regional structural dip and
localized discontinuities due to faulting, etc. Therefore, Marfurt
(2006) proposed calculating and correcting for structural dip when performing
discontinuity calculations.   While there are a number of different
methods that can be used to both calculate and correct for structural dip (see
Ch. 2 of Chopra and Marfurt (2007) for a review), dip calculations are beyond
the scope of this tutorial.  Therefore, we'll approximate a dip correction by
flattening on a pre-picked horizon in the interval of interest before applying
a discontinuity calculation (Figure 1E).

Furthermore, a number of other authors have proposed other discontinuity
attributes which are not covered in this tutorial (See Ch. 3 of Chopra and
Marfurt (2007) for a thorough review). I encourage you to try implementing
some of these methods on your own based on this tutorial and the associated
IPython/Jupyter notebook.

References
----------

Bahorich, M., and S. Farmer, 1995, 3-D seismic discontinuity for faults and stratigraphic features: The coherence cube: The Leading Edge, **14**, 1053-1058.
doi:10.1190/1.1437077 

Chopra, S., and K. J. Marfurt, 2007, Seismic Attributes for Prospect Identification and Reservoir Characterization: **SEG**
doi:10.1190/1.9781560801900

Gersztenkorn, A., and K. J. Marfurt, 1999, Eigenstructure‐based coherence computations as an aid to 3-D structural and stratigraphic mapping: GEOPHYSICS, **64**, 1468-1479.
doi:10.1190/1.1444651 

Marfurt, K., V. Sudhaker, A. Gersztenkorn, K. D. Crawford, and S. E. Nissen, 1999, Coherency calculations in the presence of structural dip: GEOPHYSICS, **64**, 104-111.
doi:10.1190/1.1444508

Marfurt, K. J., 2006, Robust estimates of 3D reflector dip and azimuth: GEOPHYSICS, **71**, no. 4, P29-P40.
doi:10.1190/1.2213049

Nova Scotia Department of Energy, 1992, Penobscot 3D Survey. Dataset accessed 19 October, 2015 at
https://opendtect.org/osr/pmwiki.php/Main/PENOBSCOT3DSABLEISLAND

Taner, M. T., And F. Koehler, 1969, Velocity Spectra—Digital Computer Derivation Applications of Velocity Functions: Geophysics, **34**, 859-881.
doi:10.1190/1.1440058 

Figures
-------

![Figure 1](images/figure_1.png)

**Figure 1**: A comparison of the discontinuity algorithms discussed using the Penebscot 3D seismic dataset (Nova Scotia Department of Energy, 1992).


![Figure 2](images/figure_2.png)

**Figure 2**: A visual explanation of the differences between the discontinuity algorithms discussed using only two traces.