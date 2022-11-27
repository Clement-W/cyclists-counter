# Bycicle Counter
UC Davis ECS 171 FInal Project

## Abstract

Group members:
- Timothy Blanton
- Sohan Patil
- Ru Han Wang
- Clément Weinreich

The goal of our project is to address the problem of adapting infrastructure to cyclist flows. On campus, for example, it is difficult to estimate the number of bicycle parking spaces needed for different areas and buildings. We therefore want to create an automatic detector and counter of cyclists using a Jetson Nano. Once our project is completed, it would be possible to capture data about the flow of cyclists at different locations, and to use this data to make predictions or estimates of cyclist flows. To complete this project, we plan to use transfer-learning on a pre-trained object detection model, which means re-train the classification layer of the deep convolutional neural network on a custom dataset in order to create our own cyclist detector. To do so, we plan to use the [Tensorflow 2 Object Detection API](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/index.html), and use one of their pre-trained model which will be a deep convolutional neural network. The dataset we will use is the [Cyclist Dataset for Object Recognition](https://www.kaggle.com/datasets/semiemptyglass/cyclist-dataset) published by [1] in 2016. It contains 13.7k labeled images of size 2048 x 1024, recorded from a moving vehicle in the urban traffic of Beijing. The labels contain the positions of the bounding boxes around the cyclists in this format: `id center_x center_y width height`. Then, we plan to implement our own algorithmic-based tracking system that would allow us to count the number of cyclists detected over a period of time.

## Setup Requirements 

Follow the steps below, or clone the repository and run the notebook [setup_install.ipynb](setup_install.ipynb).

* Clone the repository :
```
git clone git@github.com:Clement-W/bicycle-counter.git
cd bicycle-counter
```

* Download the data :
```
wget --load-cookies /tmp/cookies.txt "https://docs.google.com/uc?export=download&confirm=$(wget --quiet --save-cookies /tmp/cookies.txt --keep-session-cookies --no-check-certificate 'https://docs.google.com/uc?export=download&id=1u39ZCDroyUpguicPMUZ20eIeux2N7uql' -O- | sed -rn 's/.*confirm=([0-9A-Za-z_]+).*/\1\n/p')&id=1u39ZCDroyUpguicPMUZ20eIeux2N7uql" -O images.zip && rm -rf /tmp/cookies.txt
```
The data can also be download manually [here](https://drive.google.com/file/d/1u39ZCDroyUpguicPMUZ20eIeux2N7uql/view?usp=sharing).

* Unzip and delete the zip file :
```
unzip images.zip
rm images.zip
```

* [Optional if you don't want to train a model] Install every dependences necessary to train an object detection neural net with tensorflow object detection API. To do so, you can follow the installation instructions from the [official tensorlow object detection API](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/install.html).

Now you're all set!

## Data Exploration

The data exploration phase is split into 2 notebooks :
1. [adapt_dataset.ipynb](adapt_dataset.ipynb) 

The first notebook contains the steps to adapt the original dataset, and make it usable for our project. In fact, when we downloaded the dataset, we discovered that 1623 images out of 13674 were not labelled. These images had an empty `.txt` file as a label. Thus we decided to remove theses images, so the dataset was reduced to 12051 images. Thanks to the python library `pylabel`, we loaded the dataset as a dataframe in order to explore this new version of the dataset. This library was useful in particular to convert the label format from Yolov5 to VOC XML. As the tensorflow object detection API works with the VOC XML format, this step was mandatory. So we removed the `.txt` label and keept the new `.xml` labels. After that, we used a script from the tensorflow object detection API to split our dataset into 3 sets : 
* 90% for the train set (9760 images)
* 10% for the test set (1206 images)
* 10% of the 90% train set for the validation set (1085 images)


All these manipulations necessited to manage folders and files with command lines, that's why this notebook can't be run anymore. Before executing this notebook, the main folder looked like this:

* `images/` contains 13 674  images
* `labels/` contains 13 674 .txt files (yolo bounding box format) with this format `id center_x center_y width height`


After running this notebook the main folder looked like this:

* `adapt_dataset.ipynb` this jupyter notebook
* `tensorflow-scripts/` contains the scripts from tensorflow
    * `partition_dataset.py` python script to split a folder of images with labels into 2 subfolders train and test
* `images/` contains the data
    * `train/` contains 9760 images and labels of the train set 
    * `test/` contains 1085 images and labels of the test set
    * `validation/` contains 1206 images and labels of the validation set

<!-- #region -->
Now that the dataset is usable for the project, we can perform some data analysis on it.

2. [data_exploration.ipynb](data_exploration.ipynb)

This notebook contains all the work done to explore the dataset. In this notebook, we:
* Load the dataset as a pandas dataframe thanks to the pylabel library
* Analyze the the number of images, of bounding boxes
* Analyze the images 
* Analyze the repartitions of the bounding boxes
* Visualize some samples from train,test and validation set.

## Data Preprocessing

### Data preprocessing with tensorflow 2 object detection API

For our project, the data preprocessing was mixed with the data exploration, because the dataset needed to be modified in order to be compatible with the task we want to perform. But now that the dataset has been prepared, and that we explored it a little bit more, we can take a look at the data preprocessing we will setup in order to use the data during training.

The Tensorflow 2 Object Detection API allows to do data preprocessing and data augmentation in the training pipeline configuration. As stated in their [docs](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/configuring_jobs.md#configuring-the-trainer), all the preprocessing of the input is done in the `train_config` part of the training configuration file. The training configuration file is explained in the section [Configure the training pipeline](#configure-the-training-pipeline) of this README.md. 

So here, the important part of the configuration file is `train_config` which parametrize:
* Model parameter initialization
* Input Preprocessing
* SGD parameters

Here we will focus on the Input Preprocessing part of the config file. All this preprocessing is included in the `data_augmentation_options` tag of the `train_config`. This data_augmentation_options can take several values that are listed [here](https://github.com/tensorflow/models/blob/master/research/object_detection/protos/preprocessor.proto). And [this file](https://github.com/tensorflow/models/blob/master/research/object_detection/builders/preprocessor_builder_test.py) also explains how to write them into the config file. 

### Data Augmentation

The pipeline that specify the data augmentation options that will be applied to the images is created in [configure_training_pipeline.ipynb](configure_training_pipeline.ipynb).

First, our images are very big (2048x1024). Thus, it is important to resize them in order to make it compatible with the input layer of the neural network we'll use. The model that we chose has an input layer of 640x640. And the first layer consists in an [image-resize layer](https://www.tensorflow.org/api_docs/python/tf/keras/layers/Resizing). So we don't need to deal with the size of our images, as these images will be automatically resized to the desired input size when feeded to the network.

Now, let's focus on data augmentation. Data augmentation englobe techniques used to increase the amount of data, by adding to the dataset slightly modified copies of already existing data. Data augmentation helps to reduce overfitting by helping the network to generalize over different examples. This is closely related to oversampling. Here, we used 3 different methods to augment our data:
* **random_scale_crop_and_pad_to_square**: Randomly scale, crop, and then pad the images to fixed square dimensions. The method sample a random_scale factor from a uniform distribution between scale_min and scale_max, and then resizes the image such that its maximum dimension is (output_size * random_scale). Then a square output_size crop is extracted from the resized image. Lastly, the cropped region is padded to the desired square output_size (640x640 here) by filling the empty values with zeros.
* **random_horizontal_flip**: Randomly flips the image and detections horizontally, with a probability p. Here we chose p=0.3, so the probability that an image is horizontally flipped is 30%.
* **random_distort_color**: Randomly distorts color in images using a combination of brightness, hue, contrast and
  saturation changes. By using the parameter `color_ordering=1`, the sequence of adjustment performed is :
  1. randomly adjusting brightness
  2. randomly adjusting contrast
  3. randomly adjusting saturation 
  4. randomly adjusting hue.
  
We choosed to use the random scale,crop and pad to square data augmentation option because during the data exploration phase, we noticed that most of the cyclists were in the center of the image. Thus, to give different examples to the model, this data augmentation option will create other images where the cyclists won't be in the center of the image.

We choosed to use the horizontal_flip data augmentation option because this will create more examples to train the network. As the cyclists can come from the left,right or front of the camera, this data augmentation option will help the network to see diverse cases of cyclist positions.

We choosed to use the random_distort_color data augmentation option because during the data exploration phase, we noticed that the luminosity of the images are low, with a low contrast and a low saturation. Thus, this data augmentation option will help the network to see other examples with a different brightness, contrast, saturation and hue.
  
In the training configuration file, this will looks like this:
```py
  data_augmentation_options {
    random_horizontal_flip {
      probability: 0.3
    }
  }
  data_augmentation_options {
    random_scale_crop_and_pad_to_square {
      output_size: 640
      scale_min: 0.1
      scale_max: 2.0
    }
  }
  data_augmentation_options {
    random_distort_color {
      color_ordering: 1
    }
  }
```

Now that we have explored, preprocessed and set up a data augmentation pipeline for our data, we can move to the training job.

## Configure the training job

To follow this part, we assume you installed the required libraries to train an object detection neural net with the tensorflow 2 object detection API. If not, just follow the instructions [here](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/install.html).

### The training workspace

To train our object detection model, we followed the [documentation](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html) of the Tensorflow2 Object Detection API. Thus, we organized the training workspace `training-workspace` the same way as it is recommended:

* annotations: This folder will be used to store all *.csv files and the respective TensorFlow *.record files, which contain the list of annotations for our dataset images.

* exported-models: This folder will be used to store exported versions of our trained model(s).

* models: This folder will contain a sub-folder for each of training job. Each subfolder will contain the training pipeline configuration file *.config, as well as all files generated during the training and evaluation of our model.

* pre-trained-models: This folder will contain the downloaded pre-trained models, which shall be used as a starting checkpoint for our training jobs.

### Generate .record files [(generate_tfrecords.ipynb)](generate_tfrecords.ipynb)

The Tensolfow API use what we call tf record files to store the data. It is a simple format that contains both the images and the labels in one file. To generate these files, we followed the documentation. Everything is explained in the notebook generate_tfrecords.ipynb. In the end, this add 3 new files to the folder `training-workspace/annotations`:
* `train.record`: the train set
* `validation.record`: the validation set
* `test.record`: the test set

The .record files are associated to a `label_map` file which tells the classes that must be classified in the dataset. Here we only want to classify the cyclists, so the label map is very simple:
```py
item {
    id: 1
    name: 'cyclist'
}
```
This label map is stored in `training-workspace/annotations/label_map.pbtxt` along with the .record files.

### Download Pre-Trained Model [(download_pretrained_network.ipynb)](download_pretrained_network.ipynb)

The pre-trained object detection models of the tensorflow object detection API are listed [here](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/tf2_detection_zoo.md). Many different architecture exists such as RCNN, Faster-RCNN, SSD, etc. But today, it is EfficientNet based models (EfficientDet) that provide the best overall performances, and can work well for low latency applications. For example, `EfficientDet D1 640x640` can perform inference in 54ms on a nvidia-tesla-v100GPU, and obtain a COCO mAP (mean average precision) of 38.4 (the metrics will be discussed in the sections [Comments on training](#comments-on-training) [Evaluate the model on Test data](#evaluate-the-model-on-test-data). We can try to use different pre-trained models and compare the performances, but this wil be done for a second version of this project.

The EfficientNet architecture has been proposed by [2] (EfficientNet: Rethinking Model Scaling for Convolutional Neural Network) in 2019. Here is the model architecture of the base EfficientNet model:

<img src="markdown-images/efficientDet-architecture.png" alt="model architecture" style="height: 200px"/>

This table (from the original paper [2]) shows that the main block of this architecture is the MBConv. The MBConv is an inverted linear bottleneck layer, with depth-wise separable convolution. A depth-wise separable convolution conist of splitting a normal $k\times k$ convolution in two "simpler" convolutions, to reduce the number of parameters, and speed up the compute time. The inverted linear bottleneck layer change the output of the last convolution of a classic residual block by a linear output, before it is added to the initial activation by the skip connection. This architecture has shown better results in accuracy and in computer preformance (in FLOPS).

Thus, we decided to use `EfficientDet D1 640x640` as our pre-trained object detector. The last checkpoint of the training and the training configuration file is available in the directory `training-workspace/pre-trained-models/efficientdet_d1_coco17_tpu-32`.

### Configure the training pipeline [(configure_training_pipeline.ipynb)](configure_training_pipeline.ipynb)


The TensorFlow Object Detection API uses protobuf files to configure the training and evaluation process. The config file is split into 5 parts:
* The `model` configuration. This defines what type of model will be trained (ie. meta-architecture, feature extractor). Most of this part will not be modified as we'll use a pre-trained network, we won't modify it's architecture.
* The `train_config`, which decides what parameters should be used to train model parameters (ie. SGD parameters, input preprocessing and feature extractor initialization values). This part is very important as we can preprocess our data and do some data augmentation. 
* The `train_input_reader`, which defines what dataset the model should be trained on.
* The `eval_config`, which determines what set of metrics will be reported for evaluation.
* The `eval_input_reader`, which defines what dataset the model will be evaluated on. That's why we created a validation set for our data, in addition to the test set.

To redefine the training configuration through the file `pipeline.config`, we created the directory `efficientdet_d1_v1` into `training-workspace/models`. Then, we copied the configuration file into this directory in order to modify it, while keeping the original configuration file in the folder `training-workspace/pre-trained-models/efficientdet_d1_coco17_tpu-32`. 

The changes that have been done to this configuration file are:
* In `model`, we changed `num_classes` to 1 as we only want to predict the cyclists. This corresponds to changing the last dense layer of the neural network with only one neuron.


* In `train_config`, we changed the `batch_size` to 3 in order to fit the memory of the GPU used to train the network.
* In `train_config`, we added some data augmentation options (see below).
* In `train_config`, changed the base learning rate to 0.02 to avoid exploding gradients.
* In `train_config`, we changed the path of `fine_tune_checkpoint` to the correct path, so the weights of the model are initialized at the last checkpoint.
* In `train_config`, we changed `fine_tune_checkpoint_type` from classification to detection since we want to be training the full detection model and not only the classification part.


* In `train_config`, we changed `use_bfloat16` to false as we are not training on a TPU but a GPU.


* In `train_input_reader`, we changed `label_map_path` to the correct path of the label map for training.
* In `train_input_reader`, we changed `input_path` to the path of the training set so `train.record`.


* In `eval_input_reader`, we changed `label_map_path` to the correct path of the label map for evaluation.
* In `eval_input_reader`, we changed `input_path` to the path of the validation set so `validation.record`.

For more details about the training configuration file, check the notebook `configure_training_pipeline.ipynb`.
<!-- #endregion -->

## Train the model

[follow docs](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html#training-the-model)

## Monitor the training job using Tensorboard

[follow docs](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html#monitor-training-job-progress-using-tensorboard)

## Comments on training

(show curves and images of tensorbard)

## Evaluate the model on Test data

[follow docs](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html#evaluating-the-model-optional)

## Export the model

[follow docs](https://tensorflow-object-detection-api-tutorial.readthedocs.io/en/latest/training.html#monitor-training-job-progress-using-tensorboard)

## Counting algorithm

TODO

## Setup of the Jetson Nano

to complete

## Inference on Jetson Nano

TODO

## Results

TODO

## References

[1] X. Li, F. Flohr, Y. Yang, H. Xiong, M. Braun, S. Pan, K. Li and D. M. Gavrila. A New Benchmark for Vision-Based Cyclist Detection. In Proc. of the IEEE Intelligent Vehicles Symposium (IV), Gothenburg, Sweden, pp.1028-1033, 2016.

[2] Tan, Mingxing, and Quoc Le. "Efficientnet: Rethinking model scaling for convolutional neural networks." International conference on machine learning. PMLR, 2019.
