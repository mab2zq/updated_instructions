# The Radiomics Process:
Obtain raw data for input (CT scans, contours of particular organs) mostly done by Krishni and Marco and others

Run raw data through pyradiomics software to obtain numerical “features” that describe the raw data (voxel intensity, voxel entropy, etc.) this is what John’s code is for

Visualize this data using Jupyter notebooks (feel free to branch out if you like but this is a great platform and is very user friendly) this is where a lot of the work comes in, but lots of potential cool results

### Links:

Main Github link: [johnmatter/sbrt_radiomics](https://github.com/johnmatter/sbrt_radiomics)

Old instructions: [Basic instructions · johnmatter/sbrt_radiomics Wiki](https://github.com/johnmatter/sbrt_radiomics/wiki/Basic-instructions) 

PyRadiomics Documentation: [Radiomic Features — pyradiomics v3.0.1.post15+g2791e23 documentation](https://pyradiomics.readthedocs.io/en/latest/features.html#)

Accessing Jupyter Notebook via Rivanna [Jupyter Lab on Rivanna | Research Computing](https://www.rc.virginia.edu/userinfo/rivanna/software/jupyterlab/?msclkid=0386d39db4e811eca7c7bc2db7bf3717)

Coding with Pandas Dataframe [pandas.DataFrame — pandas 1.4.2 documentation](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html)




## Connect to rivanna and clone this repo
First, we need to connect to rivanna and clone this repository.

I usually connect to rivanna by [connecting to the UVA VPN](https://in.virginia.edu/vpn) and then using ssh in a terminal.

Throughout these instructions I will refer to `/nv/vol141/phys_nrf/YourName` as _your `/nv` directory_.

```
ssh username@rivanna.hpc.virginia.edu
cd /nv/vol141/phys_nrf/YourDirectory
git clone https://github.com/johnmatter/sbrt_radiomics
```

For Windows computers connecting to rivanna is more difficult. Some of the options are:

-Disk partition with a Linux Operating system

-Linux Subsystem for Windows

-[Cygwin64 terminal](https://www.cygwin.com/)

-[Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) with [X-server](https://teamdynamix.umich.edu/TDClient/47/LSAPortal/KB/ArticleDet?ID=1797)

-[Virtual Machine](https://www.virtualbox.org/)

-[Online FastX](https://rivanna-viz.hpc.virginia.edu:8000/auth/ssh/)

-[FastX Client](https://www.starnet.com/fastx/)

-Cmd/Powershell with [X-server](https://teamdynamix.umich.edu/TDClient/47/LSAPortal/KB/ArticleDet?ID=1797)

I would recommend the FastX Client.

## Setup your environment
Now we need to setup our environment.

The following lines will create a local copy of python and any necessary libraries in your `/nv` directory.

Any time you want to run code interactively (as opposed to via SLURM), you'll need to source this `activate` script.
```
python -m venv sbrt_radiomics_env
source sbrt_radiomics_env/bin/activate
pip install numpy matplotlib pandas pydicom pynrrd pyradiomics scipy SimpleITK
```

## Get the data
Now we need to get a copy of the raw data so we can process it. Run the following from your `/nv` directory:
```
mkdir patients

# We need a link to this directory inside the `sbrt_radiomics` directory
cd sbrt_radiomics
ln -s /nv/vol141/phys_nrf/YourName/patients data

# Now we'll create links to the DICOM files with the necessary subdirectory structure
# I created a bash script to do this
ln -s $(pwd)/src/setup/ln.sh data/
cd data
./ln.sh
```

## Try preprocessing a patient
To make sure everything works properly, we're going to try preprocessing a patient's data.

First, make sure you've `source`-ed the `activate` script from your virtual environment.

Then `cd` to your copy of the `sbrt_radiomics` repo.

Currently, you should only see a bunch of DICOM files in the `clinical` directory. Check this with:
```
ls data/BE/Pre
ls data/BE/Pre/clinical
```

Now we'll hit _go_ on the preprocess script:
```
cd src/preprocess
./preprocess.sh BE/Pre
```

It should take 10 or 20 minutes to run.

After it finishes, you should see a bunch of NRRD files and one pickle file:
* `contours.pickle`
* `ct_img.nrrd`
* `dose_in_CT_dimensions.nrrd`
* A bunch of mask NRRDs in the `masks` directory

If this is what you see, congratulations! You have successfully preprocessed some data.

You should delete `dose_in_CT_dimensions.nrrd` for this data set. If you're looking at radiomics as a function of delivered dose you will need this file.

If everything ran smoothly, you should probably also delete `contours.pickle`. These files tend to be huge, and we need to be conscious of how much data we're using as a group on `/nv`.

If you run into any problems here, shoot me an email (matter AT virginia) and I'll help you sort it out.

## Extract some radiomics
There are a bunch of ways you could extract radiomic features for any image.

For this tutorial, you will use a script I wrote (at Krishni's suggestion) that calculates radiomic features for various regions of the aorta.

From your `/nv` directory, execute the following:
```
cd sbrt_radiomics/src/radiomics/one_off/binned_in_z/
./radiomics.sh BE/Pre
```

This should take about 20 minutes to run. After it finishes, you'll see a bunch of CSV files in `data/BE/Pre/radiomics`.

Each file corresponds to a different mask.

Each line in a CSV is a different region of the aorta.

Each row is either a radiomic feature, patient info, or some info about the version of python/pyradiomics/etc we used.

## SLURM

Now that you've run one patient by hand, it's time to automate the process using [SLURM](https://www.rc.virginia.edu/userinfo/rivanna/slurm/).

I wrote some scripts to automate/standardize the process of creating SLURM jobs for preprocessing and radiomic feature extraction.

### Change the email field
On line 5 of the following files, please put your email after `--mail-user=`:
* `sbrt_radiomics/slurm/preprocess_template_slurm.txt`
* `sbrt_radiomics/slurm/radiomics_template_slurm.txt`
* `sbrt_radiomics/src/radiomics/one_off/binned_in_z/radiomics_template_slurm.txt`
* `sbrt_radiomics/src/ccr_phantom_analysis/slurm/preprocess_template_slurm.txt`
* `sbrt_radiomics/src/ccr_phantom_analysis/slurm/radiomics_template_slurm.txt`
* `sbrt_radiomics/src/ccr_phantom_analysis/slurm/random_ROI_template.txt`

This will tell SLURM to email you when your jobs start/finish/fail/etc

### Preprocess
From your `/nv` directory, do the following
```
cd sbrt_radiomics/slurm
patients=(BB1 BM CT DA DJ FL GB HD HR2 KS LC LD LK LL MB MK NJ OW PJ1 PJ2 SB SW WT)
for patient in ${patients[@]}; do
    generate_preprocess.sh $patient Pre
done
```

Now you'll see a bunch of `.slurm` files. You can submit individual jobs with `sbatch my_job.slurm`.

You can submit a whole bunch in one go with `find`'s `-exec` flag. The `{}` is a placeholder for all files (`-type f`) it finds in this directory (`./`) that match the `-name "*.slurm"`. The `\;` signifies "this is the end of what I want to do with each file."
```
find . -type f -name "*.slurm" -exec sbatch {} \;
```

After all your jobs run, you should see the same type of NRRD and pickle files for the other patients that you saw for BE.

### Radiomics
We'll follow the same procedure for extracting radiomic features. **Note you can only run this part for a patient after you've preprocessed their data.**

From your `/nv` directory, do the following
```
cd sbrt_radiomics/src/radiomics/one_off/binned_in_z
patients=(BB1 BM CT DA DJ FL GB HD HR2 KS LC LD LK LL MB MK NJ OW PJ1 PJ2 SB SW WT)
for patient in ${patients[@]}; do
    generate_radiomics.sh $patient Pre
done
```

Again, you can submit these jobs one by one using `sbatch` or by using the `find` trick.

## Visualizing the Data (using Jupyter Notebook)

#### PyRadiomics Documentation: 
[Radiomic Features — pyradiomics v3.0.1.post15+g2791e23 documentation](https://pyradiomics.readthedocs.io/en/latest/features.html#)

#### Accessing Jupyter Notebook via Rivanna 
[Jupyter Lab on Rivanna | Research Computing (virginia.edu)](https://www.rc.virginia.edu/userinfo/rivanna/software/jupyterlab/?msclkid=0386d39db4e811eca7c7bc2db7bf3717)
    
-Use phys_nrf as allocation
    
-Use python 3.8 
    
#### Coding with Pandas Dataframe 
[pandas.DataFrame — pandas 1.4.2 documentation (pydata.org)](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html)

#### Pro-tips (from Grace): 
When I first started, I spent a few hours going through each line of john’s scripts and googled what each line did -- this really helped get an idea of how to manipulate the data efficiently

Make tons of comments as you write a script or if you will be giving it to others, it helps you and others understand what you’re doing better
If an error pops up or output is strange:

-First, save code and restart kernel

-Make print statements between commands to find where things go wrong

-Check file paths and ensure there is data where you are referencing data

-Google error messages and try to find root of error

-If still in doubt and you think it’s an error with rivanna/the system, go to rivanna office hours or submit a ticket: [Support Options | Research Computing            (virginia.edu)](https://www.rc.virginia.edu/support/#office-hours) -- they are very very helpful and kind!

-If still in doubt and you think it’s an error with the code, email Krishni, John, or me

### Available Notebooks:

#### NaNcheck.ipynb
Runs through all patients and all features to check if any output is giving “NAN” (not a number)
Right now this only accounts for pre timepoints, but could be modified to include all timepoints
HUcheck_pre.ipynb
Runs through all patients for the feature “original_firstorder_mean”, which we found to be comparable to the HU value
Output is a csv file with the following columns: patient (e.g., HI), zbin (e.g., z_bin1), HU in range (e.g., GOOD or BAD), HU value (e.g., 64.5), all zbins good (e.g., GOOD or BAD)
The output can easily be modified to meet your needs

#### HU_mask_visual_check.ipynb
This code inputs the pre radiomic features from all patients across all z of the aorta
It plots a feature vs zbin for each of the different mask options John calculated (JM, JM2, aorta, aorta shrink 1 mm, aorta shrink 3mm, aorta grow 1mm)
Plotting original_firstorder_mean (HU) vs zbin for each feature helped us decide that JM2 is the best mask option (was closest to “soft tissue” which is in the 30-50 range of HU values)

##### Stable_feat_pre_check.ipynb
This code inputs the pre radiomic features from all patients across all z of the aorta and then selects the JM2 mask to analyze
It filters through all features and selects those that show stability across all patients and all z of the aorta. Those selected "Stable Features" are exported to a csv along with the associated mean and std across all patients for each z bin
We want to identify our stable features because if all patients share these similar values, then these features are the ones that will be good to look for any changes post treatment such as relatedness to cardiac events.
We consider it stable if mean/stddev > 10

#### Delta_radiomics_heatmap.ipynb
This code plots the change in stable radiomic features over time as a heatmap
Right now it's setup to plot one patient at a time and one feature at a time with rows as timepoints and columns as zbins
You can totally mess around with this, plot patients as rows or columns, plot features as rows or columns, there’s so much potential it just depends on what you want to look at

## Some warnings

### Contour names are a headache 
This codebase is very sensitive to changes in contour names contained in RTSTRUCT files.

If the names don't match the existing tricks in the code (e.g. judicious use of wildcards like *, the mask_dictionary in `generate_masks.py`) you might have a bit of a headache sorting things out.

Unfortunately I don't really have any words of wisdom here. The only way to understand why this happens and how to make the code work better is to read through the scripts and make sure you understand how they work.

### Re-preprocessing a patient
If you need to re-preprocess a patient, you should remove their mask directory. I'm not 100% sure some of my hacks in this part of the code work reliably and predictably if there are already NRRD files in the mask directory.

### Update pip installer
If the pip installer isn't updated, many of modules will not be successfully loaded.
```
pip install --upgrade pip
```
### New data directory  
In older versions of the instructions data is stored in /nv/vol141/phys_nrf/%Your_Name%/patients/pre directory, was redundant therefore instructions were updated to /nv/vol141/phys_nrf/%Your_Name%/patients. It is possible that some of the paths are not updated to this directory.
