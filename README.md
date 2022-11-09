# Top500 Benchmark - HPL Linpack

[![CI](https://github.com/geerlingguy/top500-benchmark/workflows/CI/badge.svg?branch=master&event=push)](https://github.com/geerlingguy/top500-benchmark/actions?query=workflow%3ACI)

A common generic benchmark for clusters (or extremly powerful single node workstations) is Linpack, or HPL (High Performance Linpack), which is famous for its use in rankings in the [Top500 supercomputer list](https://top500.org) over the past few decades.

I wanted to see where my various clusters and workstations would rank, historically ([you can compare to past lists here](https://hpl-calculator.sourceforge.net/hpl-calculations.php)), so I built this Ansible playbook which installs all the necessary tooling for HPL to run, connects all the nodes together via SSH, then runs the benchmark and outputs the result.

## Why not PTS?

Phoronix Test Suite includes [HPL Linpack](https://openbenchmarking.org/test/pts/hpl) and [HPCC](https://openbenchmarking.org/test/pts/hpcc) test suites. I may see how they compare in the future.

When I initially started down this journey, the PTS versions didn't play nicely with the Pi, especially when clustered. And the PTS versions don't seem to support clustered usage at all!

## Supported OSes

Currently supported OSes:

  - Ubuntu (20.04+)
  - Raspberry Pi OS (11+)
  - Debian (11+)

Other OSes may need a few tweaks to work correctly. You can also run the playbook inside Docker (see the note under 'Benchmarking - Single Node'), but performance will be artificially limited.

## Benchmarking - Cluster

Make sure you have Ansible installed (`pip3 install ansible`), then set up a `hosts.ini` file in this directory based on the `example.hosts.ini` file.

Each host should be reachable via SSH using the username set in `ansible_user`. Other Ansible options can be set under `[cluster:vars]` to connect in more exotic clustering scenarios (e.g. via bastion/jump-host).

Tweak any settings inside `config.yml` as desired (the most important being `hpl_root`—this is where the compiled MPI, ATLAS, and HPL benchmarking code will live).

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
cd ~/tmp/hpl-2.3/bin/rpi
mpirun -f cluster-hosts ./xhpl
```

> The configuration here was tested on smaller 1, 4, and 6-node clusters with 6-64 GB of RAM. Some settings in the `config.yml` file that affect the generated `HPL.dat` file may need diffent tuning for different cluster layouts!

### Benchmarking - Single Node

To run locally on a single node, clone or download this repository to the node where you want to run HPL. Make sure the `hosts.ini` is set up with the default options (with just one node, `127.0.0.1`).

Then, run the following command so the cluster networking portion of the playbook is not run:

```
ansible-playbook main.yml --tags "setup,benchmark"
```

> For testing, you can start an Ubuntu docker container:
> 
> ```
> docker run -it --rm -v $PWD:/code geerlingguy/docker-ubuntu2204-ansible:latest bash
> ```
>
> Then go into the code directory (`cd /code`) and run the playbook using the command above.

## Results

Here are a few of the results I've acquired in my testing:

| Configuration | Result | Wattage | Gflops/W |
|--- |--- |--- |--- |
| Turing Pi 2 (4x CM4 @ 1.5 GHz) | 44.942 Gflops | 24.5W | 1.83 Gflops/W |
| Turing Pi 2 (4x CM4 @ 2.0 GHz) | 51.327 Gflops | 33W | 1.54 Gflops/W |
| DeskPi Super6c (6x CM4 @ 1.5 GHz) | TODO Gflops | TODOW | TODO Gflops/W |
| M2 MacBook Air (1x M2 @ 3.5 GHz, in Docker) | 104.68 Gflops | TODOW | TODO Gflops/W |
