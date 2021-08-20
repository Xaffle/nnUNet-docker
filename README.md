# nnUNet-docker
An instruction to create a training for nnUNet. To start nnUNet training in a docker container for the first time, this is all you need. (DOGE)

## Create a docker image for nnUNet training
First of all, you should have a base image contains basic dependencies and pytorch. If you don't have one, you can pull one from docker hub, for example:  
```
docker pull pytorch/pytorch:1.8.1-cuda11.1-cudnn8-devel
```

In my practice, a pytorch-lightning image exists, a simplest Dockerfile is:  
```
FROM pytorchlightning/pytorch_lightning:latest-py3.7-torch1.8
RUN pip install nnunet
```

Elsewise if you want a modified nnUNet, the dockerfile can be:
```
FROM pytorchlightning/pytorch_lightning:latest-py3.7-torch1.8
COPY nnUNet /nnUNet
RUN cd /nnUNet \
 && pip install --no-cache-dir -e . 
```

For more applications to build docker images for nnUNet, please refer to: [hubutui/nnUNet](https://github.com/hubutui/nnUNet/tree/master/docker)

Then you can build the image:  
```
docker build -t [image_tag] -f [dockerfile_path] .
```
in my practice:
```
docker build -t nnunet:base -f /nnUNet/docker/Dockerfile .
```

## Training
Here only provide a fast step-by-step tutorial to start a training task by nnUNet docker. The detail official instruction of nnUNet training configuration and usage can be found in: [nnUNet](https://github.com/MIC-DKFZ/nnunet) and [Setting up Paths](https://github.com/MIC-DKFZ/nnUNet/blob/master/documentation/setting_up_paths.md#:~:text=nnUNet_raw_data_base%3A%20This%20is%20where%20nnU-Net%20finds%20the%20raw,in%20turn%20contains%20one%20subfolder%20for%20each%20Task).  

### Step 1: prepare data set for training
The document structure and the naming rules of data files are fixed in nnUNet. You should prepare for your data set carefully according to: [Dataset conversion instructions](https://github.com/MIC-DKFZ/nnUNet/blob/master/documentation/dataset_conversion.md). That will take you some time. Some tips should be emphasized:
1. 3 folders should be created: `nnUNet_raw_data_base`, `nnUNet_preprocessed` and `nnUNet_trained_models`, whose names can be customized;  
2. Ensure that a subfolder `nnUNet_raw_data` (whose name should not be changed) is created under `nnUNet_raw_data_base`;  
3. Source data should be located under a folder `nnUNet_raw_data_base/nnUnet_raw_data/TaskXXX_TaskName`, and the file name of images and labels should be strictly consistent; Take a one-modality-image train data as an example, the image file name is `image1_0000.nii.gz`, the label file name should be `image1.nii.gz`. if not (eg, the image file name is `image1_T1_0000.nii.gz` and the label file name is `image1.nii.gz`), even if you define right into a pair in the `dataset.json`, the training program will stopped.

### Step 2: run a training container
Start a container with the command:
```
docker run -it \
-v=(dir)/XXX:/XXX \
--gpus=all \
--rm \
--name=DMH_Task001 \
--network=host \
--ipc=host \
nnunet:base
```
**NOTE:**
1. `--ipc=host` is recommended to avoid some runtime error, refer to: [Here](https://github.com/MIC-DKFZ/nnUNet/blob/master/documentation/common_problems_and_solutions.md#nnu-net-training-in-docker-container-runtimeerror-unable-to-write-to-file-torch_781_2606105346);
2. DO NOT forget to set parameter `--gpus`, otherwise the GPU will be invisible;
3. The single device training (with command `nnUNet_train`) will run on GPU 0 inside the container, if you want to train on a specific GPU device of the host(eg. device 2), set the gpu as `--gpus="device=2"`, the device 2 will be visible inside the container as GPU 0.

### Step 3: configure the data folder to the environment variables in the container
```
export nnUNet_raw_data_base="/XXX/nnUNet_raw_data_base"
export nnUNet_preprocessed="/XXX/nnUNet_preprocessed"
export RESULTS_FOLDER="/XXX/nnUNet_trained_models"
```

### Step 4: Experiment planning and preprocessing
Refer to [Experiment planning and preprocessing](https://github.com/MIC-DKFZ/nnUNet#experiment-planning-and-preprocessing)

### Step 5: start training
start a single device training with `nnUNet_train` command, take a 3D full resolution U-Net as an example:
```
nnUNet_train 3d_fullres nnUNetTrainerV2 [`task id` or `task folder name`] FOLD --npz
```
if your task folder is in `nnUNet_raw_data_base/nnUnet_raw_data/TaskXXX_TaskName`, the task id is `XXX`, the task folder name is `TaskXXX_TaskName`.
Enjoy your nnUNet training!

## Inference
It's really easy to execute inference process. Put the images to be predicted into a folder (the path of the folder can be anywhere, eg. `XXX/input`) and run:
```
nnUNet_predict -i `XXX/input` -o 'XXX/pred' -t 1 -m 3d_fullres -f all
```
More instructions can be found [here](https://github.com/MIC-DKFZ/nnUNet#run-inference).  
**NOTE:**
The default number of epochs during training is 1000 which may be too long for specific cases, and the inference process will search for the trained model with filename contains "model_final_checkpoint" which will be not be generated untill all the training epochs finished. Change the number of epochs or change the reference model name is applicable by edit the nnUNet source code and rebuild, if you need to take a prediction with the intermediate model (whose filename contains "model_best" or "model_latest") without changing the source code, just rename any one of this two models to "model_final_checkpoint" (looks not elegant).