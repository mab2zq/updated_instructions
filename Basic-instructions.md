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

You'll also need to add the following lines to your `.bashrc`:
```
module unload python
module load gcc/7.1.0
module load intel/18.0
module load mvapich2/2.3.1
module load openmpi/3.1.4
module load python/3.6.6
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

## Do something with the features!
Once you have radiomic features extracted, it's time to load all that data into R or Python so you can do something with that information.

I'm going to leave this a little open ended for now, but for some inspiration you can look at the [jupyter](https://jupyter.org) notebooks in 
`sbrt_radiomics` that I recently started. They're still really sketchy, but they might give you some ideas about how you can load everything into one [pandas DataFrame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html) and start analyzing/plotting the data.

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
