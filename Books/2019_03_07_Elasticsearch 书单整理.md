title: Elasticsearch 书单整理
date: 2019-03-07
tags:
categories: 书单
permalink: Books/Elasticsearch-books-recommended

-------

注意，这是一个书单**整理**，不是书单**推荐**。

那么，怎么判断是否值得购买呢？主要可以通过三个方面：

1. 瞅瞅**豆瓣**评分和书籍评价
2. 看看**亚马逊**的书籍评价
3. 去**技术群**问问书籍是否值得买

对于书籍，尽量遵循买一本看一本，不要贪多，不要贪便宜。

因为 Elasticsearch 更新迭代的非常快，建议也多看看官方文档。

也所以，购买 Elasticsearch 的书籍时，可能对应的 Elasticsearch 的版本都不会很新。

## [《深入理解ElasticSearch》](https://union-click.jd.com)

> 资深软件开发专家、架构师撰写，系统且深入阐释ElasticSearch涉及的工具、方法、原则和实践，深入剖析ElasticSearch应用过程中遇到的各个层面的问题，涉及分布式索引机制、系统监控及性能优化、用户体验改善、Java API应用，以及自定义插件开发等，能为工程师与架构师快速提高ElasticSearch水平提供有效指导。
>
>本书共9章，第1章介绍Apache Lucene的工作方式、ElasticSearch的基本概念以及ElasticSearch的工作机制；第2章描述Lucene评分机制、如何进行查询重写，以及ElasticSearch的批处理API和如何使用过滤器来优化查询；第3章描述如何修改Lucene评分，如何使用不同的倒排索引格式来改变索引字段的结构；第4章阐述如何选择恰当的索引分片、路由工作机制、索引分片机制；第5章介绍如何为具体应用选择正确的目录实现，同时阐述发现、网关、恢复模块及其配置方式，以及调优ElasticSearch的缓存机制；第6章介绍JVM垃圾收集的工作原理、重要性以及如何调优；第7章介绍帮助修正查询中的拼写错误以及构建高效的自动完成机制——查询建议，还展示如何通过使用不同查询类型和ElasticSearch的其他功能来提高查询相关性；第8章重点阐释ElasticSearch的JAVA API；第9章通过演示如何开发你自己的河流和语言处理插件来介绍ElasticSearch的插件开发。

* 豆瓣评分：6.9 【27 人评价】

## [《Elasticsearch实战》](https://union-click.jd.com)

> 本书主要展示如何使用Elasticsearch构建可扩展的搜索应用程序。书中覆盖了Elasticsearch的主要特性，从使用不同的分析器和查询类型进行相关性调优，到使用聚集功能进行实时性分析，还有地理空间搜索和文档过滤等更多吸引人的特性。
> 
> 全书共分两个部分，第一部分解释了核心特性，内容主要涉及Elasticsearch的介绍，数据的索引、更新和删除，数据的搜索，数据的分析，使用相关性进行搜索，使用聚集来探索数据，文档间的关系等；第二部分介绍每个特性工作的更多细节及其对性能和可扩展性的影响，以便对核心功能进行产品化，内容主要涉及水平扩展和性能提升等。此外，本书还有6个附录（网上下载），提供了读者应该知道的特性，展示了关于地理空间搜索和聚集，如何管理Elasticsearch插件，学习在搜索结果中如何高亮查询单词，在生产环境中用来协助管理Elasticsearch的第三方的监控工具有哪些，如何使用Percolator过滤为多个查询匹配少量文档，如何使用不同的建议器来实现自动完成的功能。

* 出版时间是 2018-10 ，暂无豆瓣评分。

## [《从Lucene到Elasticsearch》](https://union-click.jd.com)

> 《从Lucene到Elasticsearch：全文检索实战》循序渐进介绍了信息检索、布尔检索、向量空间模型、tf-idf、BM25排序算法、Lucene架构、Lucene创建索引、Lucene查询、Lucene项目实战、Elasticsearch安装与配置、Elasticsearch插件安装、REST API数据操作、映射与模板、索引别名、Elasticsearch基本和高级搜索、Elasticsearch同步数据库、Elasticsearch集群管理、项目实战等内容。
> 
> 阅读《从Lucene到Elasticsearch：全文检索实战》，读者能够掌握信息检索的核心概念，应用Lucene库处理全文检索业务，掌握Elasticsearch分布式搜索引擎的使用方法与技巧。
> 
> 《从Lucene到Elasticsearch：全文检索实战》基于Lucene 6.0和Elasticsearch 5.4.0进行讲解，技术先进，示例丰富
> 
> 适合想学习信息检索技术的初学者和相关专业的大学生、研究生学习，也很适合大数据及云计算平台构建人员以及有一定基础的IT开发人员使用。

* 出版时间是 2017-11 ，暂无豆瓣评分。
* 豆瓣评论说，比较适合入门。

## [《Elasticsearch 源码解析与优化实战》](https://union-click.jd.com)

> 《Elasticsearch源码解析与优化实战》介绍了Elasticsearch的系统原理，旨在帮助读者了解其内部原理、设计思想，以及在生产环境中如何正确地部署、优化系统。系统原理分两方面介绍，一方面详细介绍主要流程，例如启动流程、选主流程、恢复流程；另一方面介绍各重要模块的实现，以及模块之间的关系，例如gateway模块、allocation模块等。本书的最后一部分介绍如何优化写入速度、搜索速度等大家关心的实际问题，并提供了一些诊断问题的方法和工具供读者参考。
> 
> 《Elasticsearch源码解析与优化实战》适合对Elasticsearch进行改进的研发人员、平台运维人员，对分布式搜索感兴趣的朋友，以及在使用Elasticsearch过程中遇到问题的人们。

