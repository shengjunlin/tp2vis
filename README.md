# tp2vis -- SJL version
Total Power to Visibilities (TP2VIS): an updated version by Sheng-Jun Lin (ASIAA)

TP2VIS requires CASA versions **5.4.0 to 5.8.0 with Python 2 compatibility**.

## Release note

### 1. tp2vispl:

This function selected the line-detected channel based on the single-dish LSRK frequency, whereas the ALMA 7m and 12m visibilities are stored in the TOPO frame. The original code attempted to extract the same frequency channel from the TOPO-frame visibilities, which may miss the actual line due to the frame mismatch. This issue is particularly critical when the SPW has a narrow bandwidth, as the LSRK frequency from the single-dish data may fall entirely outside the 7m/12m TOPO bandwidth range, causing the 7m/12m data to be skipped during the visibility extraction.

This revised version instead extracts the channel with the peak intensity from each of the single-dish-`tp2vis`ed visibilities, as well as from the 7m and 12m visibilities, under the assumption that the selected channels correspond to similar LSRK frequencies. Note that the original code identified the line-detected channel using only the first SPW. Therefore, this workaround also assumes that the target line is located in the first SPW of the visibilities from all three arrays. If that is not the case, it is recommended to remove unnecessary SPWs from measurement sets before running the code.

A comparison between the original and revised versions. Red dots indicate the single-dish data, green dots represent the 7m array, and blue dots represent the 12m array.

![plot1](figures/tp2vispl_plots.png)

### 2. New feature: Support for other single-dish telescopes:

A new variable `SDtele` is introduced to specify the single-dish telescope, allowing `tp2vis.py` to select the appropriate beam size for the deconvolution step by modifying the variables `t2v_arrays['ALMATP']` and `t2v_arrays['VIRTUAL']`.

    SDtele = 'TP'; execfile('tp2vis.py')        # load tp2vis in ALMA TP mode (dish size = 12m)
    SDtele = 'TRAO'; execfile('tp2vis.py')      # load tp2vis in TRAO mode (dish size = 13.7m)
    SDtele = 'JCMT'; execfile('tp2vis.py')      # load tp2vis in JCMT mode (dish size = 15m)
    SDtele = 'NRO'; execfile('tp2vis.py')       # load tp2vis in NRO mode (dish size = 45m)

It is now required to define `SDtele = 'TP'` before running `execfile('tp2vis.py')`, even with ALMA TP case.

## Download

Click "Clone or download" on the top for download options, or run

       git clone https://github.com/shengjunlin/tp2vis.git


## Usage

This example shows a quick look of the visibility amplitudes from three arrays.

### 1. Preparations

The following should be prepared.

* `12m.ptg`: A text file which contains the 12m (mosaic) field pointing(s).

* `12m.ms`, `7m.ms`: The 12m/7m measurement sets with only one SPW containing the target line.

* `SD.im`: The single-dish cube in the CASA format and in the units of Jy per beam.

* `SD_rms_Jybeam`: The RMS in the single-dish line-free channels in the units of Jy per beam.

