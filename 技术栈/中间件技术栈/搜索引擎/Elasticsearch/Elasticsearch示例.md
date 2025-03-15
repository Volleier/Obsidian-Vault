
```python
from elasticsearch import Elasticsearch
from datetime import datetime

# 连接到本地 Elasticsearch 实例（默认端口9200）
# 如果是远程服务器，修改为对应的地址，例如：http://remote-host:9200
es = Elasticsearch(hosts=["http://localhost:9200"])

# 检查集群健康状态
print(es.cluster.health())

# 定义索引名称和映射（数据结构）
index_name = "my_example_index"

# 创建索引（如果不存在）
if not es.indices.exists(index=index_name):
    # 定义索引映射（数据结构）
    mapping = {
        "mappings": {
            "properties": {
                "title": {
                    "type": "text",  # 全文搜索字段
                    "analyzer": "english"  # 使用英文分析器
                },
                "content": {
                    "type": "text",
                    "analyzer": "english"
                },
                "author": {
                    "type": "keyword"  # 精确值字段，适合过滤和聚合
                },
                "publish_date": {
                    "type": "date"
                },
                "views": {
                    "type": "integer"
                }
            }
        }
    }
    
    # 创建索引
    es.indices.create(index=index_name, body=mapping)
    print(f"Created index {index_name}")

# 插入测试文档
doc1 = {
    "title": "Introduction to Elasticsearch",
    "content": "Elasticsearch is a distributed, RESTful search and analytics engine.",
    "author": "John Doe",
    "publish_date": datetime(2023, 1, 15),
    "views": 1500
}

doc2 = {
    "title": "Advanced Search Techniques",
    "content": "Learn about bool queries, aggregations, and highlight in ES.",
    "author": "Jane Smith",
    "publish_date": datetime(2023, 2, 20),
    "views": 850
}

# 插入文档（自动生成ID）
es.index(index=index_name, body=doc1)
es.index(index=index_name, body=doc2)
print("Inserted sample documents")

# 强制刷新索引，使新文档立即可搜索（生产环境慎用，会影响性能）
es.indices.refresh(index=index_name)

# 执行搜索查询
search_body = {
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "content": "search engine"  # 在content字段中搜索这两个词
                    }
                }
            ],
            "filter": [
                {
                    "range": {
                        "views": {
                            "gte": 1000  # 过滤浏览量大于等于1000的文档
                        }
                    }
                }
            ]
        }
    },
    "highlight": {  # 高亮匹配内容
        "fields": {
            "content": {}
        }
    }
}

# 执行搜索
result = es.search(index=index_name, body=search_body)

# 处理搜索结果
print(f"Got {result['hits']['total']['value']} hits:")
for hit in result['hits']['hits']:
    print(f"\nScore: {hit['_score']}")  # 显示相关性得分
    print(f"Title: {hit['_source']['title']}")
    print(f"Author: {hit['_source']['author']}")
    
    # 显示高亮内容（如果有匹配）
    if 'highlight' in hit:
        print("Highlight:", " ... ".join(hit['highlight']['content']))

# （可选）删除索引（测试后清理）
# es.indices.delete(index=index_name)
```

代码主要展示了 Elasticsearch 的以下核心功能：

1. **连接管理**：通过 Elasticsearch 客户端连接到集群  
2. **索引管理**：
   - 创建带有自定义映射的索引
   - 定义不同字段类型（text/keyword/date/integer）
   - 配置分析器（英文分词）

3. **文档操作**：
   - 插入 JSON 格式的文档
   - 自动生成文档ID

4. **搜索功能**：
   - 布尔查询（must/filter）
   - 全文搜索（match query）
   - 数值范围过滤
   - 结果高亮
   - 相关性评分

5. **额外功能**：
   - 索引刷新（立即生效）
   - 集群健康检查

运行结果示例：
```bash
Created index my_example_index
Inserted sample documents
Got 1 hits:

Score: 0.5753642
Title: Introduction to Elasticsearch
Author: John Doe
Highlight: Elasticsearch is a distributed, RESTful <em>search</em> and analytics <em>engine</em>.
```

这个示例展示了 Elasticsearch 的核心能力：快速存储、搜索和分析结构化/非结构化数据