* 出版时间是 2019-01 ，暂无豆瓣评分。

## [《ELKStack权威指南(第2版)》](https://union-click.jd.com)

> ELK是Elasticsearch、Logstash、Kibana三个开源软件的组合，是目前开源界流行的实时数据分析方案，成为实时日志处理领域开源界的第壹选择。然而，ELK也并不是实时数据分析界的灵丹妙药，使用不恰当，反而会事倍功半。本书对ELK的原理进行了解剖，不仅分享了大量实战案例和实现效果，而且分析了部分源代码，使读者不仅知其然还知其所以然。读者可通过本书的学习，快速掌握实时日志处理方法，并搭建符合自己需要的大数据分析系统。本书分为三大部分，第壹部分“Logstash”介绍Logstash的安装与配置、场景示例、性能与测试、扩展方案、源码解析、插件开发等，第二部分“Elasticsearch”介绍Elasticsearch的架构原理、数据接口用例、性能优化、测试和扩展方案、映射与模板的定制、监控方案等，第三部分“Kibana”介绍Kibana3和Kibana5的特点对比，Kibana的配置、案例与源代码解析。

* 豆瓣评分：5.8 【10 人评价】

## [《Elasticsearch服务器开发（第2版）》](https://union-click.jd.com)

> 本书介绍了Elasticsearch这个优秀的全文检索和分析引擎从安装和配置到集群管理的各方面知识。本书这一版不仅补充了上一版中遗漏的重要内容，并且所有示例和功能均基于Elasticsearch服务器1.0版进行了更新。你可以从头开始循序渐进地学习本书，也可以查阅具体功能解决手头问题。

* 豆瓣评分：6.1 【51 人评价】
* 对应的 ElasticSearch 是 1.0 ，超级老的。

## [《ElasticSearch:可扩展的开源弹性搜索解决方案》](https://union-click.jd.com)

> 本书基于ElasticSearch的0.2版本，覆盖了ElasticSearch各种功能和命令的应用，全面、详细地介绍了开源、分布式、RESTful，具有全文检索功能的搜索引擎ElasticSearch。
> 
> 本书前两章着重介绍了ElasticSearch的基本功能和用法，包括ElasticSearch的安装和配置、REST API的使用方法，以及怎样使用Query DSL语句进行查询、过滤、排序等。接下来的4章是对ElasticSearch基本功能的扩展，主要介绍了如何使用统计功能来计算查询返回结果的聚集数据、如何实现自动补全功能、如何使用ElasticSearch的空间数据处理能力，以及如何使用预期搜索功能等。第7章介绍了ElasticSearch管理API的能力，如控制分片部署位置、操纵集群等功能。在第8章将学习到如何处理使用ElasticSearch过程中可能遇到的常见问题。
> 
> 本书内容丰富、全面，基本概念的讲解细致、深入浅出。各种功能和命令的介绍，都配以实践操作和详细的代码。

* 豆瓣评分：7.1 【32 人评价】
* 对应的 ElasticSearch 是 0.2 ，超级老的。

## [《Elasticsearch技术解析与实战》](https://union-click.jd.com)

> 在编写本书的时候，Elasticsearch的最新版本是2.2.0，但本书准备正式出版的时候，Elasticsearch发布了最新的5.0版本。所以本书增加了一个附录专门介绍5.0版本的特性与改进。本书前面的部分截图是2.2.0版本的，书中所有的例子和功能都可以在Elasticsearch 2.3.3下运行，大部分的功能都可以在5.0下运行，详细的新版本差别请参考附录部分。
> 
> 本书中的例子大部分都是HTTP接口的，这些接口的测试使用了Elasticsearch Head插件。如果你想使用另一种工具，请注意修改HTTP请求的格式和编码，以便适合你所选择的工具。书中例子的结构大多是JSON格式，美化后的JSON格式比较容易阅读，但美化后的JSON格式比较长，所以我们在不影响阅读的情况下，对美化后的格式做了简单调整。书中还有一小部分是Java接口，我们在实验时用的是Eclipse工具，其他主流的Java开发工具都适用。

* 豆瓣评分：4.8 【15 人评价】

## [《Elasticsearch搜索引擎开发实战》](https://union-click.jd.com)

> 本书结合Elasticsearch在工程中的实际应用，详细介绍了使用Elasticsearch开发支持中文和英文搜索引擎的相关技术，从而实现系统监控。本书共分为8章，内容涵盖了Elasticsearch搜索引擎开发的环境安装与配置，实现一个简单的网站搜索；开发中文搜索引擎；Mapping详解；源代码分析；提高搜索相关性；使用SpringBoot开发搜索界面；使用Elasticsearch和相关软件实现系统监控；搜索引擎开发案例分析。本书非常适合信息检索技术爱好者、搜索引擎开发人员和搜索引擎优化（SEO）人员阅读，也适合作为高等院校信息检索课程的教材或教学参考书。

* 出版时间是 2018-06 ，暂无豆瓣评分。
* 豆瓣评价，很差。

## [《Elasticsearch 大数据搜索引擎》](https://union-click.jd.com)

> Elasticsearch搜索集群系统在生产和生活中发挥着越来越重要的作用。本书介绍了Elasticsearch的使用、原理、系统优化与扩展应用。本书用例子说明了Java、Python、Scala和PHP的编程API，其中在Java搜索界面实现上，介绍了使用Spring实现微服务开发。为了扩展Elasticsearch的功能，本书以中文分词和英文文本分析为例介绍了插件开发方法。本书介绍了使用Elasticsearch作为数据管理平台的日志监控与分析方法，介绍了使用OCR从图像中提取文本以及问答式搜索的开发方法。

* 出版时间是 2018-01 ，暂无豆瓣评分。
* 亚马逊评价，很差。
