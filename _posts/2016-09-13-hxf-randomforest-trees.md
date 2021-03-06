---
title: "模型评估：随机森林应该有多少颗树"
author: "hxf"
date: "13 September 2016"
layout: single
header:
  overlay_color: "#5e616c"
  overlay_filter: "rgba(0,0,0,0.3),rgba(0,0,0,0.5)"
  overlay_image: "posts/hxf/randomforest-trees/header.jpg"
  padding: 10em
excerpt: 'Posted by Han Xufeng on September 13, 2016.'
comments: true
categories: [Algorithm]
tags: [RandomForest]
mathjax: true
highlight: false
description: "随机森林应该构建多少棵树?"
---


   随机森林应该构建多少棵树？恐怕很多人都思考过这个问题，网络搜索一下很难找到特别明朗的答案，有些资料上说这和你的数据集的大小有关，这特么的算什么答案（针对特定数据集应该设置多少棵树）？

   一般而言，评价一个东西自然要构建可行的评价指标体系，比如评价大小时使用均值、中位数等，评价波动时使用方差等。之所以要探究随机森林多少棵树合适的问题，其根本原因是担心树的多少可能影响模型的准确性和稳定性（当然，树枝也可能影响，这个我们暂不考虑），那就需要构建一个评价指标来衡量。比如我们选择准确性吧（也可以使用机器学习的方差（variavce）和偏差（bias）的概念评估），那么问题来了，既然进行“比较”就需要追求比较结果具有统计学意义。

   符合统计学思想，可以简单的比较正确率的均值和方差大小来衡量分类的准确性和稳定性。其实这类指标只是看起来老套但是并不“简单”，个人认为在监控、评估监督模型时还是一些传统指标比较靠谱，例如平均绝对误差（MAE）、平均平方差（MSE）、标准平均方差（NMSE）和均值等，计算简单、容易理解；只有在非监督模型中才会选择一些所谓“高大上”的指标如信息熵、复杂度和基尼值等等。

## k层交叉检验 k值取多少为妙

   如果要计算准确率的均值和方差，就需要使用多个测试集测试模型，只好祭出k层交叉检验（K-fold cross-validation）了，先将原始数据随机分成K组(一般是均分),将其中一个子集做为测试集,其余的K-1组子集作为训练集,以此重复k次，这样会得到K个模型,用这K个模型在k个测试集上的准确率（或其他评价指标）的平均数作为模型的性能评价指标。

   比如如果要测试100棵树和150棵树的随机森林模型哪个性能更好？就需要将两个特定参数的模型通过k层交叉检验，分别构建k次模型，测试k次，然后比较它们的均值、方差等指标。那么问题来了？k值应该为多少呢？我可以不负责任的告诉你，在这方面我真的不知道。

   一般来说，k值越小，训练压力越小，模型方差越小而模型的偏差越大，k值越大，训练压力越大，模型方差越大而模型偏差越小。在《The Elements of Statistical Learning》这本书中测试了一些k值，发现k值为10时模型误差趋于稳定。
   
   进行交叉检验首先要对数据分组，数据分组要符合随机且平均的原则。遵循这两条原则可以写一个交叉检验分组的自编函数：`CVgroup`，它包含三个参数：cross用于设定k值，准备做几层交叉检验？datasize用于设定数据集大小，即有多少观测值，简单理解就是数据框有多少行？    seed用来设置随机种子，所谓随机种子，如果设定了这个值，保证我现在生成的随机数，和你生成的随机数是相同的，虽然程序中有随机过程，但是可以保证你在验证我的程序时不会因为随机过程而改变结果，换句话说，我们对随机过程使用了同一套锁钥。

   我还是不太理解set.seed这个函数，他是在随机数生成之后锁定的，还是在生成之前锁定的呢？要是在随机数生成后锁定的，如何保证其它人验证程序时不会改变结果。要是在随机数生成前锁定的，那就是说set.seed函数()里一个数字代表了一个结果，那这样真的能算是随机过程吗？当然这个问题也可以设计个方案测试一下，不要不屑于做这种事情,有些时候它的确很好用。
   鉴于我们需要锻炼根据伪代码写函数的能力，这里代码尽量拆分开来说明，分组的思路是这样的：可以先产生一个1：k的序列，然后将这个序列复制成和数据编号长度（datasize）相等或略大于数据编号长度，从前往后依次抽出和数据编号等长的序列，为序列n，然后从n中非放回性随机抽样，抽出一个和数据编号长度相等的序列temp，这时temp随机地包含1：k的任何一个值且它们的个数基本相等，更重要的是它和数据编号等长，然后按照每一个k值帅选数据编号并分为一组就可以了。

   `CVgroup`函数主体的第一行产生一个空的list，用于乘装每一个小组的观测值编码（或指行编号）；2行用于设置随机种子；3行`ceiling`函数用于向上取顶，比如1.2取顶就是2；对应的函数有`floor`，向下取低，取顶后用于设定序列1：k复制的次数，因为要保证复制（rep）后的长度略大于数据大小，所以向上取顶，然后依次从中抽出前1:datasize个，形成一个和数据等长的序列n，这句目的尽量保证均分的原则；4行非放回随机抽样，抽取和数据等长的序列temp，这句的意思很简单，就是将原来的数据列随机重新摆列，保证随机的原则；5行生成一个用于筛选的序列1：k；6行生成数据编号dataseq，7行因为temp序列和dataseq序列一一对应，所以以temp筛选dataseq其结果应该也是随机的，使用`lapply`函数，将每一个x值用于匿名函数，匿名函数是一个简单的筛选，向量筛选和数据框不同，数据框筛选需有表示行列分界的“,”,如`df[temp==x,]`，而向量则不用“,”分割，因为向量算是一个只有一列的矩阵。
   
   
