# GROMACS_Cloud
GROMACS Setup on Ferdowsi Cloud Computing Service
# Installing GROMACS 2024.1 with GPU Support on Ferdowsi Cloud (Ubuntu)

This guide details the steps to compile and install GROMACS 2024.1 with NVIDIA CUDA GPU acceleration on an Ubuntu server provided by the Ferdowsi cloud computing service.

**Target Environment:**

*   Cloud Platform: Ferdowsi Cloud Computing Service
*   Operating System: Ubuntu (assumed 24.04 based on file names, adapt if needed)
*   GPU: NVIDIA GPU
*   GROMACS Version: 2024.1

## Prerequisites

1.  An active account with Ferdowsi Cloud Computing Service.
2.  An Ubuntu server instance with an assigned NVIDIA GPU.
3.  SSH access to the server instance (e.g., using PuTTY or terminal). You will need the server IP, username (`ubuntu`), and the provided password.

## Installation Steps

Follow these steps sequentially in your server's terminal after logging in via SSH.

### Step 1: Initial Server Setup & Updates

Update the package list and install essential build tools.

```bash
sudo apt update
sudo apt install build-essential -y
sudo apt upgrade -y # Optional but recommended: upgrade existing packages
    
Step 2: Install Specific GCC Version
GROMACS and certain CUDA toolkit versions require specific GCC compiler versions for compatibility.
We will install GCC/G++ version 12, as this is often compatible with common CUDA toolkits and
avoids issues with newer, potentially unsupported GCC versions.

sudo apt install gcc-12 g++-12 -y

# Verify installation (optional)
gcc-12 --version
g++-12 --version
    
Step 3: Install Helper Tools (pipx, gdown, sphinx)
Install pipx for managing Python applications in isolated environments.
Then use pipx to install gdown (for downloading files from Google Drive)
and sphinx (required for GROMACS documentation building).

sudo apt install python3-pip -y
sudo apt install pipx -y

# Add pipx to your PATH (follow on-screen instructions if any)
pipx ensurepath

# !! You might need to close and reopen your SSH session.

# Install gdown and sphinx using pipx
pipx install gdown
pipx install sphinx
    
Step 4: Install NVIDIA Driver & CUDA Toolkit (Using Local Repository)
This section assumes you have obtained the necessary CUDA local repository .deb file and the corresponding .pin file (likely downloaded via gdown as shown in the source file).
4.1 Download CUDA Repo Files
Use gdown to download the specific CUDA repository .deb file and the pinning file for your target Ubuntu version (Ubuntu 24.04 / CUDA 12.8 in this example). You need the correct Google Drive File IDs.
      # --- !!! YOU CAN REPLACE THESE IDs WITH YOUR OWNS !!! ---
PIN_FILE_ID="1B2Zvg2Rpzc8lBKVboSUKwfCedr7WUuyQ"
DEB_FILE_ID="1-UvYcXOsM7IvnGcNtkpr0271UkQpasOi"
# --- !!! ----------------------------------------- ---

gdown ${PIN_FILE_ID} # Downloads the .pin file (e.g., cuda-ubuntu2404.pin)
gdown ${DEB_FILE_ID} # Downloads the .deb file (e.g., cuda-repo-ubuntu2404-12-8-local_12.8.1-570.124.06-1_amd64.deb)

4.2 Set up CUDA Repository and Install Toolkit
Configure the system to use the downloaded local repository
and install the CUDA toolkit (version 12.8 in this example).

sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo dpkg -i cuda-repo-ubuntu2404-12-8-local_12.8.1-570.124.06-1_amd64.deb

# Copy the GPG key for the repository
sudo cp /var/cuda-repo-ubuntu2404-12-8-local/cuda-*-keyring.gpg /usr/share/keyrings/

# Update package list to include the new repository
sudo apt-get update

# Install the CUDA Toolkit (this may also install a compatible NVIDIA driver)
sudo apt-get -y install cuda-toolkit-12-8

4.3 Reboot and Verify Driver
A reboot is often required after driver installation.

sudo reboot
    

After the server reboots, log back in via SSH and verify that the NVIDIA driver is loaded and detects your GPU:

nvidia-smi

This command should output details about your GPU and the driver version. If it fails, the driver installation encountered an issue.
Step 5: Configure CUDA Environment Variables
Add the CUDA paths to your environment variables so the system can find the compiler (nvcc) and libraries.
      # Edit the .bashrc file using nano text editor
nano ~/.bashrc

# Scroll to the very end of the file and add these two lines:
export PATH=/usr/local/cuda-12.8/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}

# Save the file: Press CTRL+O, then Enter.
# Exit nano: Press CTRL+X.

# Apply the changes to your current session
source ~/.bashrc

# Verify nvcc is found and check the version
nvcc -V
# Or: nvcc --version

The output should show the CUDA toolkit version you installed (e.g., 12.8).
Step 6: Download and Extract GROMACS
Download the GROMACS 2024.1 source code and extract it.

cd ~ # Navigate to your home directory

wget ftp://ftp.gromacs.org/gromacs/gromacs-2024.1.tar.gz
tar xfz gromacs-2024.1.tar.gz

Step 7: Configure GROMACS Build (CMake)
Create a build directory and run CMake to configure the build process,
specifying GPU support and the correct compilers.

cd gromacs-2024.1
mkdir build
cd build

cmake .. -DGMX_BUILD_OWN_FFTW=ON \
         -DREGRESSIONTEST_DOWNLOAD=ON \
         -DGMX_GPU=CUDA \
         -DCMAKE_C_COMPILER=/usr/bin/gcc-12 \
         -DCMAKE_CXX_COMPILER=/usr/bin/g++-12 \
         -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda-12.8 # Optional but good practice
    

•	-DGMX_BUILD_OWN_FFTW=ON: Build GROMACS's required FFT library.
•	-DREGRESSIONTEST_DOWNLOAD=ON: Download regression tests (needed for make check).
•	-DGMX_GPU=CUDA: Enable NVIDIA CUDA GPU acceleration.
•	-DCMAKE_C_COMPILER, -DCMAKE_CXX_COMPILER: Explicitly specify the compatible GCC 12 compilers.
•	-DCUDA_TOOLKIT_ROOT_DIR: Explicitly tell CMake where CUDA is installed.
Review the output of CMake carefully for any errors.
Step 8: Compile GROMACS (make)
Compile the GROMACS source code. This can take a significant amount of time depending on the server's resources. Use the -j flag to parallelize the build using multiple cores (e.g., -j $(nproc) uses all available cores).

make -j $(nproc)

Step 9: Test GROMACS Build (make check)
Run the regression tests to ensure the build is functional. This step is highly recommended and can also take a while.

make check

Look for a summary indicating the number of tests passed, failed, or skipped. Ideally, all essential tests should pass.
Step 10: Install GROMACS (make install)
Install the compiled GROMACS binaries and libraries to the default system location (/usr/local/gromacs). Using sudo is required for writing to this location.

sudo make install

Verification
After installation, you need to source the GROMACS environment script to add the GROMACS binaries to your PATH.

source /usr/local/gromacs/bin/GMXRC
# Verify the installation by checking the version
gmx -version
    
This command should output the GROMACS version (2024.1) and mention GPU support (CUDA).
You may want to add the source /usr/local/gromacs/bin/GMXRC line to your ~/.bashrc file so GROMACS is available every time you log in:
      echo "source /usr/local/gromacs/bin/GMXRC" >> ~/.bashrc
    
Notes
•	Firewalls: If you intend to use remote visualization or other services, ensure necessary ports are open in the Ferdowsi Cloud firewall settings.
•	Installation Prefix: If you prefer not to install to /usr/local/gromacs (e.g., due to permissions or managing multiple versions), you can specify a different installation path during the cmake step using -DCMAKE_INSTALL_PREFIX=/path/to/your/install/location. If you do this, make sure to source the GMXRC file from that custom location instead.
•	CUDA/GCC Compatibility: Always ensure your chosen CUDA Toolkit version officially supports the GCC version you are using. Using -allow-unsupported-compiler with CMake is possible but not recommended as it may lead to runtime errors or incorrect results.
You should now have a working installation of GROMACS 2024.1 with GPU support on your Ferdowsi Cloud server.

