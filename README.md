# Moving Averge Batch Normalization

This reposity is the Pytorch implementation of **Moving Average Batch Normalization** on Imagenet classfication, COCO object detection and instance segmentation tasks. Notice the Imagenet classification simulate the small batch training settings by using small normalization batch size and regular SGD batch size. 

The paper has been published as an ICLR2020 conference paper (https://openreview.net/forum?id=SkgGjRVKDS&noteId=BJeCWt3KiH).

## Results

### Overall comparation of MABN and its counterparts

Top 1 Error versus Batch Size:

<img src="https://uc23c04615403e17cfed78ccacc6.previews.dropboxusercontent.com/p/thumb/AApzTU_479fgUugJranC19juFKPjXUlOgbr2D6GpIje1c0f6aXL_VLQYmQ1ojgHTqSoPhXsDcyN7-dPXL3xgw9rXdckmGKgHR09Q5ihJAVeMUu5SwFcCmJHPidCPP9mo-ILrKKnInM6ohDdiV1541Ie9ozRy-PGHOlQ8zgu0z7JndBVBgw7Ave8gN1-ixsrPw-ANRBNZwmZTgfAha4BsCvJLU3E3pjaFm8BysoHn3UbHwOWaGdUuLDPDrfCNG0cWHBkqRJaimFhwG44y7-4W2wV_2_4V8s0Wj9vtkvWzIyGuOx6nWoqY0ID9iE9L8GhNZdJZ6qR6IE2hRVSenWmVVa43hbLj4QCWOOOLcjGDE0L4O_HwddjoFipHzSkLFsuZyD5NQiaWwasvNroMGm0G41jJ/p.png?fv_content=true&size_mode=5" width="500" height="350" />

Inference Speend

| Norm | Iterations/second |
|:-------:|:-------------:|
| BN/MABN | 237.88 |
| Instance Normalization | 105.60 |
| Group Normalization | 99.37 |
| Layer Normalization | 125.44 |

### Imagenet

| Model | Normalization Batch size | Norm | Top 1 Accuracy |
|:--------:|:-----------:|:----:|:------:|
| ResNet50 | 32 | BN | 23.41 |
| ResNet50 | 2 | BN | 35.22 |
| ResNet50 | 2 | BRN | 30.29 |
| ResNet50 | 2 | MABN | 23.67 | 

### COCO 
| Backbone | Method | Training Strategy | Norm | Batch Size | AP<sup>b</sup> | AP<sup>b</sup><sub>0.50</sub> | AP<sup>b</sup><sub>0.75</sub> | AP<sup>m</sup> | AP<sup>m</sup><sub>0.50</sub> | AP<sup>m</sup><sub>0.75</sub> |
|:-------------:|:------------:|:---------:|:----:|:------:|:----:|:----:|:----:|:----:|:----:|:----:|
| R50-FPN | Mask R-CNN | 2x from scratch | BN | 2 | 32.38 | 50.44 | 35.47 | 29.07 | 47.68 | 30.75 |
| R50-FPN | Mask R-CNN | 2x from scratch | BRN | 2 | 34.07 | 52.66 | 37.12 | 30.98 | 50.03 | 32.93 |
| R50-FPN | Mask R-CNN | 2x from scratch | SyncBN | 2x8 | 36.80 | 56.06 | 40.23 | 33.10 | 53.15 | 35.24 |
| R50-FPN | Mask R-CNN | 2x from scratch | MABN | 2 | 36.50 | 55.79 | 40.17 | 32.69 | 52.78 | 34.71 |
| R50-FPN | Mask R-CNN | 2x fine-tune | SyncBN | 2x8 | 38.25 | 57.81 | 42.01 | 34.22 | 54.97 | 36.34 | 
| R50-FPN | Mask R-CNN | 2x fine-tune | MABN | 2 | 38.42 | 58.19 | 41.99 | 34.12 | 55.10 | 36.12 |


## Demo

One node with 8 GPUs.

### Imagenet
```bash
cd /your_path_to_repo/cls

# 8 GPUs Train and Test
python3 -m torch.distributed.launch --nproc_per_node=8 train.py --gpu_num=8 \
    --save /your_path_to_logs \
    --train_dir /your_imagenet_training_dataset_dir \
    --val_dir /your_imagenet_eval_dataset_dir \
    --gpu_num=8

# Only Test the trained model
python3 -m torch.distributed.launch --nproc_per_node=1 train.py --gpu_num=1 \
    --save /your_path_to_logs \
    --val_dir /your_imagenet_eval_dataset_dir \
    --checkpoint_dir /your_path_to_checkpoint \
    --test_only
```

### COCO
Please refer to [INSTALL.md](det/INSTALL.md) for installation and dataset preparation.

To use SyncBN, please do:
```bash
cd /your_path_to_repo/det/maskrcnn_benchmar/distributed_syncbn
bash compile.sh
```
You can download the pretrained model of ResNet-50 in [here](https://www.dropbox.com/sh/fbsi6935vmatbi9/AAA2jv0EBcSgySTgZnNZ3lmPa?dl=0). Notice R-50-2.pkl is the pretrained model for SyncBN while R50-wc.pth is for MABN.


```bash
cd /your_path_to_repo/det
# Train MABN from scratch
python3 -m torch.distributed.launch --nproc_per_node=8 tools/train_net.py \
    --skip-test \
    --config-file configs/e2e_mask_rcnn_R_50_FPN_mabn_2x_from_scratch.yaml \
    DATALOADER.NUM_WORKERS 2 \
    OUTPUT_DIR /your_path_to_logs

# Train MABN fine tuning (Download the pertrained model and set the path in configs at first)
python3 -m torch.distributed.launch --nproc_per_node=8 tools/train_net.py \
    --skip-test \
    --config-file configs/e2e_mask_rcnn_R_50_FPN_mabn_2x_fine_tune.yaml \
    DATALOADER.NUM_WORKERS 2 \
    OUTPUT_DIR /your_path_to_logs

# Test model
python3 -m torch.distributed.launch --nproc_per_node=8 tools/test_net.py \
       --config-file configs/e2e_mask_rcnn_R_50_FPN_mabn_2x_from_scratch.yaml \
       MODEL.WEIGHT /your_path_to_logs/model_0180000.pth \
       TEST.IMS_PER_BATCH 8
``` 

## Thanks 

This implementation of COCO is based on [maskrcnn-benchmark](https://github.com/facebookresearch/maskrcnn-benchmark). Ref to this link for more details about [maskrcnn-benchmark](https://github.com/facebookresearch/maskrcnn-benchmark).

## Citation 

If you use Moving Average Batch Normalization in your research, please cite:
```bibtex
@inproceedings{
yan2020towards,
title={Towards Stablizing Batch Statistics in Backward Propagation of Batch Normalization},
author={Junjie Yan, Ruosi Wan, Xiangyu Zhang, Wei Zhang, Yichen Wei, Jian Sun},
booktitle={International Conference on Learning Representations},
year={2020},
url={https://openreview.net/forum?id=SkgGjRVKDS}
}
```