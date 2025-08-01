---
title: Copy and transform data in Snowflake V2
titleSuffix: Azure Data Factory & Azure Synapse
description: Learn how to copy and transform data in Snowflake V2 using Data Factory or Azure Synapse Analytics.
ms.author: jianleishen
author: jianleishen
ms.subservice: data-movement
ms.topic: conceptual
ms.custom: synapse
ms.date: 05/15/2025
ai-usage: ai-assisted
---

# Copy and transform data in Snowflake V2 using Azure Data Factory or Azure Synapse Analytics

[!INCLUDE[appliesto-adf-asa-md](includes/appliesto-adf-asa-md.md)]

This article outlines how to use the Copy activity in Azure Data Factory and Azure Synapse pipelines to copy data from and to Snowflake, and use Data Flow to transform data in Snowflake. For more information, see the introductory article for [Data Factory](introduction.md) or [Azure Synapse Analytics](../synapse-analytics/overview-what-is.md).

> [!IMPORTANT]
> The [Snowflake V2 connector](connector-snowflake.md) provides improved native Snowflake support. If you are using the [Snowflake V1 connector](connector-snowflake-legacy.md) in your solution, you are recommended to [upgrade your Snowflake connector](#upgrade-the-snowflake-linked-service) before **June 30, 2025**. Refer to this [section](#differences-between-snowflake-and-snowflake-legacy) for details on the difference between V2 and V1. 

## Supported capabilities

This Snowflake connector is supported for the following capabilities:

| Supported capabilities|IR |
|---------| --------|
|[Copy activity](copy-activity-overview.md) (source/sink)|&#9312; &#9313;|
|[Mapping data flow](concepts-data-flow-overview.md) (source/sink)|&#9312; |
|[Lookup activity](control-flow-lookup-activity.md)|&#9312; &#9313;|
|[Script activity](transform-data-using-script.md) (Apply version 1.1 (Preview) when you use the script parameter)|&#9312; &#9313;|

*&#9312; Azure integration runtime &#9313; Self-hosted integration runtime*

For the Copy activity, this Snowflake connector supports the following functions:

- Copy data from Snowflake that utilizes Snowflake's [COPY into [location]](https://docs.snowflake.com/en/sql-reference/sql/copy-into-location.html) command to achieve the best performance.
- Copy data to Snowflake that takes advantage of Snowflake's [COPY into [table]](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html) command to achieve the best performance. It supports Snowflake on Azure.
- If a proxy is required to connect to Snowflake from a self-hosted Integration Runtime, you must configure the environment variables for HTTP_PROXY and HTTPS_PROXY on the Integration Runtime host. 

## Prerequisites

If your data store is located inside an on-premises network, an Azure virtual network, or Amazon Virtual Private Cloud, you need to configure a [self-hosted integration runtime](create-self-hosted-integration-runtime.md) to connect to it. Make sure to add the IP addresses that the self-hosted integration runtime uses to the allowed list. 

If your data store is a managed cloud data service, you can use the Azure Integration Runtime. If the access is restricted to IPs that are approved in the firewall rules, you can add [Azure Integration Runtime IPs](azure-integration-runtime-ip-addresses.md) to the allowed list.

The Snowflake account that is used for Source or Sink should have the necessary `USAGE` access on the database and read/write access on schema and the tables/views under it. In addition, it should also have `CREATE STAGE` on the schema to be able to create the External stage with SAS URI.

The following Account properties values must be set

| Property         | Description                                                  | Required | Default
| :--------------- | :----------------------------------------------------------- | :------- | :-------
| REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_CREATION             | Specifies whether to require a storage integration object as cloud credentials when creating a named external stage (using CREATE STAGE) to access a private cloud storage location.             | FALSE      | FALSE
| REQUIRE_STORAGE_INTEGRATION_FOR_STAGE_OPERATION            | Specifies whether to require using a named external stage that references a storage integration object as cloud credentials when loading data from or unloading data to a private cloud storage location. | FALSE | FALSE

For more information about the network security mechanisms and options supported by Data Factory, see [Data access strategies](data-access-strategies.md).

## Get started

[!INCLUDE [data-factory-v2-connector-get-started](includes/data-factory-v2-connector-get-started.md)]

## Create a linked service to Snowflake using UI

Use the following steps to create a linked service to Snowflake in the Azure portal UI.

1. Browse to the Manage tab in your Azure Data Factory or Synapse workspace and select Linked Services, then click New:

    # [Azure Data Factory](#tab/data-factory)

    :::image type="content" source="media/doc-common-process/new-linked-service.png" alt-text="Screenshot of creating a new linked service with Azure Data Factory UI.":::

    # [Azure Synapse](#tab/synapse-analytics)

    :::image type="content" source="media/doc-common-process/new-linked-service-synapse.png" alt-text="Screenshot of creating a new linked service with Azure Synapse UI.":::

2. Search for Snowflake and select the Snowflake connector.

    :::image type="content" source="media/connector-snowflake/snowflake-connector.png" alt-text="Screenshot of the Snowflake connector.":::    

1. Configure the service details, test the connection, and create the new linked service.

    :::image type="content" source="media/connector-snowflake/configure-snowflake-linked-service.png" alt-text="Screenshot of linked service configuration for Snowflake.":::

## Connector configuration details

The following sections provide details about properties that define entities specific to a Snowflake connector.

## Linked service properties

These generic properties are supported for the Snowflake linked service:

| Property         | Description                                                  | Required |
| :--------------- | :----------------------------------------------------------- | :------- |
| type             | The type property must be set to **SnowflakeV2**.              | Yes      |
| version           | The version that you specify. Recommend upgrading to the latest version to take advantage of the newest enhancements.       | Yes for version 1.1 (Preview)     |
| accountIdentifier | The name of the account along with its organization. For example, myorg-account123.   | Yes |
| database | The default database used for the session after connecting. | Yes |
| warehouse | The default virtual warehouse used for the session after connecting. |Yes|
| authenticationType | Type of authentication used to connect to the Snowflake service. Allowed values are: **Basic** (Default) and  **KeyPair**. Refer to corresponding sections below on more properties and examples respectively.  | No |
| role | The default security role used for the session after connecting. | No |
| host | The host name of the Snowflake account. For example: `contoso.snowflakecomputing.com`. `.cn` is also supported.| No |
| connectVia | The [integration runtime](concepts-integration-runtime.md) that is used to connect to the data store. You can use the Azure integration runtime or a self-hosted integration runtime (if your data store is located in a private network). If not specified, it uses the default Azure integration runtime. | No |

This Snowflake connector supports the following authentication types. See the corresponding sections for details.

- [Basic authentication](#basic-authentication)
- [Key pair authentication](#key-pair-authentication)

### Basic authentication

To use **Basic** authentication, in addition to the generic properties that are described in the preceding section, specify the following properties:

| Property         | Description                                                  | Required |
| :--------------- | :----------------------------------------------------------- | :------- |
| user | Login name for the Snowflake user. | Yes |
| password | The password for the Snowflake user. Mark this field as a SecureString type to store it securely. You can also [reference a secret stored in Azure Key Vault](store-credentials-in-key-vault.md). | Yes |

**Example:**

```json
{
    "name": "SnowflakeV2LinkedService",
    "properties": {
        "type": "SnowflakeV2",
        "typeProperties": {
            "accountIdentifier": "<accountIdentifier>",
            "database": "<database>",
            "warehouse": "<warehouse>",
            "authenticationType": "Basic",
            "user": "<username>",
            "password": {
                "type": "SecureString",
                "value": "<password>"
            },
            "role": "<role>"
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
```

**Password in Azure Key Vault:**

```json
{
    "name": "SnowflakeV2LinkedService",
    "properties": {
        "type": "SnowflakeV2",
        "typeProperties": {
            "accountIdentifier": "<accountIdentifier>",
            "database": "<database>",
            "warehouse": "<warehouse>",
            "authenticationType": "Basic",
            "user": "<username>",
            "password": {
                "type": "AzureKeyVaultSecret",
                "store": { 
                    "referenceName": "<Azure Key Vault linked service name>",
                    "type": "LinkedServiceReference"
                }, 
                "secretName": "<secretName>"
            }
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
```

### Key pair authentication

To use **Key pair** authentication, you need to configure and create a key pair authentication user in Snowflake by referring to [Key Pair Authentication & Key Pair Rotation](https://docs.snowflake.com/en/user-guide/key-pair-auth). Afterwards, make a note of the private key and the passphrase (optional), which you use to define the linked service.

In addition to the generic properties that are described in the preceding section, specify the following properties:

| Property         | Description                                                  | Required |
| :--------------- | :----------------------------------------------------------- | :------- |
| user | Login name for the Snowflake user. | Yes |
| privateKey | The private key used for the key pair authentication. <br/><br/>To ensure the private key is valid when sent to Azure Data Factory, and considering that the privateKey file includes newline characters (\n), it's essential to correctly format the privateKey content in its string literal form. This process involves adding \n explicitly to each newline.   | Yes |
| privateKeyPassphrase | The passphrase used for decrypting the private key, if it's encrypted.  | No |

**Example:**

```json
{
    "name": "SnowflakeV2LinkedService",
    "properties": {
        "type": "SnowflakeV2",
        "typeProperties": {
            "accountIdentifier": "<accountIdentifier>",
            "database": "<database>",
            "warehouse": "<warehouse>",
            "authenticationType": "KeyPair",
            "user": "<username>",
            "privateKey": {
                "type": "SecureString",
                "value": "<privateKey>"
            },
            "privateKeyPassphrase": { 
                "type": "SecureString",
                "value": "<privateKeyPassphrase>"
            },
            "role": "<role>"
        },
        "connectVia": {
            "referenceName": "<name of Integration Runtime>",
            "type": "IntegrationRuntimeReference"
        }
    }
}
```
> [!NOTE]
> For mapping data flows, we recommend generating a new RSA private key using the PKCS#8 standard in PEM format (.p8 file).


## Dataset properties

For a full list of sections and properties available for defining datasets, see the [Datasets](concepts-datasets-linked-services.md) article.

The following properties are supported for the Snowflake dataset.

| Property  | Description                                                  | Required                    |
| :-------- | :----------------------------------------------------------- | :-------------------------- |
| type      | The type property of the dataset must be set to **SnowflakeV2Table**. | Yes                         |
| schema | Name of the schema. Note the schema name is case-sensitive. |No for source, yes for sink  |
| table | Name of the table/view. Note the table name is case-sensitive. |No for source, yes for sink  |

**Example:**

```json
{
    "name": "SnowflakeV2Dataset",
    "properties": {
        "type": "SnowflakeV2Table",
        "typeProperties": {
            "schema": "<Schema name for your Snowflake database>",
            "table": "<Table name for your Snowflake database>"
        },
        "schema": [ < physical schema, optional, retrievable during authoring > ],
        "linkedServiceName": {
            "referenceName": "<name of linked service>",
            "type": "LinkedServiceReference"
        }
    }
}
```

## Copy activity properties

For a full list of sections and properties available for defining activities, see the [Pipelines](concepts-pipelines-activities.md) article. This section provides a list of properties supported by the Snowflake source and sink.

### Snowflake as the source

Snowflake connector utilizes Snowflake's [COPY into [location]](https://docs.snowflake.com/en/sql-reference/sql/copy-into-location.html) command to achieve the best performance.

If sink data store and format are natively supported by the Snowflake COPY command, you can use the Copy activity to directly copy from Snowflake to sink. For details, see [Direct copy from Snowflake](#direct-copy-from-snowflake). Otherwise, use built-in [Staged copy from Snowflake](#staged-copy-from-snowflake).

To copy data from Snowflake, the following properties are supported in the Copy activity **source** section.

| Property                     | Description                                                  | Required |
| :--------------------------- | :----------------------------------------------------------- | :------- |
| type                         | The type property of the Copy activity source must be set to **SnowflakeV2Source**. | Yes      |
| query          | Specifies the SQL query to read data from Snowflake. If the names of the schema, table and columns contain lower case, quote the object identifier in query e.g. `select * from "schema"."myTable"`.<br>Executing stored procedure isn't supported. | No       |
| exportSettings | Advanced settings used to retrieve data from Snowflake. You can configure the ones supported by the COPY into command that the service will pass through when you invoke the statement. | Yes       |
| ***Under `exportSettings`:*** |  |  |
| type | The type of export command, set to **SnowflakeExportCopyCommand**. | Yes |
| storageIntegration | Specify the name of your storage integration that you created in the Snowflake. For the prerequisite steps of using the storage integration, see [Configuring a Snowflake storage integration](https://docs.snowflake.com/en/user-guide/data-load-azure-config#option-1-configuring-a-snowflake-storage-integration). | No |
| additionalCopyOptions | Additional copy options, provided as a dictionary of key-value pairs. Examples: MAX_FILE_SIZE, OVERWRITE. For more information, see [Snowflake Copy Options](https://docs.snowflake.com/en/sql-reference/sql/copy-into-location.html#copy-options-copyoptions). | No |
| additionalFormatOptions | Additional file format options that are provided to COPY command as a dictionary of key-value pairs. Examples: DATE_FORMAT, TIME_FORMAT, TIMESTAMP_FORMAT, NULL_IF. For more information, see [Snowflake Format Type Options](https://docs.snowflake.com/en/sql-reference/sql/copy-into-location.html#format-type-options-formattypeoptions). <br/><br/>When you use NULL_IF, the NULL value in Snowflake is converted to the specified value (which needs to be single-quoted) when writing to the delimited text file in the staging storage. This specified value is treated as NULL when reading from the staging file to the sink storage. The default value is `'NULL'`. | No |

>[!Note]
> Make sure you have permission to execute the following command and access the schema *INFORMATION_SCHEMA* and the table *COLUMNS*.
>
>- `COPY INTO <location>`

#### Direct copy from Snowflake

If your sink data store and format meet the criteria described in this section, you can use the Copy activity to directly copy from Snowflake to sink. The service checks the settings and fails the Copy activity run if the following criteria isn't met:

- When you specify `storageIntegration` in the source:

  The sink data store is the Azure Blob Storage that you referred in the external stage in Snowflake. You need to complete the following steps before copying data:

    1. Create an [**Azure Blob Storage**](connector-azure-blob-storage.md) linked service for the sink Azure Blob Storage with any supported authentication types.
    
    2. Grant at least **Storage Blob Data Contributor** role to the Snowflake service principal in the sink Azure Blob Storage **Access Control (IAM)**.

- When you don't specify `storageIntegration` in the source:
  
  The **sink linked service** is [**Azure Blob storage**](connector-azure-blob-storage.md) with **shared access signature** authentication. If you want to directly copy data to Azure Data Lake Storage Gen2 in the following supported format, you can create an Azure Blob Storage linked service with SAS authentication against your Azure Data Lake Storage Gen2 account, to avoid using [staged copy from Snowflake](#staged-copy-from-snowflake).

- The **sink data format** is of **Parquet**, **delimited text**, or **JSON** with the following configurations:

    - For **Parquet** format, the compression codec is **None**, **Snappy**, or **Lzo**.
    - For **delimited text** format:
        - `rowDelimiter` is **\r\n**, or any single character.
        - `compression` can be **no compression**, **gzip**, **bzip2**, or **deflate**.
        - `encodingName` is left as default or set to **utf-8**.
        - `quoteChar` is **double quote**, **single quote**, or **empty string** (no quote char).
    - For **JSON** format, direct copy only supports the case that source Snowflake table or query result only has single column and the data type of this column is **VARIANT**, **OBJECT**, or **ARRAY**.
        - `compression` can be **no compression**, **gzip**, **bzip2**, or **deflate**.
        - `encodingName` is left as default or set to **utf-8**.
        - `filePattern` in copy activity sink is left as default or set to **setOfObjects**.
- In copy activity source, `additionalColumns` isn't specified.
- Column mapping isn't specified.

**Example:**

```json
"activities":[
    {
        "name": "CopyFromSnowflake",
        "type": "Copy",
        "inputs": [
            {
                "referenceName": "<Snowflake input dataset name>",
                "type": "DatasetReference"
            }
        ],
        "outputs": [
            {
                "referenceName": "<output dataset name>",
                "type": "DatasetReference"
            }
        ],
        "typeProperties": {
            "source": {
                "type": "SnowflakeV2Source",
                "query": "SELECT * FROM MYTABLE",
                "exportSettings": {
                    "type": "SnowflakeExportCopyCommand",
                    "additionalCopyOptions": {
                        "MAX_FILE_SIZE": "64000000",
                        "OVERWRITE": true
                    },
                    "additionalFormatOptions": {
                        "DATE_FORMAT": "'MM/DD/YYYY'"
                    },
                    "storageIntegration": "< Snowflake storage integration name >"
                }
            },
            "sink": {
                "type": "<sink type>"
            }
        }
    }
]
```

#### Staged copy from Snowflake

When your sink data store or format isn't natively compatible with the Snowflake COPY command, as mentioned in the last section, enable the built-in staged copy using an interim Azure Blob storage instance. The staged copy feature also provides you with better throughput. The service exports data from Snowflake into staging storage, then copies the data to sink, and finally cleans up your temporary data from the staging storage. See [Staged copy](copy-activity-performance-features.md#staged-copy) for details about copying data by using staging.

To use this feature, create an [Azure Blob storage linked service](connector-azure-blob-storage.md#linked-service-properties) that refers to the Azure storage account as the interim staging. Then specify the `enableStaging` and `stagingSettings` properties in the Copy activity.

- When you specify `storageIntegration` in the source, the interim staging Azure Blob Storage should be the one that you referred in the external stage in Snowflake. Ensure that you create an [Azure Blob Storage](connector-azure-blob-storage.md) linked service for it with any supported authentication when using the Azure integration runtime, or with anonymous, account key, shared access signature or service principal authentication when using the self-hosted integration runtime. In addition, grant at least **Storage Blob Data Contributor** role to the Snowflake service principal in the staging Azure Blob Storage **Access Control (IAM)**. 

- When you don't specify `storageIntegration` in the source, the staging Azure Blob Storage linked service must use shared access signature authentication, as required by the Snowflake COPY command. Make sure you grant proper access permission to Snowflake in the staging Azure Blob Storage. To learn more about this, see this [article](https://docs.snowflake.com/en/user-guide/data-load-azure-config.html#option-2-generating-a-sas-token). 

**Example:**

```json
"activities":[
    {
        "name": "CopyFromSnowflake",
        "type": "Copy",
        "inputs": [
            {
                "referenceName": "<Snowflake input dataset name>",
                "type": "DatasetReference"
            }
        ],
        "outputs": [
            {
                "referenceName": "<output dataset name>",
                "type": "DatasetReference"
            }
        ],
        "typeProperties": {
            "source": {
                "type": "SnowflakeV2Source",               
                "query": "SELECT * FROM MyTable",
                "exportSettings": {
                    "type": "SnowflakeExportCopyCommand",
                    "storageIntegration": "< Snowflake storage integration name >"                    
                }
            },
            "sink": {
                "type": "<sink type>"
            },
            "enableStaging": true,
            "stagingSettings": {
                "linkedServiceName": {
                    "referenceName": "MyStagingBlob",
                    "type": "LinkedServiceReference"
                },
                "path": "mystagingpath"
            }
        }
    }
]
```

When performing a staged copy from Snowflake, it is crucial to set the Sink Copy Behavior to **Merge Files**. This setting ensures that all partitioned files are correctly handled and merged, preventing the issue where only the last partitioned file is copied. 

**Example Configuration**

```json
{
    "type": "Copy",
    "source": {
        "type": "SnowflakeSource",
        "query": "SELECT * FROM my_table"
    },
    "sink": {
        "type": "AzureBlobStorage",
        "copyBehavior": "MergeFiles"
    }
}
```

>[!NOTE]
>Failing to set the Sink Copy Behavior to **Merge Files** may result in only the last partitioned file being copied.

### Snowflake as sink

Snowflake connector utilizes Snowflake’s [COPY into [table]](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html) command to achieve the best performance. It supports writing data to Snowflake on Azure.

If source data store and format are natively supported by Snowflake COPY command, you can use the Copy activity to directly copy from source to Snowflake. For details, see [Direct copy to Snowflake](#direct-copy-to-snowflake). Otherwise, use built-in [Staged copy to Snowflake](#staged-copy-to-snowflake).

To copy data to Snowflake, the following properties are supported in the Copy activity **sink** section.

| Property          | Description                                                  | Required                                      |
| :---------------- | :----------------------------------------------------------- | :-------------------------------------------- |
| type              | The type property of the Copy activity sink, set to **SnowflakeV2Sink**. | Yes                                           |
| preCopyScript     | Specify a SQL query for the Copy activity to run before writing data into Snowflake in each run. Use this property to clean up the preloaded data. | No                                            |
| importSettings | Advanced settings used to write data into Snowflake. You can configure the ones supported by the COPY into command that the service will pass through when you invoke the statement. | Yes |
| ***Under `importSettings`:*** |                                                              |  |
| type | The type of import command, set to **SnowflakeImportCopyCommand**. | Yes |
| storageIntegration | Specify the name of your storage integration that you created in the Snowflake. For the prerequisite steps of using the storage integration, see [Configuring a Snowflake storage integration](https://docs.snowflake.com/en/user-guide/data-load-azure-config#option-1-configuring-a-snowflake-storage-integration). | No |
| additionalCopyOptions | Additional copy options, provided as a dictionary of key-value pairs. Examples: ON_ERROR, FORCE, LOAD_UNCERTAIN_FILES. For more information, see [Snowflake Copy Options](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html#copy-options-copyoptions). | No |
| additionalFormatOptions | Additional file format options provided to the COPY command, provided as a dictionary of key-value pairs. Examples: DATE_FORMAT, TIME_FORMAT, TIMESTAMP_FORMAT. For more information, see [Snowflake Format Type Options](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table.html#format-type-options-formattypeoptions). | No |

>[!Note]
> Make sure you have permission to execute the following command and access the schema *INFORMATION_SCHEMA* and the table *COLUMNS*.
>
>- `SELECT CURRENT_REGION()`
>- `COPY INTO <table>`
>- `SHOW REGIONS`
>- `CREATE OR REPLACE STAGE`
>- `DROP STAGE`

#### Direct copy to Snowflake

If your source data store and format meet the criteria described in this section, you can use the Copy activity to directly copy from source to Snowflake. The service checks the settings and fails the Copy activity run if the following criteria isn't met:

- When you specify `storageIntegration` in the sink:

  The source data store is the Azure Blob Storage that you referred in the external stage in Snowflake. You need to complete the following steps before copying data:

    1. Create an [**Azure Blob Storage**](connector-azure-blob-storage.md) linked service for the source Azure Blob Storage with any supported authentication types. 

    2. Grant at least **Storage Blob Data Reader** role to the Snowflake service principal in the source Azure Blob Storage **Access Control (IAM)**.

- When you don't specify `storageIntegration` in the sink: 

  The **source linked service** is [**Azure Blob storage**](connector-azure-blob-storage.md) with **shared access signature** authentication. If you want to directly copy data from Azure Data Lake Storage Gen2 in the following supported format, you can create an Azure Blob Storage linked service with SAS authentication against your Azure Data Lake Storage Gen2 account, to avoid using [staged copy to Snowflake](#staged-copy-to-snowflake).

- The **source data format** is **Parquet**, **Delimited text**, or **JSON** with the following configurations:

    - For **Parquet** format, the compression codec is **None**, or **Snappy**.

    - For **delimited text** format:
        - `rowDelimiter` is **\r\n**, or any single character. If row delimiter isn't “\r\n”, `firstRowAsHeader` need to be **false**, and `skipLineCount` isn't specified.
        - `compression` can be **no compression**, **gzip**, **bzip2**, or **deflate**.
        - `encodingName` is left as default or set to "UTF-8", "UTF-16", "UTF-16BE", "UTF-32", "UTF-32BE", "BIG5", "EUC-JP", "EUC-KR", "GB18030", "ISO-2022-JP", "ISO-2022-KR", "ISO-8859-1", "ISO-8859-2", "ISO-8859-5", "ISO-8859-6", "ISO-8859-7", "ISO-8859-8", "ISO-8859-9", "WINDOWS-1250", "WINDOWS-1251", "WINDOWS-1252", "WINDOWS-1253", "WINDOWS-1254", "WINDOWS-1255".
        - `quoteChar` is **double quote**, **single quote**, or **empty string** (no quote char).
    - For **JSON** format, direct copy only supports the case that sink Snowflake table only has single column and the data type of this column is **VARIANT**, **OBJECT**, or **ARRAY**.
        - `compression` can be **no compression**, **gzip**, **bzip2**, or **deflate**.
        - `encodingName` is left as default or set to **utf-8**.
        - Column mapping isn't specified.

- In the Copy activity source: 

   -  `additionalColumns` isn't specified.
   - If your source is a folder, `recursive` is set to true.
   - `prefix`, `modifiedDateTimeStart`, `modifiedDateTimeEnd`, and `enablePartitionDiscovery` aren't specified.

**Example:**

```json
"activities":[
    {
        "name": "CopyToSnowflake",
        "type": "Copy",
        "inputs": [
            {
                "referenceName": "<input dataset name>",
                "type": "DatasetReference"
            }
        ],
        "outputs": [
            {
                "referenceName": "<Snowflake output dataset name>",
                "type": "DatasetReference"
            }
        ],
        "typeProperties": {
            "source": {
                "type": "<source type>"
            },
            "sink": {
                "type": "SnowflakeV2Sink",
                "importSettings": {
                    "type": "SnowflakeImportCopyCommand",
                    "copyOptions": {
                        "FORCE": "TRUE",
                        "ON_ERROR": "SKIP_FILE"
                    },
                    "fileFormatOptions": {
                        "DATE_FORMAT": "YYYY-MM-DD"
                    },
                    "storageIntegration": "< Snowflake storage integration name >"
                }
            }
        }
    }
]
```

#### Staged copy to Snowflake

When your source data store or format isn't natively compatible with the Snowflake COPY command, as mentioned in the last section, enable the built-in staged copy using an interim Azure Blob storage instance. The staged copy feature also provides you with better throughput. The service automatically converts the data to meet the data format requirements of Snowflake. It then invokes the COPY command to load data into Snowflake. Finally, it cleans up your temporary data from the blob storage. See [Staged copy](copy-activity-performance-features.md#staged-copy) for details about copying data using staging.

To use this feature, create an [Azure Blob storage linked service](connector-azure-blob-storage.md#linked-service-properties) that refers to the Azure storage account as the interim staging. Then specify the `enableStaging` and `stagingSettings` properties in the Copy activity.

- When you specify `storageIntegration` in the sink, the interim staging Azure Blob Storage should be the one that you referred in the external stage in Snowflake. Ensure that you create an [Azure Blob Storage](connector-azure-blob-storage.md) linked service for it with any supported authentication when using the Azure integration runtime, or with anonymous, account key, shared access signature or service principal authentication when using the self-hosted integration runtime. In addition, grant at least **Storage Blob Data Reader** role to the Snowflake service principal in the staging Azure Blob Storage **Access Control (IAM)**. 

- When you don't specify `storageIntegration` in the sink, the staging Azure Blob Storage linked service need to use shared access signature authentication as required by the Snowflake COPY command.

**Example:**

```json
"activities":[
    {
        "name": "CopyToSnowflake",
        "type": "Copy",
        "inputs": [
            {
                "referenceName": "<input dataset name>",
                "type": "DatasetReference"
            }
        ],
        "outputs": [
            {
                "referenceName": "<Snowflake output dataset name>",
                "type": "DatasetReference"
            }
        ],
        "typeProperties": {
            "source": {
                "type": "<source type>"
            },
            "sink": {
                "type": "SnowflakeV2Sink",
                "importSettings": {
                    "type": "SnowflakeImportCopyCommand",
                    "storageIntegration": "< Snowflake storage integration name >"
                }
            },
            "enableStaging": true,
            "stagingSettings": {
                "linkedServiceName": {
                    "referenceName": "MyStagingBlob",
                    "type": "LinkedServiceReference"
                },
                "path": "mystagingpath"
            }
        }
    }
]
```

## Mapping data flow properties

When transforming data in mapping data flow, you can read from and write to tables in Snowflake. For more information, see the [source transformation](data-flow-source.md) and [sink transformation](data-flow-sink.md) in mapping data flows. You can choose to use a Snowflake dataset or an [inline dataset](data-flow-source.md#inline-datasets) as source and sink type.

### Source transformation

The below table lists the properties supported by Snowflake source. You can edit these properties in the **Source options** tab. The connector utilizes Snowflake [internal data transfer](https://docs.snowflake.com/en/user-guide/spark-connector-overview.html#internal-data-transfer).

| Name | Description | Required | Allowed values | Data flow script property |
| ---- | ----------- | -------- | -------------- | ---------------- |
| Table | If you select Table as input, data flow will fetch all the data from the table specified in the Snowflake dataset or in the source options when using inline dataset. | No | String | *(for inline dataset only)*<br>tableName<br>schemaName |
| Query | If you select Query as input, enter a query to fetch data from Snowflake. This setting overrides any table that you've chosen in dataset.<br>If the names of the schema, table and columns contain lower case, quote the object identifier in query e.g. `select * from "schema"."myTable"`. | No | String | query |
| Enable incremental extract (Preview) | Use this option to tell ADF to only process rows that have changed since the last time that the pipeline executed. | No | Boolean | enableCdc |
| Incremental Column | When using the incremental extract feature, you must choose the date/time/numeric column that you wish to use as the watermark in your source table. | No | String | waterMarkColumn |
| Enable Snowflake Change Tracking (Preview) | This option enables ADF to leverage Snowflake change data capture technology to process only the delta data since the previous pipeline execution. This option automatically loads the delta data with row insert, update and deletion operations without requiring any incremental column. | No | Boolean | enableNativeCdc |
| Net Changes | When using snowflake change tracking, you can use this option to get deduped changed rows or exhaustive changes. Deduped changed rows will show only the latest versions of the rows that have changed since a given point in time, while exhaustive changes will show you all the versions of each row that has changed, including the ones that were deleted or updated. For example, if you update a row, you'll see a delete version and an insert version in exhaustive changes, but only the insert version in deduped changed rows. Depending on your use case, you can choose the option that suits your needs. The default option is false, which means exhaustive changes. | No | Boolean | netChanges |
| Include system Columns | When using snowflake change tracking, you can use the systemColumns option to control whether the metadata stream columns provided by Snowflake are included or excluded in the change tracking output. By default, systemColumns is set to true, which means the metadata stream columns are included. You can set systemColumns to false if you want to exclude them. | No | Boolean | systemColumns |
| Start reading from beginning | Setting this option with incremental extract and change tracking will instruct ADF to read all rows on first execution of a pipeline with incremental extract turned on. | No | Boolean | skipInitialLoad |

#### Snowflake source script examples

When you use Snowflake dataset as source type, the associated data flow script is:

```
source(allowSchemaDrift: true,
	validateSchema: false,
	query: 'select * from MYTABLE',
	format: 'query') ~> SnowflakeSource
```

If you use inline dataset, the associated data flow script is:

```
source(allowSchemaDrift: true,
	validateSchema: false,
	format: 'query',
	query: 'select * from MYTABLE',
	store: 'snowflake') ~> SnowflakeSource
```
### Native Change Tracking

Azure Data Factory now supports a native feature in Snowflake known as change tracking, which involves tracking changes in the form of logs. This feature of snowflake allows us to track the changes in the data over time making it useful for incremental data loading and auditing purpose. To utilize this feature, when you enable Change data capture and select the Snowflake Change Tracking, we create a Stream object for the source table that enables change tracking on source snowflake table. Subsequently, we use the CHANGES clause in our query to fetch only the new or updated data from source table. Also, it's recommended to schedule pipeline such that changes are consumed within interval of [data retention time](https://docs.snowflake.com/en/sql-reference/parameters#label-data-retention-time-in-days) set for snowflake source table else user might see inconsistent behavior in captured changes.


### Sink transformation

The below table lists the properties supported by Snowflake sink. You can edit these properties in the **Settings** tab. When using inline dataset, you'll see additional settings, which are the same as the properties described in [dataset properties](#dataset-properties) section. The connector utilizes Snowflake [internal data transfer](https://docs.snowflake.com/en/user-guide/spark-connector-overview.html#internal-data-transfer).

| Name | Description | Required | Allowed values | Data flow script property |
| ---- | ----------- | -------- | -------------- | ---------------- |
| Update method | Specify what operations are allowed on your Snowflake destination.<br>To update, upsert, or delete rows, an [Alter row transformation](data-flow-alter-row.md) is required to tag rows for those actions. | Yes | `true` or `false` | deletable <br/>insertable <br/>updateable <br/>upsertable |
| Key columns | For updates, upserts and deletes, a key column or columns must be set to determine which row to alter. | No | Array | keys |
| Table action | Determines whether to recreate or remove all rows from the destination table prior to writing.<br>- **None**: No action will be done to the table.<br>- **Recreate**: The table will get dropped and recreated. Required if creating a new table dynamically.<br>- **Truncate**: All rows from the target table will get removed. | No | `true` or `false` | recreate<br/>truncate |

#### Snowflake sink script examples

When you use Snowflake dataset as sink type, the associated data flow script is:

```
IncomingStream sink(allowSchemaDrift: true,
	validateSchema: false,
	deletable:true,
	insertable:true,
	updateable:true,
	upsertable:false,
	keys:['movieId'],
	format: 'table',
	skipDuplicateMapInputs: true,
	skipDuplicateMapOutputs: true) ~> SnowflakeSink
```

If you use inline dataset, the associated data flow script is:

```
IncomingStream sink(allowSchemaDrift: true,
	validateSchema: false,
	format: 'table',
	tableName: 'table',
	schemaName: 'schema',
	deletable: true,
	insertable: true,
	updateable: true,
	upsertable: false,
	store: 'snowflake',
	skipDuplicateMapInputs: true,
	skipDuplicateMapOutputs: true) ~> SnowflakeSink
```
#### Query Pushdown optimization
By setting the pipeline Logging Level to None, we exclude the transmission of intermediate transformation metrics, preventing potential hindrances to Spark optimizations and enabling query pushdown optimization provided by Snowflake. This pushdown optimization allows substantial performance enhancements for large Snowflake tables with extensive datasets.


> [!NOTE]
> We don't support temporary tables in Snowflake, as they are local to the session or user who creates them, making them inaccessible to other sessions and prone to being overwritten as regular tables by Snowflake. While Snowflake offers transient tables as an alternative, which are accessible globally, they require manual deletion, contradicting our primary objective of using Temp tables which is to avoid any delete operations in source schema.

## Data type mapping for Snowflake V2

When you copy data from Snowflake, the following mappings are used from Snowflake data types to interim data types within the service internally. To learn about how the copy activity maps the source schema and data type to the sink, see [Schema and data type mappings](copy-activity-schema-and-type-mapping.md).

| Snowflake data type    | Service interim data type |
|--------------------|---------------------------------------------|
| NUMBER (p,0)       | Decimal                                  |
| NUMBER (p,s where s>0) | Decimal                              |
| FLOAT              | Double                                      |
| VARCHAR            | String                                      |
| CHAR               | String                                      |
| BINARY             | Byte[]                                      |
| BOOLEAN            | Boolean                                     |
| DATE               | DateTime                                    |
| TIME               | TimeSpan                                    |
| TIMESTAMP_LTZ      | DateTimeOffset                              |
| TIMESTAMP_NTZ      | DateTimeOffset                              |
| TIMESTAMP_TZ       | DateTimeOffset                              |
| VARIANT            | String                                      |
| OBJECT             | String                                      |
| ARRAY              | String                                      |

## Lookup activity properties

For more information about the properties, see [Lookup activity](control-flow-lookup-activity.md).

## <a name="differences-between-snowflake-and-snowflake-legacy"></a> Snowflake connector lifecycle and upgrade

The following table shows the release stage and change logs for different versions of the Snowflake connector:

| Version  | Release stage | Change log |  
| :----------- | :------- |:------- |
| Snowflake V1 | End of support announced | / |  
| Snowflake V2 (version 1.0) | GA version available | • Add support for Key pair authentication. <br><br>• Add support for `storageIntegration` in Copy activity. <br><br>• The `accountIdentifier`, `warehouse`, `database`, `schema` and `role` properties are used to establish a connection instead of `connectionstring` property.<br><br>• Add support for Decimal in Lookup activity. The NUMBER type, as defined in Snowflake, will be displayed as a string in Lookup activity. If you want to covert it to numeric type in V2, you can use the pipeline parameter with [int function](control-flow-expression-language-functions.md#int) or [float function](control-flow-expression-language-functions.md#float). For example, `int(activity('lookup').output.firstRow.VALUE)`, `float(activity('lookup').output.firstRow.VALUE)`<br><br>• timestamp data type in Snowflake is read as DateTimeOffset data type in Lookup and Script activity. If you still need to use the Datetime value as a parameter in your pipeline after upgrading to V2, you can convert DateTimeOffset type to DateTime type by using [formatDateTime function](control-flow-expression-language-functions.md#formatdatetime) (recommended) or [concat function](control-flow-expression-language-functions.md#concat). For example: `formatDateTime(activity('lookup').output.firstRow.DATETIMETYPE)`, `concat(substring(activity('lookup').output.firstRow.DATETIMETYPE, 0, 19), 'Z')` <br><br>• NUMBER (p,0) is read as Decimal data type.<br><br>• TIMESTAMP_LTZ, TIMESTAMP_NTZ and TIMESTAMP_TZ is read as DateTimeOffset data type.<br><br>• Script parameters are not supported in Script activity. As an alternative, utilize dynamic expressions for script parameters. For more information, see [Expressions and functions in Azure Data Factory and Azure Synapse Analytics](control-flow-expression-language-functions.md).<br><br>• Multiple SQL statements execution in Script activity is not supported. |  
| Snowflake V2 (version 1.1) | Preview version available | • Add support for script parameters.<br><br>• Add support for multiple statement execution in Script activity. |  

### <a name="upgrade-the-snowflake-linked-service"></a> Upgrade the Snowflake connector from V1 to V2

To upgrade the Snowflake connector from V1 to V2, you can do a side-by-side upgrade, or an in-place upgrade.

#### Side-by-side upgrade

To perform a side-by-side upgrade, complete the following steps:

1. Create a new Snowflake linked service and configure it by referring to the V2 linked service properties.  
1. Create a dataset based on the newly created Snowflake linked service.
1. Replace the new linked service and dataset with the existing ones in the pipelines that targets the V1 objects. 

#### In-place upgrade

To perform an in-place upgrade, you need to edit the existing linked service payload and update dataset to use the new linked service.

1. Update the type from **Snowflake** to **SnowflakeV2**.
1. Modify the linked service payload from its V1 format to V2. You can either fill in each field from the user interface after changing the type mentioned above, or update the payload directly through the JSON Editor. Refer to the [Linked service properties](#linked-service-properties) section in this article for the supported connection properties. The following examples show the differences in payload for the V1 and V2 Snowflake linked services:

   **Snowflake V1 linked service JSON payload:**
   ```json
     {
        "name": "Snowflake1",
        "type": "Microsoft.DataFactory/factories/linkedservices",
        "properties": {
            "annotations": [],
            "type": "Snowflake",
            "typeProperties": {
                "authenticationType": "Basic",
                "connectionString": "jdbc:snowflake://<fake_account>.snowflakecomputing.com/?user=FAKE_USER&db=FAKE_DB&warehouse=FAKE_DW&schema=PUBLIC",
                "encryptedCredential": "<your_encrypted_credential_value>"
            },
            "connectVia": {
                "referenceName": "AzureIntegrationRuntime",
                "type": "IntegrationRuntimeReference"
            }
        }
    }
   ```

   **Snowflake V2 linked service JSON payload:**
   ```json
    {
        "name": "Snowflake2",
        "type": "Microsoft.DataFactory/factories/linkedservices",
        "properties": {
            "parameters": {
                "schema": {
                    "type": "string",
                    "defaultValue": "PUBLIC"
                }
            },
            "annotations": [],
            "type": "SnowflakeV2",
            "typeProperties": {
                "authenticationType": "Basic",
                "accountIdentifier": "<FAKE_Account>",
                "user": "FAKE_USER",
                "database": "FAKE_DB",
                "warehouse": "FAKE_DW",
                "encryptedCredential": "<placeholder>"
            },
            "connectVia": {
                "referenceName": "AutoResolveIntegrationRuntime",
                "type": "IntegrationRuntimeReference"
            }
        }
    }
   ```

1. Update dataset to use the new linked service. You can either create a new dataset based on the newly created linked service, or update an existing dataset's type property from **SnowflakeTable** to **SnowflakeV2Table**.

>[!NOTE]
>When transitioning linked services, the override template parameter section might only display database properties. You can resolve this by manually editing the parameters. After that the **Override template parameters** section will show the connection strings.

### Upgrade the Snowflake V2 connector from version 1.0 to version 1.1 (Preview)

In **Edit linked service** page, select 1.1 for version. For more information, see [Linked service properties](#linked-service-properties).

## Related content

For a list of data stores supported as sources and sinks by Copy activity, see [supported data stores and formats](copy-activity-overview.md#supported-data-stores-and-formats).
