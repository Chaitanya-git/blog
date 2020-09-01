---
layout: post
toc: True
comments: True
author: Chaitanya
categories: [research, computer vision, deep learning]
title: "SpineNets: A cool new way to design convolutional networks"
description: "An overview and analysis of the paper \"SpineNet: Learning Scale-Permuted Backbone for Recognition and Localization\""
---

## Introduction
Ever since their introduction in 2012, Convolutional Neural Networks have slowly become the cornerstone for solving computer vision tasks with deep learning. They have worked phenomenally well for several vision-related tasks such as image classification, object detection and instance segmentation. They have since also been applied successfully in Natural Language Processing tasks such as sentiment analysis.

However, the fundamental network architecture of convolutional networks have primarily remained the same, with each layer producing a feature map of increasingly lower resolution, but higher depth. Such an architecture has allowed CNNs to efficiently extract a large number of high-level features from images while keeping resource requirements under check while without compromising performance.

![CNN Architecture Image]({{ site.baseurl }}/images/SpineNet/cnn.png "Example of a traditional CNN network. (VGG 16)")
{: width="100%"}

Such architectures are 'scale-decreasing' since they progressively reduce the spatial resolution of the feature map. This makes sense for tasks like image classification that only require the network to report on whether or not an object exists in an image, without having to locate where in the image it is present. For such decisions, extracting high-level features were more important than maintaining spatial information.

In tasks such as object detection, a model must not only recognize objects but must also locate the object in the image. In such scenarios, scale-decreasing architectures cannot directly be applied as they sacrifice spatial information in favour of extracting semantic features. This problem has traditionally been solved using encoder-decoder networks, where an additional decoder network is attached to the scale decreasing network. This decoder tries to recover the lost spatial information to localize an object in an image.

While this has proved to perform well, the authors of the paper argue this is not optimal. They instead propose a scale permuted architecture. The rationale they provide is that for tasks like object detection, the preservation of spatial information is essential to be able to detect the location of an object in an image accurately. A scale permuted architecture will be able to preserve spatial information better by using 'cross-scale features'. Cross-scale features combine the spatial information from feature maps of different resolutions or scales. Features that are computed at a higher resolution will help detect smaller objects and those computed at lower resolutions can help detect larger objects. In a scale permuted network, the size of feature maps can both increase and decrease with depth, unlike scale decreasing architectures, where increasing the size of feature maps is not allowed.

They back their claims with experiments that show how learning an optimal permutation of the blocks of a standard ResNet-50 using Neural Architecture Search (NAS) can improve its performance. They go on to use NAS to come up with the SpineNet models that achieve better accuracy in object recognition and localization tasks and can match the previous state of the art in image classification while using less compute.

## The 'scale-permuted' architecture

![Scale permuted architecture image]({{ site.baseurl }}/images/SpineNet/scale_permuted_general.png "Scale decreasing vs Scale permuted architecture. source: [0]")
{: width="100%"}

The typical scale permuted architecture envisioned by the authors consists of two sections:
1. A stem network which uses a standard scale decreasing architecture to learn a set of base features.
2. The scale permuted layers that are stacked on top of the stem network.

The scale permuted layers/blocks are picked from a predetermined list of blocks $\{B_1, B_2, \ldots, B_n\}$. Each of these blocks has an associated level $L_i$. A block at level $L_i$ has a resolution of $1/2^i$ of the input image resolution. Each block can be fed multiple inputs from prior blocks of multiple sizes. Some of these blocks may be connected to one or more layers of the stem network. These connections are termed as cross-scale connections.

In this paper, the models tested set 5 blocks (levels $L_3$ to $L_7$) as output blocks. In contrast, the remaining blocks are designated as intermediate blocks. The output blocks are meant to provide cross-scale features that can then be fed to subnets for classification, object recognition and localization or other vision tasks. Hence, they are each connected to 1x1 Conv layers which ensure that all the output blocks output features (named $P_3, P_4, \ldots, P_7$) with the same feature depth.

The permuted ordering of blocks at different scales and the multiple input connections to each block from multiple blocks of potentially different scales allow the network to learn robust multi-scale features. Additionally, the authors also try out certain block adjustments to improve performance.

### A closer look at cross-scale connections
In regular scale decreasing convolutional networks, the convolutional layers are designed such that the size of the output feature map of one layer matches the input feature size expected by the subsequent layer. This is ensured by picking the right stride lengths and sometimes by using pooling layers downsample the feature maps as needed. If there are multiple inputs from different layers going into a layer, the inputs are designed to be of the same resolution so that they can be added and sent into the next layer.

