# Entity Match Using Azure Search And Azure Databricks

This set of Databricks notebooks will allow you to create an Azure Search Index, load master data and match entities from an external source in batch mode. In order to conduct the search a RESTful API is invoked using a Python User Defined Function (UDF) The process could be adapted to streaming mode by modifying the UDF and read operations to readstream. 

The search functionality is rich with the following options:
- geo-proximity boosting
- fuzzy matching
- Edit Distance and Levenstein scoring anhancements
