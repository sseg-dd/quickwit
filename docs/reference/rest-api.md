---
title: REST API
sidebar_position: 1
---

## API version

All the API endpoints start with the `api/v1/` prefix. `v1` indicates that we are currently using version 1 of the API.

### Parameters

Parameters passed in the URL must be properly URL-encoded, using the UTF-8 encoding for non-ASCII characters.

```
GET [..]/search?query=barack%20obama
```

### Error handling

Successful requests return a 2xx HTTP status code.

Failed requests return a 4xx HTTP status code. The response body of failed requests holds a JSON object containing an `error_message` field that describes the error.

```json
{
	"error_message": "Failed to parse query"
}
```

## Endpoints

### Search in an index

```
GET api/v1/indexes/<index id>/search?query=searchterm
```

Search for documents matching a query in the given index `<index id>`.

#### Path variable

| Variable      | Description   |
| ------------- | ------------- |
| **index id**  | The index id  |


#### Get parameters

| Variable                  | Type                 | Description                                                                                                | Default value                                                                                   |
| ------------------------- | -------------------- | -------------------------------------------------------------------------------------------------          | ----------------------------------------------------------------------------------------------- |
| **query**                 | `String`             | Query text. See the [query language doc](query-language.md) (mandatory)                                    |                                                                                                 |
| **start_timestamp**       | `i64`                | If set, restrict search to documents with a `timestamp >= start_timestamp`                                 |                                                                                                 |
| **end_timestamp**         | `i64`                | If set, restrict search to documents with a `timestamp < end_timestamp`                                    |                                                                                                 |
| **start_offset**          | `Integer`            | Number of documents to skip                                                                                | `0`                                                                                             |
| **max_hits**              | `Integer`            | Maximum number of hits to return (by default 20)                                                           | `20`                                                                                            |
| **search_field**          | `[String]`           | Fields to search on if no field name is specified in the query. Comma-separated list, e.g. "field1,field2" | index_config.search_settings.default_search_fields                                              |
| **format**                | `Enum`               | The output format. Allowed values are "json" or "prettyjson"                                               | `prettyjson`                                                                                    |
| **aggs**         				  | `JSON`               | The aggregations request. See the [aggregations doc](aggregation.md) for supported aggregations. 					| 


#### Response

The response is a JSON object, and the content type is `application/json; charset=UTF-8.`

| Field                   | Description                    | Type       |
| --------------------    | ------------------------------ | :--------: |
| **hits**                | Results of the query           | `[hit]`    |
| **num_hits**            | Total number of matches        | `number`   |
| **elapsed_time_micros** | Processing time of the query   | `number`   |


### Search stream in an index

```
GET api/v1/indexes/<index id>/search/stream?query=searchterm
```

Streams field values from ALL documents matching a search query in the given index `<index id>`, in a specified output format among the following:
 -  [CSV](https://datatracker.ietf.org/doc/html/rfc4180)
 -  [ClickHouse RowBinary](https://clickhouse.tech/docs/en/interfaces/formats/#rowbinary)

:::note

The endpoint will return 10 million values if 10 million documents match the query. This is expected, this endpoint is made to support queries matching millions of document and return field values in a reasonable response time.

:::


#### Path variable

| Variable      | Description   |
| ------------- | ------------- |
| **index id**  | The index id  |


#### Get parameters

| Variable            | Type       | Description                                                                                                      | Default value                                      |
| ----------          | ------     | -------------                                                                                                    | ---------------                                    |
| **query**           | `String`   | Query text. See the [query language doc](query-language.md) (mandatory)                                          |                                                    |
| **fast_field**      | `String`   | Name of a field to retrieve from documents. This field must be marked as "fast" in the index config. (mandatory) |                                                    |
| **search_field**    | `[String]` | Fields to search on. Comma-separated list, e.g. "field1,field2"                                                  | index_config.search_settings.default_search_fields |
| **start_timestamp** | `i64`      | If set, restrict search to documents with a `timestamp >= start_timestamp`                                       |                                                    |
| **end_timestamp**   | `i64`      | If set, restrict search to documents with a `timestamp < end_timestamp`                                          |                                                    |
| **output_format**   | `String`   | Response output format. `csv` or `clickHouseRowBinary`                                                           | `csv`                                              |


#### Response

The response is an HTTP stream. Depending on the client's capability, it is an HTTP1.1 [chunked transfer encoded stream](https://en.wikipedia.org/wiki/Chunked_transfer_encoding) or an HTTP2 stream.

It returns a list of all the field values from documents matching the query. The field must be marked as "fast" in the index config for this to work.
The formatting is based on the specified output format.

On error, an "X-Stream-Error" header will be sent via the trailers channel with information about the error, and the stream will be closed via [`sender.abort()`](https://docs.rs/hyper/0.14.16/hyper/body/struct.Sender.html#method.abort).
Depending on the client, the trailer header with error details may not be shown. The error will also be logged in quickwit ("Error when streaming search results").


### Ingest data in an index

```
POST api/v1/<index id>/ingest -d \
'{"url":"https://en.wikipedia.org/wiki?id=1","title":"foo","body":"foo"}
{"url":"https://en.wikipedia.org/wiki?id=2","title":"bar","body":"bar"}
{"url":"https://en.wikipedia.org/wiki?id=3","title":"baz","body":"baz"}'
```

Ingest a batch of documents to make them searchable in a given `<index id>`. Currently, NDJSON is the only accepted payload format. 

:::info
The payload size is limited to 10MB as this endpoint is intended to receive documents in batch.
:::

#### Path variable

| Variable      | Description   |
| ------------- | ------------- |
| **index id**  | The index id  |

#### Response

The response is a JSON object, and the content type is `application/json; charset=UTF-8.`

| Field                   | Description                        | Type       |
| --------------------    | ---------------------------------- | :--------: |
| **num_ingested_docs**   | Total number of documents ingested | `number`   |


### Ingest data with elastic compatible API

```
POST api/v1/_bulk -d \
'{ "create" : { "_index" : "wikipedia", "_id" : "1" } }
{"url":"https://en.wikipedia.org/wiki?id=1","title":"foo","body":"foo"}
{ "create" : { "_index" : "wikipedia", "_id" : "2" } }
{"url":"https://en.wikipedia.org/wiki?id=2","title":"bar","body":"bar"}
{ "create" : { "_index" : "wikipedia", "_id" : "3" } }
{"url":"https://en.wikipedia.org/wiki?id=3","title":"baz","body":"baz"}'
```

Ingest a batch of documents to make them searchable using the [Elastic Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) bulk API. This endpoint provides compatibility with tools or systems that already send data to Elastic Search for indexing. Currently, only the `create` action of the elastic bulk API is supported, all other actions such as `delete` or `update` are ignored.

:::info
The payload size is limited to 10MB as this endpoint is intended to receive documents in batch.
:::

#### Response

The response is a JSON object, and the content type is `application/json; charset=UTF-8.`

| Field                   | Description                        | Type       |
| --------------------    | ---------------------------------- | :--------: |
| **num_ingested_docs**   | Total number of documents ingested | `number`   |
