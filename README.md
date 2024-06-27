# BigQuery-Storage-Write-API-with-Geostationary-Data

This repo contains instructions and a code example of streaming geostationary data into BigQuery using the Storage Write API's default mode.

Documentation on how to use the Storage Write API can be found [HERE](https://cloud.google.com/bigquery/docs/write-api). 

Please refer and copy the files from the write-api-default-example folder to your dev environment.


To get started, we’ll first create a table in BigQuery named “example_geospatial” through the below DDL statement. The DDL also clusters the table by geo_point1 and geo_point2.
  ```
  #Replace Test_Tables with your preferred dataset name
  CREATE OR REPLACE TABLE temp.example_geospatial (
    geo_point1 GEOGRAPHY,
    geo_point2 GEOGRAPHY,
    geo_line GEOGRAPHY,
    geo_polygon GEOGRAPHY,
    geo_point1_string STRING,
    geo_point2_string STRING
    )
    CLUSTER BY
    geo_point1,geo_point2;
  ```
The Storage Write API ingests WKT or GeoJson formatted geostationary data as a STRING (as dicussed [HERE](https://cloud.google.com/bigquery/docs/write-api#data_type_conversions)). So we'll be writing all our geography data to BigQuery as a STRING, but notice that this table's first 4 columns are of the data type GEOGRAPHY, and the last two are of the type STRING. This is to illustrate that while you can write geo data as a STRING and later process it with BigQuery's geography functions, or you can natively use the GEOGRAPHY type in BigQuery as long as you format the STRING data you are writing with the Storage Write API in proper WKT or GeoJson. In this manner, BigQuery will store the data in your actual table as a GEOGRAPHY type, not a STRING type.

Now, we’ll ingest some data via the Storage Write API. In this example, we’ll use Python, so we’ll stream data as protocol buffers. For a quick refresher on working with protocol buffers, [here’s a great tutorial](https://developers.google.com/protocol-buffers/docs/pythontutorial). 

Using Python, we’ll first align our protobuf messages to the table we created using a .proto file in proto2 format. Use the sample_data.proto file from the write-api-default-example folder you downloaded to your developer environment, then run the following command within to update your protocol buffer definition:
  ```
  protoc --python_out=. sample_data.proto
  ```

Within your developer environment, use this sample default_stream_python_script.py Python script to insert some new example customer records by reading from the geo.json file and writing into the customer_records table. This code uses the BigQuery Storage Write API to stream a batch of row data by appending proto2 serialized bytes to the serialzed_rows repeated field like the example below:
  ```
  row = sample_data_pb2.SampleData()
      row.geo_point1 = "POINT (4 7)"
      row.geo_point2 = "POINT (8 12)"
      row.geo_line = "LINESTRING (20 40, 40 40, 50 40)"
      row.geo_polygon = "POLYGON((-124.49 47.35,-124.49 40.73,-116.49 40.73,-116.49 47.35,-124.49 47.35))"
      row.geo_point1_string = "POINT (4 7)"
      row.geo_point2_string = "POINT (8 12)"
  proto_rows.serialized_rows.append(row.SerializeToString())
  ```

We can now query our table with BigQuery's Geostationary functions and see it has ingested a few rows from the Storage Write API using the GEOGRAPHY type.
  ```
  SELECT
  ST_GEOHASH(geo_point1) as hashed,
  ST_MAKELINE(geo_point1,geo_point2) as make_line,
  #ST_LENGTH(geo_string_field), #Notice that this won't work because the geo_string_field is a STRING. You first have to interpret it as a GEOGRAPHY type using the below function.
  ST_GEOHASH(ST_GEOGFROMTEXT(geo_point1_string)) as hash_string
FROM
  temp.example_geospatial
  ```
