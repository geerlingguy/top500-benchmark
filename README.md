# Top500 Benchmark - HPL Linpack

[![CI](https://github.com/geerlingguy/top500-benchmark/actions/workflows/ci.yml/badge.svg)](https://github.com/geerlingguy/top500-benchmark/actions/workflows/ci.yml)

A common generic benchmark for clusters (or extremly powerful single node workstations) is Linpack, or HPL (High Performance Linpack), which is famous for its use in rankings in the [Top500 supercomputer list](https://top500.org) over the past few decades.

The benchmark solves a random dense linear system in double-precision (64 bits / FP64) arithmetic ([source](https://netlib.org/benchmark/hpl/)).

I wanted to see where my various clusters and workstations would rank, historically ([you can compare to past lists here](https://hpl-calculator.sourceforge.net/hpl-calculations.php)), so I built this Ansible playbook which installs all the necessary tooling for HPL to run, connects all the nodes together via SSH, then runs the benchmark and outputs the result.

## Why not PTS?

Phoronix Test Suite includes [HPL Linpack](https://openbenchmarking.org/test/pts/hpl) and [HPCC](https://openbenchmarking.org/test/pts/hpcc) test suites. I may see how they compare in the future.

When I initially started down this journey, the PTS versions didn't play nicely with the Pi, especially when clustered. And the PTS versions don't seem to support clustered usage at all!

## Supported OSes

Currently supported OSes:

  - Ubuntu (20.04+)
  - Raspberry Pi OS (11+)
  - Debian (11+)
  - Rocky Linux (9+)
  - AlmaLinux (9+)
  - CentOS Stream(9+)
  - RHEL (9+)
  - Fedora (38+)
  - Arch Linux
  - Manjaro

Other OSes may need a few tweaks to work correctly. You can also run the playbook inside Docker (see the note under 'Benchmarking - Single Node'), but performance will be artificially limited.

## Benchmarking - Cluster

Make sure you have Ansible installed (`pip3 install ansible`), then copy the following files:

  - `cp example.hosts.ini hosts.ini`: This is an inventory of all the hosts in your cluster (or just a single computer).
  - `cp example.config.yml config.yml`: This has some configuration options you may need to override, especially the `ssh_*` and `ram_in_gb` options (depending on your cluster layout)

Each host should be reachable via SSH using the username set in `ansible_user`. Other Ansible options can be set under `[cluster:vars]` to connect in more exotic clustering scenarios (e.g. via bastion/jump-host).

Tweak other settings inside `config.yml` as desired (the most important being `hpl_root`—this is where the compiled MPI, ATLAS/OpenBLAS/Blis, and HPL benchmarking code will live).

> **Note**: The names of the nodes inside `hosts.ini` must match the hostname of their corresponding node; otherwise, the benchmark will hang when you try to run it in a cluster. 
> 
> For example, if you have `node-01.local` in your `hosts.ini` your host's hostname should be `node-01` and not something else like `raspberry-pi`.
>
> If you're testing with `.local` domains on Ubuntu, and local mDNS resolution isn't working, consider installing the `avahi-daemon` package:
>
> `sudo apt-get install avahi-daemon`

Then run the benchmarking playbook inside this directory:

```
ansible-playbook main.yml
```

This will run three separate plays:

  1. Setup: downloads and compiles all the code required to run HPL. (This play takes a long time—up to many hours on a slower Raspberry Pi!)
  2. SSH: configures the nodes to be able to communicate with each other.
  3. Benchmark: creates an `HPL.dat` file and runs the benchmark, outputting the results in your console.

After the entire playbook is complete, you can also log directly into any of the nodes (though I generally do things on node 1), and run the following commands to kick off a benchmarking run:

```
cd ~/tmp/hpl-2.3/bin/top500
mpirun -f cluster-hosts ./xhpl
```

> The configuration here was tested on smaller 1, 4, and 6-node clusters with 6-64 GB of RAM. Some settings in the `config.yml` file that affect the generated `HPL.dat` file may need diffent tuning for different cluster layouts!

### Benchmarking - Single Node

To run locally on a single node, clone or download this repository to the node where you want to run HPL. Make sure the `hosts.ini` is set up with the default options (with just one node, `127.0.0.1`).

All the default configuration from `example.config.yml` should be copied to a `config.yml` file, and all the variables should scale dynamically for your node.

Run the following command so the cluster networking portion of the playbook is not run:

```
ansible-playbook main.yml --tags "setup,benchmark"
```

> For testing, you can start an Ubuntu docker container:
> 
> ```
> docker run --name top500 -it -v $PWD:/code geerlingguy/docker-ubuntu2404-ansible:latest bash
> ```
>
> Then go into the code directory (`cd /code`) and run the playbook using the command above.

#### Setting `performance` CPU frequency

If you get an error like `CPU Throttling apparently enabled!`, you may need to set the CPU frequency to `performance` (and disable any throttling or performance scaling).

For different OSes and different CPU types, the way you do this could be different. So far the automated `performance` setting in the `main.yml` playbook has only been tested on Raspberry Pi OS. You may need to look up how to disable throttling on your own system. Do that, then run the `main.yml` playbook again.

### Overclocking

Since I originally built this project for a Raspberry Pi cluster, I include a playbook to set an overclock for all the Raspberry Pis in a given cluster.

You can set a clock speed by changing the `pi_arm_freq` in the `overclock-pi.yml` playbook, then run it with:

```
ansible-playbook overclock-pi.yml
```

Higher clock speeds require more power and thus more cooling, so if you are running a Pi cluster with just heatsinks, you may also require a fan blowing over them if running overclocked.

### Tweaking BLIS configurations

As I found in [testing on the Cix P1 SoC](https://github.com/geerlingguy/top500-benchmark/issues/54#issuecomment-3518774461), any time you make a change to BLIS or BLAS configuration, the best option is to recompile that _and_ HPL.

For example, BLIS was automatically selecting a configuration for `armsve`, but I wanted to test the `cortexa57` configuration instead. When I did that (setting `blis_configure_options: "cortexa57"`), and recompiled BLIS, I didn't also delete the HPL configuration.

Therefore, if recompiling BLIS, make sure you _also_ recompile HPL, or the changes won't affect your benchmark runs as you expect.

Until I figure out how to automate this process, just run the following command after making changes to the configuration, to delete the appropriate files and trigger a recompile:

```
# Delete BLIS/HPL files directories.
ansible cluster -m shell -a "rm -f /opt/top500/tmp/COMPILE_BLIS_COMPLETE && rm -f /opt/top500/tmp/COMPILE_HPL_COMPLETE && rm -rf /opt/top500/tmp/hpl-2.3 && rm -rf /opt/top500/blis" -b

# Re-run the playbook with setup.
ansible-playbook main.yml --tags "setup,benchmark"
```

I realized all this after discussing some poor benchmark results where BLIS, when configured with `auto`, would pick `armsve` for the newer Armv9 architecture on the Cix P1 SoC, where that chip doesn't implement [the 256+ bit support the `armsve` configuration expects](https://github.com/flame/blis/issues/641). So using the 'older' `cortexa57` configuration was much more performant. Switching between the two requires recompiling BLIS and HPL.

## Results

Here are a few of the results I've acquired in my testing (sorted by efficiency, highest to lowest):

| Configuration | Architecture | Result | Wattage | Gflops/W |
|--- |--- |--- |--- |--- |
| [M4 Mac mini (1x M4 @ 4.4 GHz, in Docker)](https://github.com/geerlingguy/top500-benchmark/issues/47) | Arm | 299.93 Gflops | 39.6W | 7.57 Gflops/W |
| [M4 Max Mac Studio (1x M4 Max @ 4.51 GHz, in Docker)](https://github.com/geerlingguy/top500-benchmark/issues/57) | Arm | 685.00 Gflops | 120W | 5.71 Gflops/W |
| [Radxa CM5 (RK3588S2 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/31) | Arm | 48.619 Gflops | 10W | 4.86 Gflops/W |
| [Supermicro AmpereOne (A192-26X @ 2.6 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/43) | Arm | 2,745.1 Gflops | 570W | 4.82 Gflops/W |
| [Ampere Altra Dev Kit (Q64-22 @ 2.2 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/19) | Arm | 655.90 Gflops | 140W | 4.69 Gflops/W |
| [Orange Pi 5 (RK3588S 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/14) | Arm | 53.333 Gflops | 11.5W | 4.64 Gflops/W |
| [Supermicro AmpereOne (A192-32X @ 3.2 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/43) | Arm | 3,026.9 Gflops | 692W | 4.37 Gflops/W |
| [Dell Pro Max with GB10 (Nvidia GB10 @ 3.9 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/79) | Arm | 675.39 Gflops | 155W | 4.36 Gflops/W |
| [Radxa ROCK 5B (RK3588 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/8) | Arm | 51.382 Gflops | 12W | 4.32 Gflops/W |
| [Ampere Altra Developer Platform (M128-28 @ 2.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/17) | Arm | 1,265.5 Gflops | 296W | 4.27 Gflops/W |
| [Dell Pro Max with GB10 (Nvidia GB10 x2)](https://github.com/geerlingguy/top500-benchmark/issues/79) | Arm | 1,333.2 Gflops | 320W | 4.17 Gflops/W |
| [Orange Pi 5 Max (RK3588 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/39) | Arm | 52.924 Gflops | 12.8W | 4.13 Gflops/W |
| [Radxa ROCK 5C (RK3588S2 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/32) | Arm | 49.285 Gflops | 12W | 4.11 Gflops/W |
| [Ampere Altra Developer Platform (M96-28 @ 2.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/10) | Arm | 1,188.3 Gflops | 295W | 4.01 Gflops/W |
| [M1 Max Mac Studio (1x M1 Max @ 3.2 GHz, in Docker)](https://github.com/geerlingguy/top500-benchmark/issues/4) | Arm | 264.32 Gflops | 66W | 4.00 Gflops/W |
| [System76 Thelio Astra (Ampere Altra Max M128-30 @ 3.0 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/44) | Arm | 1,652.4 Gflops | 440W | 3.76 Gflops/W |
| [Minisforum MS-R1 (CIX P1 CD8180 @ 2.6 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/82) | Arm | 143.03 Gflops | 39W | 3.67 Gflops/W |
| [Raspberry Pi CM5 (BCM2712 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/48) | Arm | 32.152 Gflops | 9.2W | 3.49 Gflops/W |
| [Radxa Orion O6 (CIX P1 CD8180 @ 2.6 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/54) | Arm | 124.42 Gflops | 35.7W | 3.49 Gflops/W |
| [45Drives HL15 (Ampere Altra Q32-17 @ 1.7 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/25) | Arm | 332.07 Gflops | 100W | 3.32 Gflops/W |
| [Turing Machines RK1 (RK3588 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/22) | Arm | 59.810 Gflops | 18.1W | 3.30 Gflops/W |
| [Raspberry Pi 500 (BCM2712 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/50) | Arm | 35.586 Gflops | 11W | 3.24 Gflops/W |
| [Turing Pi 2 (4x RK1 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/27) | Arm | 224.60 Gflops | 73W | 3.08 Gflops/W |
| [Raspberry Pi 5 8 GB (BCM2712 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/18) | Arm | 35.169 Gflops | 12.7W | 2.77 Gflops/W |
| [Raspberry Pi 5 16 GB (BCM2712 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/51) | Arm | 37.888 Gflops | 13.8W | 2.75 Gflops/W |
| [Arduino Uno Q (4x Kryo-V2 @ 2.0 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/80) | Arm | 9.60 Gflops | 3.5W | 2.74 Gflops/W |
| [Intel Core Ultra 7 265K @ 5.5 GHz](https://github.com/geerlingguy/top500-benchmark/issues/86) | x86 | 741.10 Gflops | 273W | 2.71 Gflops/W |
| [LattePanda Mu (1x N100 @ 3.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/30) | x86 | 62.851 Gflops | 25W | 2.51 Gflops/W |
| [Compute Blade Cluster (CM5 16GB x10)](https://github.com/geerlingguy/top500-benchmark/issues/68) | Arm | 324.66 Gflops | 132W | 2.46 Gflops/W |
| [Radxa X4 (1x N100 @ 3.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/38) | x86 | 37.224 Gflops | 16W | 2.33 Gflops/W |
| [Framework Desktop Mainboard (AMD AI Max+ 395 @ 5.1 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/66) | x86 | 307.84 Gflops | 135.7W | 2.27 Gflops/W |
| [Raspberry Pi CM4 (BCM2711 @ 1.5 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/33) | Arm | 11.433 Gflops | 5.2W | 2.20 Gflops/W |
| [GMKtec NucBox G3 Plus (1x N150 @ 3.6 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/52) | Intel | 62.067 Gflops | 28.5W | 2.18 Glops/W |
| [Framework 13 AMD (AMD Ryzen AI 340 @ 4.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/83) | AMD | 88.787 Gflops | 41.6W | 2.13 Gflops/W |
| [Framework Desktop Cluster (4x AMD AI Max+ 395 @ 5.1 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/66) | x86 | 1,084 Gflops | 536.6 | 2.02 Gflops/W |
| [Supermicro Ampere Altra (M128-30 @ 3.0 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/3) | Arm | 953.47 Gflops | 500W | 1.91 Gflops/W |
| [Turing Pi 2 (4x CM4 @ 1.5 GHz)](https://www.jeffgeerling.com/blog/2021/turing-pi-2-4-raspberry-pi-nodes-on-mini-itx-board) | Arm | 44.942 Gflops | 24.5W | 1.83 Gflops/W |
| [Sipeed NanoCluster (4x CM5 @ 2.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/63) | Arm | 112.25 Gflops | 62W | 1.81 Gflops/W |
| [Lenovo M710q Tiny (1x i5-7400T @ 2.4 GHz)](https://www.jeffgeerling.com/blog/2023/rock-5-b-not-raspberry-pi-killer-yet) | x86 | 72.472 Gflops | 41W | 1.76 Gflops/W |
| [Raspberry Pi 400 (BCM2711 @ 1.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/49) | Arm | 11.077 Gflops | 6.4W | 1.73 Gflops/W |
| [Raspberry Pi 4 (BCM2711 @ 1.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/13) | Arm | 11.889 Gflops | 7.2W | 1.65 Gflops/W |
| [Turing Pi 2 (4x CM4 @ 2.0 GHz)](https://www.jeffgeerling.com/blog/2021/turing-pi-2-4-raspberry-pi-nodes-on-mini-itx-board) | Arm | 51.327 Gflops | 33W | 1.54 Gflops/W |
| [DeskPi Super6c (6x CM4 @ 1.5 GHz)](https://www.jeffgeerling.com/blog/2022/pi-cluster-vs-ampere-altra-max-128-core-arm-cpu) | Arm | 60.293 Gflops | 40W | 1.50 Gflops/W |
| [Orange Pi CM4 (RK3566 4-core)](https://github.com/geerlingguy/top500-benchmark/issues/23) | Arm | 5.604 Gflops | 4.0W | 1.40 Gflops/W |
| [DeskPi Super6c (6x CM4 @ 2.0 GHz)](https://www.jeffgeerling.com/blog/2022/pi-cluster-vs-ampere-altra-max-128-core-arm-cpu) | Arm | 70.338 Gflops | 51W | 1.38 Gflops/W |
| Custom PC (AMD 5600x @ 3.7 GHz) | x86 | 229 Gflops | 196W | 1.16 Gflops/W |
| [HiFive Premier P550 (ESWIN EIC7700X @ 1.4 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/56) | RISC-V | 7.181 Gflops | 8.9W | 0.81 Gflops/W |
| [DC-ROMA RISC-V Mainboard II (P550 @ 1.8 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/77) | RISC-V | 17.759 Gflops | 32.5W | 0.55 Gflops/W |
| [Milk-V Mars CM (JH7110 4-core)](https://github.com/geerlingguy/top500-benchmark/issues/20) | RISC-V | 1.99 Gflops | 3.6W | 0.55 Gflops/W |
| [Lichee Console 4A (TH1520 4-core)](https://github.com/geerlingguy/top500-benchmark/issues/20) | RISC-V | 1.99 Gflops | 3.6W | 0.55 Gflops/W |
| [Milk-V Jupiter (SpacemiT X60 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/37) | RISC-V | 5.66 Gflops | 10.6W | 0.55 Gflops/W |
| [Sipeed Lichee Pi 3A (SpacemiT K1 8-core)](https://github.com/geerlingguy/top500-benchmark/issues/42) | RISC-V | 4.95 Gflops | 9.1W | 0.54 Gflops/W |
| [Milk-V Mars (JH7110 4-core)](https://github.com/geerlingguy/top500-benchmark/issues/35) | RISC-V | 2.06 Gflops | 4.7W | 0.44 Gflops/W |
| [Raspberry Pi Zero 2 W (RP3A0-AU @ 1.0 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/26) | Arm | 0.370 Gflops | 2.1W | 0.18 Gflops/W |
| [Dell Optiplex 780 (Intel Q8400 @ 2.66 GHz)](https://github.com/geerlingguy/top500-benchmark/issues/81) | x86 | 31.224 Gflops | 180W | 0.17 Gflops/W |
| [M2 Pro MacBook Pro (1x M2 Pro, in Asahi Linux)](https://github.com/geerlingguy/top500-benchmark/issues/21#issuecomment-1792425949) | Arm | 296.93 Gflops | N/A | N/A |
| M2 MacBook Air (1x M2 @ 3.5 GHz, in Docker) | Arm | 104.68 Gflops | N/A | N/A |

You can [enter the Gflops in this tool](https://hpl-calculator.sourceforge.net/hpl-calculations.php) to see how it compares to historical top500 lists.

> **Note**: My current calculation for efficiency is based on average power draw over the course of the benchmark, based on either a Kill-A-Watt (pre-2024 tests) or a ThirdReality Smart Outlet monitor. The efficiency calculations may vary depending on the specific system under test.

### Other Listings

Over the years, as I find other people's listings of HPL results—especially those with power usage ratings—I will add them here:

  - [VMW Research Group GFLOPS/W listing](https://web.eece.maine.edu/~vweaver/group/green_machines.html)
