---
layout: post
title:  The First Step of Open GeoAgent - A Technical Dive into Finding Useful Geospatial Data Sources
date: 2025-05-27 21:01:00
description: how to use RAG on open earth data
tags: LLMs, Remote Sensing
categories: sample-posts
thumbnail: assets/img/26d11da3-af08-4e94-8d88-e043334c6cdb.png
---

## abstract
Efficiently discovering and preparing optimal geospatial datasets from heterogeneous, large-scale open repositories like Google Earth Engine (GEE), Microsoft Planetary Computer, NASA Earth Data, and AWS Open Data is a critical bottleneck for advanced geospatial analysis and AI applications. Open GeoAgent's initial phase tackles this by implementing a robust system for discovering and preparing relevant geospatial data sources.

This post details a two-stage architecture:

1. A **data processing pipeline** converting STAC (SpatioTemporal Asset Catalog) metadata to GeoParquet, indexed by DuckDB for high-performance spatio-temporal querying.
2. An **intelligent retrieval layer** using a multi-document Retrieval Augmented Generation (RAG) system built with LlamaIndex to identify task-specific collections and extract necessary parameters.
3. scripts and geoparquet data link: https://huggingface.co/datasets/Yusin/GeoAgent-Geoparquet

## Part 1: High-Performance Data Preparation with STAC, GeoParquet, and DuckDB
The foundation of our data discovery pipeline is the standardized STAC metadata. To optimize for analytical queries, STAC Items from target collections across major open data providers are transformed into GeoParquet files.

**1. STAC to GeoParquet Transformation:**

- **Why GeoParquet?** GeoParquet is an open, cloud-optimized, columnar format for geospatial vector data. Its columnar nature is key, allowing efficient storage and partial reads. This structure enables query engines like DuckDB to leverage predicate pushdown (filtering data at the source before reading it into memory) and columnar vectorization for significantly faster I/O and query processing. This is particularly advantageous compared to row-oriented formats or iterating through individual STAC items via an API for large-scale filtering.
- **Organization**: Typically, each set of GeoParquet files corresponds to STAC items from a specific data collection, maintaining a clear and organized data lake structure. The schema of the GeoParquet files is derived directly from the STAC Item structure, including common metadata fields, asset links, and the geometry.

**2. DuckDB for Accelerated Indexing and Querying:**

DuckDB, an in-memory OLAP (Online Analytical Processing) DBMS, is employed for its exceptional speed, ease of integration (especially its Python bindings and direct Parquet reading capabilities), and rich SQL dialect.

Key DuckDB features utilized:

- Direct Parquet Querying: DuckDB can directly query one or more Parquet files, including those stored in cloud object storage.

  Python

  ```
  import duckdb
  conn = duckdb.connect()
  conn.sql("SELECT count(id) FROM 'path/to/your/collection_geoparquet/*.parquet';")
  ```

- Spatial Extension: This extension is crucial for geospatial filtering. After installation (INSTALL spatial; LOAD spatial), powerful spatial SQL operations can be performed.

  Python

  ```
  # Example Python usage with DuckDB
  import duckdb
  
  # It's good practice to install and load extensions once per session if needed.
  # For persistent storage, these might be set in the database configuration.
  conn = duckdb.connect()
  conn.execute("INSTALL spatial;")
  conn.execute("LOAD spatial;")
  
  # Define WKT for an Area of Interest (AOI) and time range
  aoi_wkt = "POLYGON((-5.0 47.0, -5.0 48.0, -4.0 48.0, -4.0 47.0, -5.0 47.0))" # Example polygon
  start_time = "2023-03-01T00:00:00Z"
  end_time = "2023-08-31T23:59:59Z"
  
  query = f"""
  SELECT id, properties_datetime, assets_B04_href, properties_eo_cloud_cover
  FROM read_parquet('path_to_your_geoparquet_files/*.parquet', union_by_name=True)
  WHERE "sar:product_type" = 'GRD' -- Example for Sentinel-1
    AND "sar:instrument_mode" = 'IW' -- Example for Sentinel-1
    AND ST_Intersects(geometry, ST_GeomFromText('{aoi_wkt}'))
    AND properties_datetime >= '{start_time}'
    AND properties_datetime <= '{end_time}'
    AND properties_eo_cloud_cover < 20; -- Example cloud cover filter
  """
  result = conn.execute(query).fetchdf()
  print(result.head())
  ```

