# Importing Into A Local PostgreSQL Database

**Prerequisites**:
  - [PostgreSQL](https://www.postgresql.org/) is installed and running
  - (optional) for geospatial queries, [PostGIS](https://postgis.net/) must be installed
  - The [metadata](../../../Metadata) files have been [downloaded](../../Download)
<br/>

First create a new database. From a terminal run:
If using PostGIS:
```bash
createdb -T template_postgis inaturalist-open-data
```
Otherwise
```bash
createdb inaturalist-open-data
```

<br/>

Download [structure.sql](../../structure.sql) and apply it to the new database. From a terminal run:
```bash
psql inaturalist-open-data < structure.sql
```
<br/>

Import the CSV files into the created tables (adjust the paths according to the local enironment). Import time will vary depending on the configuration of the database. These are large files and it may take several minutes to import. Connect to a PostgreSQL console with `psql inaturalist-open-data` and run:
```sql
COPY observations FROM 'observations.csv' DELIMITER E'\t' QUOTE E'\b' CSV HEADER;
COPY photos FROM 'photos.csv' DELIMITER E'\t' QUOTE E'\b' CSV HEADER;
COPY taxa FROM 'taxa.csv' DELIMITER E'\t' QUOTE E'\b' CSV HEADER;
COPY users FROM 'users.csv' DELIMITER E'\t' QUOTE E'\b' CSV HEADER;
```
<br/>

Create some indices that may help improve the speed of some queries. From the PostgreSQL console run:
```sql
CREATE INDEX index_photos_photo_uuid ON photos USING btree (photo_uuid);
CREATE INDEX index_photos_observation_uuid ON photos USING btree (observation_uuid);
CREATE INDEX index_taxa_taxon_id ON taxa USING btree (taxon_id);
CREATE INDEX index_users_user_id ON users USING btree (user_id);
CREATE INDEX index_observations_user_id ON observations USING btree (user_id);
CREATE INDEX index_observations_taxon_id ON taxa USING btree (taxon_id);
```
<br/>

At this point the database should be set up, populated with data, and indexed. For example, this query should return 10 photos of animals and details from their associated observations. From the PostgreSQL console run:

```sql
SELECT
  t.taxon_id,
  t.name taxon_name,
  'https://www.inaturalist.org/observations/' || o.observation_uuid as observation_url,
  u.login,
  'https://inaturalist-open-data.s3.amazonaws.com/photos/' || p.photo_id || '/medium.' || p.extension as photo_url
FROM taxa t
JOIN observations o ON (t.taxon_id = o.taxon_id)
JOIN photos p ON (o.observation_uuid=p.observation_uuid)
JOIN users u ON (o.user_id = u.user_id)
WHERE t.ancestry LIKE '48460/1/%'
LIMIT 10;
```

### Geospatial Queries, PostGIS
To make geospatial queries on observations, PostGIS must be installed and the database created with the `template_postgis` template. Create and populate a PostGIS geometry with the latitude and longitude columns in observations table. This `UPDATE` will likely to take longer to run than it took to import the data. From the PostgreSQL console run:
```sql
ALTER TABLE observations ADD COLUMN geom public.geometry;

UPDATE observations SET geom = ST_GeomFromText('POINT(' || longitude || ' ' || latitude || ')', 4326);

```
<br/>

After that finishes, geospatial queries can be made with the new `observations.geom` column. For example, this query should return 10 observations within a bounding box. From the PostgreSQL console run:
```sql
SELECT COUNT(*) FROM observations WHERE ST_Within(geom, 'POLYGON((-162 18, -153 18, -153 23, -162 23, -162 18))'::geography::geometry);
```