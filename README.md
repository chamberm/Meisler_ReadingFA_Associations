# Meisler_and_Gabrieli2021
Code used in Meisler and Gabrieli 2021

This repository includes the code used to preprocess the neuroimaging data, segment white matter tracts, perform tractometry, and compare cohort characteristics. The preprocessing and white matter segmentation is agnostic to the data set. That is, it can be run on **any BIDs** compliant dataset with T1w and diffusion data. The tractometry and cohort characteristic code is tailored to work with a Healthy Brain Network (HBN) phenotypic query, but can be adapted to work with a custom file.

## Requirements
### Additional requirements if downloading HBN data
- Data Usage Agreement (if working with Healthy Brain Network data), used to access neuroimaging and phenotypic data
- Amazon Web Services (AWS) Command Line version 2 (or any version with s3 capabilities), used to download neuroimaging data
### In general
- BIDS-compliant dataset with _at least_ T1w and DWI images
- Singularity (used to compile and run Docker Images)
  - QSIPrep docker container (`singularity build qsiprep.simg docker://pennbbl/qsiprep:0.13.0RC1`)
  - TractSeg docker container (`singularity build tractseg.simg docker://brainlife/tractseg:2.2`)
  - Combined MrTrix 3.0.2 w/ FSL 6.0.4 and ANTs 2.3.4 container (`singularity build mrtrix_fsl.simg docker://sclove/mrtrix3.0.2`)
  - If using your own single-shelled DTI data (not HBN), then you will also need mrtrix3tissue (`singularity build mrtrix3t.simg docker://vnmd/mrtrix3tissue_5.2.8:20201111)
  - Singularity 3.6.3 was used in this study. The above reference the images used in this study, but later versions of these software may introduce improvements that should be used in future research.
- SLURM job scheduler, used for parallelizing jobs.
- Python environment with Jupyter capabilities and the following packages: numpy, scipy, pandas, glob, matplotlib, json, filecmp (most are standard Anaconda packages)
- FreeSurfer license (https://surfer.nmr.mgh.harvard.edu/fswiki/License)

## Step 0: Download and prepare HBN data (if using HBN)
- Obtain a Data Usage Agreement: http://fcon_1000.projects.nitrc.org/indi/cmi_healthy_brain_network/Pheno_Access.html#DUA
- In your terminal, navigate in to the directory where you want your BIDS HBN data to live. Then, download the neuroimaging data with: `aws s3 cp s3://fcp-indi/data/Projects/HBN/MRI/Site-RU/ $path/to/HBN_dir --exclude "derivatives/*" --recursive`, where  `$path/to/HBN_dir` should be replaced by the path to your directory. This downloads the raw data from the Rutgers University site as used in our study, excluding preprocessed derivatives. You can change `RU` to look at other sites (for example `CBIC` for the Cornell site).
- To prepare the HBN data for QSIPrep, we need to update the DWI JSON files so the DWI fieldmaps can be assosciated with the DWI NIFTI files. This involves:
  1) Rename the fieldmap files to match BIDs conventions. In your BIDs code directory, download`rename_fmaps.sh`, update the variable `bids` in the first line to direct to your BIDs directory, and run the code.
  2) Add "IntendedFor" fields in the JSON to associate fieldmaps with NIFTIs. Download `add_intended_for.ipynb` to your BIDS code directory, update the variable `bids` in the first line to direct to your BIDs directory, and run the code/notebook.
- Open a fieldmap JSON file to make sure this field has been added, and that the name convention matches BIDS specification (https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/01-magnetic-resonance-imaging-data.html).
  - `sub-<label>[_ses-<label>][_acq-<label>][_ce-<label>]_dir-<label>[_run-<index>]_epi.json`
  - `sub-<label>[_ses-<label>][_acq-<label>][_ce-<label>]_dir-<label>[_run-<index>]_epi.nii[.gz]`
- Using the HBN Loris portal (https://data.healthybrainnetwork.org/main.php) make a phenotypic query with _at least_ the following fields: "Basic_Demos", "TOWRE", and "EHQ"
  - If you have proof of a Data Use Agreement, we can share the phenotypic file we used in the study

## Step 1: Run QSIPrep to Preprocess DWI
- Download the `code/QSIPrep` folder to your BIDS code folder. In it, add your FreeSurfer license as `license.txt` (case-sensitive).
- In the `submit_job_array.sh` script, udpate the variable `base` in the beginning of the script to direct to your BIDS directory. Additionally, in the last line of the script, you can update the parameter after `%` to limit how many jobs can be active at a time. We set this to 100 as a default, but you can alter this or delete it to not set a limit.
- In the `ss_qsiprep.sh` script, change the SBATCH header to match your desired parameters (e.g. memory usage, time-to-wall, etc). Then update the variable `IMG` to the path of your QSIPrep singularity image. Set the variable `scratch` to where you want intermediate files to be stored during the QSIPrep workflow. Review the command at the bottom of the script to make sure you understand how QSIPrep is being run and change any parameters you would like.
- In terminal, navigate to the QSIPrep code folder, and run `./submit_job_array.sh` to begin QSIPrep.

## Step 2: Run TractSeg segmentation
- Download the `code/TractSeg` folder to your BIDS code folder.
- In the `submit_job_array.sh` script, udpate the variable `base` in the beginning of the script to direct to your BIDS directory. Additionally, in the last line of the script, you can update the parameter after `%` to limit how many jobs can be active at a time. We set this to 100 as a default, but you can alter this or delete it to not set a limit.
- If using your own data, change the `single_multi` (either 'single' or 'multi') and `flip_x` variables ('yes' or 'no') to match your data. That is, single shell data only has one non-0 b-value, and sometimes, the x-gradient needs to be flipped to correct its orientation. The `flip_x` may be hard to know _apriori_, but typically Siemens data need it flipped. If unsure, leave it as 'yes', and if tracts do not look right, try changing it.
- In the `ss_tractseg.sh` script, update the SBATCH header as needed to accomodate your memory usage needs, and update the first few variables to the paths to your singularity containers.
- In terminal, navigate to the TractSeg code folder, and run `./submit_job_array.sh` to begin TractSeg.

## Setp 3: Perform Tractometry and Compare Cohort Characteristics
Save your HBN phenotypic data as `HBN_query.csv` and place it in your BIDS code directory. Download `HBN_query.ipynb` to your BIDS code directory, open it, and follow the directions in the notebook. This is mainly tailored towards analyzing HBN data but can be adapted to look at other data.

## Questions? Feel free to either open an issue in this repository or email Steven Meisler (smeisler@g.harvard.edu) with any problems, suggestions, or anything else!

## Extra documentation
- Healthy Brain Network Data Portal (http://fcon_1000.projects.nitrc.org/indi/cmi_healthy_brain_network/index.html)
- TractSeg Website (https://github.com/MIC-DKFZ/TractSeg)
- QSIPrep Website (https://qsiprep.readthedocs.io/en/latest/)

## License

MIT License

Copyright (c) 2021 Steven Meisler

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
