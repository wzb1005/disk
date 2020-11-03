# DISK
<p align="center">
  <img src="teaser.png" />
</p>

Official code release for [DISK: learning local features with policy gradient](https://arxiv.org/abs/2006.13566). If you use this code in your work, please cite us as
```latex
@article{tyszkiewicz2020disk,
  title={DISK: Learning local features with policy gradient},
  author={Tyszkiewicz, Micha{\l} J and Fua, Pascal and Trulls, Eduard},
  journal={arXiv preprint arXiv:2006.13566},
  year={2020}
}
```

## Table of contents
1. [Installation](#installation)
2. [Inference](#inference)
3. [Training](#training)
4. [Extending](#extending)

## Installation
1. Clone this repo **recursively**
2. `cd` into this repo: the next step uses relative paths
3. Execute `pip install --user -r requirements.txt`

## Inference
### Feature extraction
To extract features, execute
```
python detect.py h5_artifacts_destination images_directory
```
This should create `h5_artifacts_destination/keypoints.h5`
and `h5_artifacts_destination/descriptors.h5` compatible with the IMW benchmark.
The model by default uses a 4-layer U-Net architecture which means that the image
dimensions have to be a multiple of 16: for this reason you will probably want to
specify the `--height` and `--width` flags to scale the input images accordingly.
The images will be scaled *preserving* their aspect ratio (by 0-padding the missing
values) and the keypoint locations will be rescaled and filtered with respect to
the original image dimensions.

You can use `--help` to learn about other options, in particular it is possible
to specify the weights file with `--model-path`. We provide `save-depth.pth`, the
checkpoint trained with depth-based reward and reported in the paper (default), as
well as `save-epipolar.pth`, the savepoint trained with epipolar reward and shown in
supplementary material.

### Keypoint matching
Execute
```
python match.py h5_artifacts_destination
```
(or use `--help` to learn about other options). This should create `h5_artifacts_destination/matches.h5`
**incompatible** with the IMW benchmark: instead of saving matches as `{image_name_1}-{image_name_2}`, it
saves them as `{image_name_1}/{image_name_2}`, which creates [HDF groups](https://docs.h5py.org/en/stable/high/group.html#creating-groups)
and therefore allows this approach to scale to large image collections (saving HDFs with > 300k top-level groups
becomes painfully slow due to hashing overhead).

### Viewing results
The `view_h5.py` script can be used to view artifacts generated by `detect.py` and `match.py`.

### Exporting to [COLMAP](colmap.github.io/)
After features are detected and matched, the results can be converted into a COLMAP-compatible database format with `colmap/h5_to_db.py h5_artifacts_location raw_images_location`. Note that the features are inserted **WITHOUT** their descriptors, so our `match.py` has to be used to perform the matching beforehand. At the same time, `match.py` doesn't run pose estimation, so the exhaustive feature matching stage of the COLMAP pipeline **still has to be ran**. An example pipeline use below:
```bash
# assume we have the images in scene/images
python detect.py --height 1024 --width 1024 --n 2048 scene/h5 scene/images
python match.py --rt 0.95 --save-threshold 100 scene/h5
python colmap/h5_to_db.py --database-path scene/database.db scene/h5 scene/images

# don't use GPU since we aren't computing the descriptor distance matrices anyway,
# only RANSAC
colmap exhaustive_matcher --database_path scene/database.db --SiftMatching.use_gpu 0
mkdir scene/sparse
colmap mapper --database_path scene/database.db --image_path scene/images --output_path scene/sparse
```

Please try `h5_to_db.py --help` for extra additional options.

## Training
### The training script
Assuming data is available, `python train.py DATASETS_LOCATION` starts training. The `--reward` switch allows for choosing the reward scheme (`depth` or `epipolar`). For more information, execute `python train.py --help`.

### Reproducing our results
The data we used for training and validation can be downloaded by executing the `download_dataset` script (~164 gb). It will download the data into `datasets.epfl.ch/disk-data/` and this is the path that should be given to `train.py`. The default settings of the script will learn with the inverse softmax matching temperature `inverse_T` (called `θ_M` in the paper) annealed from 15 to 50 over the course of first 20 epochs. We then pick the best checkpoint according to validation AUC, as reported by `python compute_validation_auc.py TENSORBOARD_LOG_FILE`. Following this schedule allowed us to obtain 0.50432 stereo AUC and 0.72624 multiview AUC on IMW2020 test set with 2k features, slightly less than reported in the paper (0.51315 and 0.72705, respectively).

The paper results (available as `depth-save.pth`, the default checkpoint in `detect.py`) were obtained through an ad-hoc schedule of annealing θ_M between 15 and 25 over 10 epochs and then training for further 40 epochs. We picked the best checkpoint obtained this way (39th) and fine-tuned it with a schedule of `θ_M=25+epoch_number`, for another 50 epochs, obtaining the best model at 20th epoch (`θ_M=45`). We default to the currently presented mode of training for simplicity, while disclosing the original process.

#### Low GPU memory training
We performed our experiments with 32GB version of Nvidia V100 GPUs. However, running `python train.py --substep 2 --batch-size 1 --chunk-size 10000 --warmup 500` should be functionally equivalent with that setup and fit within 11/12gb GPUs (note that training in this mode may take on the order of 2 weeks!).

### Custom data preparation
Alternatively, one can use a custom dataset laid out in the proper format, as explained more in depth [here](https://github.com/jatentaki/disk/blob/release/disk/data/disk_dataset.py). We provide a script to automate that process in the case of photo collections posed with COLMAP.

#### Creating new datasets by importing from COLMAP
A new dataset (for instance with custom scenes) can be created by importing from COLMAP outputs. One should run COLMAP on the images, including steps of image rectification and patch match depth estimation. This should leave the user with a directory structured as
```bash
$ tree colmap_output
colmap_output/
├── images
│   ├── 2020_07_25__12_09_03.jpg
│   ├── 2020_07_25__12_09_05.jpg
│   ├── ...
├── run-colmap-geometric.sh
├── run-colmap-photometric.sh
├── sparse
│   ├── cameras.bin
│   ├── images.bin
│   └── points3D.bin
└── stereo
    ├── consistency_graphs
    ├── depth_maps
    │   ├── 2020_07_25__12_09_03.jpg.geometric.bin
    │   ├── 2020_07_25__12_09_03.jpg.photometric.bin
    │   ├── 2020_07_25__12_09_05.jpg.geometric.bin
    │   ├── 2020_07_25__12_09_05.jpg.photometric.bin
    │   ├── ...
    ├── fusion.cfg
    ├── normal_maps
    │   ├── ...
    └── patch-match.cfg
```

one can then execute `python colmap/colmap2dataset.py colmap_output --name my_scene` to create an extra "dataset" directory:

```bash
tree colmap_output/dataset/
├── calibration
│   ├── calibration_2020_07_25__12_09_03.jpg.h5
│   ├── calibration_2020_07_25__12_09_05.jpg.h5
│   ├── ..
├── dataset.json
└── depth
    ├── 2020_07_25__12_09_03.h5
    ├── 2020_07_25__12_09_05.h5
    ├── ..
```
The `dataset.json` is a file for instantiating DISK dataloaders and it contains a collection of *absolute* paths to contents of `colmap_output/dataset` and `colmap_output/images`, so those should **not be moved** afterwards. `colmap_output/stereo` and `colmap_output/sparse` can be safely deleted to conserve disk space.

In case one wants to merge multiple scenes into a single dataset, she can execute `python colmap/merge_datasets.py my_scene_1/dataset/dataset.json my_scene_2/dataset/dataset.json ...` in order to obtain a single file called `merged.json` which contains all the scenes (and still references the files in their original locations for each of the scenes!). Scenes with repeating names (as given by the `--name` flag of `colmap2dataset`) will be renamed to unique (but non-informative) names.

## Extending
We tried to keep the code easy to understand and reasonably documented. One particular feature is the extensive use of [`torch_dimcheck`](https://github.com/jatentaki/torch-dimcheck) (the `@dimchecked` function decorator): please refer to the repository for extra information. Please open an issue if problems are encountered.