But the feature map sizes may not match so easily with scale permuted layers as the subsequent layer may have a wide variety of input resolutions. Providing input from multiple layers is also an issue as the two input features may not be of the same resolution in a scale permuted network. Hence the input features are first processed to have the same input feature length ($\alpha C_{in}$, where $\alpha$ is a scaling parameter), after which spatial resampling (i.e. the spatial resolution is changed) is performed using nearest neighbour upsampling or downsampled using a 3x3 Conv with stride 2 followed by a max-pooling operation. These features are now passed through another 1x1 Conv to make sure that the output feature depth is of a compatible value. The feature vectors are now added together before being sent to the next layer.

![Cross scale connections image]({{ site.baseurl }}/images/SpineNet/cross_scale.png "An illustration of scale decreasing and scale permuted feature blocks. (source: [0])")
{: width="100%"}

## Building (or learning) scale-permuted architectures
A natural complication that arises when trying to build a network with a scale permuted architecture is finding an optimal permutation of blocks to solve the task at hand. Due to a large number of permutations possible within the feature blocks, it’s prohibitively expensive to either hand-design an optimal architecture or try out every permutation in the search space. This problem is alleviated by learning the optimal permutation using Neural Architecture Search.

When using NAS, the entire network architecture is learnt in a three-step process to simplify the procedure:
    1. Learn an optimal permutation of feature blocks of different scales.
    2. Learn a set of optimal cross-scale connections between the blocks.
    3. Optionally optimize the learnt architecture by learning a set of block adjustments for each block.

### Search space for learning the optimal architecture
When designing a meta-architecture for a neural network, one needs to ensure that the search space for the learned model architecture is kept reasonably small to reduce the amount of compute and memory resources required to search for or learn the optimal model. This paper identifies and places constraints on three significant factors that decide the size of the search space for learning scale permuted architectures.

#### Scale permutations
Since a block in any given ordering of blocks can only connect to blocks that come before it in that ordering, the exact arrangement of blocks becomes vital as this decides which cross-scale connections the final network may have.
If the scale permuted network had N blocks, this would mean we would have to try out N! different permutations before we could find the optimal ordering. In this paper, the authors significantly reduce the size of the search space by restricting blocks permutations to the set of intermediate and output blocks. In other words, the 5 output blocks and the N-5 intermediate blocks are permuted among themselves. This brings down the size of the search space to (N-5)!5!

#### Cross-scale connections
If any given block were allowed to receive input from any arbitrary block before it, there would be $2^i$ connection patterns the $i$th block could have. This would mean the number of connection patterns for the entire network would be $2^{N-1}$.
The authors thus restrict each block to have only two cross-scale input connections, which significantly brings down the size of the search space to $\displaystyle\prod_{i=m}^{N+m-1} {^iC_2}$.

#### Block adjustments
Since the initially selected list of blocks may not be the optimal choice of blocks to be used when building a scale permuted model, the authors further allow NAS to search through a small set of block adjustment options to improve efficiency. The authors allow two types of adjustments to the scale permuted blocks:
1. Level adjustment: 
     The authors allow the intermediate blocks to deviate from the current level they are in by \{-1, 0, 1, 2\}. This results in a search space size of $4^{N-5}$.
2. Type adjustment:
    The authors also allow every block to take one of the two types: \{ bottleneck block (default), residual block \}. This results in a search space size of $2^N$. Residual blocks pass on the inputs they have received, along with their outputs. This allows blocks further down the line to utilize features from more than two previous blocks if needed.

## Experiments and Results
The authors of the paper test a multitude of models using the scale permuted architecture on Object detection and image classification. The models they test are mainly of two types:
1. Scale permuted ResNets
2. Custom SpineNet models that can take full advantage of the scale permuted architecture

### Scale permuted ResNet-50
![Scale permuted resnets]({{site.baseurl}}/images/SpineNet/scale_permuted_resnet.png "An illustration of the scale permuted resnets with a scale decreasing stem network attached to a scale permuted component, along with their performance on the COCO dataset. (source: [0])")
The authors train and test a few models based on the original ResNet-50 architecture to study the effects of scale permutation on model performance on vision-related tasks.

These models were built by allocating a part of the original ResNet-50 blocks to a scale decreasing stem network and allocating the rest to the scale permuted section of the network. These models are named as R[$N$]-SP[$M$] to indicate that the model has $N$ feature layers in the stem network and $M$ layers in the scale permuted part of the network. A tabulation of the models trained and the block allocations are listed below:

