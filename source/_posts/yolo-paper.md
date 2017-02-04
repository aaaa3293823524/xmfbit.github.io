---
title: YOLO 论文阅读
date: 2017-02-04 18:49:22
tags:
    - paper
    - yolo
---
YOLO(**Y**ou **O**nly **L**ook **O**nce)是一个流行的目标检测方法，和Faster RCNN等state of the art方法比起来，主打检测速度快。截止到目前为止（2017年2月初），YOLO已经发布了两个版本，在下文中分别称为[YOLO V1](https://arxiv.org/abs/1506.02640)和[YOLO V2](https://arxiv.org/abs/1612.08242)。YOLO V2的代码目前作为[Darknet](http://pjreddie.com/darknet/yolo/)的一部分开源在[GitHub]()。在这篇博客中，记录了阅读YOLO两个版本论文中的重点内容，并着重总结V2版本的改进。

![YOLO V2的检测效果示意](/img/yolo2_result.png)
<!-- more -->

## YOLO V1
这里不妨把YOLO V1论文["You Only Look Once: Unitied, Real-Time Object Detection"](https://arxiv.org/abs/1506.02640)的摘要部分意译如下：

> 我们提出了一种新的物体检测方法YOLO。之前的目标检测方法大多把分类器重新调整用于检测（这里应该是在说以前的方法大多将检测问题看做分类问题，使用滑动窗提取特征并使用分类器进行分类）。我们将检测问题看做回归，分别给出bounding box的位置和对应的类别概率。对于给定输入图像，只需使用CNN网络计算一次，就同时给出bounding box位置和类别概率。由于整个pipeline都是同一个网络，所以很容易进行端到端的训练。
> YOLO相当快。base model可以达到45fps，一个更小的model（Fast YOLO），可以达到155fps，同时mAP是其他可以实现实时检测方法的两倍。和目前的State of the art方法相比，YOLO的localization误差较大，但是对于背景有更低的误检（False Positives）。同时，YOLO能够学习到更加泛化的特征，例如艺术品中物体的检测等任务，表现很好。

和以往较为常见的方法，如HoG+SVM的行人检测，DPM，RCNN系方法等不同，YOLO接受image作为输入，直接输出bounding box的位置和对应位置为某一类的概率。我们可以把它和目前的State of the art的Faster RCNN方法对比。Faster RCNN方法需要两个网络RPN和Fast RCNN，其中前者负责接受图像输入，并提出proposal。后续Fast RCNN再给出这些proposal是某一类的概率。也正是因为这种“直来直去”的方法，YOLO才能达到这么快的速度。也真是令人感叹，网络参数够多，数据够多，什么映射关系都能学出来。。。下图就是YOLO的检测系统示意图。在test阶段，经过单个CNN网络前向计算后，再经过非极大值抑制，就可以给出检测结果。
![YOLO V1检测系统示意图](/img/yolo1_detection_system.png)

### 基本思路
![基础思路示意图](/img/yolo1_basic_idea.png)

- 网格划分：将输入image划分为$S \times S$个grid cell，如果image中某个object box的中心落在某个grid cell内部，那么这个cell就对检测该object负责（responsible for detection that object）。同时，每个grid cell同时预测$B$个bounding box的位置和一个置信度。这个置信度并不只是该bounding box是待检测目标的概率，而是该bounding box是待检测目标的概率乘上该bounding box和真实位置的IoU的积。通过乘上这个交并比，反映出该bounding box预测位置的精度。如下式所示：

$$\text{confidence} = P(\text{Object})\times \text{IoU}_{\text{pred}}^{\text{truth}}$$

- 网络输出：每个bounding box对应于5个输出，分别是$x,y,w,h$和上述提到的置信度。其中，$x,y$代表bounding box的中心离开其所在grid cell边界的偏移。$w,h$代表bounding box真实宽高相对于整幅图像的比例。$x,y,w,h$这几个参数都已经被bounded到了区间$[0,1]$上。除此以外，每个grid cell还产生$C$个条件概率，$P(\text{Class}_i|\text{Object})$。注意，我们不管$B$的大小，每个grid cell只产生一组这样的概率。在test的非极大值抑制阶段，对于每个bounding box，我们应该按照下式衡量该框是否应该予以保留。
$$\text{confidence}\times P(\text{Class}_i|\text{Object}) = P(\text{Class}_i)\times \text{IoU}_{\text{pred}}^{\text{truth}}$$

- 实际参数：在PASCAL VOC进行测试时，使用$S = 7$, $B=2$。由于共有20类，故$C=20$。所以，我们的网络输出大小为$7\times 7 \times 30$

### 网络模型结构
Inspired by GoogLeNet，但是没有采取inception的结构，simply使用了$1\times 1$的卷积核。base model共有24个卷积层，后面接2个全连接层，如下图所示。
![YOLO的网络结构示意图](/img/yolo1_network_arch.png)

另外，Fast YOLO使用了更小的网络结构（9个卷积层，且filter的数目也少了），其他部分完全一样。

### 训练
同很多人一样，这里作者也是先在ImageNet上做了预训练。使用上图网络结构中的前20个卷积层，后面接一个average-pooling层和全连接层，在ImageNet 1000类数据集上训练了一周，达到了88%的top-5准确率。

由于[Ren的论文](https://arxiv.org/abs/1504.06066)提到给预训练的模型加上额外的卷积层和全连接层能够提升性能，所以我们加上了剩下的4个卷积层和2个全连接层，权值为随机初始化。同时，我们把网络输入的分辨率从$224\times 224$提升到了$448 \times 448$。

在最后一层，我们使用了线性激活函数；其它层使用了leaky ReLU激活函数。如下所示：
$$
f(x)=
\begin{cases}
x, &\text{if}\ x > 0 \\\\
0.1x, &\text{otherwise}
\end{cases}
$$

很重要的一点就是定义很好的loss function，为此，作者提出了一下几点：

- loss的形式采用误差平方和的形式（真是把回归进行到底了。。。）
- 由于很多的grid cell没有目标物体存在，所以给有目标存在的bounding box和没有目标存在的bounding box设置了不同的比例因子进行平衡。具体来说，
$$\lambda_{\text{coord}} = 5，\lambda_{\text{noobj}} = 0.5$$
- 直接使用$w$和$h$，这样大的box的差异在loss中占得比重会和小的box不平衡，所以这里使用$\sqrt{w}$和$\sqrt{h}$。
- 上文中已经提到，同一个grid cell会提出多个bounding box。在training阶段，我们只想让一个bounding box对应object。所以，我们计算每个bounding box和ground truth的IoU，以此为标准得到最好的那个bounding box，其他的认为no obj。

loss函数的具体形式见下图（实在是不想打这一大串公式。。）。其中，$\mathbb{1}_i$表示是否有目标出现在第$i$个grid cell。

$\mathbb{1}_{i,j}$表示第$i$个grid cell的第$j$个bounding box是否对某个目标负责。
![YOLO的损失函数定义](/img/yolo1_loss_fun.png)

在进行反向传播时，由于loss都是二次形式，所以导数形式都是统一的。下面是Darknet中[detection_layer.c](https://github.com/pjreddie/darknet/blob/master/src/detection_layer.c)中训练部分的代码。在代码中，计算了cost（也就是loss）和delta（也就是反向的导数），

``` c
if(state.train){
    float avg_iou = 0;
    float avg_cat = 0;
    float avg_allcat = 0;
    float avg_obj = 0;
    float avg_anyobj = 0;
    int count = 0;
    *(l.cost) = 0;
    int size = l.inputs * l.batch;
    memset(l.delta, 0, size * sizeof(float));
    for (b = 0; b < l.batch; ++b){
        int index = b*l.inputs;
        // for each grid cell
        for (i = 0; i < locations; ++i) {   // locations = S * S = 49
            int truth_index = (b*locations + i)*(1+l.coords+l.classes);
            int is_obj = state.truth[truth_index];
            // for each bbox
            for (j = 0; j < l.n; ++j) {     // l.n = B = 2
                int p_index = index + locations*l.classes + i*l.n + j;
                l.delta[p_index] = l.noobject_scale*(0 - l.output[p_index]);
                // 因为no obj对应的bbox很多，而responsible的只有一个
                // 这里统一加上，如果一会判断该bbox responsible for object，再把它减去
                *(l.cost) += l.noobject_scale*pow(l.output[p_index], 2);  
                avg_anyobj += l.output[p_index];
            }

            int best_index = -1;
            float best_iou = 0;
            float best_rmse = 20;
            // 该grid cell没有目标，直接返回
            if (!is_obj){
                continue;
            }
            // 否则，找出responsible的bounding box，计算其他几项的loss
            int class_index = index + i*l.classes;
            for(j = 0; j < l.classes; ++j) {
                l.delta[class_index+j] = l.class_scale * (state.truth[truth_index+1+j] - l.output[class_index+j]);
                *(l.cost) += l.class_scale * pow(state.truth[truth_index+1+j] - l.output[class_index+j], 2);
                if(state.truth[truth_index + 1 + j]) avg_cat += l.output[class_index+j];
                avg_allcat += l.output[class_index+j];
            }

            box truth = float_to_box(state.truth + truth_index + 1 + l.classes);
            truth.x /= l.side;
            truth.y /= l.side;
            // 找到最好的IoU，对应的bbox是responsible的，记录其index
            for(j = 0; j < l.n; ++j){
                int box_index = index + locations*(l.classes + l.n) + (i*l.n + j) * l.coords;
                box out = float_to_box(l.output + box_index);
                out.x /= l.side;
                out.y /= l.side;

                if (l.sqrt){
                    out.w = out.w*out.w;
                    out.h = out.h*out.h;
                }

                float iou  = box_iou(out, truth);
                //iou = 0;
                float rmse = box_rmse(out, truth);
                if(best_iou > 0 || iou > 0){
                    if(iou > best_iou){
                        best_iou = iou;
                        best_index = j;
                    }
                }else{
                    if(rmse < best_rmse){
                        best_rmse = rmse;
                        best_index = j;
                    }
                }
            }

            if(l.forced){
                if(truth.w*truth.h < .1){
                    best_index = 1;
                }else{
                    best_index = 0;
                }
            }
            if(l.random && *(state.net.seen) < 64000){
                best_index = rand()%l.n;
            }

            int box_index = index + locations*(l.classes + l.n) + (i*l.n + best_index) * l.coords;
            int tbox_index = truth_index + 1 + l.classes;

            box out = float_to_box(l.output + box_index);
            out.x /= l.side;
            out.y /= l.side;
            if (l.sqrt) {
                out.w = out.w*out.w;
                out.h = out.h*out.h;
            }
            float iou  = box_iou(out, truth);

            //printf("%d,", best_index);
            int p_index = index + locations*l.classes + i*l.n + best_index;
            *(l.cost) -= l.noobject_scale * pow(l.output[p_index], 2);  // 还记得我们曾经统一加过吗？这里需要减去了
            *(l.cost) += l.object_scale * pow(1-l.output[p_index], 2);
            avg_obj += l.output[p_index];
            l.delta[p_index] = l.object_scale * (1.-l.output[p_index]);

            if(l.rescore){
                l.delta[p_index] = l.object_scale * (iou - l.output[p_index]);
            }

            l.delta[box_index+0] = l.coord_scale*(state.truth[tbox_index + 0] - l.output[box_index + 0]);
            l.delta[box_index+1] = l.coord_scale*(state.truth[tbox_index + 1] - l.output[box_index + 1]);
            l.delta[box_index+2] = l.coord_scale*(state.truth[tbox_index + 2] - l.output[box_index + 2]);
            l.delta[box_index+3] = l.coord_scale*(state.truth[tbox_index + 3] - l.output[box_index + 3]);
            if(l.sqrt){
                l.delta[box_index+2] = l.coord_scale*(sqrt(state.truth[tbox_index + 2]) - l.output[box_index + 2]);
                l.delta[box_index+3] = l.coord_scale*(sqrt(state.truth[tbox_index + 3]) - l.output[box_index + 3]);
            }

            *(l.cost) += pow(1-iou, 2);
            avg_iou += iou;
            ++count;
        }
    }

```

## YOLO V2