### k层交叉检验数据集

```r
library(plyr)
CVgroup <- function(k, datasize, seed) {
  cvlist <- list()
  set.seed(seed)
  n <- rep(1:k, ceiling(datasize/k))[1:datasize]
  temp <- sample(n, datasize)
  x <- 1:k
  dataseq <- 1:datasize 
  cvlist <- lapply(x, function(x) dataseq[temp==x])
  return(cvlist)
}

k <- 5
datasize <- nrow(iris)
cvlist <- CVgroup(k = k, datasize = datasize, seed = 1408)
```

最后三行用于对著名的iris鸢尾花数据集分组，设定k值；`nrow`用于获得鸢尾花数据集有多少行即datasize，最后一行使用函数对数据编号分组。莺尾花数据是R里自带的，萼片长度为因变量，数据集中其他变量为自变量，这一任务的性质为回归的预测而非分类。用情感分析的数据电脑会秒炸的，虽然我已经习惯用复杂数据来练习，但是这里我们暂时不强调数据（我的手里也没有那么多好用的数据-_-)。

随机数种子取1024，因为我最近看了部电影《幻影凶间1024》对这个数字印象比较深。下面就验证一下随机森林的树数对模型准确性的影响，首先要确定树的序列，比如要测试60棵树至200棵树，可以每隔20棵设定一个特定的树数（根据数据而定），然后对每种设定树数的随机森林做一次k层检验，这样就会得到k个预测结果集，如果测试n种树数，那么就要得到k*n个预测结果集。按照上面的结果，我们需要进行一个嵌套循环，循环的外层是任何一种设定的树数，比如对60棵树循环一次，对80棵树也要循环一次等等，依次循环到n，循环的内层针对某一设定树数的随机森林进行k层交叉检验循环，比如使用分组1和其余k-1组数据对60棵树的随机森林进行一次建模和测试，然后使用分组2和其余k-1组数据对60棵树的随机森林再做一次建模和测试，依次循环到k。
    
    
### 随机森林k层交叉检验

