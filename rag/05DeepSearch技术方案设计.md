

# 一、Deepsearch介绍
Deepsearch的核心理念是通过搜索、阅读和推理三个环节中不断循环往复，直到找到最优答案。搜索环节通过检索网页、内部知识库、结构性数据库获取知识；而阅读环节专注于知识的分析；推理环节则负责评估当前的状态，并决定是否应该将原始问题拆成更小的问题。

# 二、流程图
[流程图 73837726f6f54ed6b00abedc79427123 mxGraph]

用更长的等待时间，换取更高质量。更具实用性的结果

# 三、案例
**用户的问题**：建国路×大望路路口和建国路×东三环路口的早高峰周期有什么差异
**使用传统rag技术回答**：
    1.用户问题或者改写后问题直接检索知识库，且只检索一次，模型根据召回切片答复
    2. 由于知识库中不直接存在比较结果，导致召回切片无效，答复效果差
**深度检索答复过程**：
1.大模型将用户问题拆解成多个子问题
2.检索第一个问题，建国路×大望路路口早高峰周期是多少？
3.检索第二个问题，建国路×东三环路口早高峰周期是多少？
4.检索第三个问题，建国路×大望路路口和建国路×东三环路口的早高峰周期有什么差异？
5.反思是否存在知识差距，继续检索或模型根据前面检索片段答复问题
**以下是模型qwen3-32b实际执行日志信息和最终答案**：

![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=49fbf49a786c4322bb4d7b889fb8831d&docGuid=ACYu127-QHL_LX)
![](https://rte.weiyun.baidu.com/wiki/attach/image/api/imageDownloadAddress?attachId=0191b4f5986e41b4bd977a4d04bfc830&docGuid=ACYu127-QHL_LX)
二、代码实现

参考代码库deep-search

URL:[https://github.com/zilliztech/deep-search](https://github.com/zilliztech/deep-search)
