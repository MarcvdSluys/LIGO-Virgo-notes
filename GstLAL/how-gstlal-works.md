- [Preprocessing](#org1e24a9b)
      - [Measuring the PSD](#orge989883)
      - [Generating injections](#org6fb18b6)
      - [Generating the SVD template bank](#org3657f02)
    - [Filtering](#orgf88e45d)
      - [Filtering data only](#org81347e8)
      - [Filtering data with injections](#org6fa0ba3)
    - [Ranking](#orge01a1a8)
      - [Summary pages](#org02c426f)
    - [Links](#org348fb91)

-   Stefano Schmidt, <span class="timestamp-wrapper"><span class="timestamp">[2022-03-02 Wed]</span></span>
-   See Stefano's run 20220208 on CIT as an example
    -   <https://ldas-jobs.ligo.caltech.edu/~stefano.schmidt/gstlal/run_nonprecessing_20220208>

-   Opera sunt omnis divisa in partes tres:
    1.  preprosessing
    2.  filtering
    3.  ranking


<a id="org1e24a9b"></a>

# Preprocessing


<a id="orge989883"></a>

## Measuring the PSD

1.  take a data segment, divide it in &sim; 10 batches of &sim; 1000s
    -   `reference_psd`
2.  take binwise median of all batches
    -   `median_psd`
    -   `gstlal_median_of_psds`
    -   Welch method: take 32s, FFT, average
    -   file: `MEDIAN_PSD-STARTGPS-LENGTH.xml.gz`
        -   xml contains:
            1.  process table
            2.  psd tables
            3.  one table for each detector
        -   use `python-ligo-lw` (pypi) to deal with xml files
    -   used to compute SNR and whiten the bank
    -   in `median_psd/` subdir 12381 = first five digits of GPS time


<a id="org6fb18b6"></a>

## Generating injections

1.  Uses `split_injection`
2.  injection file: e.g. BBH with injection GPS file
3.  split into several batches (0-100s), each with (probably) some injections
4.  goal: to compute expected SNR (?)
5.  `--usertag` BBH, BNS etc, referring to injection settings in `config.yml`
    -   `config.yml`: `combine: true/false` treat as a single one or not
6.  `lalaps-inspinj` in `Makefile`
7.  file e.g. `bbh_injections.xml`


<a id="org3657f02"></a>

## Generating the SVD template bank

-   bank file: list of BBH components
-   `sgnl_inspiral:table`: for **both** bank and triggers
-   bank is initially list of parameters, not of waveforms
    -   `gst_svd_bank` creates set of waveforms from list of parameters
    -   GSTlal does time-domain filtering
        -   SNR time series: match for a range of time shifts - should see SNR of &sim; 2 + spike
        -   time-domain filtering is slower than frequency-domain filtering
            -   Gstreamer is designed for time-domain filtering
            -   however, you need to do SVD to speed it up
        -   filter data with SVDecomposed data - see paper <https://arxiv.org/abs/1604.04324> -> LLOID method
        -   do this for each part of the total bank
            -   each part consists of &sim; 1000 waveforms
            -   each waveform has &sim; 30000 data points
        -   see dir `split_bank/`
        -   time-domain waveforms: `filter/svd_bank/`
            -   whitened by PSD
            -   whitening: weighting by noise in frequency domain, FFT back to time domain
-   `config.yml`:
    -   `svd` > `approximant`: notation: - `x:y:TaylorF2` means lower chirp-mass limit - upper chirp-mass limit - approximant
    -   `sampling_max`: sampling rates applied below `_64Hz`, below `_256Hz`, etc.
    -   `overlap`: between tiles of the SVD bank
    -   $$ \chi = \frac{\chi_1 M_1 + \chi_2 M_2}{M_1+M_2} $$
-   PE: use &chi; because component spins are degenerate


<a id="orgf88e45d"></a>

# Filtering

There are two types of filtering: with and without injections.


<a id="org81347e8"></a>

## Filtering data only

-   `gstlal_inspiral`: take raw data, filter them using raw SNR time series
-   uses `reference_psd`
-   creates `GSTLAL_TRIGGERS`, `GSTLAL_DIST_STATS` files
-   applied separately for each detector
-   calls gstreamer, black box where the magic happens


<a id="org6fa0ba3"></a>

## Filtering data with injections

-   `gstlal_inspiral_injection`: same, but on data with injections
-   creates e.g. `GSTLAL_TRIGGERS_BBH`, `GSTLAL_DIST_STATS` files
-   want to compare SNR timeseries of data with and without injections to understand how a **signal** trigger looks
-   injections are needed to validate the pipeline every time again, for each analysis
-   also needed for development


<a id="orge01a1a8"></a>

# Ranking

-   Goal of ranking: is a trigger a signal, or a noise feature?
-   trigger: SNR > threshold (e.g. 4)
    -   for such low SNR threshold there may be a trigger every 10s
-   Want likelihood ratio: $$ \frac{p(\mathrm{trigger}|\mathrm{signal})}{p(\mathrm{trigger}|\mathrm{noise})} $$ Ingredients:
    -   coincidences between detectors
    -   &theta; = SVD bank bin
    -   &xi;<sup>2</sup> = compare measured and expected time-series SNRs (labelled as &chi;<sup>2</sup>)
        -   high number indicates a higher probability of a noise feature, hence a lower probability of a signal
    -   new: $\Delta t$, &Delta;&phi; between detectors must be correlated
    -   mass model: astrophyscial priors?
-   In `config.yml`:
    -   prior > mass-model (postprocessing), dtdphi (`dist_stats`)
-   likelihood ratio &rarr; FAR (yr<sup>-1</sup>) &rarr; P<sub>astro</sub>


<a id="org02c426f"></a>

## Summary pages

1.  **Search sensitivity** tab
    -   &sim; volume of universe you would have covered with your search
    -   BNS horizon: BNS 1.4+1.4 at SNR &sim; 10
        -   increases when observing longer, hence $V*T$
2.  **Backgound** tab (for debugging; to check pipeline):
    1.  high &chi;<sup>2</sup>/SNR<sup>2</sup> in $\ln P$ noise plot: cannot have any signals
    2.  in $\ln P$ noise plot: SNR $\gtrsim$ 10: looks like detection
    3.  noise vs. signal plot: focus on $\ln L$ ratio > 0
    4.  "background" means *noise triggers*
    5.  sharp SNR line between missed and recovered injections
        -   e.g. <https://ldas-jobs.ligo.caltech.edu/~stefano.schmidt/gstlal/run_nonprecessing_20220208/background.html>
3.  **Money plots** tab
    -   measure whether the number of triggers at certain FAR matches the expectation from noise
    -   where the thick line veers of to right (probably at N=1): detection


<a id="org348fb91"></a>

# Links

-   Stefano's example run: <https://ldas-jobs.ligo.caltech.edu/~stefano.schmidt/gstlal/run_nonprecessing_20220208>
-   O3 offline runs overview: <https://docs.google.com/spreadsheets/d/10O-H2Fyg9K5X2EfSM68slHQ36Cl6FbfTBDJxeP4fF2o>
-   Offline searches: <https://ldas-jobs.ligo.caltech.edu/~gstlalcbc.offline/>
-   GW 150914: <https://pycbc.org/pycbc/latest/html/gw150914.html>
-   Analysis framework: <https://arxiv.org/abs/1604.04324>
-   Likelihood-ratio ranking statistic: <https://arxiv.org/abs/1504.04632>
-   Basis for HM search: <https://arxiv.org/abs/1709.09181>