```r
data <- iris
pred <- data.frame() #存储预测结果
library(plyr)
library(randomForest)
m <- seq(60, 500, by = 20)##如果数据量大尽量间隔大点，间隔过小没有实际意义
for (j in m) {
  progress.bar <- create_progress_bar("text")
  progress.bar$init(k)
  for (i in 1:k) {
  train <- data[-cvlist[[i]],]
  test <- data[cvlist[[i]],]
  model <- randomForest(Sepal.Length ~ ., data = train, ntree = j)
  prediction <- predict(model, subset(test, select = - Sepal.Length))
  randomtree <- rep(j, length(prediction))
  kcross <- rep(i, length(prediction))
  temp <- data.frame(cbind(subset(test, select = Sepal.Length), prediction, randomtree, kcross))
  pred <- rbind(pred, temp)
  #print(paste("随机森林：", j))
  #progress.bar$step()
  }
}
```

 1行创建数据集；2行创建一个空数据框，用于存放测试集编号、树数、真实值和预测值；5行生成一个包含欲测试树数的向量，这里测试60-500棵树；6行第一层循环，每种树数循环一次，7、8行用plyr包的`create_progress_bar`函数创建一个进度条，8行设置任务数，比如下面的循环要进行k次，则任务数为k；9行内层循环开始，即k层交叉检验；10行筛选data中行编号与交叉检验分组cvlist里第i个元素存储的行编号相同的数据，将其删除后余下的数据（其余k-1组）作为训练集，而筛选行编号相同的作为测试集（11行）；12行使用训练集构建随机森林模型，树数为j；13行使用上步构建的模型预测测试集的萼片长度，`subset`函数删除了测试集中真实的萼片长度，14行将树数j重复至与预测值等长，赋值给randomtree；15行将i重复至与预测值等长，赋值给kcross；16行将真实值、预测值、随机森林树数、测试组编号捆绑在一起组成新的数据框temp；17行将temp按行和pred合并；18行打印通知：循环至树数j的随机森林模型，19行输出进度条，告知完成了这个任务的百分之几（为避免刷屏。18,19行用#号标注，结果可自行尝试，去掉#号就可运行）。这样我们就可以根据pred记录的结果进行方差分析等等，进一步研究树数对随机森林准确性及稳定行的影响。

 R里不要轻易使用循环，上面的循环也可以使用lapply家族函数进行改造，甚至使用并行计算都可以（并行我还没有尝试）

```r
## apply家族提高效率

data <- iris
library(plyr)
library(randomForest)
j <- seq(10, 5000, by = 20)
i <- 1:k
i <- rep(i, times = length(j))
j <- rep(j, each = 5)
x <- cbind(i, j)
cvtest <- function(i, j) {
  train <- data[-cvlist[[i]],]
  test <- data[cvlist[[i]],]
  model <- randomForest(Sepal.Length ~ ., data = train, ntree = j)
  prediction <- predict(model, subset(test, select = - Sepal.Length))
  temp <- data.frame(cbind(subset(test, select = Sepal.Length), prediction))
}
system.time(pred <- mdply(x, cvtest))

##    user  system elapsed 
##  299.85   12.59  312.91

names(pred)[1:2]<-c("kcross","randomtree")
```
当循环数较少时，apply家族并不比循环有效率上的优势，但一旦比赛由百米变成了马拉松，apply家族的优势就展现出来了，这就是所谓的路遥知马力吧。
  
## 统计指标选取 


$$ MAE = \frac{ \sum{ \left|pred - obs\right|} }{n} $$

$$ MSE = \frac{ \sum{(pred - obs)^2} }{n} $$

$$ NMSE = \frac{ MSE }{ \frac{ \sum{ (\overline{obs} - obs)^2 } }{n} } $$

既然是预测回归任务，就可以使用常见的统计指标进行模型的筛选、评估、监控，这里仅回顾平均绝对误差（MAE）、均方差（MSE）、标准化平均绝对方差（NMSE）这三个评价指标。