- CQL2 to SQL Translation: To maintain compatibility with existing STAC API workflows and allow users to leverage familiar query languages, the pygeofilter  library, along with its  library, along with its  pygeofilter-duckdb backend, can parse CQL2-JSON filters and translate them into DuckDB SQL WHERE  clauses. This allows users to define filters once and apply them to both STAC APIs and the GeoParquet/DuckDB backend.

  Python

  ```
  # Conceptual Python usage with pygeofilter
  from pygeofilter.parsers.cql2_json import parse as json_parse
  from pygeofilter.backends.duckdb import to_sql_where
  from pygeofilter.util import IdempotentDict
  
  cql2_filter = {
    "op": "and",
    "args": [
      {"op": "between", "args": [{"property": "eo:cloud_cover"}, 0, 21]},
      {"op": "between", "args": [{"property": "datetime"}, "2023-02-01T00:00:00Z", "2023-02-28T23:59:59Z"]},
      {"op": "s_intersects", "args": [{"property": "geometry"}, {"type": "Polygon", "coordinates": [[[...]]]}]}
    ]
  }
  # field_mapping can be used if property names differ from GeoParquet column names
  sql_where_clause = to_sql_where(json_parse(cql2_filter), IdempotentDict())
  # full_query = f"SELECT * FROM read_parquet('{geoparquet_path}') WHERE {sql_where_clause}"
  ```

This spatio-temporal pre-filtering significantly prunes the search space, making the subsequent semantic retrieval step more focused and computationally feasible.

## Part 2: Intelligent Collection Selection via Multi-Document RAG with LlamaIndex

Even after spatio-temporal filtering, multiple data collections might meet the basic criteria. Selecting the *most suitable* collection for a nuanced task (e.g., "which LiDAR dataset is best for sub-meter vertical accuracy canopy height models in a temperate forest region?") requires deeper semantic understanding of each collection's specifications, processing levels, and suitability for specific applications. A Multi-Document Retrieval Augmented Generation (RAG) system, built using LlamaIndex, provides this capability.

**1. Architecture Overview:**

The system employs a two-tiered agent structure:

- **Collection-Specific Agents (`FunctionAgent`)**: Each candidate geospatial data collection, represented by its detailed metadata, user guides, scientific papers describing it, or even API documentation, is managed by a dedicated LlamaIndex `FunctionAgent`.
- **Top-Level Orchestrator Agent (`FunctionAgent` or `ReActAgent`)**: This agent receives the user's primary query and intelligently routes sub-queries to the appropriate collection-specific agents.

**2. Building Collection-Specific Agents:**

The process, inspired by LlamaIndex's multi-document agent patterns, involves creating specialized agents for each data collection:

- **Document Processing**: For each collection, relevant documents (e.g., landing pages, technical specifications, usage tutorials) are loaded and parsed into nodes using `SentenceSplitter`.

- Index Creation:

  - `VectorStoreIndex`: Built from these nodes using an embedding model (e.g., `OpenAIEmbedding(model="text-embedding-3-small")`). This index allows for semantic search within the collection's documentation (e.g., "find information about radiometric correction").
  - `SummaryIndex`: Optionally, a `SummaryIndex` can be built to generate concise summaries of each document/collection. This is useful for providing quick overviews to the top-level agent or the user.

- Query Engines and Tools:

  - The indices are exposed as query engines (e.g., `vector_index.as_query_engine()`, `summary_index.as_query_engine()`).
  - These query engines are then wrapped into `QueryEngineTool` instances. Each tool is given a name and a description that outlines its specific capabilities (e.g., "Useful for answering specific factual questions about Collection X's processing levels").

