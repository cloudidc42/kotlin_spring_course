# Part 74: Search กับ Elasticsearch

## Elasticsearch — Full-text Search และ Analytics สำหรับ Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- Setup Elasticsearch กับ Spring Boot
- Spring Data Elasticsearch
- Full-text search
- Aggregations และ analytics
- Sync ข้อมูลจาก DB ไป Elasticsearch
- สร้าง Product search

---

## 📖 1. Elasticsearch Concepts

```
Elasticsearch Concepts
├── Index (เหมือน Database/Table)
├── Document (เหมือน Row)
├── Field (เหมือน Column)
├── Mapping (เหมือน Schema)
├── Shard (การแบ่ง Index)
└── Replica (สำเนาเพื่อ redundancy)
```

### ทำไมใช้ Elasticsearch?

- **Full-text search** ที่ดีกว่า SQL LIKE
- **Relevance scoring** — เรียงผลตามความเกี่ยวข้อง
- **Aggregations** — analytics แบบ real-time
- **Autocomplete** — suggest ขณะพิมพ์
- **Faceted search** — filter ด้วย multiple criteria

---

## ⚙️ 2. Setup

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200

volumes:
  es_data:
```

### Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
    implementation("co.elastic.clients:elasticsearch-java:8.11.0")
}
```

### application.yml

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    connection-timeout: 5s
    socket-timeout: 30s

elasticsearch:
  index:
    products: products
    articles: articles
```

---

## 📄 3. Document Model

```kotlin
import org.springframework.data.annotation.Id
import org.springframework.data.elasticsearch.annotations.*

@Document(indexName = "products")
@Setting(settingPath = "/es/product-settings.json")
@Mapping(mappingPath = "/es/product-mapping.json")
data class ProductDocument(
    @Id
    val id: String,
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    val name: String,
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    val description: String,
    
    @Field(type = FieldType.Keyword)
    val category: String,
    
    @Field(type = FieldType.Keyword)
    val brand: String,
    
    @Field(type = FieldType.Double)
    val price: Double,
    
    @Field(type = FieldType.Integer)
    val stock: Int,
    
    @Field(type = FieldType.Keyword)
    val tags: List<String> = emptyList(),
    
    @Field(type = FieldType.Nested)
    val attributes: List<ProductAttribute> = emptyList(),
    
    @Field(type = FieldType.Float)
    val rating: Float = 0f,
    
    @Field(type = FieldType.Integer)
    val reviewCount: Int = 0,
    
    @Field(type = FieldType.Boolean)
    val isActive: Boolean = true,
    
    @Field(type = FieldType.Date, format = [DateFormat.date_time])
    val updatedAt: java.time.Instant = java.time.Instant.now()
)

data class ProductAttribute(
    @Field(type = FieldType.Keyword)
    val name: String,
    
    @Field(type = FieldType.Keyword)
    val value: String
)
```

### Index Settings

```json
// src/main/resources/es/product-settings.json
{
  "analysis": {
    "analyzer": {
      "thai_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "thai_stop"]
      },
      "autocomplete_analyzer": {
        "type": "custom",
        "tokenizer": "autocomplete_tokenizer",
        "filter": ["lowercase"]
      }
    },
    "tokenizer": {
      "autocomplete_tokenizer": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 20,
        "token_chars": ["letter", "digit"]
      }
    },
    "filter": {
      "thai_stop": {
        "type": "stop",
        "stopwords": ["และ", "หรือ", "แต่", "ที่", "ใน", "ของ"]
      }
    }
  },
  "number_of_shards": 1,
  "number_of_replicas": 1
}
```

---

## 🔍 4. Spring Data Elasticsearch Repository

```kotlin
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository
import org.springframework.data.elasticsearch.annotations.Query

interface ProductSearchRepository : ElasticsearchRepository<ProductDocument, String> {

    // Simple field search
    fun findByCategory(category: String): List<ProductDocument>
    fun findByBrand(brand: String): List<ProductDocument>
    fun findByPriceBetween(minPrice: Double, maxPrice: Double): List<ProductDocument>

