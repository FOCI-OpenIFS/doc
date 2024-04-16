Running FOCI-OpenIFS 
====================

You will need
-------------

* Patience
* Access to an HPC
* Miniconda / Miniforge
* ESM-Tools (GitHub account is recommended, but not required)
* Access to gitlab.dkrz.de, git.geomar.de 

Prerequisites
-------------

* Get ESM-Tools (https://github.com/esm-tools/esm_tools/) 

```bash
module load git-lfs

mkdir esm
cd esm

git clone https://github.com/esm-tools/esm_tools
```

* Install Perl (util in ESM-Tools)

```bash
cd esm_tools/utils/
./install_perl.sh
```

* Install ESM-Tools

```bash
cd ../
./install.sh
```

If you get an issue with `LC_ALL` being unset, then just set it in your `.bashrc` file. 
See also "INSTALLING" on the ESM-Tools git page. 

If it complains about the python version, then you can install your own Miniconda environment and build ESM-Tools that way. 
First install Miniconda:

```bash
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -o Miniconda3.sh 
bash Miniconda3.sh -b -p ${HOME}/miniconda3
${HOME}/miniconda3/bin/conda init bash
```

Log out and log in again for the changes to take effect. 

Make sure you have an older Python
```bash
conda install python=3.10
```

Then install ESM-Tools
```bash
cd esm/esm
./install.sh
```

Switch to olaf branch
```bash
cd
cd esm/esm_tools/
git checkout feature/ibs_olaf
```


Set up access tokens for git servers
------------------------------------

Github: 
* Log into github.com. 
* Profile picture upper right corner -> Settings -> Developer settings -> Personal access tokens -> Generate new token (classic)
* Click "repo" box. Name it "olaf". Click Generate token. 
* Copy code

Gitlab (DKRZ, GEOMAR, etc)
* Log into gitlab e.g. gitlab.dkrz.de or git.geomar.de
* Click your profile logo near the `+` sign in the top left. 
* Choose "Preferences" -> "Access Tokens" -> "Add new token". 
* Name it "olaf", empty the "expiration date" box, click "read_repository" and "write_repository", "Create personal access token". 
* Copy code

Make a `${HOME}/.netrc` file on Olaf with contents
```bash
machine github.com login <github_username> password <token_code_from_github> 
machine gitlab.dkrz.de login <dkrz_username> password <token_code_from_dkrz> 
machine git.geomar.de login <geomar_email> password <token_code_from_geomar> 
```

Save and close the file. 


Set up the environment
----------------------

```bash
cd 
cd esm
mkdir -vp /proj/internal_group/iccp/$USER/esm-experiments/
ln -sfv /proj/internal_group/iccp/$USER/esm-experiments esm-experiments
```

Install model
-------------

Get the code

```bash
cd esm
mkdir models
cd models
esm_master get-focioifs-4.0 
``` 

Compile 
```bash
esm_master comp-focioifs-4.0
``` 

Run an experiment
-----------------

```bash



