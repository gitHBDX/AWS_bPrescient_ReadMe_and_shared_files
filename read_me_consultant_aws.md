### Important Links:
- for this ReadMe: https://github.com/gitHBDX/AWS_bPrescient_ReadMe_and_shared_files/blob/main/read_me_consultant_aws.md
- for all ReadMes: https://github.com/gitHBDX/AWS_bPrescient_ReadMe_and_shared_files
- for the shared fileds: https://biomarkers-my.sharepoint.com/:f:/g/personal/mkahraman_hb-dx_com/IgCzDpSrYlKMQY0ZfDv6mBiHAeeueBSTaLzCt1dFrPRBCPI?e=iQwOVp

### Logging In

1. Go to the AWS SSO portal URL: `https://d-9967459aa9.awsapps.com/start`
2. Sign in with the email address
3. On first login, click on "forgot password" you'll be prompted to set a password and configure MFA
4. Click on the **SageMaker Studio** application

### Using SageMaker Studio

1. After logging in, you'll see your JupyterLab space. Click **Run** to start it
2. The first boot takes 2-5 minutes
3. A template notebook will be in your home directory
4. Environment variables are pre-configured:
   - `DATA_BUCKET` -- S3 bucket with datasets (read-only)
   - `UPLOADS_BUCKET` -- S3 bucket for your outputs (read/write)
   - `CODECOMMIT_CLONE_URL` -- Git repo URL

### Main folder to get HBDx datasets and notebooks


Download the Dataset
Create a local data directory and sync from S3:

```bash
mkdir ~/data

# you can run this command regurlarly to have an updated folder
aws s3 sync s3://$DATA_BUCKET/data/ ~/data

ls -lsh ~/data/
```

### Installing Python Packages

Packages are installed via CodeArtifact (a PyPI proxy). From a terminal:

```bash
# Configure pip (run once per session) ### you can skip it
aws codeartifact login --tool pip \
  --domain <project_name> \
  --domain-owner <account_id> \
  --repository <project_name>-pypi \
  --region <aws_region>

# Then install packages normally
pip install pandas scikit-learn matplotlib
```

### Cloning the Code Repository

```bash
# template
git clone <codecommit_clone_url>

# hummingbird repo: main hummingbird repo with ground functions
git clone https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/hummingbird
# if needed, change to main
git checkout master

# classifynder repo: machine learning
git clone https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/classifynder
# if needed, change to main
git checkout master

# hummgingbird_snRNA_anno repo: feature annotation pipeline
git clone https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/hummingbird_sRNA_anno
# if needed, change to main
git checkout main

# bPrescient_HBDx_1862 repo: the bPrescient-team is developing here their own code
git clone https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/bPrescient_HBDx_1862
# if needed, change to main
git checkout main
```

Git credentials are automatically configured by the lifecycle config.

Setup your git credentials
```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

### Important Limitations

- **No internet access** -- you cannot access external websites, APIs, or download from the internet
- **No data download** -- you cannot copy data out of the environment to your local machine
- **Instance types are restricted** -- only pre-approved instance types are available, see section below "Allowed Instance Types"
- **Sessions auto-stop** after 2 hours of inactivity to save costs
- **All actions are logged** via CloudTrail
- **Conda package installation via internet works** Internet traffic to install condapackages is allowed

### Allowed Instance Types

Warning: datasets are large (around 16GB)! takes quite a while to load! use a large instance with enough memory

```bash
locals {
  allowed_instance_types = [
    "ml.t3.medium",
    "ml.t3.large", # use this for inital setup and playing around
    "ml.t3.xlarge", 
    "ml.t3.2xlarge", 
    "ml.m5.large",
    "ml.m5.xlarge",
    "ml.m5.2xlarge", # if you need more RAM and power, use this one for a smooth running space (CPU-based)
    "ml.g4dn.xlarge",
    "ml.g4dn.2xlarge",
    "ml.g6e.2xlarge",
    "system",
  ]
}
```

### Create two hbdx/hummingbird-environments (conda) and install hummingbird

To install Conda packages from SageMaker in this environment, configure Conda to use the internal proxy:

Edit ~/.condarc:

  ```bash
  nano ~/.condarc
  ```

!!! The content should be ONLY this !!!
After editing, to save, hit "CNTRL+X" and write "yes" and hit "ENTER"

  ```bash
  channels:
    - conda-forge
  offline: false
  proxy_servers:
    http: http://10.0.100.233:3128
    https: http://10.0.100.233:3128
  ```

To get the updated yml-files:

  ```bash
  cd ~/hummingbird
  git pull
  ```

Notes for CPU:
- use hbdx.cpu.aws.yaml for cpu-instances
- the created env will called "hbdx_cpu"

  ```bash
  conda env create -f ~/hummingbird/hbdx.cpu.aws.yml 
  ```


Notes for GPU:
- use hbdx.cuda.aws.yaml for gpu-instances
- the created env will called "hbdx_cuda"

  ```bash
  conda env create -f ~/hummingbird/hbdx.cuda.aws.yml 
  ```


Install the `hummingbird` package and `classifynder` package
while still being in the hummingbird folder (which contains the environment files)

  ```bash
  conda activate hbdx_cpu

  cd ~/hummingbird
  pip install --no-build-isolation --no-deps -e .

  cd ~/classifynder
  pip install --no-build-isolation --no-deps -e .
  ```

After installation to use the installed environment in jupyternotebook, do this:

  - Install ipykernel for the hbdx_cpu-environment

  ```bash
  python -m ipykernel install --user --name hbdx_cpu --display-name "hbdx_cpu"
  ```
  - Select the right kernel in the dropbdow of newly started jupyternotebook
  - In case, it is still not showing, stop the Jupyterlab instance and start it again. Then you should see it in the kernel-dropdown of a jupyternotebook



### Example: A test for loading an anndata with the hummingbird-environment

1. Choose an instance type with enough memory, e.g. "ml.t3.xlarge" because our anndatas are around 7 GB big.
2. Open the terminal in sagemaker-studio and download the data
3. Load a anndata in a jupyter-notebook

Download the data
```bash
# if you haven't done it yet in the sagemaker-terminal
pip install anndata
mkdir ~/data
aws s3 sync s3://$DATA_BUCKET/data/ ~/data
ls -lsh ~/data
```

Loading an anndata
```python
import hummingbird as hbdx
path = "/home/sagemaker-user/data/datasets/LC__ngs__DI_HB_GEL__wo_Boston_II_Grosshansdorf-26.02.0.h5ad"
ad = hbdx.io.load(path)
ad.obs
```