Please see [below](#preparations) for the details.

### 2. Load and run TP2VIS

    !ln -s $TP2VIS_folder/tp2vis.py       # Make a symbolic link at the working directory
    SDtele = 'TRAO'; execfile('tp2vis.py')  # Load tp2vis in TRAO mode
    tp2vis(infile='SD.im', outfile='SD.ms', ptg='12m.ptg', maxuv=13.0, rms=SD_rms_Jybeam, deconv=True)

"`maxuv`" sets the maximum uv distance in meter for the output single-dish visibilities. A value slightly smaller than the dish size is perferred (e.g., 10m for the ALMA TP 12m dish). Other appropriate values should be given for the other single dishes.

    tp2vispl(mslist=['SD.ms', '7m.ms', '12m.ms'], outfig='plot_tp2viswt_rms.png')

The resulting plot shows the visibility amplitudes are well connected from the short to long baselines.

<img src="figures/tp2vispl_amp.png" width="300">


# tp2vis -- The original public distribution
Total Power to Visibilities (TP2VIS): an ALMA Cycle 4 Development Study

Jin Koda, Peter Teuben, Adele Plunkett, Tsuyoshi Sawada, Crystal Brogan, Ed Formalont

This project provides tools to create visibilities from a single dish cube using the method of [Koda et al. 2011](http://adsabs.harvard.edu/abs/2011ApJS..193...19K) and Koda et al. 2019 (PASP in press).
The TP visibilities can then be combined with the interferometric visibilities in a joint deconvolution using for example CASA's [**tclean()**](https://casa.nrao.edu/casadocs/latest/global-task-list/task_tclean/about) method.
TP2VIS requires **CASA 5.4 or above** and as powerful a computer as what the CASA Feather guide requires.

Our github repo for distribution : https://github.com/tp2vis/distribute


## Release note

This release assumes that users have experience on interferometer data reduction with CASA so that they can catch any anomaly by themselves. This release is ready for scientific use, but please let us know if you encounter any problem.

The following acknowledgement would be appreciated if you decide to use TP2VIS: "This work made use of TP2VIS (Koda et al. 2011 ApJS, 139, 19; Koda et al. 2019, PASP in press)".


## Download

Click "Clone or download" on [top page](https://github.com/tp2vis/distribute) for download options, or run

       git clone https://github.com/tp2vis/distribute.git

You need only one script "tp2vis.py".

## Usage

To give you a quick idea how to run TP2VIS, here are the basic flow of commands in CASA, broken into 6 pieces. This document gives an overall flow, but look at examples listed below for more info.


### 1. Preparations:<a name="preparations"></a>

#### 1.1: Make a pointing (**ptg**) file

       listobs('calibrated_12m.ms',listfile='calibrated_12m.log')

after which you can cut and paste the appropriate **Fields**
in the following format into a pointing file, **12m.ptg**, which has to be a simple text file:

       cat 12m.ptg
       
       J2000 05h39m45.660s -70d07m57.524s
       J2000 05h39m54.340s -70d07m57.524s
       J2000 05h39m41.320s -70d07m19.175s
       ...

This can be a little cumbersome, so in the examples listed below you can see examples using awk and grep.

#### 1.2: Find a reasonable RMS

We need to know the RMS in the TP cube from some line free channels. For example, you might be able to use the first 10 channels of your TP cube

       importfits(fitsimage='tp.fits',imagename='tp.im')              # convert to CASA image format
       imstat('tp.im',axes=[0,1])['rms'][:10].mean()
       -> 0.67

#### 1.3: Cutting down the 12m & 7m MS dataset sizes.

Cut down unnecessary spws from measurement sets as tp2vis assumes all spws to be used for imaging.
CASA tasks such as
[**split()**](https://casa.nrao.edu/casadocs/latest/global-task-list/task_split/about)
and
[**mstransform()**](https://casa.nrao.edu/casadocs/latest/global-task-list/task_mstransform/about)
can be useful.


### 2. Load and run TP2VIS:

       execfile('tp2vis.py')                                          # load tp2vis 
       tp2vis('tp.im','tp.ms','12m.ptg',rms=0.67)                     # make visibilities

### 3. [optional] Expert mode Weighting Schemes

       tp2viswt('tp.ms', ...)                                         # set weights
       tp2vispl(['12m.ms','7m.ms','tp.ms'])                           # (optional) plot weights

"tp2viswt" shows weight statistics and manipulates weights. There are several modes how you can set weight, described in [example1](example1.md). "tp2vispl" plots the weights.

### 4. Some CASA workarounds to get files recognized properly

In case tclean crashes in the next step (perhaps due to an inconsistency in CASA versions), there is no single workaround. The below worked in some cases (more in [example1](example1.md), section 4):

       concat(vis=['12m.ms','7m.ms','tp.ms'], concatvis='all.ms',copypointing=False)

The "copypointing=False" option is important. No worries, important pointing info still remain in 'all.ms' (one of CASA mysteries!).

### 5. finally the joint deconvolution using CASA's ``tclean()``

       tclean(vis='all.ms', imagename='all_clean', ...)                     # run clean the way you like it

Where ... represents the large number of options to control the deconvolution. For example, users may try "robust" and "uvtaper" options.

Make dirty images as well

       tclean(vis='all.ms', imagename='all_dirty', niter=0, ...)                     # make dirty map

### 6. Correction for beam size mismatch

This step is for general interferometer imaging, not only for TP2VIS.

Once dirty and cleaned images are generated, one may correct for the discrepancy between dirty and clean/restore beam areas (see [**Jorsater and van Moorsel 1995**](http://adsabs.harvard.edu/abs/1995AJ....110.2037J)).

       tp2vistweak('all_dirty','all_clean')                   # adjust dirty beam size in residual image

It assumes that all outputs from tclean() are kept intact (no name change). It creates ``.tweak.image`` and ``.tweak.residual``, which have a correct flux scale.

## Examples

* [example1:](example1.md)  M100 (data from CASA guide on [**Feather**](https://casaguides.nrao.edu/index.php/M100_Band3_Combine_4.3))

## Comparisons of 7m+12m vs TP+7m+12m clean maps

**The spiral galaxy M100:**

Negative sidelobes around strong emissions in the 7m+12m map (left), but not in the TR+7m+12m map (right).

![plot1](figures/M100_07m12m_TP07m12m_clean.png)

**A giant molecular cloud in the Large Magellanic Cloud:**

Most extended emissions are not recovered in the 7m+12m map (left), but recovered in the TR+7m+12m map (right).

![plot1](figures/Cloud197_07m12m_TP07m12m_clean.png)


## References

* Koda et al. 2011, ApJS, 193, 19 : http://adsabs.harvard.edu/abs/2011ApJS..193...19K

* Koda, Teuben, Sawada, Plunkett & Fomalont 2019, PASP, 131, 54505: https://ui.adsabs.harvard.edu/abs/2019PASP..131e4505K

* CASA reference manual and cookbook : http://casa.nrao.edu/docs/cookbook/
  * Measurement Set: https://casa.nrao.edu/casadocs/latest/reference-material/measurement-set

* Jorsater and van Moorsel 1995, AJ, 110, 2037 : http://adsabs.harvard.edu/abs/1995AJ....110.2037J


## Acknowledgements

We thank the NRAO staff, in particular, Remy Indebetouw, Kumar Golap, Jennifer Donovan Meyer, Crystal Brogan, and John Carpenter for their help. We also thank Kazuki Tokuda, Fumi Egusa, Manuel Fernández, and Mercedes Vazzano for feedback on an early version of TP2VIS.