Model | Stem network \{$L_2, L_3, L_4, L_5$\} | scale-permuted network \{$L_2, L_3, L_4, L_5, L_6, L_7$\} |
| --- | --- | --- |
R50         | \{3,4,6,3\} | \{-\}           |
R35-SP18    | \{2,3,5,1\} | \{1,1,1,1,1,1\} |
R23-SP30    | \{2,2,2,1\} | \{1,2,4,1,1,1\} |
R14-SP39    | \{1,1,1,1\} | \{2,3,5,1,1,1\} |
R0-SP53     | \{2,0,0,0\} | \{1,4,6,2,1,1\} |
SpineNet-49 | \{2,0,0,0\} | \{1,2,4,4,2,2\} |
{: style="margin-left: 10%;font-size: 100%;"}

By gradually increasing the fraction of the ResNet-50 blocks allocated to the scale permuted sections, the authors study the benefits of the scale permuted architecture. Since they make use of the same feature blocks present in the regular ResNet-50, they ensure that the amount of compute used by the proposed models remains the same and that any improvements seen are solely due to the scale permutation introduced.

The search space for these models only includes models with varying block permutations and cross-scale permutations and block adjustments have been avoided. However, a small adaptation was introduced to the scale permuted models to allow generating multi-scale outputs. The adaption involves removing one $L_5$ block from the set of blocks available and adding an $L_6$ and $L_7$ block and setting the output feature dimensions to 256 for $L_5$, $L_6$ and $L_7$ blocks. These adaptations are necessary to be able to follow the general architecture mentioned earlier.

After testing these models on the object detection on the COCO dataset, the authors observed that as more blocks were allocated to the scale permuted section, the performance of the network improved. The SpineNet-49 model, which was learnt with block adjustments enabled, performed the best as it was able to change the blocks it was using in order to take full advantage of the scale permuted architecture.

This experiment, in my opinion, clearly shows the effectiveness of the scale permuted architecture.

### SpineNets
The authors also recognize that the scale permuted ResNets aren’t the best showcase of the potential of a scale permuted architecture as the feature blocks used by ResNets are likely to be suboptimal choices for training scale permuted networks. Therefore, they train another set of models called SpineNets, which were also learnt using NAS. However, while learning these architectures, block adjustment was also introduced into the search space, on top of scale permutation and cross-scale connections. This allows choosing a more optimal set of blocks to take advantage of the scale permuted architecture.

#### Generating larger SpineNets
The larger SpineNet models are generated from the base SpineNet-49 architecture instead of using NAS to relearn the block permutations and cross-scale connections as using NAS is also computationally expensive. This is done by using a technique called ‘block repeat’.

![Block repeat illustration]({{ site.baseurl }}/images/SpineNet/block_repeat.png "Illustration of how block repeat works. (source: [0])")
{: width="100%"}

The block repeat method works by repeating each feature block in the model multiple times. When a given block $B_k$ is repeated, the repeated instances are connected sequentially. The input going into $B_k$ is connected to the first block in the new group. The output of the last block in the repeated group is sent to all the subsequent blocks that were previously taking the output of $B_k$ as input.
### Object detection and instance segmentation
The authors also tested the scale permuted architecture on object detection and instance segmentation tasks by using SpineNets as a backbone model for RetinaNet and Mask-RCNN. Here they once again they found that SpineNets perform better when compared to ResNet-FPNs as they use a scale permuted architecture.

#### Single-stage object detection with RetinaNet
RetinaNet is a single-stage network for object detection. It utilizes an FPN as a backbone model to extract multi-scale features. It then passes it to two different subnets for classifying objects and locating them. Single-stage detectors often work by sampling many regions within an image as per a pre-defined policy and trying to find objects within them. A large number of regions sampled have no object in them, which makes it harder to train the network due to the imbalance between the individual classes and no object regions. The RetinaNet model handles this by using a focal-loss that increases the weightage given to rare classes, thereby alleviating the imbalance problem.

The authors use a SpineNet as a backbone for a RetinaNet and train the system to recognize objects and draw bounding boxes around them using the popular COCO dataset. Their experiments show that SpineNets outperform other backbone network architectures such as ResNet-FPN and NAS-FPN in both accuracy and efficiency.
![Object detection results graph]({{site.baseurl}}/images/SpineNet/RetinaNet_graph.png "Performance of scale permuted and scale decreasing models on the COCO dataset. (source: [0])")

#### Two-stage object detection with Mask-RCNN
Mask-RCNN is a two-stage detector model. It too uses an FPN model as a backbone to generate feature maps from an image. However, it passes these features to two separate networks. The first network scans these features and gives out region proposals or regions of interest. The generated ROI is combined with the image features and given to the second network, which looks at features in the ROIs to predict object classes, bounding boxes and segmentation masks.