### 回归模型的筛选指标

```r
maefun <- function(pred, obs) mean(abs(pred - obs))
msefun <- function(pred, obs) mean((pred - obs)^2)
nmsefun <- function(pred, obs) mean((pred - obs)^2)/mean((mean(obs) - obs)^2)
library(dplyr)
eval <- pred %>% group_by(randomtree, kcross) %>% 
  summarise(mae = maefun(prediction, Sepal.Length),
            mse = msefun(prediction, Sepal.Length),
            nmse = nmsefun(prediction, Sepal.Length))
```

前三行分别构建了计算mae、mse、nmse的三个函数，第四行开始使用dplyr包里的管道函数`%>%`将数据集传递给`group_by`函数，按照随机森林的树数randomtree、交差检验测试集编号kcross这两列进行数据分组，然后使用`summarise`函数将数据集相应的变量应用到三个指标计算函数，最后得到了一个包含每一种特定树数的随机森林在特定测试集上的三个评价指标集。

   检验不同树数的随机森林三个指标是否存在显著的差异，其实就是进行单因子方差分析，在进行方差分析之前首先要检验方差齐性，因为在方差分析的F检验中，是以各个实验组内总体方差齐性为前提的；方差齐性通过后进行方差分析，如果组间差异显著，再通过多重比较找出哪些组之间存在差异。

### 单因子方差分析

```r
eval$randomtree <- as.factor(eval$randomtree)
bartlett.test(mae ~ randomtree, data = eval)

## 
##  Bartlett test of homogeneity of variances
## 
## data:  mae by randomtree
## Bartlett's K-squared = 0.6086, df = 249, p-value = 1

temp <- aov(mae ~ randomtree, data = eval)##可以选择前100行
summary(temp)

##               Df Sum Sq   Mean Sq F value Pr(>F)
## randomtree   249 0.0036 0.0000143   0.006      1
## Residuals   1000 2.5074 0.0025074

#TukeyHSD(temp)
```

  1行首先要将分组变量转化为因子，2行使用bartlett方法检验指标mae的方差齐性，为什么检验方差齐性，其目的是保证各组的分布一致，如果各组的分布都不一致，比较均值还有什么意义，F越小（p越大，大于P0.05），就证明没有差异，说明方差齐；`aov`函数对mae指标进行方差分析，summary显示差异不显著，说明不同树数的随机森林的mae指标差异不显著（p远远大于0.05），即没有必要做多重检验了，但为了展示整个分析流程，我们使用`TukeyHSD`进行多重比较，（也是为了避免刷屏，用#标注了）各组间的p,adj值都远远大于0.05，组间不存在差异，以上就是分析评估代码，感兴趣的话你可以稍作修改进一步检验mse、nmse这两个指标。
  
统计检验让我们坚信各种树数的随机森林之间的差异不显著，但是很多人总是坚信眼见为实，那我们不妨将三个指标随树数的变化趋势可视化，使用折线图分析一下它们的差异。
   
### 折线图分析

```r
library(extrafont)
loadfonts(device="win")
#载入windows自带的字体
library(ggplot2)
library(reshape2)
eval <- aggregate(cbind(mae, mse, nmse) ~ randomtree, data = eval, mean)
eval <- melt(eval, id = "randomtree")
eval$randomtree <- as.numeric(as.character(eval$randomtree))
p <- ggplot(eval, aes(x = randomtree, y = value, color = variable)) + 
  geom_line(size = 1.3)+facet_wrap(~ variable, nrow = 3, scales = "free")+ylab("")+xlab("随机森林树数")+theme(legend.position="none")
p
```
![折线图](/images/posts/hxf/randomforest-trees/ggplot.png)

  怎么样，有了图表是不是显得于说服力一点。R制作图表的问题以后再说，它是有很多门道的，比如我就用了太多默认参数，默认就是随便，就是不专业，数据可视化过程的重要性并不亚于分析过程。

