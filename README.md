# Entity Matching with Geospatial Proximity-boosting Using Azure Cognitive Search And Azure Databricks

This set of Databricks notebooks will allow you to create an Azure Search Index, load master data and match entities from an external source in batch mode. In order to conduct the search, a RESTful API is invoked using a Python User Defined Function (UDF) in Databricks. The process could be adapted to streaming mode by modifying the UDF and read operations to readstream. The matching notebook includes a threshold parameter (Widget) which can be used to define the cut-off score for which anything result below that score is considered a false-positive.

The search functionality is rich with the following options:
- geo-proximity boosting. If GeoJson 
- fuzzy matching
- Edit Distance, Jaro Winkler, length ratio scoring anhancements
- Stop word demotion (reduces affect on score)

Getting Started
===============

First [create](https://docs.microsoft.com/en-us/azure/search/search-create-service-portal) a basic Azure Search service or using an existing service. Make note of the resource enddpoint/URL and admin key. 
Launch a Databricks workspace and import the dbc archive of notebooks into your workspace. 

Read the additional information below before running the demo.

![architecture](media/architecture.png)
================================================================================

Scenario
========

Acme Marketing want to recommend the best breweries to their customers in Greenville County, South Carolina, United States. In order to do this they will harvest information from social networking/recommendation sites such as Foursquare or Trip Advisor and assign these reviews to  a master list of Breweries in the area. The master list was obtained via an open data set from [openupstate.org](https://data.openupstate.org/map/breweries) and slightly modified to suit their requirements. The problem with the data from the external social review sites is that the names (and sometimes locations) are not always accurate therefore they needs a robust search mechanism to cater for different brewery names and inexact location data. In the future, they know they will need to cater for [phonetic searches](https://azure.microsoft.com/en-us/blog/custom-analyzers-in-azure-search/) and [key phrase extraction](https://docs.microsoft.com/en-us/azure/search/cognitive-search-skill-keyphrases).

Solution
========

Acme have identified that Azure Cognitive Search (aka Knowledge Mining) supports a rich set of search capabilities including a geospatial distance boosting function which promotes breweries closest to the reference location provided to the top of the search results.
To run the matching solution they will load their master list into an Azure Search index and load the social review data into a Spark dataframe.  Using a UDF they can generate a best match to the master brewery list and then tag breweries with popularity scores and key phrases. Using this data they plan to run their recommendation algorithm to market the breweries to their customers. They have chosen to use Azure Databricks for this Spark based processing.

Benefits of Azure Search
========================

-   Fully managed PaaS service -- lowest cost of ownership

-   Process could be fully scripted end-to-end (CI/CD pipeline)

    -   Create Search service (ARM), create index, data source indexer via API or SDK

-   Pricing & usage: usage is billed per hour

    -   Can terminate service when not required to reduce cost

-   Scale: ability to scale (online) up and down during peak processing

-   Ability to incorporate Cognitive Services using skillsets

-   Geospatial support & Scoring Profiles

-   Simplified implementation & reduce query complexity

Data Ingestion
==============

-   Geospatial data

    -   Ensure data is in GeoJSON format.

    -   Eg: \"location\":{\"type\": \"Point\", \"coordinates\": \[49.5328469,8.7268988\]}
    
    -   If your data is in another format you can use Spark to reformat, e.g.
    
    *\"location\":{\"lat\":50.141925900000004,\"lon\":8.418495100000001}*
    
    can be converted by creating a new field called geojson_location using the Spark dataframe API code:
    
    *.withColumn(\"geojson_location\",struct(lit(\"Point\").as(\"type\"),array($\"location.lon\",$\"location.lat\").as(\"coordinates\")))*

-   Improving ingestion time:

    -   Consider multiple indexers. 50 indexers can be spawned on the standard tier

    -   Partitioning of flat folder structure into **n** number of folders with **n** indexers

    -   Increasing partitions = increases costs but can provide better ingestion rates and reduced query time

    -   Increase replicas will not improve ingestion. Used for increased query throughput (requests per second) due to additional compute & load balancing

    -   Further reading:

https://docs.microsoft.com/en-gb/azure/search/search-limits-quotas-capacity

https://docs.microsoft.com/en-us/azure/search/search-performance-optimization

Azure Search Ingestion Options
==============================

![https://docs.microsoft.com/en-us/azure/search/search-what-is-data-import
](media/ingestion.png)

Index creation and loading using REST APIs
==========================================

![](media/createandload.png)
================================================================================

Parallel loading operation using indexers
=========================================

![](media/parallelloading.png)

Geo-spatial Queries
===================

-   Requires the Edm.GeographyPoint data type

-   Filtering:

    1.  Geo.distance returns the distance in kilometers between two points

    2.  Geo.intersects returns true if a given point is within a given polygon

    3.  Distance boosting increases the score (rank) when closer to geo point

![](media/geofiltering.png)

Compared to

![](media/geoboosint.png)

AWS ES to Azure Search Challenges
=================================

-   Handling of geospatial data

    -   Elastic Search: Supports [5 different](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html) geo-point formats including geo-point expressed as object

    -   Azure Search:

        -   Loading only supports [GeoJSON Point](https://docs.microsoft.com/en-gb/rest/api/searchservice/Supported-data-types) format (Lon, Lat)

        -   Queries only support [Well-known Text (WKT)](https://docs.microsoft.com/en-us/azure/search/search-query-odata-geo-spatial-functions) point

POINT(lon lat): geo.distance(\[FIELD\_NAME\],geography\'POINT(\"\[LON\] \[LAT\]\")\') le 1

-   No ability to port existing Lucene queries

-   Analyzers: Only one analyzer can be applied to a field at a time

    -   Eg: stopwords and German language analyzer could not be applied simultaneously

-   Kibana (self service analytics UI) vs "Search App" (preview) for standard search UI -- no analytics
