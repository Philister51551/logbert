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

### 调参记录
| result                                                       | dataset | window_size | Mask Ratio | MLKP | VHM   | Layers | num_workers |      |      |      |      |
| ------------------------------------------------------------ | ------- | ----------- | ---------- | ---- | ----- | ------ | ----------- | ---- | ---- | ---- | ---- |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 6890, TN: 550568, FP: 2800, FN: 3757<br/>Precision: 71.10%, Recall: 64.71%, F1-measure: 67.76%<br/>elapsed_time: 730.9638893604279 | HDFS    | 20/128      | 0.45       | True | False | 2      | 0           |      |      |      |      |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 7132, TN: 551095, FP: 2273, FN: 3515<br/>Precision: 75.83%, Recall: 66.99%, F1-measure: 71.14%<br/>elapsed_time: 721.8566524982452 | HDFS    | 20          | 0.45       | True | False | 4      | 0           |      |      |      |      |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8781, TN: 549328, FP: 4040, FN: 1866<br/>Precision: 68.49%, Recall: 82.47%, F1-measure: 74.83%<br/>elapsed_time: 967.1170127391815 | HDFS    | 20          | 0.65       | True | False | 4      | 0           |      |      |      |      |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8784, TN: 550443, FP: 2925, FN: 1863<br/>Precision: 75.02%, Recall: 82.50%, F1-measure: 78.58%<br/>elapsed_time: 930.6453251838684 | HDFS    | 128         | 0.65       | True | False | 4      | 16          |      |      |      |      |
| best threshold: 0, best threshold ratio: 0.0<br/>TP: 8781, TN: 549329, FP: 4039, FN: 1866<br/>Precision: 68.49%, Recall: 82.47%, F1-measure: 74.84%<br/>elapsed_time: 859.1613080501556 | HDFS    | 128         | 0.65       | True | False | 4      | 0           |      |      |      |      |

