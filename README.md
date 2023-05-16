## AQUA: Scalable Rowhammer Mitigation by Quarantining Aggressor Rows at Runtime
作者: Anish Saxena, Gururaj Saileshwar, Prashant J. Nair, and Moinuddin K. Qureshi

论文地址 [MICRO 2022](https://www.microarch.org/micro55/)

论文源码地址 [aqua_rowhammer_mitigation](https://github.com/Anish-Saxena/aqua_rowhammer_mitigation)

### 介绍

原论文的源码基础上，在自己的服务器上复现，主要做一些本地环境改动以及依赖软件版本变更。

### Gem5 CPU模拟器性能评估要求，实验环境和原作者不一样:
   - **硬件配置:**
     - 操作系统：CentOS Linux release 7.6.1810 `(命令：cat /etc/centos-release)`
     - CPU架构：x86_64 `(命令：lscpu | grep -i Architecture)`
     - CPU型号：Intel(R) Xeon(R) CPU E5-2630 v4 @ 2.20GHz `(命令：cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c)`
     - vCPU数：40 `(命令：cat /proc/cpuinfo | grep process | wc -l)`
   - **软件依赖:** Gem5需要依赖gcc, Python, scons.
     - gcc v8.3.1、scons v3.0.5、Python v3.6.8
     - Scons-3.0.5安装，跟作者的安装方式不太一样 [下载链接](https://pypi.org/project/SCons/3.0.5/#files). 下载`scons-3.0.5-py2.py3-none-any.whl` 然后 `pip3 install scons-3.0.5-py2.py3-none-any.whl`.
     - 数据分析库perl `(yum install perl-List-MoreUtils)`
   - **基准测试集:** [SPEC-2017](https://www.spec.org/cpu2017/).

### Steps for Gem5 Evaluation
Here you will recreate results in Figure 3, 6, 7, 9, 10, and 11, by executing the following steps:
- **Compile Gem5:** `cd gem5 ; scons -j50 build/X86/gem5.opt`
- **Set Paths** in `scripts/env.sh`. You will set the following :
    - `GEM5_PATH`: the full path of the gem5 directory (that is, `<current-dirctory>/gem5`).
    - `SPEC17_PATH`: the path to your SPECint-CPU 2017 installation. 
    - `CKPT_PATH`: the path to a new folder where the checkpoints will be created next (for example, `<current-directory/cpts>`).
    - Optionally, modify `MAX_GEM5_PARALLEL_RUNS` to set the maximum number of parallel gem5 threads the system can support. For example, for a 30-core system, set the variable to 30.
    - Please source the paths as: `source scripts/env.sh` after modifying the file.
- **Test Creating and Running Checkpoints:** For each program the we need to create a checkpoint of the program state after the initialization phase of the program is complete, which will be used to run the simulations with different Rowhammer defense configurations. 
    - To test the checkpointing process, run `cd scripts; ./ckptscript_test.sh perlbench 4 2017;`: this will create a checkpoint after 1Mn instructions (should complete in a couple of minutes).
      * In case the `ckptscript_test.sh` fails with the error `$SPEC17_PATH/SPEC2017_inst/benchspec/CPU/500.perlbench_r/run/run_base_refrate_<config-name>-m64.<run-number>/perlbench_r_base.<config-name>-m64: No such file or directory` (or similar error message), it indicates the script is unable to find the run-directory for perlbench. Please follow the steps outlined in [README_SPEC_INSTALLATION.md](./README_SPEC_INSTALLATION.md) to ensure the run-directories are properly set up for all the SPEC-benchmarks.
- **Run All Experiments:** for all the benchmarks, run `cd scripts; ./run_all.sh`. This will run the following scripts:
    - `create_checkpoints.sh` - This creates checkpoints for single-core and multi-core benchmarks.
      * **Create Checkpoint:** For each benchmark, the checkpoints will be created using `./ckptscript.sh <BENCHMARK> <NUM-CORES> <SPEC-VERSION>`. 
      	- By default, `ckptscript.sh` is run for 52 programs in parallel (18 single-core SPEC workloads, 18 multi-core SPEC workloads, and 14 multi-core MIXED workloads). 
      	- For each program, the execution is forwarded by 25 Billion Instructions (by when the initialization of the program should have completed) and then the architectural state (e.g. registers, memory) is checkpointed. Subsequently, when each RH defense is simulated, these checkpoints will be reloaded.
      	- This process can take 24-36 hours for each benchmark. Hence, all the benchmarks are run in parallel by default.
      	- Please see `configs/example/spec17_benchmarks.py` for list of benchmarks supported.
    - `run_<baseline-or-aqua-rrs>_experiments.sh` - These scripts run all baseline, AQUA, and RRS configurations for all 34 benchmarks.
      * **Run experiments**: Once all the checkpoints are created, the scripts will internally use `./runscript.sh <BMARK> <CONFIG-NAME> <NUM-CORES> <SPEC-VERSION> <RH-DEFENSE-PARAMETERS>`, where each Rowhammer defense configuration (AQUA and RRS) is simulated with different parameters (for scalability studies) for each benchmark.
      	- The arguments for `runscript.sh` are as follows:
          -  `BMARK`: The benchmark to be simulated, like perlbench.
          -  `CONFIG-NAME`: Any string that will be used to identify this run, and the name for the results-folder of this run.
          -  `NUM-CORES`: Number of cores, 4 for multi-core runs and 1 for single-core baseline runs.
          - `SPEC-VERSION`: Fixed to be 2017.
          - `RH-DEFENSE-PARAMETERS`: `--rh_defense` enables the defense, `--rh_mitigation=RQ` selects AQUA as the defense (RRS in place of RQ selects RRS defense), and  `--rh_actual_threshold=1000` specifies 1K as the rowhammer threshold. Additional runtime parameters like FPT-cache size, quarantine region size, etc. are present in `run_<aqua-or-rrs>_experiments.sh` scripts. In absence of `--rh_defense` parameter, the baseline configuration is run.
      	- Each configuration is simulated for 250Mn instructions. This takes 8-24 hours per benchmark, per configuration. Benchmarks in 2-3 configurations are run in parallel for a total of up to `MAX_GEM5_PARALLEL_RUNS` (defined in `scripts.env.sh`) parallel Gem5 runs at a time.
- **Parse the results:** for Figures 3, 6, 7, 9, and 11 (Figure 10 remains **TODO**). Run the following script:
    - `cd stats_scripts; ./collect_all_data.sh` This will parse the gem5 stat files and collect the relevant stats for all figures mentioned above. The data is stored in `data/` directory inside `stats_scripts` and will be used by plotting scripts. The script in turn calls `Figure_*.sh` which collect data required to plot that particular figure. 
    - Note that we will supply the script to collect data for Figure 10 by end of this week. We seem to have lost that script during refactoring the code. All other scripts will work as intended.
<!-- -`cd stats_scripts; ./data_perf.sh`. This will compare the normalized performance (using weighted speedup metric) vs baseline.
- The normalized peformance results will be stored in `stats_scripts/data/perf.stat`. 
    - Script to collect the LLC misses-per-thousand-instructions (MPKI) for each of the schemes is also available in `stats_scripts/data_mpki.sh`.
    - `./run.sensitivity.cachesz.sh` - This runs the evaluations for sensitivity to LLC-Size from 2MB to 64MB (shared between 4-cores)
      * Experiments are run using the script `./runscript.sh`
      * Results for normalized Perf vs. LLCSz can be generated using `cd stats_scripts; ./data_LLCSz.sh`. 
      * Results are stored in `stats_scripts/data/perf.LLCSz.stat`.
    - `./run.sensitivity.encrlat.sh` - This runs the evaluations for Encryption-latencies from 1 to 5 (used in cache-indexing).
      * Experiments are run using the script `./runscript.sh`
      * Results for normalized Perf vs. EncrLat can be generated using `cd stats_scripts; ./data_EncrLat.sh`. 
      * Results are stored in `stats_scripts/data/perf.EncLat.stat`. -->
- **Visualize the results:** `cd graph_scripts; jupyter-notebook aqua_MICRO22_plots.ipynb` This will open a Jupyter Notebook session in the browser and will enable you to plot all the graphs. Remember to run the stats script command above before plotting the results.
  - In the notebook, select the code for a given figure and click on Run either from the menu or within the top left corner of the selected cell.
- **Note on Simulation Time:** Running all experiments takes almost 3-4 days on a system with 72 cores. Here is the order in which to reduce the simulation costs, if required:
    - To shorten experiment run time, you may reduce instruction count (`MAX_INSTS`) in `runscript.sh`to 100Mn. This reduces the simulation time to 3.5 days with 72 cores.
    - You may skip the scalability configuration for RRS (Figure 3) by commenting out the configurations in `RRS scalability configs` section in `run_rrs_experiments.sh`. This further reduces the simulation time to 3 days at the expense of Figure 3.
    - You may skip the scalability configuration for AQUA (Figure 11) by commenting out the configurations in `AQUA scalability configs` section in `run_aqua_experiments.sh`. This further reduces the simulation time to 2.5 days at the expense of Figure 3.
    - While discouraged, you may skip the comparison of AQUA with RRS (Figures 6 and 7) by commenting out the AQUA configurations in `AQUA SRAM config` section in `run_aqua_experiments.sh` _and_ commenting out the `run_rrs_experiments.sh` script from `run_all.sh` script. This further reduces the simulation time to 2 days at the expense of Figures 6 and 7.
    - **NOTE-1:** The dominant simulation cost at this point is running the checkpointing process, which takes about 1.5 days, and baseline, AQUA-SRAM, and AQUA-Memory-Mapped configurations, which takes about 12 hours, and is difficult to reduce further without sacrificing on workloads. 
    - **NOTE-2:** Note on plotting results.

### Acknowledgements

This artifact has been adapted from the artifact repository of [MIRAGE (USENIX Security 2021)](https://github.com/gururaj-s/mirage).