    // Full-text search
    @Query("""
        {
          "multi_match": {
            "query": "?0",
            "fields": ["name^3", "description^1", "brand^2", "tags^2"],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
    """)
    fun searchByText(query: String): List<ProductDocument>
}
```

---

## 🔬 5. Advanced Search Service

```kotlin
import co.elastic.clients.elasticsearch.ElasticsearchClient
import co.elastic.clients.elasticsearch.core.SearchRequest
import org.springframework.stereotype.Service

@Service
class ProductSearchService(
    private val elasticsearchClient: ElasticsearchClient,
    private val productSearchRepository: ProductSearchRepository
) {

    fun search(request: ProductSearchRequest): ProductSearchResponse {
        val searchRequest = SearchRequest.of { s ->
            s.index("products")
                .query { q ->
                    q.bool { b ->
                        // Full-text search
                        if (request.query.isNotBlank()) {
                            b.must { m ->
                                m.multiMatch { mm ->
                                    mm.query(request.query)
                                        .fields("name^3", "description", "brand^2", "tags^2")
                                        .type(co.elastic.clients.elasticsearch._types.query_dsl.TextQueryType.BestFields)
                                        .fuzziness("AUTO")
                                        .minimumShouldMatch("75%")
                                }
                            }
                        }

                        // Filters
                        val filters = mutableListOf<co.elastic.clients.elasticsearch._types.query_dsl.Query>()

                        request.category?.let { category ->
                            filters.add(co.elastic.clients.elasticsearch._types.query_dsl.Query.of { fq ->
                                fq.term { t -> t.field("category").value(category) }
                            })
                        }

                        request.brand?.let { brand ->
                            filters.add(co.elastic.clients.elasticsearch._types.query_dsl.Query.of { fq ->
                                fq.term { t -> t.field("brand").value(brand) }
                            })
                        }

                        if (request.minPrice != null || request.maxPrice != null) {
                            filters.add(co.elastic.clients.elasticsearch._types.query_dsl.Query.of { fq ->
                                fq.range { r ->
                                    r.field("price").apply {
                                        request.minPrice?.let { gte(it.toString()) }
                                        request.maxPrice?.let { lte(it.toString()) }
                                    }
                                }
                            })
                        }

                        if (filters.isNotEmpty()) {
                            b.filter(filters)
                        }

                        b.filter { fq ->
                            fq.term { t -> t.field("isActive").value(true) }
                        }

                        b
                    }
                }
                .aggregations("categories", { a ->
                    a.terms { t -> t.field("category").size(20) }
                })
                .aggregations("brands", { a ->
                    a.terms { t -> t.field("brand").size(20) }
                })
                .aggregations("price_ranges", { a ->
                    a.range { r ->
                        r.field("price")
                            .ranges(
                                { rng -> rng.to("1000") },
                                { rng -> rng.from("1000").to("5000") },
                                { rng -> rng.from("5000").to("20000") },
                                { rng -> rng.from("20000") }
                            )
                    }
                })
                .sort { so ->
                    when (request.sortBy) {
                        "price_asc" -> so.field { f -> f.field("price").order(co.elastic.clients.elasticsearch._types.SortOrder.Asc) }
                        "price_desc" -> so.field { f -> f.field("price").order(co.elastic.clients.elasticsearch._types.SortOrder.Desc) }
                        "rating" -> so.field { f -> f.field("rating").order(co.elastic.clients.elasticsearch._types.SortOrder.Desc) }
                        else -> so.score { it.order(co.elastic.clients.elasticsearch._types.SortOrder.Desc) }
                    }
                }
                .from(request.page * request.size)
                .size(request.size)
        }

        val response = elasticsearchClient.search(searchRequest, ProductDocument::class.java)

        return ProductSearchResponse(
            products = response.hits().hits().mapNotNull { it.source() },
            total = response.hits().total()?.value() ?: 0,
            categories = extractAggregation(response, "categories"),
            brands = extractAggregation(response, "brands"),
            priceRanges = extractRangeAggregation(response, "price_ranges")
        )
    }

    fun autocomplete(prefix: String): List<String> {
        val request = SearchRequest.of { s ->
            s.index("products")
                .suggest { suggest ->
                    suggest.suggesters("product-suggest") { sugg ->
                        sugg.prefix(prefix)
                            .completion { c ->
                                c.field("name_suggest").size(10)
                            }
                    }
                }
        }

        val response = elasticsearchClient.search(request, ProductDocument::class.java)
        return response.suggest()["product-suggest"]
            ?.flatMap { it.completion().options() }
            ?.mapNotNull { it.source()?.name }
            ?: emptyList()
    }

    private fun extractAggregation(
        response: co.elastic.clients.elasticsearch.core.SearchResponse<ProductDocument>,
        name: String
    ): List<FacetItem> {
        return response.aggregations()[name]?.sterms()?.buckets()?.array()
            ?.map { bucket -> FacetItem(bucket.key().stringValue(), bucket.docCount()) }
            ?: emptyList()
    }

    private fun extractRangeAggregation(
        response: co.elastic.clients.elasticsearch.core.SearchResponse<ProductDocument>,
        name: String
    ): List<RangeFacetItem> {
        return response.aggregations()[name]?.range()?.buckets()?.array()
            ?.map { bucket -> RangeFacetItem(bucket.key(), bucket.docCount()) }
            ?: emptyList()
    }
}
```

---

## 🔄 6. Sync DB กับ Elasticsearch

```kotlin
// Event-driven sync ด้วย Spring Events
@Service
class ProductSyncService(
    private val productSearchRepository: ProductSearchRepository
) {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun onProductCreated(event: ProductCreatedEvent) {
        val doc = event.product.toDocument()
        productSearchRepository.save(doc)
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun onProductUpdated(event: ProductUpdatedEvent) {
        val doc = event.product.toDocument()
        productSearchRepository.save(doc)
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun onProductDeleted(event: ProductDeletedEvent) {
        productSearchRepository.deleteById(event.productId.toString())
    }
}

// Full reindex job
@Service
class ElasticsearchReindexJob(
    private val productRepository: ProductRepository,
    private val productSearchRepository: ProductSearchRepository
) {

    @Scheduled(cron = "0 0 2 * * *") // ทุกคืน 2 AM
    fun reindexAll() {
        productSearchRepository.deleteAll()
        val batchSize = 500

        var page = 0
        do {
            val products = productRepository.findAll(PageRequest.of(page++, batchSize))
            if (products.hasContent()) {
                productSearchRepository.saveAll(products.content.map { it.toDocument() })
            }
        } while (products.hasNext())
    }
}
```

---

## 📊 7. สรุปตาราง Elasticsearch Query Types

| Query Type | ใช้สำหรับ | ตัวอย่าง |
|-----------|---------|---------|
| `match` | Full-text search | `{"match": {"name": "laptop"}}` |
| `multi_match` | หลาย fields | `{"fields": ["name^3", "desc"]}` |
| `term` | Exact match | `{"term": {"category": "electronics"}}` |
| `range` | ช่วงค่า | `{"range": {"price": {"gte": 100}}}` |
| `bool` | รวมหลาย query | `{"must": [], "filter": []}` |
| `fuzzy` | ค้นหาแม้พิมพ์ผิด | `{"fuzzy": {"name": "labtop"}}` |
| `nested` | Nested objects | `{"nested": {"path": "attrs"}}` |
| `completion` | Autocomplete | `{"completion": {"field": "suggest"}}` |

---

## 💡 Best Practices

1. **ใช้ Filter** แทน Query สำหรับ exact matches — เร็วกว่าเพราะ cacheable
2. **Field boosting** (`name^3`) เพื่อให้ผลลัพธ์ที่เกี่ยวข้องมากขึ้น
3. **Bulk indexing** สำหรับ reindex — เร็วกว่า index ทีละ document มาก
4. **Async sync** อย่า sync ใน transaction หลัก
5. **Index aliases** เพื่อ zero-downtime reindexing

---

## 🔄 8. Zero-downtime Reindexing ด้วย Aliases

```bash
# สร้าง index ใหม่
PUT /products_v2
{...mapping...}

# Reindex จาก v1 ไป v2
POST /_reindex
{
  "source": {"index": "products_v1"},
  "dest": {"index": "products_v2"}
}

# Switch alias
POST /_aliases
{
  "actions": [
    {"remove": {"index": "products_v1", "alias": "products"}},
    {"add": {"index": "products_v2", "alias": "products"}}
  ]
}
```

### Kotlin Reindex Service

```kotlin
@Service
class ElasticsearchMigrationService(
    private val elasticsearchClient: ElasticsearchClient
) {

    fun reindex(sourceIndex: String, targetIndex: String): Long {
        val response = elasticsearchClient.reindex { r ->
            r.source { s -> s.index(sourceIndex) }
                .dest { d -> d.index(targetIndex) }
                .conflicts(co.elastic.clients.elasticsearch._types.Conflicts.Proceed)
        }
        return response.total()
    }

    fun switchAlias(alias: String, fromIndex: String, toIndex: String) {
        elasticsearchClient.indices().updateAliases { u ->
            u.actions(
                co.elastic.clients.elasticsearch.indices.AliasAction.of { a ->
                    a.remove { r -> r.index(fromIndex).alias(alias) }
                },
                co.elastic.clients.elasticsearch.indices.AliasAction.of { a ->
                    a.add { ad -> ad.index(toIndex).alias(alias) }
                }
            )
        }
    }
}
```

---

## 🔔 9. Elasticsearch Percolator (Reverse Search)

Percolator ช่วยให้ query เป็น document และ document เป็น input — มีประโยชน์สำหรับ alert systems

```kotlin
// เก็บ query ที่ผู้ใช้สนใจ
data class PriceAlert(
    val userId: String,
    val maxPrice: Double,
    val category: String
)

@Service
class PriceAlertService(
    private val elasticsearchClient: ElasticsearchClient
) {

    fun registerAlert(alert: PriceAlert) {
        // Index the query as a percolator document
        elasticsearchClient.index { i ->
            i.index("price_alerts")
                .id("alert_${alert.userId}_${alert.category}")
                .document(mapOf(
                    "query" to mapOf(
                        "bool" to mapOf(
                            "must" to listOf(
                                mapOf("term" to mapOf("category" to alert.category)),
                                mapOf("range" to mapOf("price" to mapOf("lte" to alert.maxPrice)))
                            )
                        )
                    ),
                    "userId" to alert.userId
                ))
        }
    }

    fun findMatchingAlerts(product: ProductDocument): List<String> {
        // ค้นหา queries ที่ match กับ document
        val response = elasticsearchClient.search({ s ->
            s.index("price_alerts")
                .query { q ->
                    q.percolate { p ->
                        p.field("query")
                            .document(product)
                    }
                }
        }, Map::class.java)

        return response.hits().hits()
            .mapNotNull { it.source()?.get("userId")?.toString() }
    }
}
```

---

## 📊 10. สรุปตาราง Elasticsearch Use Cases

| Use Case | Query Type | หมายเหตุ |
|---------|-----------|---------|
| Product search | `multi_match` + `bool` | Fuzzy + boosting |
| Autocomplete | `completion` + edge ngram | Prefix search |
| Faceted search | `terms` agg + filter | Category/brand |
| Price range | `range` filter | Numeric fields |
| Similar products | `more_like_this` | Content similarity |
| Trending items | `terms` agg + date | Recent orders |
| User alerts | `percolate` | Reverse search |
| Log analysis | `date_histogram` | Time-series |

---

*Part 74/100+ | Kotlin & Spring Boot Complete Course*
