
# bigquery-autoload
**_Auto-load files from Google Cloud Storage to Google BigQuery tables_**

A Google Cloud Function providing a **simple and configurable** way to automatically load data  from GCS files into Big Query tables.

It features a **_convention over configuration_** approches, the table name is automatically derived from the file's name, ignoring date and timestamp suffix.
If the default behaviour does not suit your needs, it can be modified, or overriden for certain files through mapping files or custom metadata.

# Quickstart
### First-time configuration
* Create a new `bq-autoload` Google Cloud Storage bucket
  Any file uploaded to this bucket will be automatically loaded into a BigQuery table
  
* Download the default mapping file [`mappings/000_default_mappings.json`](mappings/000_default_mappings.json) and edit the `_global_config` mapping configuration to replace the `projectId` property to match yours
   ```json
   // mappings/000_default_mappings.json
   [
     {
        "id":"_global_config",
	    "configuration.load.destinationTable.projectId":"__REPLACE_WITH_YOUR_PROJECT_ID__",
        // ...
     },
     // ...
   ]
  ```
  
* Upload the modified file to the gcs bucket as `gs://bq-autoload/mappings/000_default_mappings.json`
  ```bash
  $> gsutil cp "mappings/000_default_mappings.json" \
               "gs://bq-autoload/mappings/000_default_mappings.json"
  ```
 
* Deploy a cloud function triggered by changes on this GCS bucket
  ```bash
  $> gcloud functions deploy "bigquery-autoload" \
               --trigger-bucket "bq-autoload"
               --source "https://github.com/tfabien/bigquery-autoload" \
               --runtime "nodejs10"
  ```
  
  ### Loading data
  
* Upload the [samples/cities.csv](samples/cities.csv) file to the `bq-autoload` bucket
  ```bash
  $> gsutil cp "samples/cities_20190506.csv" "gs://bq-autoload/"
  ```
* A few seconds later, the corresponding `cities` BigQuery table has been created with data from the sample CSV file
  ```bash
  $> bq query "SELECT * FROM Staging.cities LIMIT 10"
  Waiting on bqjob_r7d361af5_0000016a8c759436_1 ... (0s) Current status: DONE
  +------+------+------+-------+------+------+------+------+-----------------+-------+
  | LatD | LatM | LatS |  NS   | LonD | LonM | LonS |  EW  |      City       | State |
  +------+------+------+-------+------+------+------+------+-----------------+-------+
  |   39 |   45 |    0 | false |   75 |   33 |    0 | W    | Wilmington      |  DE   |
  |   41 |   15 |    0 | false |   77 |    0 |    0 | W    | Williamsport    |  PA   |
  |   26 |   43 |   11 | false |   80 |    3 |    0 | W    | West Palm Beach |  FL   |
  |   36 |   40 |   11 | false |  121 |   39 |    0 | W    | Salinas         |  CA   |
  |   50 |   25 |   11 | false |  104 |   39 |    0 | W    | Regina          |  SA   |
  |   42 |   16 |   12 | false |   71 |   48 |    0 | W    | Worcester       |  MA   |
  |   29 |   25 |   12 | false |   98 |   30 |    0 | W    | San Antonio     |  TX   |
  |   35 |   56 |   23 | false |   77 |   48 |    0 | W    | Rocky Mount     |  NC   |
  |   32 |   30 |   35 | false |   93 |   45 |    0 | W    | Shreveport      |  LA   |
  +------+------+------+-------+------+------+------+------+-----------------+-------+
    ```
# Mapping files
When fired, the Cloud Function creates a BigQuery load job for the triggering file.

The configuration for this job results from the combination of all mappings matching the file's pattern, and all custom metadata prefixed with `bigquery.` on the file.

## Mapping file structure
A mapping file is a list of mapping configurations.

A mapping configuration is defined as follows:
```json
{
  /**
   *A unique id identifying the mapping configuration
  **/
  "id": "_unique_id_",
  
  /**
   * A Regex pattern.
   * If this expression matches the filepath, this mapping configuration will be active.
   * Multiple mapping configuration can match a single file, all active maping configuration will be merged to determine the resulting effective configuration.
   * If (named) capturing groups are defined in the expression, they will be available for usage in configuration value templating
  **/
  "match": "gs:\/\/.+\/MyFile\.csv" //,
  
  /**
   * Any number of configuration options for the load job
   * This are the options that will be merged into the effective job configuration if this mapping is active
   * @see https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs#configuration.load for the list of available options
  **/
  "configuration.load.destinationTable.tableId": "MyTable",
  "configuration.load.writeDisposition":"WRITE_TRUNCATE"
}
```
## Adding a new mapping file
All files of the autoload bucket matching the _(glob)_ pattern **`/mappings/**/*.json`** will be aggregated to obtain the full mapping configuration

Therefore, you can add any number of arbitrary JSON files to the `/mappings/` directory, defining the mappings you want to use.
These mappings can be specific to a single file, or a file pattern matching multiple similar files you want to process the same way.

