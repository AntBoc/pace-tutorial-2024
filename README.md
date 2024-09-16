# pace-tutorial
This repo contains files for tutorial on P(GR)ACE and p(gr)acemaker

Supporting slides [here](https://ruhr-uni-bochum.sciebo.de/s/nqjrW3vPIEI2ZFs)

# Preparation

From dashboard (https://www.mahti.csc.fi/pun/sys/dashboard/)
*  Start Jupyter 
    * Partition: interactive
    * Resources: Cores - 16
    * Time: 4 hours
    * Settings: Python - tensorflow
    * Module version: tensorflow/2.15
    * Jupyter type - lab
    * Working directory:  /user/[YOUR_USER_NAME]

* Connect to Jupyter

## Installation of pace(grace)maker (python-ace and tensorpotential)

## PYTHON-ACE
```bash
git clone --depth 1 --branch feature/grace_fs https://github.com/ICAMS/python-ace.git
cd python-ace
pip install .
cd ..
```

## GRACE-TENSORPOTENTIAL
```bash
git clone --depth 1 https://github.com/ICAMS/grace-tensorpotential.git
cd grace-tensorpotential
pip install .
cd ..
```

### PACEMAKER DOCS

Documentation on pacemaker and installation instruction without GRACE
https://pacemaker.readthedocs.io/en/latest/pacemaker/install/ 

## LAMMPS

```bash
git clone --depth 1 --branch grace https://github.com/yury-lysogorskiy/lammps.git
cd lammps/
mkdir build
cd build/
cmake -DCMAKE_BUILD_TYPE=Release -D BUILD_MPI=ON
 -DPKG_ML-01-PACE=ON -DPKG_MC=ON -DPKG_MANYBODY=ON ../cmake

# Check that you have line `TensorFlow library is FOUND at ...` after previous command

cmake --build . -- -j 
make install

cd ../..
```
Path to the TensorFlow lib should be automatically set, but in case of problems try setting it manually

```bash
export LD_LIBRARY_PATH=/usr/local/lib64/python3.9/site-packages/tensorflow:$LD_LIBRARY_PATH
```

## Tutorial materials

```bash
git clone https://github.com/AntBoc/pace-tutorial-2024.git
cd pace-tutorial-2024
```

# TUTORIAL: pacemaker

## DFT data collection

```bash
cd 01-PACE/00-AlLi-VASP-DATA
tar zxvf AlLi_vasp_data.tar.gz
cd  AlLi_vasp_data
pace_collect --free-atom-energy auto --output-dataset-filename AlLi.pkl.gz
cd ../..
```

## pacemaker: automatic input file generation

Prepare the folder and copy the dataset
```bash
cd 01-AlLi-FIT
cp ../00-AlLi-VASP-DATA/AlLi.pkl.gz .
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
pacemaker input.yaml
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
cd pace-tutorial-2024/02-AlLi-LAMMPS
lmp -in in.lammps
```
You should see `extrapolative_structures.dump` file. If it is empty, then no extrapolation occurs. Also check `c_max_pace_gamma` column in `log.lammps` output.
Try to increase supercell size, change temperature profile, switch to NPT, change atom types to get extrapolative structures.
Remember, that LAMMPS will overwrite `extrapolative_structures.dump` file, so copy it beforehand.

## pace_select: selection of structures for active learning

```bash
pace_select -p AlLi.yaml -a AlLi.asi -e "Al Li" -m 20 -o selected/POSCAR extrapolative_structures.dump
```
In case of problems with maxvolpy library, go to python-ace/lib/maxvolpy and try to run

```bash
pip install .
```

## (optional) Active exploration

Check the Jupyter notebook in `python-ace/examples/active_learning/Active_Exploration.ipynb` and adapt it for Al and/or Li and for your `AlLi.yaml/asi` potential

## (optional) Using ACE models in python with ASE

Check out the `01-PACE/pace_basic_validation.ipynb` for basic manipulations with a trained model in python

# TUTORIAL: gracemaker

## gracemaker: run fit
```bash
cd 02-GRACE/01-HEA25-FS-FIT
gracemaker input.yaml -sf 
```
OR

after termination of gracemaker, run the following command to export model to LAMMPS format

```bash
gracemaker -r -s -sf
```
If a problem with `yaml` appers, run
```bash
pip install --upgrade pyyaml
```

## construction of active set

```bash
cd seed/1/
pace_activeset -d training_set.pkl.gz FS_model.yaml
```
Now you can rename potential and active set files for using it in further examples:
```bash
mv FS_model.yaml HEA25_FS_model.yaml
mv FS_model.asi HEA25_FS_model.asi
```

OR

We will use previously trained models available

## Run LAMMPS with GRACE-FS
```bash
cd 02-GRACE/02-HEA25-LAMMPS/10-elems-HEA25
orterun -n 2 --oversubscribe lmp -in in.lammps.ext
```

## (Optional) Universal GRACE
```bash
cd 02-GRACE/02-HEA25-LAMMPS/10-elems-mp-1layer
```

Download universal GRACE models
```bash
grace_download
```
Identify the path to the 1-LAYER model

```bash
grace_download | grep mp-1layer
```
copy this path and inset into the `in.lammps` :
```
...
pair_style grace
pair_coeff      * * /PATH/TO/HOME/.cache/grace/train_1.5M_test_75_grace_1layer_v2_7Aug2024 Ag Au Cu Ir Ni Pd Pt Rh Ru Sc
...
```

Run lammps WITHOUT mpi (on GPU, ideally):
```bash
lmp -in in.lammps
```

## (Optional) Visualize elements distribution

Check out notebook `02-GRACE/02-HEA25-LAMMPS/Visualize_elemt_distrb.ipynb` for visualization of the
elements distribution before and after the MDMC run. 

## Links
ACE users support [group](https://acesupport.zulipchat.com/join/tgxhoxi2v3q6ddzutae4sjfz/)

Pacemaker (and gracemaker soon) [documentation](https://pacemaker.readthedocs.io/)

# Further reading

* [Online documentation](https://pacemaker.readthedocs.io/)
* [Bochkarev A., Lysogorskiy Y., and Drautz R. Graph Atomic Cluster Expansion for Semilocal Interactions beyond Equivariant Message Passing](https://doi.org/10.1103/PhysRevX.14.021036)
* [Lysogorskiy Y., Bochkarev A., Mrovec M., Drautz R., Active learning strategies for atomic cluster expansion models, Phys. Rev. Materials 7, 043801 (2023)](https://journals.aps.org/prmaterials/abstract/10.1103/PhysRevMaterials.7.043801)
* [Bochkarev, A., Lysogorskiy, Y., Menon, S., Qamar, M., Mrovec, M. and Drautz, R. Efficient parametrization of the atomic cluster expansion. Physical Review Materials 6(1) 013804 (2022)](https://journals.aps.org/prmaterials/abstract/10.1103/PhysRevMaterials.6.013804)
* [Lysogorskiy, Y., Oord, C. v. d., Bochkarev, A., Menon, S., Rinaldi, M., Hammerschmidt, T., Mrovec, M., Thompson, A., Csányi, G., Ortner, C. and  Drautz, R. Performant implementation of the atomic cluster expansion (PACE) and application to copper and silicon. npj Computational Materials 7(1), 1-12 (2021)](https://www.nature.com/articles/s41524-021-00559-9)
* [Drautz, R. Atomic cluster expansion for accurate and transferable interatomic potentials. Physical Review B, 99(1), 014104 (2019)](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.99.014104)
* HEA25 dataset: N. Lopanitsyna et al 2023 PRM 7(4) 045802  [link](https://link.aps.org/doi/10.1103/PhysRevMaterials.7.045802)
* HEA25S dataset: A. Mazitov et al 2024 J. Phys. Mater. 7 025007 [link](https://iopscience.iop.org/article/10.1088/2515-7639/ad2983)

