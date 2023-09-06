# pace-tutorial
This repo contains files for tutorial on PACE and pacemaker

# Preparation

From dashboard (https://www.puhti.csc.fi/pun/sys/dashboard/) 
*  Start Jupyter 
    * Partition: gpu or interactive
    * Resources: Cores - 8, Memory - 32 GB, Local disk - 40
    * Settings: Python - tensorflow

## Terminal in JupyterLab
- New Laucnher (Ctrl+Shift+L) - Terminal


## Installation of pacemaker (python-ace and tensorpotential)

```bash
cd
mkdir tools
export PATH=~/.local/bin/:$PATH
```

### python-ace
```bash
cd tools
git clone https://github.com/ICAMS/python-ace.git
cd python-ace
pip install . --user
cd lib/maxvolpy
python setup.py install --user
```

### tensorpotential
```bash
cd tools 
git clone https://github.com/ICAMS/TensorPotential.git
cd TensorPotential
pip install . --user
```

## LAMMPS
```bash
cd tools/
git clone --depth 1  --branch develop https://github.com/lammps/lammps.git lammps
cd lammps/
mkdir build
cd build/
cmake -DCMAKE_BUILD_TYPE=Release -D BUILD_MPI=ON -DPKG_ML-PACE=ON  ../cmake
cmake --build . -- -j 
make install
```

## Tutorial materials

```bash
cd
git clone https://github.com/yury-lysogorskiy/pace-tutorial.git
cd pace-tutorial
tar zxvf AlLi_vasp_data.tar.gz
```

# TUTORIAL

## DFT data collection

```bash
cd  AlLi_vasp_data
pace_collect --free-atom-energy auto --output-dataset-filename AlLi.pkl.gz
```

## pacemaker: automatic input file generation

Prepare the folder and copy the dataset
```bash
cd pace-tutorial
mkdir AlLi_fit
cd AlLi_fit
cp ../AlLi_vasp_data/AlLi.pkl.gz
```
and run automatic input file generation
```bash
pacemaker -t
```

You will see following requests and should reply:
```
Generating 'input.yaml'
Enter training dataset filename (ex.: data.pckl.gzip, [TAB] – autocompletion): AlLi.pkl.gz
Enter test set fraction or size (ex.: 0.05 or [ENTER] - no test set): 0.05
Please enter list of elements (ex.: "Cu", "AlNi", [ENTER] - determine from dataset): [ENTER]
Enter number of functions per element ([ENTER] - default 700): 300
Enter cutoff (Angstrom, default:7.0): 6.0
Enter weighting scheme type - `uniform` or `energy` ([ENTER] - `uniform`): energy
Input file is written into `input.yaml`
```

## pacemaker: run fit

If you have `input.yaml` file in the current folder, then just execute:

```bash
pacemaker
```

At the end you should find `output_potential.yaml` (if program terminates successfully) and `interim_potential_0.yaml` (always contains latest version of potential)
You can rename one of these files into `AlLi.yaml`

```bash
mv interim_potential_0.yaml AlLi.yaml
```

## pace_activeset: calculation of active set

In order to compute active set, run

```bash
pace_activeset -d fitting_data_info.pckl.gzip AlLi.yaml
```

## LAMMPS: run MD with uncertainty indication

Go to `AlLi-LAMMPS` folder and execute LAMMPS there

```bash
cd pace-tutorial/AlLi-LAMMPS
lmp -in in.lammps
```
You should see `extrapolative_structures.dump` file. If it is empty, then no extrapolation occurs. Also check `c_max_pace_gamma` column in `log.lammps` output.
Try to increase supercell size, change temperature profile, switch to NPT, change atom types to get extrapolative structures.
Remember, that LAMMPS will overwrite `extrapolative_structures.dump` file, so copy it beforehand.

## pace_select: selection of structures for active learning

```bash
pace_select -p AlLi.yaml -a AlLi.asi -e "Al Li" -m 20 -o selected/POSCAR extrapolative_structures.dump
```

## (optional) Active exploration

Check the Jupyter notebook in `~/tools/python-ace/examples/Active_Exploration.ipynb` and adapt it for Al and/or Li and for your `AlLi.yaml/asi` potential
