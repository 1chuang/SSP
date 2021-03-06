Large-scale Point Cloud Semantic Segmentation with Superpoint Graphs
=========

This is the official PyTorch implementation of our paper *Large-scale Point Cloud Semantic Segmentation with Superpoint Graphs* <http://arxiv.org/abs/1711.09869>.

<img src="http://imagine.enpc.fr/~simonovm/largescale/teaser.jpg" width="900">


## Code structure
* `./partition/*` - Partition code (geometric partitioning and superpoint graph construction)
* `./learning/*` - Learning code (superpoint embedding and contextual segmentation).


## Requirements

1. Install [PyTorch](https://pytorch.org) and [torchnet](https://github.com/pytorch/tnt) with `pip install git+https://github.com/pytorch/tnt.git@master`.

2. Install additional Python packages: `pip install future python-igraph tqdm transforms3d pynvrtc cupy h5py sklearn plyfile scipy`.

3. Install Boost (1.63.0 or newer) and Eigen3, in Conda: `conda install -c anaconda boost; conda install -c omnia eigen3; conda install eigen; conda install -c r libiconv`.

4. Compile the ```libply_c``` and ```libcp``` libraries:
```
cd partition/ply_c
cmake . -DPYTHON_LIBRARY=$CONDAENV/lib/libpython3.so -DPYTHON_INCLUDE_DIR=$CONDAENV/include/python3.6m -DBOOST_INCLUDEDIR=$CONDAENV/include -DEIGEN3_INCLUDE_DIR=$CONDAENV/include/eigen3
make
cd ..
cd cut-pursuit
cmake . -DPYTHON_LIBRARY=$CONDAENV/lib/libpython3.so -DPYTHON_INCLUDE_DIR=$CONDAENV/include/python3.6m -DBOOST_INCLUDEDIR=$CONDAENV/include -DEIGEN3_INCLUDE_DIR=$CONDAENV/include/eigen3
make
```
where `$CONDAENV` is the path to your conda environment. The code was tested on Ubuntu 14.04 with Python 3.6 and PyTorch 0.2.

## S3DIS

Download [S3DIS Dataset](http://buildingparser.stanford.edu/dataset.html) and extract `Stanford3dDataset_v1.2_Aligned_Version.zip` to `$S3DIR_DIR/data`, where `$S3DIR_DIR` is set to dataset directory.

### Partition

To compute the partition run

```cd partition; python partition_S3DIS.py --S3DIS_PATH $S3DIR_DIR```

### Training

First, reorganize point clouds into superpoints by:

```python learning/s3dis_dataset.py --S3DIS_PATH $S3DIR_DIR```

To train on the all 6 folds, run
```
for FOLD in 1 2 3 4 5 6; do \
CUDA_VISIBLE_DEVICES=0 python learning/main.py --dataset s3dis --S3DIS_PATH $S3DIR_DIR --cvfold $FOLD --epochs 350 --lr_steps '[275,320]' \
--test_nth_epoch 50 --model_config 'gru_10,f_13' --ptn_nfeat_stn 14 --nworkers 2 --odir "results/s3dis/best/cv${FOLD}"; \
done
```
The trained networks can be downloaded [here](http://imagine.enpc.fr/~simonovm/largescale/models_s3dis.zip), unzipped and loaded with `--resume` argument.

To test this network on the full test set, run
```
for FOLD in 1 2 3 4 5 6; do \
CUDA_VISIBLE_DEVICES=0 python learning/main.py --dataset s3dis --S3DIS_PATH $S3DIR_DIR --cvfold $FOLD --epochs -1 --lr_steps '[275,320]' \
--test_nth_epoch 50 --model_config 'gru_10,f_13' --ptn_nfeat_stn 14 --nworkers 2 --odir "results/s3dis/best/cv${FOLD}" --resume RESUME; \
done
```


## Semantic3D

Download all point clouds and labels from [Semantic3D Dataset](http://www.semantic3d.net/) and place extracted training files to `$SEMA3D_DIR/data/train`, reduced test files into `$SEMA3D_DIR/data/test_reduced`, and full test files into `$SEMA3D_DIR/data/test_full`, where `$SEMA3D_DIR` is set to dataset directory. The label files of the training files must be put in the same directory than the .txt files.

### Partition

To compute the partition run

```cd partition; python partition_Semantic3D.py --SEMA3D_PATH $SEMA3D_DIR```

It is recommended that you have at least 24GB of RAM to run this code. Otherwise, either use swap memory of increase the ```voxel_width``` parameter to increase pruning.

### Training

First, reorganize point clouds into superpoints by:

```python learning/sema3d_dataset.py --SEMA3D_PATH $SEMA3D_DIR```

To train on the whole publicly available data and test on the reduced test set, run
```
CUDA_VISIBLE_DEVICES=0 python learning/main.py --dataset sema3d --SEMA3D_PATH $SEMA3D_DIR --db_test_name testred --db_train_name trainval \
--epochs 500 --lr_steps '[350, 400, 450]' --test_nth_epoch 100 --model_config 'gru_10,f_8' --ptn_nfeat_stn 11 \
--nworkers 2 --odir "results/sema3d/trainval_best"
```
The trained network can be downloaded [here](http://imagine.enpc.fr/~simonovm/largescale/model_sema3d_trainval.pth.tar) and loaded with `--resume` argument.


To test this network on the full test set, run
```
CUDA_VISIBLE_DEVICES=0 python learning/main.py --dataset sema3d --SEMA3D_PATH $SEMA3D_DIR --db_test_name testfull --db_train_name trainval \
--epochs -1 --lr_steps '[350, 400, 450]' --test_nth_epoch 100 --model_config 'gru_10,f_8' --ptn_nfeat_stn 11 \
--nworkers 2 --odir "results/sema3d/trainval_best" --resume RESUME
```

We validated our configuration on a custom split of 11 and 4 clouds. The network is trained as such:
```
CUDA_VISIBLE_DEVICES=0 python learning/main.py --dataset sema3d --SEMA3D_PATH $SEMA3D_DIR --epochs 450 --lr_steps '[350, 400]' --test_nth_epoch 100 \
--model_config 'gru_10,f_8' --ptn_nfeat_stn 11 --nworkers 2 --odir "results/sema3d/best"
```

To upsample the prediction to the unpruned data and write the .labels files for the reduced test set, run:

```python partition/write_Semantic3D.py --SEMA3D_PATH $SEMA3D_DIR --odir "results/sema3d/best --db_test_name testred```
