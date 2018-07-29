# Transfer ViciDial MySQL call log into Google BigQuery & Data Studio Analytics
This documents describes the process to export enriched `vicidial_log` MySQL table into Google BigQuery to use it with Google Data Studio in order to create an Analytics Report.

I have enriched call log table by joining `vicidial_campaigns`, `vicidial_lists`, `vicidial_statuses`. This will allow to report campaign name, list name and call status name instead just their IDs in order for the report to be more descriptive. 
I also created a custom table `area_codes` where I loaded CSV dump to match lead phone area code to the city & state location in order to render map reports.

### Create area_codes table & import data
```
CREATE TABLE `area_codes` (
  `area_code` varchar(255) COLLATE utf8_unicode_ci NOT NULL DEFAULT '',
  `city` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  `state` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  `country` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  `latitude` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  `longitude` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  `latlng` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`area_code`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```

I used this CSV dump I found online in order to populate the table: 
https://github.com/ravisorg/Area-Code-Geolocation-Database/blob/master/us-area-code-cities.csv

### Export MySQL table into CSV file
Using mysql client I dumped the table data into a CSV file.
This process took about 5 minutes to dump approximately 40M rows.

```
USE asterisk;
SELECT 
    vicidial_log.*,
    vicidial_campaigns.campaign_name,
    vicidial_lists.list_name,
    vicidial_statuses.status_name,
    SUBSTR(vicidial_log.phone_number,1,3) AS areacode,
    area_codes.city, area_codes.state, area_codes.latlng

FROM vicidial_log 

JOIN vicidial_campaigns 
ON (vicidial_log.campaign_id=vicidial_campaigns.campaign_id)

JOIN vicidial_lists 
ON (vicidial_log.list_id=vicidial_lists.list_id)

JOIN vicidial_statuses 
ON (vicidial_log.status=vicidial_statuses.status)

JOIN area_codes 
ON (SUBSTR(vicidial_log.phone_number,1,3)=area_codes.area_code)

INTO OUTFILE '/var/lib/mysql/dump.csv' 
CHARACTER SET 'utf8' 
FIELDS TERMINATED BY '\t' 
OPTIONALLY ENCLOSED BY '';
```

### Compress data before transferring to Google Cloud Storage
```
gzip /var/lib/mysql/dump.csv
```

### Transfer data to Google Cloud Storage
```
gsutil -qmo GSUtil:parallel_composite_upload_threshold=10M mv dump.csv.gz gs://your_storage_bucket_name/
```

## Import CSV dump into BogQuery dataset
You will need a `schema.json` file to describe required data schema for the table.

```
bq load --replace=true --field_delimiter="\t" --null_marker="\N" --quote="" PROJECT_NAME:DATA_SET.TABLE_NAME gs://your_storage_bucket_name/dump.csv.gz schema.json
```
