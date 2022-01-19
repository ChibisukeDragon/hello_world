# hello-world
just a hello-world
just a try
want to see what happened

$\color{red}{说明：删除线用于READ.md的说明，以下带有删除线的说明在README.md中需要删除}$  

# 3D nested_unet模型PyTorch离线推理指导

## 1 环境准备 

### 1.1 安装环境
安装必要的依赖。
```
pip3.7 install -r requirements.txt  
```

### 1.2 获取，修改与安装开源模型代码  
```
git clone https://github.com/MrGiovanni/UNetPlusPlus.git
cd UNetPlusPlus/pytorch
pip install git+https://github.com/MIC-DKFZ/batchgenerators.git
git reset 3da7e6f03164a92e696cb6da059b1cd771b0346d --hard
pip install -e .
```
注：由于该模型需要将命令注册到环境中才能找到正确的函数入口，即使第一步中已经安装过环境所需求的包，在第二步中仍然需要一步pip来将代码注册到环境中。除此之外，每次将代码文件进行大幅度地增减时，“pip install -e .”都是必须的，否则很可能出现“import nnunet“错误。

如果在执行“pip install -e .”或在后面的实验过程中，仍然出现了除了nnunet以外的其他环境包或模块的安装或导入错误，则很可能需要重新手动安装部分包。我们在多个服务器上，已观测到仍然可能出现异常的包有以下但不仅限于：decorator, sympy, SimpleITK, matplotlib, batchgenerators==0.21, pandas, scikit-image。使用诸如“pip install batchgenerators==0.21”的方式来重新安装界面报错提示中指定的包。

### 1.3 准备数据集及环境设置

