# LogBERT: Log Anomaly Detection via BERT

### [ARXIV](https://arxiv.org/abs/2103.04475) 

This repository provides the implementation of Logbert for log anomaly detection. 
The process includes downloading raw data online, parsing logs into structured data, 
creating log sequences and finally modeling. 

![alt](img/log_preprocess.png)

## Configuration

- Ubuntu 20.04
- NVIDIA driver 460.73.01 
- CUDA 11.2
- Python 3.8
- PyTorch 1.9.0

## Installation

This code requires the packages listed in requirements.txt.
An virtual environment is recommended to run this code

On macOS and Linux:  

```
python3 -m pip install --user virtualenv
python3 -m venv env
source env/bin/activate
pip install -r ./environment/requirements.txt
deactivate
```

Reference: https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/

An alternative is to create a conda environment:

```
    conda create -f ./environment/environment.yml
    conda activate logbert
```

Reference: https://docs.conda.io/en/latest/miniconda.html

## Experiment

Logbert and other baseline models are implemented on [HDFS](https://github.com/logpai/loghub/tree/master/HDFS), [BGL](https://github.com/logpai/loghub/tree/master/BGL), and [thunderbird]() datasets

### HDFS example

```shell script
cd HDFS

sh init.sh

# process data
python data_process.py

#run logbert
python logbert.py vocab
python logbert.py train
python logbert.py predict

#run deeplog
python deeplog.py vocab
# set options["vocab_size"] = <vocab output> above
python deeplog.py train
python deeplog.py predict 

#run loganomaly
python loganomaly.py vocab
# set options["vocab_size"] = <vocab output> above
python loganomaly.py train
python loganomaly.py predict

#run baselines

baselines.ipynb
```

### Folders created during execution

```shell script 
~/.dataset //Stores original datasets after downloading
project/output //Stores intermediate files and final results during execution
```

### mac上传文件到server

```
ssh ...
# 上传文件
scp -P 端口号 本地文件路径 root@地址:/root/autodl-tmp/
# 上传文件夹
scp -P 端口号 -r 本地文件夹 root@地址:/root/autodl-tmp/
# 本地运行
scp -P 43355 /Users/wangzhi/Downloads/dataset/HDFS_v1.zip root@region-41.seetacloud.com:/root/.dataset/

```

### 监控AutoDL状态
```
# (实时动态查看GPU（每秒刷新一次）)
watch -n 1 nvidia-smi
# (实时动态查看CPU)
htop
```

### Baseline调参记录
| result                                                       | dataset | is_logkey | hypersphere _loss | hypersphere _loss_test | window_size | mask_ratio | Layers | batch_size | num_workers | num_candidates |
| ------------------------------------------------------------ | ------- | --------- | ----------------- | ---------------------- | ----------- | ---------- | ------ | ---------- | ----------- | -------------- |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8781, TN: 549329, FP: 4039, FN: 1866<br/>Precision: 68.49%, Recall: 82.47%, F1-measure: 74.84%<br/>elapsed_time: 883.600507736206 | HDFS    | True      | True              | False                  | 128         | 0.65       | 4      | 192        | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8746, TN: 547511, FP: 5857, FN: 1901<br/>Precision: 59.89%, Recall: 82.15%, F1-measure: 69.28%<br/>elapsed_time: 932.270750284195 | HDFS    | True      | True              | False                  | 128         | 0.65       | 4      | 64         | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8361, TN: 550526, FP: 2842, FN: 2286<br/>Precision: 74.63%, Recall: 78.53%, F1-measure: 76.53%<br/>elapsed_time: 1017.3592114448547 | HDFS    | True      | True              | False                  | 128         | 0.65       | 4      | 32         | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8366, TN: 549240, FP: 4128, FN: 2281<br/>Precision: 66.96%, Recall: 78.58%, F1-measure: 72.30%<br/>elapsed_time: 1098.0065379142761 | HDFS    | True      | True              | True                   | 128         | 0.65       | 4      | 32         | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8369, TN: 550850, FP: 2518, FN: 2278<br/>Precision: 76.87%, Recall: 78.60%, F1-measure: 77.73%<br/>elapsed_time: 1035.9144899845123 | HDFS    | True      | False             | False                  | 128         | 0.65       | 4      | 32         | 0           | 6              |
|                                                              |         |           |                   |                        |             |            |        |            |             |                |
|                                                              |         |           |                   |                        |             |            |        |            |             |                |
|                                                              |         |           |                   |                        |             |            |        |            |             |                |

### 消融实验调参记录

| result                                                       | dataset | is_logkey  | hypersphere _loss | hypersphere _loss_test | window_size | mask_ratio | test_ratio | Layers | batch_size | num_workers | num_candidates |
| ------------------------------------------------------------ | ------- | ---------- | ----------------- | ---------------------- | ----------- | ---------- | ---------- | ------ | ---------- | ----------- | -------------- |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 833, TN: 55094, FP: 242, FN: 231<br/>Precision: 77.49%, Recall: 78.29%, F1-measure: 77.89%<br/>elapsed_time: 111.56463170051575 | HDFS    | True       | False             | False                  | 128         | 0.65       | 0.1        | 4      | 32         | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 825, TN: 54950, FP: 386, FN: 239<br/>Precision: 68.13%, Recall: 77.54%, F1-measure: 72.53%<br/>elapsed_time: 118.4754638671875 | HDFS    | True       | True              | True                   | 128         | 0.65       | 0.1        | 4      | 32         | 0           | 6              |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 49, TN: 55208, FP: 128, FN: 1015<br/>Precision: 27.68%, Recall: 4.61%, F1-measure: 7.90%<br/>elapsed_time: 41.09920597076416 | HDFS    | True;False | True;True         | True;True              | 128         | 0.65       | 0.1        | 4      | 32         | 0           | 6              |



## 代码改进方案

1. Cursor+Composer 用OpenRouter代替解决方案
2. 超参数搜索
3. 保存每次训练、预测的超参数和F1