- **Agent Definition**: A `FunctionAgent` (often powered by an LLM like OpenAI's `gpt-4o`) is instantiated for each collection. It's equipped with the tools created above and a system prompt that directs it to use these tools to answer questions *exclusively* about its assigned data collection, avoiding reliance on prior knowledge. An asynchronous function `build_agent_per_doc` typically encapsulates this logic.

**3. Top-Level Orchestrator Agent:**

- **Tool Aggregation**: The specialized collection agents are exposed as tools to the top-level agent. This is done by wrapping the `agent.run` (or `agent.arun` for async) method of each collection agent into a `FunctionTool` using `FunctionTool.from_defaults`. The `description` of each `FunctionTool` is critical; it's often derived from a summary of the collection the agent represents, enabling the top-level agent to make informed decisions about which tool (i.e., which collection agent) to engage for a given query.

- Tool Retrieval and Reranking :

  1. An `ObjectIndex.from_objects(all_tools, index_cls=VectorStoreIndex)` is created to index all the collection-agent tools.
  2. When the top-level agent receives a query, an initial set of relevant tools (collection agents) is retrieved using `obj_index.as_node_retriever(similarity_top_k=N)`.
  3. This retrieved set can be further refined using a postprocessor like `CohereRerank(top_n=M, model="rerank-english-v3.0")` (or similar) to improve the precision of tool selection.
  4. A `CustomObjectRetriever` (as demonstrated in LlamaIndex examples) can be implemented. This custom retriever can not only fetch the top N tools post-reranking but also dynamically inject a "comparison" or "query planning" sub-agent/tool. This sub-agent would take the original query and the selected tools as input, enabling it to explicitly compare information from multiple collections if the user's query requires it (e.g., "Compare Sentinel-2 and Landsat 9 for vegetation monitoring").

- **Query Execution**: The top-level agent (e.g., a `FunctionAgent` or `ReActAgent` with an LLM like `gpt-4o`) then utilizes these selected and reranked tools (i.e., collection agents) to synthesize an answer to the user's query. It effectively delegates sub-questions or information gathering tasks to the most relevant collection specialists.

**4. Output:**

The system aims to identify the most suitable collection(s) for the user's task. Furthermore, it can be prompted to extract key parameters (e.g., specific band names, asset keys for direct access, relevant processing levels, or even Python code snippets for data loading) based on the context retrieved by the RAG pipeline.

Okay, I've revised the blog post to be more technical and included instructions for the flowchart.

------

## The First Step of Open GeoAgent: A Technical Dive into Finding Useful Geospatial Data Sources

Efficiently discovering and preparing optimal geospatial datasets from heterogeneous, large-scale open repositories like Google Earth Engine (GEE), Microsoft Planetary Computer, NASA Earth Data, and AWS Open Data is a critical bottleneck for advanced geospatial analysis and AI applications. Open GeoAgent's initial phase tackles this by implementing a robust system for discovering and preparing relevant geospatial data sources.

This post details a two-stage architecture:

1. A **data processing pipeline** converting STAC (SpatioTemporal Asset Catalog) metadata to GeoParquet, indexed by DuckDB for high-performance spatio-temporal querying.
2. An **intelligent retrieval layer** using a multi-document Retrieval Augmented Generation (RAG) system built with LlamaIndex to identify task-specific collections and extract necessary parameters.

### Part 1: High-Performance Data Preparation with STAC, GeoParquet, and DuckDB

The foundation of our data discovery pipeline is the standardized STAC metadata. To optimize for analytical queries, STAC Items from target collections across major open data providers are transformed into GeoParquet files.

**1. STAC to GeoParquet Transformation:**

- **Why GeoParquet?** GeoParquet is an open, cloud-optimized, columnar format for geospatial vector data. Its columnar nature is key, allowing efficient storage and partial reads. This structure enables query engines like DuckDB to leverage predicate pushdown (filtering data at the source before reading it into memory) and columnar vectorization for significantly faster I/O and query processing. This is particularly advantageous compared to row-oriented formats or iterating through individual STAC items via an API for large-scale filtering.
- **Organization**: Typically, each set of GeoParquet files corresponds to STAC items from a specific data collection, maintaining a clear and organized data lake structure. The schema of the GeoParquet files is derived directly from the STAC Item structure, including common metadata fields, asset links, and the geometry.

**2. DuckDB for Accelerated Indexing and Querying:**

DuckDB, an in-memory OLAP (Online Analytical Processing) DBMS, is employed for its exceptional speed, ease of integration (especially its Python bindings and direct Parquet reading capabilities), and rich SQL dialect.

Key DuckDB features utilized:

- Direct Parquet Querying: DuckDB can directly query one or more Parquet files, including those stored in cloud object storage.

  Python

  ```
  import duckdb
  conn = duckdb.connect()
  conn.sql("SELECT count(id) FROM 'path/to/your/collection_geoparquet/*.parquet';")
  ```

- Spatial Extension: This extension is crucial for geospatial filtering. After installation (INSTALL spatial; LOAD spatial;), powerful spatial SQL operations can be performed.

  Python

  ```
  # Example Python usage with DuckDB
  import duckdb
  
  # It's good practice to install and load extensions once per session if needed.
  # For persistent storage, these might be set in the database configuration.
  conn = duckdb.connect()
  conn.execute("INSTALL spatial;")
  conn.execute("LOAD spatial;")
  
  # Define WKT for an Area of Interest (AOI) and time range
  aoi_wkt = "POLYGON((-5.0 47.0, -5.0 48.0, -4.0 48.0, -4.0 47.0, -5.0 47.0))" # Example polygon
  start_time = "2023-03-01T00:00:00Z"
  end_time = "2023-08-31T23:59:59Z"
  
  query = f"""
  SELECT id, properties_datetime, assets_B04_href, properties_eo_cloud_cover
  FROM read_parquet('path_to_your_geoparquet_files/*.parquet', union_by_name=True)
  WHERE "sar:product_type" = 'GRD' -- Example for Sentinel-1
    AND "sar:instrument_mode" = 'IW' -- Example for Sentinel-1
    AND ST_Intersects(geometry, ST_GeomFromText('{aoi_wkt}'))
    AND properties_datetime >= '{start_time}'
    AND properties_datetime <= '{end_time}'
    AND properties_eo_cloud_cover < 20; -- Example cloud cover filter
  """
  result = conn.execute(query).fetchdf()
  print(result.head())
  ```

- CQL2 to SQL Translation: To maintain compatibility with existing STAC API workflows and allow users to leverage familiar query languages, the 

  ```
  pygeofilter
  ```

   library, along with its 

  ```
  pygeofilter-duckdb
  ```

   backend, can parse CQL2-JSON filters and translate them into DuckDB SQL 

  ```
  WHERE
  ```

   clauses. This allows users to define filters once and apply them to both STAC APIs and the GeoParquet/DuckDB backend.

  Python

  ```
  # Conceptual Python usage with pygeofilter
  from pygeofilter.parsers.cql2_json import parse as json_parse
  from pygeofilter.backends.duckdb import to_sql_where
  from pygeofilter.util import IdempotentDict
  
  cql2_filter = {
    "op": "and",
    "args": [
      {"op": "between", "args": [{"property": "eo:cloud_cover"}, 0, 21]},
      {"op": "between", "args": [{"property": "datetime"}, "2023-02-01T00:00:00Z", "2023-02-28T23:59:59Z"]},
      {"op": "s_intersects", "args": [{"property": "geometry"}, {"type": "Polygon", "coordinates": [[[...]]]}]}
    ]
  }
  # field_mapping can be used if property names differ from GeoParquet column names
  sql_where_clause = to_sql_where(json_parse(cql2_filter), IdempotentDict())
  # full_query = f"SELECT * FROM read_parquet('{geoparquet_path}') WHERE {sql_where_clause}"
  ```

- STAC Item Reconstruction

  : Query results from DuckDB (often retrieved as Arrow tables or Pandas DataFrames) can be efficiently converted back to 

  ```
  pystac.Item
  ```

   objects using the 

  ```
  stac_geoparquet.arrow._api.stac_table_to_items
  ```

   function. This ensures seamless integration with downstream STAC-aware tools and libraries.

  Python

  ```
  # Conceptual Python snippet (based on your provided material)
  # import json
  # import pystac
  # from stac_geoparquet.arrow._api import stac_table_to_items
  # db_result_arrow_table = conn.execute(full_query).fetch_arrow_table()
  # stac_items = []
  # for item_dict in stac_table_to_items(db_result_arrow_table):
  #     item_dict['assets'] = json.loads(item_dict['assets']) # Assets might be stored as JSON strings
  #     stac_items.append(pystac.Item.from_dict(item_dict))
  ```

This spatio-temporal pre-filtering significantly prunes the search space, making the subsequent semantic retrieval step more focused and computationally feasible.

### Part 2: Intelligent Collection Selection via Multi-Document RAG with LlamaIndex

Even after spatio-temporal filtering, multiple data collections might meet the basic criteria. Selecting the *most suitable* collection for a nuanced task (e.g., "which LiDAR dataset is best for sub-meter vertical accuracy canopy height models in a temperate forest region?") requires deeper semantic understanding of each collection's specifications, processing levels, and suitability for specific applications. A Multi-Document Retrieval Augmented Generation (RAG) system, built using LlamaIndex, provides this capability.

**1. Architecture Overview:**

The system employs a two-tiered agent structure:

- **Collection-Specific Agents (`FunctionAgent`)**: Each candidate geospatial data collection, represented by its detailed metadata, user guides, scientific papers describing it, or even API documentation, is managed by a dedicated LlamaIndex `FunctionAgent`.
- **Top-Level Orchestrator Agent (`FunctionAgent` or `ReActAgent`)**: This agent receives the user's primary query and intelligently routes sub-queries to the appropriate collection-specific agents.

**2. Building Collection-Specific Agents:**

The process, inspired by LlamaIndex's multi-document agent patterns, involves creating specialized agents for each data collection:

- **Document Processing**: For each collection, relevant documents (e.g., landing pages, technical specifications, usage tutorials) are loaded and parsed into nodes using `SentenceSplitter`.

- Index Creation

  :

  - `VectorStoreIndex`: Built from these nodes using an embedding model (e.g., `OpenAIEmbedding(model="text-embedding-3-small")`). This index allows for semantic search within the collection's documentation (e.g., "find information about radiometric correction").
  - `SummaryIndex`: Optionally, a `SummaryIndex` can be built to generate concise summaries of each document/collection. This is useful for providing quick overviews to the top-level agent or the user.

- Query Engines and Tools

  :

  - The indices are exposed as query engines (e.g., `vector_index.as_query_engine()`, `summary_index.as_query_engine()`).
  - These query engines are then wrapped into `QueryEngineTool` instances. Each tool is given a name and a description that outlines its specific capabilities (e.g., "Useful for answering specific factual questions about Collection X's processing levels").

- **Agent Definition**: A `FunctionAgent` (often powered by an LLM like OpenAI's `gpt-4o`) is instantiated for each collection. It's equipped with the tools created above and a system prompt that directs it to use these tools to answer questions *exclusively* about its assigned data collection, avoiding reliance on prior knowledge. An asynchronous function `build_agent_per_doc` typically encapsulates this logic.

**3. Top-Level Orchestrator Agent:**

- **Tool Aggregation**: The specialized collection agents are exposed as tools to the top-level agent. This is done by wrapping the `agent.run` (or `agent.arun` for async) method of each collection agent into a `FunctionTool` using `FunctionTool.from_defaults`. The `description` of each `FunctionTool` is critical; it's often derived from a summary of the collection the agent represents, enabling the top-level agent to make informed decisions about which tool (i.e., which collection agent) to engage for a given query.

- Tool Retrieval and Reranking :

  1. An `ObjectIndex.from_objects(all_tools, index_cls=VectorStoreIndex)` is created to index all the collection-agent tools.
  2. When the top-level agent receives a query, an initial set of relevant tools (collection agents) is retrieved using `obj_index.as_node_retriever(similarity_top_k=N)`.
  3. This retrieved set can be further refined using a postprocessor like `CohereRerank(top_n=M, model="rerank-english-v3.0")` (or similar) to improve the precision of tool selection.
  4. A `CustomObjectRetriever` (as demonstrated in LlamaIndex examples) can be implemented. This custom retriever can not only fetch the top N tools post-reranking but also dynamically inject a "comparison" or "query planning" sub-agent/tool. This sub-agent would take the original query and the selected tools as input, enabling it to explicitly compare information from multiple collections if the user's query requires it (e.g., "Compare Sentinel-2 and Landsat 9 for vegetation monitoring").

- **Query Execution**: The top-level agent (e.g., a `FunctionAgent` or `ReActAgent` with an LLM like `gpt-4o`) then utilizes these selected and reranked tools (i.e., collection agents) to synthesize an answer to the user's query. It effectively delegates sub-questions or information gathering tasks to the most relevant collection specialists.

**4. Output:**

The system aims to identify the most suitable collection(s) for the user's task. Furthermore, it can be prompted to extract key parameters (e.g., specific band names, asset keys for direct access, relevant processing levels, or even Python code snippets for data loading) based on the context retrieved by the RAG pipeline.

### Conclusion and Future Work

This two-stage architecture – combining high-performance pre-filtering with DuckDB/GeoParquet and sophisticated semantic retrieval with LlamaIndex RAG – forms a robust and technically sound first step for Open GeoAgent. It addresses the dual challenges of massive data volume and the need for nuanced semantic relevance in data discovery, paving the way for more automated, intelligent, and insightful geospatial data utilization. Future enhancements will focus on loading data from queried data collection to local data analysis.







