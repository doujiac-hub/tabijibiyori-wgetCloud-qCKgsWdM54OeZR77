
这一章我们介绍GraphRAG范式，算着时间也是该到图谱了，NLP每一轮新模型出来后，往往都是先研究微调，然后各种预训练方案，接着琢磨数据，各种主动学习半监督，弱监督，无监督，再之后就到图谱和对抗学习\~


前一阵Graph RAG的风吹得呼呼的，经常被问你们也Graph RAG了么？但Graph RAG虽好但并非RAG的Silver Bullet，它有特定适合的问题和场景，更适合作为RAG中的一路召回，用来解决实体密集，依赖全局关系的信息召回。所以这一章我们来聊聊GraphRAG的实现和具体解决哪些问题。


## Graph RAG和Naive RAG的效果对比



> * [https://graphrag\-demo.deepset.ai/](https://github.com)


我们先基于Deepset提供的Graph RAG的demo来看几个效果对比。Demo基于美股的季报构建了图谱，然后横向对比同一个问题使用GPT4\-O基于图谱回答和使用朴素RAG的区别。我们来看3个Graph RAG会出现显著优势的问题类型


问题1：Which companies bought GPUs, write one line summary for each company


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637013-1570398364.png)


问题2：Compare Tesla and Apple Inc, answer the question in a structure and concise way


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637154-1037135281.png)


问题3： What do these reports talk about?


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637045-694911007.png)


以上3个Demo展示了Graph RAG最核心的优势和适配的场景，范围从大到小分别是


* Dataset Global Info：依赖数据全局结构化信息，回答“有哪些？”,“the best、most、top”类问题
* Subgroup Abstract Info：依赖对全局数据进行分类划分，对局部进行信息抽象，可以回答一些衍生的“某一类，某主题”，“有几类”问题
* Entity and Relationship Info: 依赖实体和实体间的关联信息，回答“A的各个方面”，“A vs B”对比差异类问题。这里也可以从实体延伸到文档，包括多文档和文档间的关联系信息


## 静态图谱：微软Graph RAG实现



> * [https://github.com/microsoft/graphrag](https://github.com)
> * GRAPH Retrieval\-Augmented Generation: A Survey
> * From Local to Global: A Graph RAG Approach to Query\-Focused Summarization


阿里的graph RAG综述对GRAG的流程进行了分类，Graph RAG其实就是在RAG的召回内容中加入图谱召回，来优化回答，主要包含以下三个部分：图谱构建，图数据召回，图增强回答。所以不同Graph RAG论文的差异也主要就在以上三个部分的不同实现和排列组合，下面我们看下微软GraphRAG的具体实现。


### step1\.图谱构建


第一步是对文档进行chunking，分块的目标是因为大模型在处理太长的上文时会导致实体抽取的召回率较低，论文对比了不同的chunk大小从600\-2400字，随着chunk变大，段落中能检测到的实体量级会逐渐降低。


第二步是使用大模型对分段的内容进行实体抽取，通用领域直接使用以下指令进行未指定实体类型的广义实体（entity，type，description），和实体三元组（source，target，relation）抽取，而对于垂直领域会需要依赖few\-shot来提升抽取效果。这里论文会使用模型进行多轮反思“针对抽取结果是否有未识别出的实体？”如果存在则进行补充抽取，来提升对于构建图谱最关键的实体三元组抽取的召回率。



```
GRAPH_EXTRACTION_PROMPT = """
-Goal-
Given a text document that is potentially relevant to this activity and a list of entity types, identify all entities of those types from the text and all relationships among the identified entities.

-Steps-
1. Identify all entities. For each identified entity, extract the following information:
- entity_name: Name of the entity, capitalized
- entity_type: One of the following types: [{entity_types}]
- entity_description: Comprehensive description of the entity's attributes and activities
Format each entity as ("entity"{{tuple_delimiter}}{{tuple_delimiter}}{{tuple_delimiter}})

2. From the entities identified in step 1, identify all pairs of (source_entity, target_entity) that are *clearly related* to each other.
For each pair of related entities, extract the following information:
- source_entity: name of the source entity, as identified in step 1
- target_entity: name of the target entity, as identified in step 1
- relationship_description: explanation as to why you think the source entity and the target entity are related to each other
- relationship_strength: an integer score between 1 to 10, indicating strength of the relationship between the source entity and target entity
Format each relationship as ("relationship"{{tuple_delimiter}}{{tuple_delimiter}}{{tuple_delimiter}}{{tuple_delimiter}})

3. Return output in {language} as a single list of all the entities and relationships identified in steps 1 and 2. Use **{{record_delimiter}}** as the list delimiter.

4. If you have to translate into {language}, just translate the descriptions, nothing else!

5. When finished, output {{completion_delimiter}}.

-Examples-
######################
{examples}

-Real Data-
######################
entity_types: [{entity_types}]
text: {{input_text}}
######################
output:"""

```

使用以上Promtp抽取出的实体结果如下


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637055-1885659343.png)


有了实体三元组，就可以直接进行图谱构建了，这里graphrag直接使用NetworkX构建无向图。


### step2\.图谱划分和描述生成


有了图，下一步就是如何描述图谱信息，在大模型之前我们更多是采用模版，来把实体和实体关系信息转化成文本，而在LLM时代有了更多可能。这里微软的特色是多了一步对图谱进行划分和描述。


图谱划分，也叫社群发现，之所以要做社群发现，其实就来自GraphRag要解决全局主体，汇总类，关联类的问题，而这种问题就是通过先对全局进行层次划分，对所有局部主题（社群）都预先进行信息汇总实现的，这部分信息既包含了局部结构信息例如主营业务为GPU的有几家公司，也包含了局部语义例如抽象主题和概念。