Named capturing groups are available as variables for replacement inside the values of the configuration options.
[Mustache](https://mustache.github.io/) templating engine is used for the `String` values of the configuration.

>**Example:**
> The following file will instruct biquery-autoloader to load data from `export_{table}_{yyyyMMdd}.csv` into the `{table}` table, and use truncate data dwrite disposition at load rather than append. 
> 
> _eg: `/export_cities_20190506.csv` will be loaded into the `cities` table_
>```bash
>$> cat /mappings/export_TABLE_yyyyMMdd.json
>```
>```json
>[
>   {
>      "id": "export_{table}_{yyyyMMdd}.csv",
>      "match": "\/export_(?<TABLE>.*)_\\d{8}\.csv$",
>      "configuration.load.destinationTable.tableId": "{{{TABLE}}}",
>      "configuration.load.writeDisposition":"WRITE_TRUNCATE"
>   }
>]
>```

## Default behaviour
The `mapping/000_default_mappings.json` file defines the default configuration for the BigQuery `load` job
You can edit this file to modify or add default behaviour.

#### "_global_config"
```json
{
  "id": "_global_config",
  "match": ".*",
  "configuration.load.destinationTable.projectId":"__REPLACE_WITH_YOUR_PROJECT_ID__",
  "configuration.load.destinationTable.datasetId":"Staging",
  "configuration.load.writeDisposition":"WRITE_APPEND"	  
}
```
This configuration always applies and defines the global configuration options such as project id, dataset id, and write disposition

#### "_source_uri"
```json
{
  "id": "_source_uri",
  "match": "(?<FILE_URI>.*)",
  "configuration.load.sourceUris[0]":"{{{FILE_URI}}}"
}
```
This configuration always applies and defines the source uri for the load job.

#### "_table_naming"
```json
{
   "id":"_table_naming",
   "match":"^gs\\:\\/\\/(?<FILE_BUCKET>[^\\/]+)(?:\\/(?<FILE_SUBDIR>.*))?\\/(?<FILE_NAME>.*?)(?:_(?<FILE_DATE>[\\d\\-\\_]+)*)?(?:\\.(?<FILE_EXTENSION>.*))?$",
   "configuration.load.destinationTable.tableId":"{{{FILE_NAME}}}"
}
```
This configuration applies to all files and defines the table naming pattern for this file.

The table name is derived from the file name minus it's extension and optionnal `yyyyMMdd-HHmmSS` timestamp suffix, using named cature groups inside the regular expression.

> **Example:**
> The following files will all be loaded inside the same **`cities`** table:
> - `gs://bq-autoload/cities.csv`
> - `gs://bq-autoload/cities.json`
> - `gs://bq-autoload/cities_20190506.csv`
> - `gs://bq-autoload/cities_20190506-063000.csv`

You can change the `configuration.load.destinationTable.tableId`property to alter this naming convention if the provided default does not suite your needs.
Depending on the variables you want to use, you may also need to alter the `match` property to add new capturing groups.

> **Example:**
> Use the first subdirectory as `datasetId`, and the second one as `tableId`, ignoring the file name
> The `gs://bq-autoload/Public/Cities/export_cities_2019-05-06.csv` will be loaded into the `Public.Cities` table
> ```json
> {
>   "id":"_table_naming",
>   "match":"^(?<FILE_URI>gs\\:\\/\\/[^\\/]+(?<DATASET_ID>\\/[^\\/]+)\\/(?<TABLE_ID>[^\\/]+)\\/.*)$",
>   "configuration.load.destinationTable.datasetId":"{{{DATASET_ID}}}"
>   "configuration.load.destinationTable.tableId":"{{{TABLE_ID}}}"
>}
>```
### `_format_{csv|json|avro|...}`
Load options for the CSV file format:
```json
{
   "id":"_format_csv",
   "match":".+\\.csv$",
   "configuration.load.sourceFormat":"CSV",
   "configuration.load.autodetect":"True",
   "configuration.load.skipLeadingRows":"1",
   "configuration.load.ignoreUnknownValues":"True"
}
``` 

Load options for the JSON file format:
```json
{
    "id":"_format_json",
    "match":".+\\.js(?:on)?$",
    "configuration.load.sourceFormat":"NEWLINE_DELIMITED_JSON",
    "configuration.load.ignoreUnknownValues":"True"
}
```

Load options for the AVRO file format:
```json
{
   "id":"_format_avro",
   "match":".+\\.avro$",
   "configuration.load.sourceFormat":"AVRO",
   "configuration.load.useAvroLogicalTypes":"True"
}
```

# Custom metadata
Any custom metadata prefixed with `bigquery.` will be added to the `load` job configuration
Any option specified in custom metadata will take precedence and override existing configuration options if present

>**Example:**
> Changing the table name and dataset by setting custom metadata at upload time
> *Note: Custom metadata keys must be prefixed with `x-goog-meta-` when using gsutil to upload the file*
>```bash
>  gsutil -h "x-goog-meta-bigquery.configuration.load.destinationTable.datasetId: Test" \
>         -h "x-goog-meta-bigquery.configuration.load.destinationTable.tableId: City" \
>         cp "samples/cities_20190506.csv" "gs://bq-autoload/"
>```