In line with the previous experiment, the authors have also trained Mask-RCNN models with various scale-decreasing models and SpineNets as backbones. Here too, SpineNets outperform their scale-decreasing counterparts by a large margin. Moreover, they do so while also being smaller in size.

#### Real-time object detection
Unlike the previous two experiments, where the performance of SpineNets with its predecessors were compared, here the authors instead quantify the inference latency that can be achieved when using SpineNet models of different sizes in an end-to-end object detection pipeline running with NVIDIA Tensor RT on a V100 GPU.

The end to end pipeline performs additional steps like pre-processing, bounding box and score generation, post-processing and non-maximum suppression along with running the object detection model. The authors were able to run this pipeline in real-time, at over 30 frames per second.

### Image classification
Although improving image classification performance was not the primary goal of the SpineNet architecture, the authors have nevertheless tested SpineNet’s performance on image classification. They train and test SpineNet based image classifiers on two datasets, namely the popular ImageNet dataset and a more challenging dataset named the iNaturalist dataset. The iNaturalist dataset is a fine-grained classification dataset that requires networks to detect and classify different species of plants and animals. The dataset is extremely challenging because the classes are imbalanced and different classes are extremely similar visually. The dataset has over 5000 species of plants and animals.

![iNaturalist data example]({{ site.baseurl }}/images/SpineNet/iNaturalistExample.png "Example of two visually similar classes in the iNaturalist dataset (source: [2])")
{: width="100%"}

The experiments showed that SpineNet models can match the performance of a ResNet on the ImageNet dataset while using much fewer FLOPs. On the more challenging iNaturalist dataset SpineNets outperform ResNets by a large margin.

On the iNaturalist dataset, the authors found that there is an almost 5% improvement in Top-1 classification accuracy. To explain this improvement, the authors generate a new dataset from the iNaturalist dataset (named iNaturalist-bbox) in which each image consists of only the object to be detected cropped out and centered in the image, with only a small amount of the original image around it to provide local context. This ensures that all the objects are now visible in the same scale, which means both SpineNets and ResNets should perform equally well when trained to classify on this dataset, if the original improvement was due to SpineNet's ability to detect objects at different scales. The results obtained are tabulated below.


| Model         | Top-1%   | Top-5% |
| ------------- | ---      | ---    |
| SpineNet-49   | 63.9     | 86.9   |
| ResNet-50     | 59.6     | 83.3   |
{: style="margin-left: 35%;font-size: 100%;"}

Here we see that even on the modified dataset, SpineNet-49 shows an improvement of approximately 4.3% in Top-1 classification accuracy, even though all the objects are of the same scale. The authors suggest that the classification accuracy may have improved because SpineNet is able to extract fine-grained details from the image with allows to classify between classes that only have very subtle differences and also because it could use a more compact feature representation which makes it more robust against overfitting.

## Conclusion
In the SpineNet paper, the authors have identified a problem that exists in the current models that are being used for object detection and related computer vision tasks and have come up with an innovative solution to solve the identified issues. They use thoughtful experiments to show that their architectural improvements do indeed lead to noticeable performance improvement, and show how more optimizations can be made to take full advantage of the scale permuted architecture.

### Key Takeaways
To summarize, the key takeaways from the paper are:
1. Current scale decreasing models are suboptimal for solving computer vision tasks such as object detection where resolution information is essential.
2. Using the scale permuted architecture introduced by the authors can lead to more optimal networks, that are more efficient and perform better on object detection and related tasks.
3. The improvements brought about using a scale permuted model translates to tasks like fine-grained image classification, where resolution isn't important for the final classification, but can still help in extracting fine-grained features.

## References
- [0] X. Du et al., "SpineNet: Learning Scale-Permuted Backbone for Recognition and Localization", _IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)_, 2020. Available: 10.1109/cvpr42600.2020.01161.
- [1] T. Lin, P. Goyal, R. Girshick, K. He and P. Dollar, "Focal Loss for Dense Object Detection", _IEEE Transactions on Pattern Analysis and Machine Intelligence_, vol. 42, no. 2, pp. 318-327, 2020. Available: 10.1109/tpami.2018.2858826.
- [2] G. Van Horn et al., "The iNaturalist Species Classification and Detection Dataset", _2018 IEEE/CVF Conference on Computer Vision and Pattern Recognition_, 2018. Available: 10.1109/cvpr.2018.00914.
