## 1. Install Anaconda
 Install Anaconda version 5.3.1 for supporting python version 3.7
 * **Download Anaconda installaion script**:
    ```
        wget https://repo.anaconda.com/archive/Anaconda3-5.3.1-Linux-x86_64.sh
    ```
 * **Run script file**:
    ```
        ./home/hadoop/Anaconda3-5.3.1-Linux-x86_64.sh
    ```
    Follow the installation on command, when done, to activating Anaconda: 
    ```
        source ~/.bashrc
    ```
## 2. Create Conda environment
 * **Create Conda environment**:
    ```
        conda create -n my-env python=3.7
    ```
    _Must specific the python version, if not, Conda will automatically install the lastes version_
 * **Activate Conda environment**:
    ```
        conda activate my-env
    ```
 * **Package environment**:
    Package a conda environment using conda-pack
    Install conda-pack if not exists:
    ```
        conda install -c conda-forge conda-pack
    ```
    Then package the current environment (activate first):
    ```
        conda-pack -p /path/to/save/gz/file
    ```
   Replicate conda environment to all nodes (Remember to edit _**.bashrc**_ file and put them in same directory)

## 3. Install dash-yarn && run first dask job
 * **Install dask-yarn with conda**:
    ```
        conda install -c conda-forge dask-yarn
    ```
 * **Submit python job with dask-yarn cli**:
    From terminal, run command:
    ```
         dask-yarn submit --environment python:///opt/anaconda/envs/analytics/bin/python  --worker-count 8   --worker-vcores 2   --worker-memory 4GiB dask_submit_test.py
    ```
    _**Note:**_ Environment format is **_python:///path_to_env/bin/python_**. Get the path to environment by execute the command:
    ```
        conda env list
    ```
 * **Start YARN cluster**:
  To start a YARN cluster, create an instance of YarnCluster. This constructor takes several parameters, leave them empty to use the defaults defined in the Configuration:
    ```
    from dask_yarn import YarnCluster
    import dask.dataframe as dd
    
    from dask.distributed import Client
    
    cluster = YarnCluster(environment='python:///opt/anaconda/envs/analytics/bin/python',
                      worker_vcores=2,
                      worker_memory="4GiB")
    ```
     By default no workers are started on cluster creation. To change the number of workers, use the YarnCluster.scale()method. When scaling up, new workers will be requested from YARN. When scaling down, workers will be intelligently selected and scaled down gracefully, freeing up resources: 
     ```
        cluster.scale(4)
     ```
    Read csv file:
    ```
        data = dd.read_csv('hdfs://10.56.239.63:9000/user/dan/Global_Mobility_Report.csv',dtype={'sub_region_2': 'object'})
    ```
    This will create a dask dataframe, to show the result, use
    ```
        data.head()
    ```
    Converto pandas dataframe by **compute** method:
    ```
        df = data.compute()
    ```
    
    
