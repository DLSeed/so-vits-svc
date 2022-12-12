# SoftVC VITS Singing Voice Conversion
## Update
> 针对sovits3.0 48khz模型推理显存占用大的问题，可以切换到[32khz的分支](https://github.com/innnky/so-vits-svc/tree/32k) 版本训练32khz的模型\
> 目前发现一个较大问题，3.0推理时显存占用巨大，6G显存基本只能推理30s左右长度音频\
> 断音问题已解决，音质提升了不少\
> 2.0版本已经移至 sovits_2.0分支\
> 3.0版本使用FreeVC的代码结构，与旧版本不通用\
> 目前音质上依然与[DiffSVC](https://github.com/prophesier/diff-svc) 有较大的差距
## 模型简介
歌声音色转换模型，通过SoftVC内容编码器提取源音频语音特征，与F0同时输入VITS替换原本的文本输入达到歌声转换的效果。同时，更换声码器为 [NSF HiFiGAN](https://github.com/openvpi/DiffSinger/tree/refactor/modules/nsf_hifigan) 解决断音问题
## 注意
当前分支是48khz的版本，推理时显存占用较大，经常会出现爆显存的问题，如果爆显存需要手动将音频切片逐片段转换，推荐切换到[32khz的分支](https://github.com/innnky/so-vits-svc/tree/32k) 训练32khz版本的模型

## 预先下载的模型文件
+ soft vc hubert：[hubert-soft-0d54a1f4.pt](https://github.com/bshall/hubert/releases/download/v0.1/hubert-soft-0d54a1f4.pt)
  + 放在hubert目录下
+ 预训练底模文件 [G_0.pth](https://huggingface.co/innnky/sovits_pretrained/resolve/main/G_0.pth) 与 [D_0.pth](https://huggingface.co/innnky/sovits_pretrained/resolve/main/D_0.pth)
  + 放在logs/48k 目录下
  + 预训练底模为必选项，因为据测试从零开始训练有概率不收敛，同时底模也能加快训练速度
  + 预训练底模训练数据集包含云灏 即霜 辉宇·星AI 派蒙 绫地宁宁，覆盖男女生常见音域，可以认为是相对通用的底模
  + 底模删除了optimizer speaker_embedding 等无关权重, 只可以用于初始化训练，无法用于推理
```shell
# 一键下载
# hubert
wget -P hubert/ https://github.com/bshall/hubert/releases/download/v0.1/hubert-soft-0d54a1f4.pt
# G与D预训练模型
wget -P logs/48k/ https://huggingface.co/innnky/sovits_pretrained/resolve/main/G_0.pth
wget -P logs/48k/ https://huggingface.co/innnky/sovits_pretrained/resolve/main/D_0.pth

```


## 数据集准备
仅需要以以下文件结构将数据集放入dataset_raw目录即可
```shell
dataset_raw
├───speaker0
│   ├───xxx1-xxx1.wav
│   ├───...
│   └───Lxx-0xx8.wav
└───speaker1
    ├───xx2-0xxx2.wav
    ├───...
    └───xxx7-xxx007.wav
```

## 数据预处理
1. 重采样至 48khz

```shell
python resample.py
 ```
2. 自动划分训练集 验证集 测试集 以及自动生成配置文件
```shell
python preprocess_flist_config.py
# 注意
# 自动生成的配置文件中，说话人数量n_speakers会自动按照数据集中的人数而定
# 为了给之后添加说话人留下一定空间，n_speakers自动设置为 当前数据集人数乘2
# 如果想多留一些空位可以在此步骤后 自行修改生成的config.json中n_speakers数量
# 一旦模型开始训练后此项不可再更改
```
3. 生成hubert与f0
```shell
python preprocess_hubert_f0.py
```
执行完以上步骤后 dataset 目录便是预处理完成的数据，可以删除dataset_raw文件夹了

## 训练
```shell
python train.py -c configs/config.json -m 48k
```

## 训练中追加说话人数据
基本类似预处理过程
1. 将新追加的说话人数据按之前的结构放入dataset_raw目录下，并重采样至48khz
```shell
python resample.py
 ```
2. 使用`add_speaker.py`重新生成训练集、验证集，重新生成配置文件
```shell
python add_speaker.py
```
3. 重新生成hubert与f0
```shell
python preprocess_hubert_f0.py
```
之后便可以删除dataset_raw文件夹了

## 推理

使用[inference_main.py](inference_main.py)
+ 更改model_path为你自己训练的最新模型记录点
+ 将待转换的音频放在raw文件夹下
+ clean_names 写待转换的音频名称
+ trans 填写变调半音数量
+ spk_list 填写合成的说话人名称

