# Install Python 3.8 on Ubuntu 18.04
**Install Python3.8**
`sudo apt install python3.8`

**Set python3 to Python3.8**
Add Python3.6 & Python 3.8 to update-alternatives

- `sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1`
- `sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 2`

Update Python 3 to point to Python 3.8

`sudo update-alternatives --config python3` Enter 2 for Python 3.8  

Test the version of python
```
python3 --version
Python 3.8.0
```

**Fix error: ImportError: No module named apt_pkg**

- `cd /usr/lib/python3/dist-packages/`
- `sudo ln -s apt_pkg.cpython-36m-x86_64-linux-gnu.so apt_pkg.so`

**Ensure to update Python in all Spark worker nodes to the same version**
```Exception: Python in worker has different version 3.6 than that in driver 3.8, PySpark cannot run with different minor versions. Please check environment variables PYSPARK_PYTHON and PYSPARK_DRIVER_PYTHON are correctly set.
```
