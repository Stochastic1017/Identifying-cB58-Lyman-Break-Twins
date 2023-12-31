# Finding the Cosmic Needle: Unraveling cB58 Resemblance in a Galactic Haystack Using High-Throughput Computing

## Introduction

<p align="justify">
This project involves the identification of a novel gravitationally lensed Lyman-break galaxy that closely mirrors the target spectrum CB58. Lyman-break galaxies are those undergoing active star formation at high redshifts, selected based on the distinct appearance of the galaxy in various imaging filters, influenced by the position of the Lyman limit. The spectral data were captured through the Sloan Digital Sky Survey (SDSS), a significant multi-spectral imaging and spectroscopic redshift survey conducted with a dedicated 2.5-m wide-angle optical telescope at the Apache Point Observatory in New Mexico, United States. The dataset was generously provided by Christy Tremonti, an astronomer affiliated with the University of Wisconsin - Madison and a member of the Sloan Digital Sky Survey III (SDSS-III) collaboration. There are approximately 2.5 million spectral data ($\approx$ 281 gigabytes .fits files), and we are interested the ten noisy spectras that most closely resemble the template spectra by calculating red-shifted distance metrics for each spectral data in parallel using University of Wisconsin - Madison's Center For High-Throughput Computing via HTCondor. 
</p>

## Reading and interpreting .fits files in R

The `R` library `FITSio` contains the function `ReadFrameFromFITS()` that allows us to load .fits as a dataframe in R. Supposing we wish to load an `arbitrary.fits` as a dataframe, we write the following code:

``` r
require("FITSio")
df <- readFrameFromFITS("arbitrary.fits")
print(df)
```

The data frame for each of the spectra has these following columns:

1. `flux` is light intensity at a given wavelength. It is theoretically nonnegative, but with noise it can be negative.
2. `loglam `is `log(x=wavelength, base=10)`, so `wavelength = 10^loglam`. 
3. `ivar` (“inverse variance”) is $\frac{1}{S_i^2}$ where $S_i^2$ is an estimated variance of the $i$-th flux.
4. `and_mask` is 0 for a good observation. We exclude data with nonzero `and_mask`.
5. `or_mask` is 0 for a good observation.
6. `wdisp` is the resolution of the spectrograph at that wavelength.
7. `sky` is the spectrum of the sky, mostly already subtracted out from flux.
8. `model` is SDSS’s hypothesis of the true spectrum, shifted to the redshift of the object.

We use only `flux` and its corresponding `index` for this project. 

## Standardization

We standardize all `flux` values as follows before computing a distance metric:
$$z = \frac{x - \mu}{\sigma} \sim N(0,1)$$

``` r
standardize <- function(flux)
{
  return (scale(flux, center = mean(flux), scale = sd(flux)))
}
```

## Distance Metric

We implement Minkowski distances with p=2, i.e., euclidean distances at each flux of a spectra red-shifted.

``` r
minkowski  <- function(x, y, p)
{
  return (sum(abs((x-y)^p))^(1/p))
}
```

Our target spectra (cB58) has 2181 size vector values of `flux`. For noisy spectras that we compare to cB58, if the vector size is smaller than 2181, it is ignored. Else we compute distances at each red-shifted cB58 onto that noisy spectrum until we find the minimum distance and save the result.

``` r
standardize_minkowski <- function(cB58, spectra, p)
{
  n <- length(cB58)
  m <- length(spectra)
  dist  <-  c()

  if (n > m) # cB58 is larger than the other spectra
  {
    next
  }
  
  else (m > n)
  {
    cB58 <- standardize(cB58)
    for (i in 1:(m-n+1)) # Red-shifting at each index
    {
      temp  <- minkowski(cB58, standardize(spectra[i:(i+n-1)])[, 1], p)
      dist  <- append(dist, temp)
    }
  }
  return (c(min(dist), which(dist == min(dist))))
}
```

To allow the code to loop through each .fits file in a directory, and write an output .csv file whose name is the data directory name in the following format:
  * `distance` : Measure of the distance from this spectrum to the template.
  * `i` : red-shifted index in the spectrum at which your alignment with the template begins.
  * `spectrumID` : the spectrum ID, e.g., spec-1353-53083-0579.fits

``` r
files <- list.files(dir, pattern = 'fit*') # save all files as a list

for (file in files)
{
  cat("On File:", file, "\n")
  path_to_file = paste(sep = "", sprintf('%s/', dir), file)
  noisy = readFrameFromFITS(path_to_file) # Interested spectrum in this iteration
  result  <- rbind(result, c(standardize_minkowski(cB58$FLUX, noisy$flux, p = 2), file))
}

result <- na.omit(result)
write.csv(result, file = sprintf("%s.csv", dir), row.names = FALSE)
```

