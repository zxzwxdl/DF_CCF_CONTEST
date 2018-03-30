# 2018“云移杯- 景区口碑评价分值预测 (baseline 0.53362)

## 前言
实在太忙，找实习，天池，华为等比赛都放在一块了，该方案初赛第9，进入复赛之后就先放下了。此处记录从春节到3月份关于NLP的学习感悟，供大家交流学习。

## 任务
根据每个用户的评论，预测他们对景区的情感值（1~5）。

## 思路
1. 分类问题：通过分类器学习评论与情感值的复杂映射关系。
2. 回归问题：情感值实际是有先后等级关系，因此可以采用回归大法，直接预测。

注意：分类可以采用softmax多分的手段，实测效果很差。因此，我最终还是采用了回归大法。

## 特征 (Feature Engineering 0209.xlsx)
特征很重要，自然语言处理作为非结构化数据的代表需要处理成计算机能够认识的语言，才能送入分类器来学习。首先需要对中文评论进行分词，此处采用两种开源分词：

-  结巴分词，pyhton版本，可直接在python处理。
-  hankcs分词，java版本，NLP大神的开源大作，链接如下：https://github.com/hankcs/HanLP

分词之后，有两种处理手段：

1. 把每个词当作一个标签，进行one-hot-code编码，也就是bag of words，变成一个稀疏矩阵，采用ridge or lasso等LR模型进行学习。
2. word2vec，该方法能够避免one-hot-code编码的稀疏性，且可以计算每个词之间的距离，得到近义词，反义词等。当然它还不仅如此，比如：king - man + women = queue

针对第一种bag of words得到每个单词对应的标签，比如： 喜欢 -> 23, 不喜欢 -> 24, 杭州 -> 68，注意在进行标记时，我是根据每个词出现的频率来打标签的，这里可以简单理解为"杭州"出现的频率 > "不喜欢"出现的频率 > "喜欢"出现的频率

### 统计特征
根据情感值标签，统计每个词出现的频次，从大到小排序：
1. label_1: 不认 大众化 斑斑驳驳 过团 找罪 船下 双倍 透顶 一百 误导 差价 一片狼藉 夸耀 太矮 再行 倒闭 值不值 谁家 三十块 明教 质...
2. label_2: 小得 望了望 五毛钱 欲望 关了 形同虚设 多钱 无法比拟 次数 脏乱差 差太多 人多车 还花 金领 佳音 室内环境 宰死 王小二 帮到...
3. label_3: 聊胜于无 名声大噪 兵谏亭 超强 不太值 好深 耗时间 白跑一趟 慈城 做礼拜 请问 体会出 景點 看提 啥子 金融机构 亦昌冶坊 可不...
4. label_4: 显贵 古庙 菽庄花园 没钱 门市 总归 明月山 土家族 早年 天主教堂 虎丘 桥边 紫藤 九门 麋鹿 兴坪 冰挂 古装 偷偷 留着 一探...
5. label_5: 叩拜 海鲜 热播 泰国 环岛 只选 藏兵 广州市区 亲眼看到 妃子 铺满 千奇百怪 水底 大会堂 内景 西域 忘不掉 透明 加深 慈禧...

对于每个情感值标签，出现词的集合是不一样的，简单统计它们的频次之后，可取topK的词进行离散化，比如由label_1构成的top5词集合为{不认，大众化，斑斑驳驳 ，过团，找罪},接着针对每一条评论，如果这些词出现在这条评论里，则为1，否则为0，对应能够生成5个特征。label_2至label_5同理。

### 地点特征
往往景区差评成群聚现象，好的地方大都好评，差的地方整体呈现脏乱差现象，因此：
1. 取整个词料库top100的地点进行离散化
2. 差评景区离散化

### 情绪窗口
采用snownlp做情绪预测，输入一条评论能够得到(0~1)之间的情感值，越接近1情感越积极。直接做整条评论的情感值特征提取效果不佳，采用如下trick：

- 固定一个情感窗口，如窗口大小为2，则根据评论"我 不 喜欢 这个 地方"，能够得到【我不】，【不喜欢】，【喜欢这个】，【这个地方】四条独立的组合，取情感值的最大，最小，均值，能够有效提取情感值较差的词组合。


### tf-idf
如果单纯的根据频次统计词集合，会出现大量无意义的词，如{的，是}等等这些词会占据绝大部分，因此可以借助停用词表来过滤这些无意义的词。不过实测效果不好，这需要一个很强大的停用词表，而每种NLP任务的停用词表千变万化，一套大而全的停用词往往得不到一个针对性的效果。

解决思路：采用关键词提取法，tf-idf会对一条评论的每个词进行打分，根据打分进行排序，就能得到topK的关键词集合。核心思想如下：首先一个词在该条评论中出现的次数与该分成正比，这样某个词出现的次数越高说明越能成为这条评论的关键词（tf）。此时，过滤出了真正的关键词和无意义词，为了再过滤无意义词，可以根据整个文档进行统计，词在文档中出现的频率越高，该打分应该越小（idf）。公式如下：

- 词频（tf） = 某个词在评论中出现的次数 / 评论的总词数
- 逆文档率（idf）= log(词料库评论的总数 / (出现该词的评论数 + 1))
- 词（tf-idf）= 词频（tf） * 逆文档率（idf）

再进行统计特征，效果提升了近0.001个点。详见：/utils/TFIDF.py

### doc2vec
上述特征缺少了前后词之间的上下文关系，为了提取上下文信息，可采用doc2vec来提取，输入一条评论 -> 200维的vector，把这200维当特征直接丢到模型中训练即可。此处采用java版的hankcs doc2vec，详见：https://github.com/hankcs/HanLP

部分特征提取在：/explore/Feature Explore0209.ipynb

## 模型
模型就很简单了，对于非结构化数据，直接转成了固定维数的结构化数据，可直接送入模型

- lightGBM进行5折bagging，baseline: 0.52451
- xgboost单模型全集训练，baseline: 0.52832
- textCNN， baseline: 0.51386
- random foreset 多分转二分，给stacking做融合

textCNN详见链接：https://github.com/brightmart/text_classification

小小的创新：
在做textCNN的时候，会进行一个sequence padding处理，此处并非简单的截断和随机填补成固定长度。而是在截断时，根据tf-idf的关键词列表，删除无意义词，填补时，根据tf-idf的topK词进行取整翻倍，效果比传统sequence padding好。

## 模型融合
采用stacking，在做上述几个单模型时，都会进行stacking特征的预提取，最终用xgb进行第二层的学习，随机堆了200多个lgbm模型和一些开源模型后能够提升到0.53362。

### 后记
这个项目未经整理，且用了JAVA和PYTHON同时编写，无法执行，自己也没太多精力写个傻瓜式的执行顺序。核心思路阐述完毕，代码中的trick可自行查看。欢迎交流，联系方式：daimenSong@gmail.com