该模型是依赖于[UNET官方代码仓](https://github.com/MIC-DKFZ/nnUNet)而进行的二次开发，依据UNET的描述，整个实验流程大体可描述为“数据格式转换->数据预处理->训练->验证->推理”。中间不可跳过，因为每一个后续步骤都依赖于前一个步骤的结果。您可以参照官方说明进行数据集设置，但过于繁琐。下面我们将描述其中的核心步骤及注意事项，必要时通过提供中间结果文件来帮助我们跳过一些步骤。

#### 1.3.1 设置nnunet环境变量

参照UNET的描述，在硬盘空间充足的路径下，我们以/root/heyupeng/为例，在该路径下创建一个新的文件夹environment，用于存放相关实验数据。在其中再创建三个子文件夹：nnUNet_raw_data_base、nnUNet_preprocessed和RESULTS_FOLDER。最后修改/root/.bashrc文件，设置环境变量。
```
export nnUNet_raw_data_base="/root/heyupeng/environment/nnUNet_raw_data_base"
export nnUNet_preprocessed="/root/heyupeng/environment/nnUNet_preprocessed"
export RESULTS_FOLDER="/root/heyupeng/environment/RESULTS_FOLDER"
```
使用source命令来刷新环境变量，并且再载入NPU的环境变量。
```
source /root/.bashrc
source env_npu.sh
```
注：我们十分推荐将以上文件夹置于SSD上。如果使用的是机械硬盘，我们观察到该模型会占据大量的IO资源，导致系统卡顿。如果您还希望使用您设备上可用的GPU，则还需要额外添加以下环境变量。
```
export CUDA_VISIBLE_DEVICES=0,1,2,3
```

#### 1.3.2 获取数据集

[获取Medical Segmentation Decathlon](http://medicaldecathlon.com/)，下载其中的第三个子任务集Task03_Liver.tar，放到/root/datasets目录下，并解压。该数据集中的Task03_Liver在后续实验过程中会被裁剪展开，使用约260GB的存储空间。
```
mv ./Task03_Liver.tar /root/heyupeng/environment/
cd /root/heyupeng/environment/
tar xvf Task03_Liver.tar
```
至此，在environment（后文中environment均指代/root/heyupeng/environment/）文件夹内，文件结构应像如下所示，它们与上一节在.bashrc中设置的环境变量路径要保持相同：
```
environment/
├── nnUNet_preprocessed/
├── nnUNet_raw_data_base/
├── RESULTS_FOLDER/
├── Task03_Liver/
└── Task03_Liver.tar
```

#### 1.3.3 数据格式转换

在environment文件夹内，使用nnunet的脚本命令，对解压出的Task03_Liver文件夹中的数据进行数据格式转换。该脚本将运行约5分钟，转换结果将出现在nnUNet_raw_data_base子文件夹中。
```
nnUNet_convert_decathlon_task -i Task03_Liver -p 8
```
如果您的设备性能较差或者该命令在较长时间后都未结束，您可以将参数-p的数值调小，这将消耗更多的时间。

注：若您在之后的实验过程中想要重置实验或者数据集发生严重问题（例如读取数据时遇到了EOF等读写错误），您可以将nnUNet_preprocessed、nnUNet_raw_data_base和RESULTS_FOLDER下的文件全部删除，并从本节开始复现后续过程。

#### 1.3.4 实验计划与预处理

nnunet十分依赖数据集，这一步需要提取数据集的属性，例如图像大小、体素间距等，并生成后续实验的配置文件。若删减数据集图像，都将使后续实验配置发生变化。使用nnunet的脚本命令，对nnUNet_raw_data_base中的003任务采集信息。这个过程将持续半小时至六小时不等，具体时间依赖于设备性能，转换结果将出现在nnUNet_preprocessed子文件夹中。
```
nnUNet_plan_and_preprocess -t 003 --verify_dataset_integrity
```
我们观察到，该过程很可能意外中断却不给予用户提示信息，这在系统内存较小时会随机发生，请您确保该实验过程可以正常结束。如果您的设备性能较差或者该命令在较长时间后都未能正常结束，您可以改用下面的命令来降低系统占用，而这将显著提升运行时间。
```
nnUNet_plan_and_preprocess -t 003 --verify_dataset_integrity -tl 1 -tf 1
```
注：若在后续的实验步骤中出现形如“RuntimeError: Expected index [2, 1, 128, 128, 128] to be smaller than self [2, 3, 8, 8, 8] apart from dimension 1”的错误，请删除environment/nnUNet_preprocessed/Task003_Liver/以及environment/nnUNet_raw_data_base/nnUNet_cropped_data/下的所有文件，然后重新完成本节内容。

#### 1.3.5 拷贝实验配置文件

由于nnunet的实验计划与预处理中，对数据集的划分存在随机性，为了保证后续实验的可控性，我们提供了一份可用的实验配置文件，即设定了训练集、验证集的划分。请将这些文件覆盖到environment中。
```
cp -rf ### ### 我拷贝了两个plans的pkl和splits_final.pkl
cp -rf ### ###
```
值得注意的是，在nnUNet_preprocessed中，我们额外提供了一份change_batch.py文件，用于修改整个实验的batch_size，默认值为2，但我们必须将其修改为1，而这个属性存储在nnUNet_preprocessed下的.pkl文件中。修改batch_size的考量，一方面是受限于系统内存和显存容量，另一方面是简化推理的整体流程。使用###下的change_batch.py将配置文件中的batch_size修改为1。【###这句话可能要删掉，因为我可以直接提供bz1的配置文件。。。】
```
python change_batch.py ### 这个我目前还没做，拷贝过去就是bz1了
```
在environment中创建一个新的子文件夹名为input，用于存放待推理的图像。同时也创建一个output文件夹用于存放模型的推理输出，请勿在以上两个文件夹中存放多余无关的文件。除此之外，多创建一个bin_files文件夹备用。splits_final.pkl中存储了对数据的划分，我们需要将其中划分出来的验证集图像拷贝到指定路径下，用作我们的待推理图像，使用create_testset.py来完成文件的迁移复制。
```
cd environment
mkdir input output bin_files
python create_testset.py ./input
```

#### 1.3.6 获取权重文件

该模型采用了五重交叉验证的方法，因此作者提供的预训练的权重文件也分为五个文件夹，分别代表着5个fold（交叉）的结果。实测后，各个fold的精度都相差不大，浮动大约在1%以内，鉴于计算资源的考虑，整个实验过程我们只采用fold 0（第一个交叉实验）的结果。

下载预训练过的[模型参数权重](https://drive.google.com/drive/folders/1mY6nOoL9ddHyFqNIKqBN9l3GhPegye8H)，在environment下创建一个新的子文件夹download_models用于存放下载得到的压缩包，将该压缩包解压后得到五个文件夹及一个配置文件：fold_0, fold_1, fold_2, fold_3, fold_4, plans.pkl。将其中的fold_0文件夹和plans.pkl拷贝至environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/下，请提前创建相关子文件夹，最终文件结构目录如下：
```
environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/
├── fold_0/
│   ├── ...
│   ├── model_final_checkpoint.model
│   ├── model_final_checkpoint.model.pkl
│   └── ...
└── plans.pkl
```

#### 1.3.7 设置推理实验相关路径

后续推理实验通常要使用多个路径参数，使用时十分容易造成混淆。因为在前文中我们已经设置了nnunet环境变量，所以我们可以认为该模型的相关路径都是稳定的，不会经常变动。为了让后续的实验更加便捷，我们可以在程序中设置好路径作为默认参数。

打开项目代码中的UNetPlusPlus/pytorch/nnunet/inference/infer_path.py，修改其中设定的几个变量，我们推荐将这些变量指向environment文件夹下，但不是强求：
 - INFERENCE_INPUT_FOLDER：存放待推理图像的文件夹。
 - INFERENCE_OUTPUT_FOLDER：推理完成后，存放推理结果的文件夹。
 - INFERENCE_SHAPE_PATH：存放文件all_shape.txt的目录。在后续实验过程中会被介绍到，在当前指定目录下会生成一个all_shape.txt，存储着当前待推理图像的情报。
 - INFERENCE_BIN_INPUT_FOLDER：存放输入.bin文件的目录。
 - INFERENCE_BIN_OUTPUT_FOLDER：存放输出.bin文件的目录，这些文件是310的全部输出.bin文件。可在2.5节后观察310的默认输出路径后再设置该变量。但在这里我们可以提前预知地进行设置。
 
 最终的修改示例如下：
```
INFERENCE_INPUT_FOLDER = '/root/heyupeng2/environment/input/'
INFERENCE_OUTPUT_FOLDER = '/root/heyupeng2/environment/output'
INFERENCE_SHAPE_PATH = '/root/heyupeng2/environment/'
INFERENCE_BIN_INPUT_FOLDER = '/root/heyupeng2/environment/bin_files/'
INFERENCE_BIN_OUTPUT_FOLDER = '/root/heyupeng2/result/dumpOutput_device0/'
```
接下来修改程序脚本中的路径。打开###下的clear2345.sh，该脚本用于删除310输出结果中后缀带有2、3、4、5的冗余.bin文件（保留后缀带有1的.bin文件），并将所有的.bin文件都移动到同一个文件夹下（例如放置于device0卡的输出路径），便于后续的结果合并搜索指定子图结果。我们将该脚本中的rm命令参数替换为正确的310输出路径，该路径要与上文中的INFERENCE_BIN_OUTPUT_FOLDER保持一致。其中的mv命令，用于将4卡的输出结果全部移动到1卡上，也要保持正确。部分示例如下：
```
# 删除多余的输出.bin文件
rm -rf /root/heyupeng2/result/dumpOutput_device0/*_2.bin###这里还要改！！！！
rm -rf /root/heyupeng2/result/dumpOutput_device0/*_3.bin
rm -rf /root/heyupeng2/result/dumpOutput_device0/*_4.bin
rm -rf /root/heyupeng2/result/dumpOutput_device0/*_5.bin
# 将其他文件夹的.bin结果移动到同一个目录下
mv result/dumpOutput_device1/* result/dumpOutput_device0/
mv result/dumpOutput_device2/* result/dumpOutput_device0/
mv result/dumpOutput_device3/* result/dumpOutput_device0/
```
之后找到clearall.sh，前两个rm命令用于将所有的输入、输出.bin文件删除，以释放硬盘，应于2.2.3中的bin_save_folder和bin_real_folder保持一致。而最后的mv命令则是将environment下的验证集图像搬运到测试集中。###这里找时间改下描述，现在有点歧义

找到clearall.sh
修改其中的路径
```
###后续添加一些描述
```

#### 1.3.8 拷贝实验结果
推理需要对验证集中未经训练的27张图像进行推理，理论上在NPU上完成全部的推理，可能需要3-4天时间。由于推理过程过于繁琐，我们额外提供了一份含有在fold 0设置下的全部推理结果的附加文件，也包含在NPU上的完整推理流程下的推理结果。后文将以编号11的图像为例，讲解如何进行单幅图像的推理，而其他编号的图像也可以遵循同样的方法来得到，进而复现出所有的推理结果。所有验证集图像的编号如下：
```
# 文件名形如liver_3_0000.nii.gz、liver_128_0000.nii.gz
3, 5, 11, 12, 17, 19, 24, 25, 27, 38, 40, 41, 42, 44, 51, 52, 58, 64, 70, 75, 77, 82, 101, 112, 115, 120, 128
```
本质上，若在INFERENCE_INPUT_FOLDER中存在输入图像而在INFERENCE_OUTPUT_FOLDER中不存在结果图像，二者的差集便是模型需要进行推理的内容，接着模型便是随机挑选一张未经推理的图像进行推理，这个随机性是由多个进程的IO读取速率来决定的。我们将NFERENCE_OUTPUT_FOLDER中的某个指定编号的文件删除掉，就可以对该图像进行一次推理流程了。将###下的NPU推理结果拷贝至输出结果文件夹INFERENCE_OUTPUT_FOLDER中，并删除其中的编号为11的结果，模拟已经完成了其余26张图像的推理，并准备开始对编号11的图像进行推理。
```
cp -rf ### /root/heyupeng2/environment/output/
rm /root/heyupeng2/environment/output/liver_11.nii.gz
```

#### 1.3.9 开始模型训练【应该是不需要这一步的，但是这个可能和验证有关系，或许要放到最后一节】###
由于nnunet的后续实验依赖于前继实验的结果，所以我们必须完成至少一次完整的训练流程才能进行推理。您可以尝试按照官方的教程从头开始实验，使用如下命令即可。但从观测到的实验现象来看，作者的代码并不能达到他所描述的效果。而且模型的训练周期特别久，仅从推理的业务需求出发，我们使用上面获得的
```
nnUNet_train 3d_fullres nnUNetTrainerV2 003 0
```

因此我们可以使用上面获取的权重文件来模拟我们已经完成了训练过程。将刚才获得的fold 0文件夹拷贝至environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/中，执行nnunet的训练脚本命令。

注：首次运行该命令时，模型将开始对数据集解包，这将消耗比平时更多的时间。

### 1.4 获取[benchmark工具](https://gitee.com/ascend/cann-benchmark/tree/master/infer) 
将benchmark.x86_64或benchmark.aarch64放到当前工作目录。

## 2 离线推理 

### 2.1 生成om模型
首先让模型载入预训练好的权重，将其转化为onnx模型，输出为nnunetplusplus.onnx。其中参数--pre_mode的取值含义如下，我们在此将其置为-1：
 - --pre_mode = -1：将预训练的模型转化为onnx模型，输出文件为--file_path。
 - --pre_mode = 1：推理模式1，将INFERENCE_INPUT_FOLDER下的待推理图像切割子图，生成输入.bin文件，存放到INFERENCE_BIN_INPUT_FOLDER下。
 - --pre_mode = 2：推理模式2，将INFERENCE_BIN_OUTPUT_FOLDER下的输出.bin文件合并出推理结果，推理结果存放到INFERENCE_OUTPUT_FOLDER下。
```
python predict_simple2.py --pre_mode -1 --file_path /root/heyupeng2/environment/nnunetplusplus.onnx
```
之后我们需要将onnx转化为om模型，需要在310上执行，先使用npu-smi info查看设备状态，确保device空闲后，执行以下命令。这将生成batch_size为1的om模型，其输入文件为nnunetplusplus.onnx，输出文件为nnunetplusplus.om。
```
atc --framework=5 --model=nnunetplusplus.onnx --output=nnunetplusplus --input_format=NCDHW --input_shape="image:1,1,128,128,128" --log=debug --soc_version=Ascend310
```

### 2.2 删除指定的待推理图像的结果文件【是否需要保留？因为1.3.8已经说过了】
在1.3.8节中，我们说明过：若在INFERENCE_INPUT_FOLDER中存在输入图像而在INFERENCE_OUTPUT_FOLDER中不存在结果图像，二者的差集便是模型需要进行推理的内容，而本教程便是以编号11的图像为例进行的推理实验，并且已经删除了在INFERENCE_OUTPUT_FOLDER中的liver_11.nii.gz。如果您想指定其他的推理图像，删除其他编号的结果文件即可。
```
# 全部验证集图像的编号：3, 5, 11, 12, 17, 19, 24, 25, 27, 38, 40, 41, 42, 44, 51, 52, 58, 64, 70, 75, 77, 82, 101, 112, 115, 120, 128
rm /root/heyupeng2/environment/output/liver_11.nii.gz
```

### 2.3 数据预处理后切割子图，生成待输入bin文件

遵从UNET的实验流程，一张待推理的图像会被切割出1000至4000张的子图，我们需要将这些子图存储为.bin文件，存放在INFERENCE_BIN_INPUT_FOLDER下。使用predict_simple2.py，推理模式设为1。
```
python predict_simple2.py --pre_mode 1
```
该程序执行成功后，会在INFERENCE_BIN_INPUT_FOLDER下生成大量的.bin文件，并且在INFERENCE_SHAPE_PATH下生成一个all_shape.txt文件，该文件存储了当前待输入图像的部分属性信息，这些信息将在后续的实验过程中帮助输出.bin的结果合并。

注：请确保有充足的硬盘空间。若使用310设备，遵从UNET的实验流程设计，推理一副图像，预计消耗200GB至800GB（多为300GB左右，上限受原始图像尺寸影响，800GB是一个估计上限）的额外存储空间，耗时半小时至两小时不等。待推理的图像共有27张，不可能一次性将所有图像都推理完毕，因此我们只能采用逐个图像推理，之后立即做结果合并，然后删除掉使用过的bin文件，重复此过程。

### 2.4 生成info文件

对上述生成的预处理数据.bin，生成对应的info文件，作为benchmark工具推理的输入，将结果命名为nnunetplusplus.info。
```
python gen_dataset_info.py bin ./environment/bin_files nnunetplusplus_prep_bin.info 128 128
```
这个操作同时会额外生成四个子文件：sth1.info, sth2.info, sth3.info, sth4.info。它们是对nnunetplusplus_prep_bin.info的不重叠的有序拆分，有了这些拆分的info文件，便于我们同步使用4个310设备进行推理，加快实验进度。

### 2.5 使用benchmark工具进行推理

确保device空闲后，使用benchmark工具同步开启一个或四个进程进行推理。参数-device_id指代了使用的设备编号，-om_path指代了使用的om模型，-input_text_path指代了采用的info文件，-output_binary=True指代了将结果保存为.bin。
```
# 方法一：使用总的nnunetplusplus_prep_bin.info，使用1个310进行推理
./benchmark.x86_64 -model_type=vision -device_id=0 -batch_size=1 -om_path=./environment/nnunetplusplus.om -input_text_path=nnunetplusplus_prep_bin.info -input_width=128 -input_height=128 -output_binary=True -useDvpp=False

# 方法二：使用拆分的四个info，使用4个310进行推理，全部推理结束后必须使用clear2345.sh脚本。可以通过打开四个session来完成
./benchmark.x86_64 -model_type=vision -device_id=0 -batch_size=1 -om_path=./environment/nnunetplusplus.om -input_text_path=sth1.info -input_width=128 -input_height=128 -output_binary=True -useDvpp=False
./benchmark.x86_64 -model_type=vision -device_id=1 -batch_size=1 -om_path=./environment/nnunetplusplus.om -input_text_path=sth2.info -input_width=128 -input_height=128 -output_binary=True -useDvpp=False
./benchmark.x86_64 -model_type=vision -device_id=2 -batch_size=1 -om_path=./environment/nnunetplusplus.om -input_text_path=sth3.info -input_width=128 -input_height=128 -output_binary=True -useDvpp=False
./benchmark.x86_64 -model_type=vision -device_id=3 -batch_size=1 -om_path=./environment/nnunetplusplus.om -input_text_path=sth4.info -input_width=128 -input_height=128 -output_binary=True -useDvpp=False
```
这会在当前路径下，自动生成result文件夹，里面有形如dumpOutput_device0的子文件夹，存放着对info文件中记录的每个输入.bin的推理输出.bin，“device0”指在第0个设备上的运行结果。而生成的形如perf_vision_batchsize_1_device_0.txt的文件，则记录了310推理过程中的部分指标与性能。

注：本节内容将会产生大量的输出.bin文件，请使用df -h及时观测硬盘剩余空间。如果实验进行到一半，硬盘空间紧张，请查阅下节内容。

### 2.6 清除多余的结果

上节中使用的benchmark工具，对每张输入.bin会输出五个输出结果.bin文件，而只有其中之一是我们所需要的结果。执行以下脚本将多余的.bin删除。
```
bash clear2345.sh
```
注：clear2345.sh脚本可与前一节同步使用。及时使用df -h命令查看硬盘剩余空间，适时调用该脚本清理多余的后缀为2、3、4、5的输出.bin文件，使得该实验仍可以在存储空间较小的设备上运行。以4卡并行为例，每半小时运行一次该脚本，可以清理出约150GB-200GB的存储空间。在前一节内容全部完成后，也要调用一次该脚本，将4卡的结果都移动到dumpOutput_device0文件夹中，保证dumpOutput_device0文件夹中保留有全部的输出.bin文件。

### 2.7 将结果.bin文件合并为最终推理结果

将result/dumpOutput_device0/下的.bin文件做结果合并。生成的推理结果会输出到INFERENCE_OUTPUT_FOLDER下。
```
python predict_simple2.py --pre_mode 2
```

### 2.8 重复实验

截止目前，我们已经完成了1张编号为11的待推理图像的推理结果，执行以下脚本删除实验过程中的输入输出.bin文件，释放硬盘空间。
```
bash clearall.sh
```
若用户希望复现其他结果，请重复2.2至2.7的步骤，直至全部的验证集图片都推理完毕。

### 2.9 精度评测

推理完成后，我们需要对全部结果做精度验证。将INFERENCE_OUTPUT_FOLDER下的结果拷贝至environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/fold_0/validation_raw/下，请用户自行创建相关子文件夹。
```
cp -rf environment/output/* environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/fold_0/validation_raw/
```
请确保有27个结果图像已经位于上述的validation_raw文件夹中。然后使用nnUNet_train脚本命令开始评测精度，--validation_only表明我们不需要重新训练，直接进入验证步骤。
```
nnUNet_train 3d_fullres nnUNetTrainerV2 003 0 --validation_only
```
注：首次运行该命令时，模型将开始对数据集解包，这将消耗比平时更多的时间。

实验的精度将记录在environment/RESULTS_FOLDER/nnUNet/3d_fullres/Task003_Liver/nnUNetPlusPlusTrainerV2__nnUNetPlansv2.1/fold_0/validation_raw/summary.json中，您可以参照如下的结构树来找到其中的Dice指标。
```
summary.json
├── "author"
├── "description"
├── "id"
├── "name"
├── "results"
│   ├── "all"
│   └── "mean"
│       ├── "0"
│       ├── "1"
│       │   ├── ...
│       │   ├── "Dice": 0.9655123016429166
│       │   └── ...
│       └── "2"
│           ├── ...
│           ├── "Dice": 0.719350267858144
│           └── ...
├── "task": "Task003_Liver"
└── "timestamp"
```

### 2.10 性能评测

GPU上的性能使用onnx_infer.py来计算，需要在T4服务器上执行。
```
python onnx_infer.py nnunetplusplus.onnx 1,1,128,128,128
```
NPU上的性能使用benchmark工具来计算，需要在310服务器上执行
```
/benchmark.x86_64 -round=20 -om_path=nnunetplusplus.om -device_id=0 -batch_size=1
```

 **评测结果：**   
| 模型      | 官网pth精度  | 310离线推理精度  | 基准性能    | 310性能    |
| :------: | :------: | :------: | :------:  | :------:  | 
| 3D nested_unet bs1  | [Liver 1_Dice (val):95.80, Liver 2_Dice (val):65.60](https://github.com/MrGiovanni/UNetPlusPlus/tree/master/pytorch) | Liver 1_Dice (val):96.55, Liver 2_Dice (val):71.97 |  0.3731fps | 0.9414fps | 

备注：
该模型不支持batch 16，甚至batch 2都难以在310上使用。所以该教程全程使用了batch 1。
