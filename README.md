# It is alpha version for evaluation.

# Platform Resource Manager

Platform Resource Manager is a suite of software packages to help you to co-locate best-efforts jobs with latency-critical jobs on a node and in a cluster.    
The suite provides an analysis tool [analyze] to build a model for platform resource contention detection. 
The suite also provides an agent to monitor and control platform resources (CPU Cycle, Last Level Cache, Memory Bandwidth, etc.. ) on each node.    

## Requirements

 - Python 3.6.x
 - Python lib: numpy, pandas, scipy, scikit-learn, docker, prometheus-client
 - Golang compiler
 - gcc
 - git
 - Docker

## Environment Setup
Assuming all requirements are installed and configured properly, follow the steps below to setup a working environment.

**Install intel-cmt-cat tool**

     git clone https://github.com/intel/intel-cmt-cat
     cd intel-cmt-cat
     make
     sudo make install

**Build Platform Resource Manager**

     git clone https://github.com/intel/platform-resource-manager
     cd platform-resource-manager
     ./setup.sh
     cd eris

**Prepare workload configuration file**

In order to use resource manager tool, you will need to provide a workload configuration json file in advance. Each row in file describes name, id, type (Best-Effort, Latency-Critical), request CPU count of one task (Container).  The following is an example file demonstrating the file format.  
```json
{
    "cassandra_workload": {
        "cpus": 10,
        "type": "latency_critical"
    },
    "django_workload": {
        "cpus": 8,
        "type": "latency_critical"
    },
    "memcache_workload_1": {
        "cpus": 2,
        "type": "latency_critical"
    },
    "memcache_workload_2": {
        "cpus": 2,
        "type": "latency_critical"
    },
    "memcache_workload_3": {
        "cpus": 2,
        "type": "latency_critical"
    },
    "stress-ng": {
        "cpus": 2,
        "type": "best_efforts"
    },
    "tensorflow_training": {
        "cpus": 1,
        "type": "best_efforts"
    }
}
```
 
## Command Line Arguments

**eris agent command line arguments summary**
 
    usage: eris.py [-h] [-v] [-g] [-d] [-c] [-r] [-i] [-e] [-n] [-p]
                   [-u UTIL_INTERVAL] [-m METRIC_INTERVAL] [-l LLC_CYCLES]
                   [-q QUOTA_CYCLES] [-k MARGIN_RATIO] [-t THRESH_FILE]
                   workload_conf_file
    
    eris agent monitor container CPU utilization and platform metrics, detect
    potential resource contention and regulate task resource usages
    
    positional arguments:
      workload_conf_file    workload configuration file describes each task name,
                            type, id, request cpu count
    
    optional arguments:
      -h, --help            show this help message and exit
      -v, --verbose         increase output verbosity
      -g, --collect-metrics
                            collect platform performance metrics (CPI, MPKI,
                            etc..)
      -d, --detect          detect resource contention between containers
      -c, --control         regulate best-efforts task resource usages
      -r, --record          record container CPU utilizaton and platform metrics
                            in csv file
      -i, --key-cid         use container id in workload configuration file as key
                            id
      -e, --enable-hold     keep container resource usage in current level while
                            the usage is close but not exceed throttle threshold
      -n, --disable-cat     disable CAT control while in resource regulation
      -x, --exclusive-cat   use exclusive CAT control while in resource regulation
      -p, --enable_prometheus
                            allow eris send metrics to prometheus
      -u UTIL_INTERVAL, --util-interval UTIL_INTERVAL
                            CPU utilization monitor interval (1, 10)
      -m METRIC_INTERVAL, --metric-interval METRIC_INTERVAL (1, 60)
                            platform metrics monitor interval
      -l LLC_CYCLES, --llc-cycles LLC_CYCLES
                            cycle number in LLC controller
      -q QUOTA_CYCLES, --quota-cycles QUOTA_CYCLES
                            cycle number in CPU CFS quota controller
      -k MARGIN_RATIO, --margin-ratio MARGIN_RATIO
                            margin ratio related to one logical processor used in
                            CPU cycle regulation
      -t THRESH_FILE, --thresh-file THRESH_FILE
                            threshold model file build from analyze.py tool


**analyze tool command line arguments**

    usage: analyze.py [-h] [-v] [-t THRESH]
                      [-f {quartile,normal,gmm-strict,gmm-normal}]
                      [-m METRIC_FILE]
                      workload_conf_file
    
    This tool analyzes CPU utilization and platform metrics collected from eris
    agent and build data model for contention detect and resource regulation.
    
    positional arguments:
      workload_conf_file    workload configuration file describes each task name,
                            type, id, request cpu count
    
    optional arguments:
      -h, --help            show this help message and exit
      -v, --verbose         increase output verbosity
      -t THRESH, --thresh THRESH
                            threshold used in outlier detection
      -f {gmm-strict,gmm-normal}, --fense-type {gmm-strict,gmm-normal}
                            fense type used in outlier detection
      -m METRIC_FILE, --metric-file METRIC_FILE
                            metrics file collected from eris agent
      -u UTIL_FILE, --util-file UTIL_FILE
                            Utilization file collected from eris agent
      -o, --offline         do offline analysis based on given metrics file
      -i, --key-cid         use container id in workload configuration file as key
                            id


## Typical Usage


Step 1 - Run latency critical tasks and stress workloads on one node, the CPU utilization will be recorded in util.csv and platform metrics will be recorded in metrics.csv

    sudo python eris.py --collect-metrics --record workload.json

Step 2 - Analyze data collected from eris agent, build data model for resource contention detection and regulation. Model file threshold.json will be generated.

    sudo python analyze.py workload.json

Step 3 - Add best-efforts task to node, restart monitor and detect potential resource contention

    sudo python eris.py --collect-metrics --record --detect workload.json

optionally, user can enable resource regulation on best-efforts tasks as well

    sudo python eris.py --collect-metrics --record --detect --control workload.json