## Visualizing Template cB58 Spectrum

The standardized template cB58 spectra looks as follows:

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/Standardized_cB58.png" width="700" height="500">

## Data Preprocessing

The 2.5 million .fits files were written into 2459 .tgz files, where each .tar file contained approximately 1000 .fits files ($\approx$ 100 mb each), and the data was stored in one of CHTC's approved distinct location (located in `~/data/tgz`). The template spectra is named as `cB58_Lyman_break.fit` (located in `~/data`), and each of the .tar file is of the form `[0-9][0-9][0-9][0-9].tgz` (example 5377.tgz).

## Bash and Shell Scripts

Various bash/shell scripts were written that performed various tasks before, during, and after running parallel jobs to find the top 10 closest spectras to cB58:

1. [`list.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/list.sh): Finds out the names of all .tgz files in the directory `~/data/tgz`, and writes them in the directory `~/minkowski/files`. This is done **before** job is submitted via HTCondor.
2. [`executable.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/executable.sh): Unpacks `R` (4.1.3), unpacks the `FITSio` package, tells bash where `R` and its packages are, unpacks the current .tgz file (like 3586.tgz), and runs [`minkowski_spectra.R`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/R/minkowski_spectra.R) on that directory (like 3586). This is done **during** the job, in parallel at each compute node of the CHTC, after running `list.sh`.
3. [`merge.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/merge.sh): Merges all 2459 .csv files into one, and writes the best 100 spectra to `100_minkowski_best.csv`. This is done **after** all the parallel `executable.sh` jobs are run.
4. [`pull_fits.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/pull_fits.sh): Applies regex to extract the respective .tgz files and spectrumID from [`100_minkowski_best.csv`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/100_minkowski_best.csv) and the files are copied to the directory `~/minkowski/best`. This is dome **after** running `merge.sh`.

## Condor Submit Script

We first write `minkowski_spectre.R` such that it can take command terminal arguements as inputs of the form `<template spectrum>` (cB58_Lyman_break.fit) `<data directory>` (unpacked .tgz file):

``` r
print('Inputing command line prompts ...')
args = (commandArgs(trailingOnly = TRUE))
if(length(args) == 2){
  template = args[1]
  dir = args[2]
} else {
  cat('usage: Rscript minkowski_spectra.R <template spectrum> <data directory>\n', file = stderr())
  stop()
}
```

The corresponding .sub file to send the 2459 parallel jobs to CHTC clusters (called [`condor.sub`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/condor/condor.sub))

``` shell
universe = vanilla
log    = log/minkowski-chtc_$(file).log
error  = error/minkowski-chtc_$(file).err
output = output/minkwoski-chtc_$(file).out

executable = ./executable.sh

arguments = cB58_Lyman_break.fit $(file)

should_transfer_files = YES
when_to_transfer_output = ON_EXIT
transfer_input_files = http://proxy.chtc.wisc.edu/SQUID/chtc/el8/R413.tar.gz,
                       https://pages.stat.wisc.edu/~jgillett/DSCP/CHTC/callingR/packages_FITSio.tar.gz,
                       ~/data/cB58_Lyman_break.fit,
                       ~/data/tgz/$(file).tgz,
                       minkowski_spectra.R

request_cpus = 2
request_memory = 500MB
request_disk = 500MB

queue file from files
```

The [`output`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/tree/main/output) and [`log`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/tree/main/log) files can be found for the first 50 jobs run in parallel. 

## Output Visualization and Interpretation

The top ten cB58 resemblance noisy spectras are given below:

| distance     | i            | spectrumID   |
|--------------|--------------|--------------|
| 39.778417825 | 1257         | spec-6052-56092-0294.fits |
| 39.949355491 | 589          | spec-5436-56015-0440.fits |
| 40.136308215 | 185          | spec-6452-56366-0270.fits |
| 40.722211411 | 1526         | spec-5783-56017-0903.fits |
| 41.189031229 | 29           | spec-3804-55267-0936.fits |
| 41.511091326 | 1080         | spec-5172-56071-0705.fits |
| 41.526470563 | 754          | spec-6184-56267-0854.fits |
| 41.552732019 | 1156         | spec-3665-55247-0718.fits |
| 41.714083619 | 1164         | spec-4221-55443-0556.fits |
| 42.019932552 | 356          | spec-3866-55623-0134.fits |

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-6052-56092-0294.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-5436-56015-0440.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-6452-56366-0270.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-5783-56017-0903.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-3804-55267-0936.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-5172-56071-0705.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-6184-56267-0854.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-3665-55247-0718.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-4221-55443-0556.png" width="700" height="500">

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/spec-3866-55623-0134.png" width="700" height="500">