社群发现有很多种算法, 有基于modularity，有基于层次聚类，还有基于随机游走的各种算法，这里论文选用了Leiden，是对Louvain算法的优化，也属于modularity类型的算法，会生成互斥的多个子图。


针对每个子图，会使用以下Prompt指令让模型生成社群的相关总结，这里一个子图其实就对应一个topic，可以是一个事件，一个主体相关的所有信息，一类话题等等。以下prompt除了生成summary，还会生成finding，也就对应前面提到的抽象，主题，话题类型的信息抽象。



```
COMMUNITY_REPORT_SUMMARIZATION_PROMPT = """
{persona}

# Goal
Write a comprehensive assessment report of a community taking on the role of a {role}. The content of this report includes an overview of the community's key entities and relationships.

# Report Structure
The report should include the following sections:
- TITLE: community's name that represents its key entities - title should be short but specific. When possible, include representative named entities in the title.
- SUMMARY: An executive summary of the community's overall structure, how its entities are related to each other, and significant points associated with its entities.
- REPORT RATING: {report_rating_description}
- RATING EXPLANATION: Give a single sentence explanation of the rating.
- DETAILED FINDINGS: A list of 5-10 key insights about the community. Each insight should have a short summary followed by multiple paragraphs of explanatory text grounded according to the grounding rules below. Be comprehensive.

.....此处省略输出格式和few-shot
"""

```

得到的针对每个社群的结构化总结如下，后面会把title\+summary\+findings\['summary']\+findings\['explanation']整体拼接的文本作为该局部信息的总结。这里其实是整个流程中最消耗模型的部分，并且虽然论文并未提及，但是当图谱有新增节点信息时，这里的report也需要进行增量更新，需要识别到出现变更的子图，并对应去更新子图的report


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184636920-804174477.png)


### step3\. 图谱信息回答


最后就是如何使用以上信息，当用户的query进来，微软的论文中采用了针对以上生成的report，全部用于回答，再对回答进行筛选的逻辑。但其实也可以先加入召回逻辑，基于用户query去召回相关的子图report，虽然肯定会有一些损失，但是可以大幅降低耗时和成本。微软的实现是


* Concatenation：把所有report打散，拼接成多个chunk，每个chunk作为一段context上文
* Map：使用以上多个context和用户query得到多个中间回答，然后使用大模型对所有中间回答进行打分0\-100
* Reduce：以上打分降序，保留窗口长度限制内的所有答案拼接作为上文，使用大模型生成最终回答。


效果上论文对比了在podcast和news article数据集上，分别使用不同level的子图生成的report作为context（C0\-C3），以及直接使用图节点信息（TS）来对比naive RAG（SS）的效果。以下是使用大模型分别评估回答的全面性，多样性，有用性和直接性，在以上4个评估角度上，两两对比的胜率。使用图信息的所有方案均超越naive rag\~


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637121-1931175018.png)


## LightRAG



> * [https://github.com/HKUDS/LightRAG](https://github.com):[FlowerCloud机场](https://yunbeijia.com)
> * LightRAG: Simple and Fast Retrieval\-Augmented Generation


LightRAG是港大刚出的新RAG框架，对比微软的graph rag实现，它更多在信息召回层面做了优化。这里我们只看下lightrag和graph rag的核心差异点：对图索引的构建和图信息召回


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637291-1977892352.png)


在图谱构建的环节二者基本是一致的，差异在于LightRAG为了构建召回索引，在graphRAG抽取实体和关系的Prompt指令中加入了high\-level关键词抽取的指令，用于抽取可以描述图局部抽象信息的关键词，该关键词作为索引可以直接用于主题，概念等问题的信息召回。对比微软使用子图report来描述局部信息，lightrag在抽取时使用关键词来描述局部信息，更加轻量级，但对于范围更大的子图信息描述会有不足。



```
GRAPH_EXTRACTION_PROMPT = """
...同上

- relationship_keywords: one or more high-level key words that summarize the overarching nature of the relationship, focusing on concepts or themes rather than specific details
Format each relationship as ("relationship"{tuple_delimiter}{tuple_delimiter}{tuple_delimiter}{tuple_delimiter}{tuple_delimiter})

...同上
"""

```

而在检索阶段，lightrag加入了两路图信息召回


* low level：用于回答细节类的问题，例如谁写了傲慢与偏见，问题专注于具体实体，关系
* high level：用于回答全局类，概念类问题，需要掌握全局，抽象信息，例如人工智能如何影响当代教育


针对以上两个角度，lightrag会使用指令让大模型分别生成两类检索关键词，一类针对具体实体进行检索，一类针对主题概念进行检索，对应上面实体抽取过程中生成的high level keywords，prompt和few\-shot如下


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637339-1387120077.png)


使用以上两类关键词会分别进行两路召回，再对信息进行合并


* low level：使用基于query生成low level关键词，去检索entity，因为low level针对的是实体导向的细节信息。这里论文是对实体进行了向量化，使用实体名称\+描述过embedding模型
* high level：使用基于query生成的high level关键词，去检索relation，因为在前面抽取时针对关系抽取了局部的抽象关键词，而relation的向量化使用了这些关键词还有关系的描述，因此主题类的局部召回可以通过召回关系来实现


![image](https://img2024.cnblogs.com/blog/1326688/202410/1326688-20241027184637355-2012407951.png)


拼接上文和大模型回答之类的都大差不差就不细说了，感兴趣的同学可以去扒拉扒拉代码\~


**想看更全的大模型论文·微调预训练数据·开源框架·AIGC应用 \>\>** [**DecryPrompt**](https://github.com)


