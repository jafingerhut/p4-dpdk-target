
# P4 DPDK Target Components

p4-dpdk-target repo contains the P4-DPDK target specific code that supports
the target independent Table Driven Interface (TDI).

## p4-dpdk-target works in conjunction with following repos:

   
#### Table Driven Interface:
        (https://github.com/p4lang/tdi)
        TDI (Table Driven Interface) provides the target independent frontend
        interface while p4-dpdk-target provides the target dependent backend
        for DPDK SWX pipeline.
        p4-dpdk-target submodules tdi repo.  p4-dpdk-target isn't currently
        defined in p4-dpdk-target/include/bf_rt directory. 

#### DPDK SWX Pipeline:
        (https://github.com/DPDK/dpdk/blob/main/lib/pipeline)
        Version: 22.07-rc2
        
        P4 DPDK SWX pipeline implements the P4 pipeline in a DPDK software switch.
        The software switch is driven by artifacts generated by the P4 compiler
        for P4 DPDK SWX pipeline.
        p4-dpdk-target submodules dpdk repo to build P4 DPDK pipeline.
        Note about DPDK Patches (p4-dpdk-target/src/lld/dpdk/patch):
           DPDK patches 007-Fix-pipeline-structs-initialization.patch
           is a temporary solution and will be removed. 
           The fixes are being worked upon and will be committed soon.

#### P4 Compiler DPDK Backend:
        (https://github.com/p4lang/p4c/tree/main/backends/dpdk)
        P4C DPDK backend creates the artifact files used by p4-dpdk-target 
        and DPDK SWX pipeline.
        p4-dpdk-target uses bfrt.json to define the program independent frontend 
        bf_rt interface.
        context.json is used to map the P4 runtime info to the P4 DPDK target
        specific info.
        The P4 DPDK spec file from P4C DPDK, which is used by DPDK SWX pipeline
        to setup the software pipeline, is also loaded through p4-dpdk-target.

#### P4-OVS:
        (https://github.com/ipdk-io/p4-ovs)
        The remote control plane interfaces like P4 Runtime and OpenConfig are
        supported through P4-OVS.  P4-OVS maps these remote protocol messages 
        and maps them to tdi_rt interface.

## Code Organization

The drivers are P4 independent. At the time of device initialization, 
they take artifacts associated with P4 as inputs.

**p4-dpdk-target is organized as follows**

    include/
        Header files associated with various driver modules
        bf_rt:
            bf_rt defines the interim frontend interface with bfrt.
        tdi_rt:
            tdi_rt defines the frontend interface with TDI.

    src/
        Driver source files. Largely they are categorized into these submodules.

        pipe_mgr:
            P4 Pipeline Mgmt API (e.g. tables, actions, stats, meters)
        port_mgr:
            Port Mgmt API
        lld:
            Low Level Driver (e.g. DPDK library)
        dvm:
            Device Mgmt (e.g. device/port instantiation, HA sequences)
        tdi_rt:
            P4 program independent and object driven implementation with TDI.
        bf_rt:
            P4 program independent and object driven implementation.
        bf_pal:
            Platform related code to support device and port management.
        ctx_json:
            Utility used for parsing cJSON structures in context JSON.

    bf_switchd/
        Reference implementation of application to exercise the drivers.

    third-party
    	tdi:
        	Target independent frontend interface.

    tools/
        Scripts to help dependencies installation, ports setup and executing.

    dependencies/
        Python requirements files used by the driver.    

    doc/
        Files used for doxygen.


## Building and installing

The building of drivers produces a set of libraries that need to be
loaded (or linked to) the application.

Note: SDE is the top level directory with this p4-dpdk-target, syslibs and utils repo cloned in next step

#### Create install directory under SDE
```
    sudo -s
    mkdir install
```
#### Set environment variables
To set environment variables for SDE, see below :-
```
    cd p4-dpdk-target/tools/setup
    source p4sde_env_setup.sh <path to SDE directory>
```
Ensure SDE, SDE_INSTALL and LD_LIBRARY_PATH environment variables are set correctly

#### Install dependent packages
To Install the dependencies for p4-driver on the platform Fedora 33, see below:-

Note:- Make sure that your yum repository proxy and environment proxies are set properly and you have sudo access.
```
    pip3 install distro (dependency)
    cd p4-dpdk-target/tools/setup
    python3 install_dep.py
```

#### Building P4 DPDK target
 ```
    cd $SDE/p4-dpdk-target
    git submodule update --init --recursive --force
    ./autogen.sh
    ./configure --prefix=$SDE_INSTALL (For both bfrt and TDI build)
    ./configure --prefix=$SDE_INSTALL --with-generic-flags=yes (For TDI build)
    make -j
    make install
```
#### Artifacts installed

Here're the artifacts that get installed for p4-driver.

**Build artifacts:**
```
    header files for driver API to $SDE_INSTALL/include/

    libdriver.[a,la,so] to $SDE_INSTALL/lib/
        driver library to manage the device
    bf_switchd to $SDE_INSTALL/bin
        reference implementation to exercise driver API
```
## Running Reference App

    bf_switchd is a reference application that can be configured through a debug CLI.

**Running of bf_switchd**
```
    Need to set PYTHONPATH and PYTHONHOME for bfrt_python/tdi_python to work.
    export SDE_INSTALL=<path to install directory>
    export LD_LIBRARY_PATH=$SDE_INSTALL/lib:$SDE_INSTALL/lib64:$SDE_INSTALL/lib/x86_64-linux-gnu
    export PYTHONPATH=$SDE_INSTALL/lib/python3.8/:$SDE_INSTALL/lib/python3.8/lib-dynload:$SDE_INSTALL/lib/python3.8/site-packages
    export PYTHONHOME=$SDE_INSTALL/lib/python3.8
    cd $SDE_INSTALL/bin
    ./bf_switchd --install-dir $SDE_INSTALL --conf-file $SDE_INSTALL/share/dpdk/<P4 program name>/<P4 program name>.conf --init-mode=cold --status-port 7777
```
#### Config file used by bf_switchd
    bf_switchd uses a config file to initialize.  Following are some of the key fields:
        bfrt-config: Use file generated from the P4C DPDK for P4 pipeline definition.
        port-config: Optional file to describe initial ports topology.
        context:     Use file generated from the P4C DPDK to map P4 to P4 DPDK.
        config:      Use file generated from the P4C DPDK to setup P4 DPDK pipeline.
     {
        "chip_list": [
            {
                "id": "asic-0",
                "chip_family": "dpdk",
                "instance": 0
            }
        ],
        "instance": 0,
        "p4_devices": [
            {
                "device-id": 0,
                "eal-args": "dummy -n 4 -c 3",
                "mempools": [
                   {
                       "name": "MEMPOOL0",
                       "buffer_size": 2304,
                       "pool_size": 1024,
                       "cache_size": 256,
                       "numa_node": 0
                   }
                ],
                "p4_programs": [
                    {
                        "program-name": "<P4 program name>",
                        "bfrt-config": "share/dpdk/<P4 program name>/tdi.json",
                        "port-config": "share/dpdk/<P4 program name>/ports_topology.json",
                        "p4_pipelines": [
                            {
                                "p4_pipeline_name": "pipe",
                                "core_id": 1,
                                "numa_node": 0,
                                "context": "share/dpdk/<P4 program name>/pipe/context.json",
                                "config": "share/dpdk/<P4 program name>/pipe/<P4 program name>.spec",
                                "pipe_scope": [
                                        0,
                                        1,
                                        2,
                                        3
                                ],
                                "path": "share/dpdk/<P4 program name>"
                            }
                        ]
                    }
                ]
            }
        ]
    }


#### CLI interface

Once bf_switchd starts running, bfshell is available.
```
    bfshell> tdi_python

    tdi.<P4 program>.enable
    pipe = tdi.<P4 program>.pipe.ingress
    from netaddr import IPAddress
    pipe.<P4 table>.<P4 action>(<action parameters>)
    eg) pipe.ipv4_lpm.add_with_send(dst_addr=IPAddress('192.168.2.0'),dst_addr_p_length=24, port=1)
```