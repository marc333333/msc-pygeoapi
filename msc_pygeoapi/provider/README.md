This plugin requires the following patch to pygeoapi master branch:

```diff
diff --git a/pygeoapi/api.py b/pygeoapi/api.py
index ed21350..fc87017 100644
--- a/pygeoapi/api.py
+++ b/pygeoapi/api.py
@@ -511,9 +511,17 @@ class API:
                     })
                 if dataset is not None:
                     LOGGER.debug('Creating extended coverage metadata')
-                    p = load_plugin('provider', get_provider_by_type(
-                        self.config['resources'][dataset]['providers'],
-                        'coverage'))
+                    try:
+                        p = load_plugin('provider', get_provider_by_type(
+                            self.config['resources'][dataset]['providers'],
+                            'coverage'))
+                    except ProviderConnectionError:
+                        exception = {
+                           'code': 'NoApplicableCode',
+                           'description': 'connection error (check logs)'
+                        }
+                        LOGGER.error(exception)
+                        return headers_, 500, to_json(exception, self.pretty_print)
 
                     collection['crs'] = [p.crs]
                     collection['domainset'] = p.get_coverage_domainset()
@@ -1313,6 +1321,13 @@ class API:
             query_args['subsets'] = subsets
             LOGGER.debug('Subsets: {}'.format(query_args['subsets']))
 
+        if 'bbox' in args:
+            LOGGER.debug('Processing bbox parameter')
+            query_args['bbox'] = args['bbox'].split(',')
+        if 'datetime' in args:
+            LOGGER.debug('Processing datetime parameter')
+            query_args['datetime'] = args['datetime']
+
         LOGGER.debug('Querying coverage')
         try:
             data = p.query(**query_args)
```

Sample configuration

```yaml
hrdps-temperature:
    type: collection
    title: Global Deterministic Prediction System sample
    description: Global Deterministic Prediction System sample
    keywords:
        - hrdps
    extents:
        spatial:
            bbox: [-152.962, 27.6163, -43.8168, 69.9581]
            crs: http://www.opengis.net/def/crs/OGC/1.3/CRS84
    links:
        - type: text/html
          rel: canonical
          title: information
          href: https://eccc-msc.github.io/open-data/msc-data/nwp_gdps/readme_gdps_en
          hreflang: en-CA
    providers:
        - type: coverage
          name: msc_pygeoapi.provider.gdr_rasterio.GDRRasterioProvider
          data: http://localhost:9200/geomet-data-registry
          options:
              #DATA_ENCODING: COMPLEX_PACKING  # GDPS
              DATA_ENCODING: SIMPLE_PACKING
          format:
              name: GRIB
              mimetype: application/x-grib2
```