# 远监督数据集构造流程


![](http://www.bbvdd.com/d/20190314161746ddc.png)

## 运行顺序
0. 下载原始数据, 解压后放在该目录下, 即  `kg_data/baike_triples.txt`
1. 在kg_data目录下运行jupyter notebook
```
jupyter notebook .
```
2. 按顺序执行 data_process.ipynb
3. 匹配实体
``` 
python EntityMatcher.py
```
4. 分词
```
python SentenceSegment.py
```
5. 按顺序执行 add_relation.ipynb


## 原始数据

* 数据来源

原始数据采用了中文通用百科知识图谱（CN-DBpedia）公开的部分数据, 包含900万+的百科实体以及6600万+的三元组关系。其中摘要信息400万+, 标签信息1980万+, infobox信息4100万+。

**下载地址：** http://www.openkg.cn/dataset/cndbpedia

**源链接：** http://kw.fudan.edu.cn/cndbpedia

* 数据格式

下载压缩包后解压为baike_triples.txt文件, 文件的每一行为一个三元组。第一个元素为实体名称, 第二个元素为关系或属性指代词, 第三个元素为其对应的值。

```
"1+8"时代广场   中文名  "1+8"时代广场
"1+8"时代广场   地点    咸宁大道与银泉大道交叉口
"1+8"时代广场   实质    城市综合体项目
"1+8"时代广场   总建面  约11.28万方
北京   中文名称   北京
北京   BaiduTAG  北京, 简称“京”, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。北京地处中国华北地区, 中心位于东经116°20′、北纬39°56′, 东与天津毗连, 其余均与河北相邻, 北京市总面积16410.54平方千米。
北京   所属地区   中国华北地区
"1.4"河南兰考火灾事故   地点    河南<a>兰考县</a>城关镇
"1.4"河南兰考火灾事故   时间    2013年1月4日
"1.4"河南兰考火灾事故   结果    一人重伤
```

## 构建实体字典

原始数据的每行第一个元素为实体, **根据正则表达式筛选全部为中文字符的实体**, 转换为字典格式。

处理后数据格式如下：

```python
{
    "北京":{
        "中文名称": "北京",
        "BaiduCard": "北京, 简称“京”, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。北京地处中国华北地区, 中心位于东经116°20′、北纬39°56′, 东与天津毗连, 其余均与河北相邻, 北京市总面积16410.54平方千米。",
        "所属地区": "中国华北地区"
    },
    ...
}
```

## 获取句子集合、句子预处理

实体的`BaiduCard`属性为实体的百度百科简介, 通常为多个句子。根据实体字典获取句子集合, 存为列表格式。

对所有句子进行预处理, **去除所有中文字符、中文常用标点之外的所有字符,  并对多个句子进行拆分**, 存为列表格式。

处理后数据格式如下：
```python
[
    "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
    "北京地处中国华北地区, 中心位于东经、北纬, 东与天津毗连, 其余均与河北相邻, 北京市总面积平方千米。",
    ...
]
```

## 句子匹配实体

对每一个句子, 遍历实体集合, 根据**字符串匹配**保存所有出现在句子中的实体。**过滤掉没有实体或仅有一个实体出现的句子**, 数据处理为`[[sentence, [entity1,...]], ...]`的格式。

处理后数据格式如下：
```python
[
    [
        "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
        [
            "北京",
            "中华人民共和国",
            "政治"
        ]
    ], 
    ...
]
```

## 句子分词

使用`Python`的`jieba`库进行中文分词, 对中文句子进行分词。将数据处理为`[[sentence, [entity1,...], [sentence_seg]], ...]`的格式。

收集所有分词后的句子, 作为语料库使用`Python`的`word2vec`库训练词向量。

**jieba 使用教程：** https://github.com/fxsjy/jieba 

**word2vec 使用教程：** https://radimrehurek.com/gensim/models/word2vec.html

### 定义用户字典
为防止实体被错误分词, 将所有实体(实体字典的键集合)写入到文件`dict.txt`作为用户字典。
### 定义停用词
定义文件`stop_word.txt`, 在分词过程中对句子去除中文停用词。(网上资源较多)

### 训练词向量

处理后数据格式如下：
```python
[
    [
        "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
        [
            "北京",
            "中华人民共和国",
            "政治"
        ],
        [
            "北京",
            "简称",
            "京",
            ...
        ]
    ], 
    ...
]
```

## 句子、实体对筛选

对分词后的句子重新对实体进行筛选, 对每一个句子的实体列表中的实体, 若其没有在分词后的句子中出现, 则去除该实体。对筛选后的实体集合两两组合, 数据处理为`[[sentence, entity_head, entity_tail, [sentence_seg]]]`的格式。（一个句子可能被用于多个样本。）

此处先对句子匹配实体, 去除不符合条件的句子后然后分词, 再用分词后的句子匹配实体的主要原因是：
1. 某些实体名称可能是另一实体的子集, 如“北京”和“北京大学”。在句子“北京大学是中国的著名大学。”中, 出现的实体应仅为“北京大学”。
2. 分词时间较长, 不对句子进行初步筛选, 直接对所有句子先分词再匹配实体, 这样效率较低。

处理后数据格式如下：
```python
[
    [
        "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
        "北京",
        "中华人民共和国",
        [
            "北京",
            "简称",
            ...
        ]
    ], 
    [
        "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
        "北京",
        "政治",
        [
            "北京",
            "简称",
            ...
        ]
    ], 
    ...
]
```

## 添加关系标签

根据对原始数据集的分析, 人工预定义了23种出现频率较高关系, 见附录1, 其中'NA'表示两实体没有关系或存在其他关系。同时, 原始数据中的关系/属性并没有对齐（如妻子、夫人对应同一种关系）, 人工编写规则对关系对齐、聚合。

遍历上一步的每一条数据, 根据实体字典和人工定义的关系对齐表进行关系标注。数据处理为`[[sentence, entity_head, entity_tail, relation, [sentence_seg]]]`的格式。

处理后数据格式如下：
```python
[
    [
        "北京, 简称京, 是中华人民共和国省级行政区、首都、直辖市, 是全国的政治、文化中心。",
        "北京",
        "中华人民共和国",
        "/地点/地点/首都"
        [
            "北京",
            "简称",
            ...
        ]
    ], 
    ...
]
```
## 数据集格式转化

对数据进行格式转化, 并添加'id'、'type'或其它等等属性。数据处理为:
```python
[
    {
        "head": {
            "word": "北京",
            "id": "666",
            ...
        },
        "tail": {
            "word": "中华人名共和国",
            "id": "6",
            ...
        },
        "relation": "/地点/地点/首都",
        "sentence": "北京 简称 京 是 中华人民共和国 省级 行政区 首都 直辖市 是 全国 的 政治 文化 中心",
        ...
    }
]
```

## 划分训练集、测试集

每种标签按照3:1的比例划分训练集、测试集。

# 一些问题
1. 句子较多, 匹配实体、分词、训练词向量时间较长（400W句子匹配8W实体, 使用8个线程约需1~2小时？）, 建议先使用较少数据预测下运行时间, 使用多线程或者数据子集进行操作。
2. 部分数据清洗工作较为简单粗暴, 存在改进空间。
3. 关系种类较少, 关系对齐规则较为简单, 且原始数据中存在部分噪声（如BaiduTAG被错误分类）, 数据集中存在噪声。


# 附录


**关系对齐/聚合：** 对于三元组(head, relation, tail), 其属于关系 `/人物/地点/出生地` 的条件是head属于人物类别,  tail属于地点类别,  relation为 `/人物/地点/出生地` 对应关系指代词集合中的某一个。

 

实体类别|BaiduTAG中至少含有以下类别中的一个
:-:|:-:
人物|人物、歌手、演员、作家
机构|机构、企业、公司、学校、部门、大学
地点|地点、地理、城市、国家、地区
其它| 不限制



<table>
    <tr>
        <th width=20px>序号</th>
        <th>关系类别</th>
        <th>关系指代词集合</th>
    </tr>
    <tr>
        <td align="center" width=10px>0</td>
        <td width=200px align="center">NA</td>
        <td>原始数据不存在关系</td>
    </tr>
    <tr>
        <td width=50px align="center">1</td>
        <td width=200px align="center">/人物/人物/家庭成员</td>
        <td>父亲、母亲、丈夫、妻子、儿子、女儿、哥哥、妹妹、姐姐、弟弟、孙子、孙女、爷爷、奶奶、外婆、外公、家人、家庭成员 ,夫人、对象、夫君</td>
    </tr>
    <tr>
        <td width=50px align="center">2</td>
        <td width=200px align="center">/人物/人物/社交关系</td>
        <td> 朋友、好友、同学、合作、搭档、经纪人、师从</td>
    </tr>
    <tr>
        <td width=50px align="center">3</td>
        <td width=200px align="center">/人物/地点/出生地</td>
        <td>出生地、出生于、来自、歌手出生地、作者出生地、出生在、作者出生地、出生</td>
    </tr>
    <tr>
        <td width=50px align="center">4</td>
        <td width=200px align="center">/人物/地点/居住地</td>
        <td>居住地、主要居住地、居住、现居住、目前居住地、现居住于、居住地点、居住于</td>
    </tr>
    <tr>
        <td width=50px align="center">5</td>
        <td width=200px align="center">/人物/地点/国籍</td>
        <td>国籍、国家</td>
    </tr>
    <tr>
        <td width=50px align="center">6</td>
        <td width=200px align="center">/人物/组织/毕业于</td>
        <td>毕业院校、毕业于、毕业学院、本科毕业院校、最后毕业院校、毕业高中、毕业地点、本科毕业学校、知名校友</td>
    </tr>
    <tr>
        <td width=50px align="center">7</td>
        <td width=200px align="center">/人物/组织/属于</td>
        <td>隶属单位、经纪公司、隶属关系、行政隶属、隶属学校、隶属大学、隶属地区、所属公司、签约公司、任职公司、工作单位、所属</td>
    </tr>
    <tr>
        <td width=50px align="center">8</td>
        <td width=200px align="center">/人物/其它/职业</td>
        <td>职业</td>
    </tr>
    <tr>
        <td width=50px align="center">9</td>
        <td width=200px align="center">/人物/其它/民族</td>
        <td>民族</td>
    </tr>
    <tr>
        <td width=50px align="center">10</td>
        <td width=200px align="center">/组织/人物/拥有者</td>
        <td>拥有、拥有者</td>
    </tr>
    <tr>
        <td width=50px align="center">11</td>
        <td width=200px align="center">/组织/人物/创始人</td>
        <td>创始人、创始、主要创始人、集团创始人</td>
    </tr>
    <tr>
        <td width=50px align="center">12</td>
        <td width=200px align="center">/组织/人物/校长</td>
        <td>校长、现任校长、学校校长、总校长</td>
    </tr>
    <tr>
        <td width=50px align="center">13</td>
        <td width=200px align="center">/组织/人物/领导人</td>
        <td>领导、现任领导、领导单位、主要领导、领导人、主要领导人</td>
    </tr>
    <tr>
        <td width=50px align="center">14</td>
        <td width=200px align="center">/组织/组织/周边</td>
        <td>周围景观、周边景点</td>
    </tr>
    <tr>
        <td width=50px align="center">15</td>
        <td width=200px align="center">/组织/地点/位于</td>
        <td>所属地区、国家、地区、地理位置、位于、区域、地点、总部地点、所在地、所在区域、位于城市、总部位于、酒店位于、学校位于、最早位于、地址、所在城市、城市、主要城市、坐落于</td>
    </tr>
    <tr>
        <td width=50px align="center">16</td>
        <td width=200px align="center">/地点/人物/相关人物</td>
        <td>相关人物、知名人物、历史人物</td>
    </tr>
    <tr>
        <td width=50px align="center">17</td>
        <td width=200px align="center">/地点/地点/位于</td>
        <td>所属地区、所属国、所属洲、所属州、所属国家、最大城市、地区、地理位置、位于、区域、地点、总部地点、所在地、所在区域、位于城市、总部位于、酒店位于、学校位于、最早位于、地址、所在城市、城市、主要城市、坐落于</td>
    </tr>
    <tr>
        <td width=50px align="center">18</td>
        <td width=200px align="center">/地点/地点/毗邻</td>
        <td>毗邻、东邻、邻近行政区、相邻、紧邻、邻近、北邻、南邻、邻国</td>
    </tr>
    <tr>
        <td width=50px align="center">19</td>
        <td width=200px align="center">/地点/地点/包含</td>
        <td>包含、包含国家、包含人物、下辖地区、下属、</td>
    </tr>
    <tr>
        <td width=50px align="center">20</td>
        <td width=200px align="center">/地点/地点/首都</td>
        <td>首都</td>
    </tr>
    <tr>
        <td width=50px align="center">21</td>
        <td width=200px align="center">/地点/组织/景点</td>
        <td>著名景点、主要景点、旅游景点、特色景点</td>
    </tr>
    <tr>
        <td width=50px align="center">22</td>
        <td width=200px align="center">/地点/其它/气候</td>
        <td> 气候类型、气候条件、气候、气候带</td>
    </tr>
</table>