# ETL - Synthea to OMOP

## Synthea Data Generation
Data is generated using [keep modules](https://mitre.github.io/fhir-for-research/modules/synthea-customizing). The main idea of these modules consists on normal synthea generation but only keeping the generated patients that satisfy concrete defined conditions. Three keep modules are created via [Synthea Module Builder](https://synthetichealth.github.io/module-builder/) depending on the diseases desired: CVD, Diabetes and None of the previous (**/keep_modules**). The idea is that each of them construct 1/3 (~333/1000) of the final data simulating the trials.

To do so, the keep module desired is copied to the synthea folder (outside of the rest of the folders, just inside synthea folder) and the following command is executed:

`run_synthea -a 18-75 -p 333 --exporter.csv.export=true -k {keep_module.json}`

After each set is created (CVD ~2h, Diabetes ~10 min, Not Diabetes or CVD ~1min) they are merged into the final dataset using `merge_datasets.ipynb`.

## ETL-Synthea ODHSI

For converting Synthea dataset into OMOP, the [ETL-Synthea by OHDSI](https://github.com/OHDSI/ETL-Synthea/tree/main) is used. The tutorial consists on the following steps:
1. Create a database in progres (`synthea_omop`)
2. Create the OMOP schema in the database (`omop_schema`)
3. Create a native schema for Synthea data (`native`)
4. Install [JDBC postgres drivers](https://jdbc.postgresql.org/download/) and save on a selected folder (`/PathToDrivers`)
5. Download vocabularies from [Athenea](https://athena.ohdsi.org/vocabulary/download-history) and save them in a located folder (`/PathToVocab`)
``` r
devtools::install_github("OHDSI/ETL-Synthea")
library(ETLSyntheaBuilder)

cd <- DatabaseConnector::createConnectionDetails(dbms = "postgresql",server = "localhost/synthea_omop", user = "postgres", password = "postgres", port = 5432, pathToDriver = "~/PathToDrivers")
syntheaVersion <- "3.3.0"
syntheaSchema  <- "native"
cdmSchema      <- "omop_schema"
cdmVersion     <- "5.4"
syntheaFileLoc <- "~/PathToSyntheaData"
vocabFileLoc   <- "~/PathToVocab"
```
6. Create CDM Tables:
``` r
ETLSyntheaBuilder::CreateCDMTables(connectionDetails = cd, cdmSchema = cdmSchema, cdmVersion = cdmVersion)
```
7. Create Synthea tables executing `create_synthea_tables.sql` due to a syntax error in OHDSI R function ("start"_date on payer_transitions):
``` r
sqlFilePath <- "~/create_synthea_tables.sql"
sqlQuery <- readChar(sqlFilePath, file.info(sqlFilePath)$size)
conn <- DatabaseConnector::connect(connectionDetails)
DatabaseConnector::executeSql(conn, sqlQuery)
DatabaseConnector::disconnect(conn)
```
8. Load Synthea Tables with generated data (trial_data):
``` r
ETLSyntheaBuilder::LoadSyntheaTables(connectionDetails = cd, syntheaSchema = syntheaSchema, syntheaFileLoc = syntheaFileLoc)
```
9. Load vocabulary from CSV files
``` r
ETLSyntheaBuilder::LoadVocabFromCsv(connectionDetails = cd, cdmSchema = cdmSchema, vocabFileLoc = vocabFileLoc)
```
10. Execute and poblate CDM tables
``` r
ETLSyntheaBuilder::CreateMapAndRollupTables(connectionDetails = cd, cdmSchema = cdmSchema, syntheaSchema = syntheaSchema, cdmVersion = cdmVersion, syntheaVersion = syntheaVersion)

# Optional Steps
ETLSyntheaBuilder::CreateExtraIndices(connectionDetails = cd, cdmSchema = cdmSchema, syntheaSchema = syntheaSchema, syntheaVersion = syntheaVersion)

ETLSyntheaBuilder::LoadEventTables(connectionDetails = cd, cdmSchema = cdmSchema, syntheaSchema = syntheaSchema, cdmVersion = cdmVersion, syntheaVersion = syntheaVersion)
```

On [ETL-Synthea/vignettes](https://github.com/OHDSI/ETL-Synthea/tree/main/vignettes) it can be seen how the mapping is actually done for each table of the OMOP CDM, in order to see posible failures and where data actually came from.