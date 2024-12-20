## Concise Guide For Setting Up `SEOBNREPHMv5`

### Easy

```bash
conda env create -f env.yml
conda activate seobnrv5
git clone [...]
cd [...]
pip install .
```


## Lagecy

### 0. Install GSL and BLAS Libraries

First, ensure that the required native libraries for GSL $(\geqslant 2.7)$ and BLAS are installed:

```bash
apt-get update
apt-get install libgsl-dev libblas-dev
```

---

### 1. Create a Conda Environment

1. Create a file named `env.yml` and paste the following configuration:

   ```yaml
   name: seobnrv5
   dependencies:
     - numpy=1.26
     - hdf5
     - pip
     # python >= 3.9 is ok
     - python=3.12
     - ipykernel
     - rich
     - black
     - scipy
     - matplotlib
     - fftw
     - lalsuite
     - bilby
     - pycbc
     - astropy>=6.0
     - scri=2022.9
     - llvmlite 
     - numba
     - pathos
     - scikit-optimize
   ```

2. Create and activate the environment:

   ```bash
   conda env create -f env.yml
   conda activate seobnrv5
   ```

This environment provides Python 3.9, numerical libraries, and gravitational waveform analysis tools like `bilby`, `gwsurrogate`, and `lalsuite`.

---

### 2. Install `pygsl` 

`pygsl` provides Python wrappers for the GSL (GNU Scientific Library). We will build and install `pygsl` from source:

1. Clone the `pygsl` repository:

   ```bash
   git clone https://github.com/pygsl/pygsl.git
   cd pygsl
   ```

2. Generate wrappers, configure, and install:

   ```bash
   python setup.py gsl_wrappers
   python setup.py config
   python setup.py install
   cd ..
   ```

After this step, `pygsl` should be installed and ready to use in your conda environment.

---

### 3. Install `pyseobnr`

`pyseobnr` is the Python package providing SEOBNREPHMv5 functionality. We need to modify it to use `pygsl` instead of `pygsl_lite`.

1. Clone the `pyseobnr` repository:

   ```bash
   git clone https://git.ligo.org/waveforms/software/pyseobnr.git
   cd pyseobnr
   ```

2. Update `pyseobnr` source code to use `pygsl`:

   - In all relevant source files, replace `pygsl_lite` imports with `pygsl`:
   
     **Before:**
     ```python
     import pygsl_lite.errno as errno
     from pygsl_lite import roots, errno
     import pygsl_lite.odeiv2 as odeiv2
     from pygsl_lite import spline
     ```
     
     **After:**
     ```python
     import pygsl.errno as errno
     from pygsl import roots, errno
     import pygsl.odeiv2 as odeiv2
     from pygsl import spline
     ```

   - Update the ODE API calls:

     **Before:**
     ```python
     step = odeiv2.pygsl_lite_odeiv2_step
     _control = odeiv2.pygsl_lite_odeiv2_control
     evolve = odeiv2.pygsl_lite_odeiv2_evolve
     ```

     **After:**
     ```python
     step = odeiv2.pygsl_odeiv2_step
     _control = odeiv2.pygsl_odeiv2_control
     evolve = odeiv2.pygsl_odeiv2_evolve
     ```

3. Install `pyseobnr` and its dependencies (using the `[checks]` option):

   ```bash
   pip install .[checks]
   ```


---

### Verification

After completing all setup steps, verify that `pygsl` and `pyseobnr` are correctly installed and functioning as intended. Save the following code as `test_waveform.py` and run `python test_waveform.py`:

   ```python
   import matplotlib.pyplot as plt
   import numpy as np
   import warnings

   warnings.filterwarnings("ignore", "Wswiglal-redir-stdio")  # silence LAL warnings
   from pyseobnr.generate_waveform import GenerateWaveform, generate_modes_opt

   # Input parameters
   q = 5.3
   chi_1 = 0.9
   chi_2 = 0.3
   omega_start = 0.0137  # Orbital frequency in geometric units with M=1
   eccentricity = 0.4
   rel_anomaly = 2.3

   t, modes = generate_modes_opt(
       q,
       chi_1,
       chi_2,
       omega_start,
       eccentricity=eccentricity,
       rel_anomaly=rel_anomaly,
       approximant="SEOBNRv5EHM",
   )

   # Print available modes keys
   print("Available modes:", modes.keys())

   # Plot the real part of the (2,2) mode
   plt.figure()
   plt.plot(t, modes["2,2"].real)
   plt.xlabel("Time (M)")
   plt.ylabel(r"$\Re[h_{22}]$")
   plt.grid(True)
   plt.title("SEOBNRv5EHM Mode (2,2)")
   plt.show()
   ```

   If successful, this should produce a plot of the (2,2) mode waveform.  
   For reference, compare your output with the example at:  
   [https://waveforms.docs.ligo.org/software/pyseobnr/source/notebooks/example_SEOBNRv5EHM.html](https://waveforms.docs.ligo.org/software/pyseobnr/source/notebooks/example_SEOBNRv5EHM.html)

If everything runs without errors and the generated waveform looks similar to the reference example, your SEOBNREPHMv5 environment setup is complete and correct.

### 4. Install `PyCBC` Plugins

To enable SEOBNRv5 waveforms in PyCBC, you'll need to install additional plugins:

1. Clone the PyCBC SEOBNR plugin repository:

   ```bash
   git clone https://github.com/yi-fan-wang/pycbc-plugin-seobnr.git
   cd pycbc-plugin-seobnr/
   ```

2. Install the plugin:

   ```bash
   pip install .
   ```

3. Verify the installation by checking available approximants. Create and run a test script:

   ```python
   from pycbc.waveform import td_approximants, fd_approximants
   
   # Filter time-domain approximants
   td_apx = [
       approx for approx in td_approximants() 
       if approx.startswith("SEOBNRv5")
   ]
   
   # Filter frequency-domain approximants
   fd_apx = [
       approx for approx in fd_approximants() 
       if approx.startswith("SEOBNRv5")
   ]
       
   print("Time-domain approximants:", td_apx)
   print("\nFrequency-domain approximants:", fd_apx)
   ```

   You should see the following available approximants:

   Time-domain approximants:
   ```plaintext
   ['SEOBNRv5_ROM_NRTidalv3', 'SEOBNRv5_ROM', 'SEOBNRv5E_tdtaper', 'SEOBNRv5HM', 'SEOBNRv5PHM']
   ```

   Frequency-domain approximants:
   ```plaintext
   ['SEOBNRv5_ROM', 'SEOBNRv5HM_ROM', 'SEOBNRv5_ROM_NRTidalv3', 'SEOBNRv5_ROM_INTERP', 
    'SEOBNRv5E', 'SEOBNRv5EHM', 'SEOBNRv5HM', 'SEOBNRv5PHM', 'SEOBNRv5PHM_INTERP']
   ```

If you see these approximants listed, the PyCBC plugins are correctly installed and ready to use.


