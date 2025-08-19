# Ed-Fi Data Management Service (DMS) Relational-First Design with JSON Flattening

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [Core Architecture Principles](#core-architecture-principles)
3. [Metadata Schema Design](#metadata-schema-design)
4. [Application-Side JSON Flattening](#application-side-json-flattening)
5. [Database-Side JSON Flattening](#database-side-json-flattening)
6. [Relational Schema Design Patterns](#relational-schema-design-patterns)
7. [Database-Side JSON Generation](#database-side-json-generation)
8. [Performance Architecture](#performance-architecture)
9. [Implementation Roadmap](#implementation-roadmap)
10. [Technical Appendices](#technical-appendices)

---
Yeah, OK.
## Executive Overview

The Ed-Fi Data Management Service (DMS) API requires a sophisticated approach to handling diverse JSON document structures while maintaining high performance and flexibility. This design document presents a metadata-driven architecture that enables dynamic JSON-to-relational flattening and relational-to-JSON reconstruction without requiring code generation.

The core innovation is treating the mapping logic as **data** (metadata) rather than **code**, creating a generic engine that interprets metadata files to perform bidirectional transformations between JSON documents and relational database structures. This approach is particularly suited to the Ed-Fi DMS, which is inherently metadata-driven with endpoints and document shapes declared through metadata.

### Key Architectural Decisions

1. **Metadata-Driven Processing**: All JSON-to-relational and relational-to-JSON transformations are governed by declarative metadata files
2. **Database-Side Operations**: Leverage native database JSON capabilities for maximum performance
3. **Consistent Naming Conventions**: Establish predictable patterns that simplify SQL generation
4. **No Code Generation**: Runtime dynamic processing eliminates the need for compile-time code generation

### Technology Stack

- **Application Layer**: C# with .NET 8.0
- **JSON Processing**: Newtonsoft.Json (Json.NET) for dynamic traversal
- **Database Access**: Dapper micro-ORM for high-performance database operations
- **Databases**: SQL Server (primary) and PostgreSQL (secondary) with native JSON support
- **Metadata Format**: JSON-based configuration files

---

## Core Architecture Principles

### 1. The Metadata-Driven Interpreter Pattern

The application functions as a generic engine that performs these key operations:

1. **Metadata Loading**: Read and parse mapping documents that describe JSON-to-relational transformations
2. **Dynamic JSON Parsing**: Parse incoming JSON documents into traversable structures without predefined types
3. **Rule Application**: Apply mapping rules from metadata to extract values from JSON
4. **SQL Generation**: Dynamically construct and execute SQL statements (`CREATE TABLE`, `INSERT`, `SELECT`)

### 2. Bidirectional Transformation Support

The same metadata supports both directions of data flow:

- **Write Path (JSON → Relational)**: Decompose nested JSON documents into normalized relational tables
- **Read Path (Relational → JSON)**: Reconstruct nested JSON documents from relational data

### 3. Database-First Philosophy

Leverage the database's native JSON capabilities wherever possible:

- Use database-side JSON parsing for writes (`OPENJSON` in SQL Server, `jsonb_populate_record` in PostgreSQL)
- Use database-side JSON generation for reads (`FOR JSON` in SQL Server, `json_build_object` in PostgreSQL)

### 4. Performance by Design

- Minimize network round-trips through batch operations
- Use database-side processing to reduce data transfer
- Implement pagination to handle large datasets
- Optimize query patterns based on relationship cardinality

---

## Metadata Schema Design

The metadata schema is the cornerstone of the system, describing how JSON documents map to relational structures.

### Metadata Structure

```json
{
  "schemaVersion": "1.0",
  "documentType": "Order",
  "entities": [
    {
      "tableName": "Orders",
      "jsonPath": "$.order",
      "properties": [
        {
          "columnName": "OrderId",
          "dataType": "NVARCHAR(50)",
          "jsonPath": "$.id",
          "isPrimaryKey": true
        },
        {
          "columnName": "OrderDate",
          "dataType": "DATETIME",
          "jsonPath": "$.date"
        },
        {
          "columnName": "CustomerDetails_Name",
          "dataType": "NVARCHAR(255)",
          "jsonPath": "$.customer_details.name"
        },
        {
          "columnName": "CustomerDetails_Email",
          "dataType": "NVARCHAR(255)",
          "jsonPath": "$.customer_details.email"
        },
        {
          "columnName": "EventId",
          "dataType": "NVARCHAR(50)",
          "jsonPath": "$.purchase_event_id",
          "isGlobal": true
        }
      ]
    },
    {
      "tableName": "OrderItems",
      "jsonPath": "$.order.items",
      "properties": [
        {
          "columnName": "OrderItemId",
          "dataType": "INT IDENTITY(1,1)",
          "isPrimaryKey": true,
          "isAutoGenerated": true
        },
        {
          "columnName": "ProductSKU",
          "dataType": "NVARCHAR(50)",
          "jsonPath": "$.sku"
        },
        {
          "columnName": "Quantity",
          "dataType": "INT",
          "jsonPath": "$.quantity"
        },
        {
          "columnName": "UnitPrice",
          "dataType": "DECIMAL(18, 2)",
          "jsonPath": "$.price"
        }
      ],
      "relationships": [
        {
          "parentTable": "Orders",
          "foreignKeyColumn": "OrderId",
          "parentPrimaryKey": "OrderId",
          "cascadeDelete": true
        }
      ]
    }
  ],
  "indexingStrategy": {
    "primaryIndexes": ["OrderId", "OrderItemId"],
    "secondaryIndexes": [
      {
        "tableName": "Orders",
        "columns": ["OrderDate", "CustomerDetails_Name"]
      }
    ]
  }
}
```

### Metadata Elements Explained

#### Entity Definition
- **tableName**: The target relational table name
- **jsonPath**: JSONPath expression to locate the data in the source JSON
- **properties**: Array of column definitions
- **relationships**: Foreign key relationships to parent tables

#### Property Definition
- **columnName**: The database column name
- **dataType**: SQL data type specification
- **jsonPath**: Relative JSONPath from the entity's context
- **isPrimaryKey**: Identifies primary key columns
- **isAutoGenerated**: Indicates database-generated values
- **isGlobal**: Pulls value from a higher level in the JSON tree

#### Relationship Definition
- **parentTable**: Referenced parent table name
- **foreignKeyColumn**: Column name for the foreign key
- **parentPrimaryKey**: Primary key column in parent table
- **cascadeDelete**: Enable cascade delete operations

---

## Application-Side JSON Flattening

### Implementation Architecture

The application-side flattening engine consists of several key components:

#### 1. JSON Document Parser

```csharp
public class JsonDocumentParser
{
    private readonly JObject _rootDocument;

    public JsonDocumentParser(string jsonString)
    {
        _rootDocument = JObject.Parse(jsonString);
    }

    public JToken SelectToken(string jsonPath)
    {
        return _rootDocument.SelectToken(jsonPath);
    }

    public IEnumerable<JToken> SelectTokens(string jsonPath)
    {
        var token = _rootDocument.SelectToken(jsonPath);
        if (token is JArray array)
            return array;
        return token != null ? new[] { token } : Enumerable.Empty<JToken>();
    }
}
```

#### 2. Metadata-Driven Processor

```csharp
public class MetadataProcessor
{
    private readonly MetadataSchema _metadata;
    private readonly IDbConnection _connection;

    public async Task ProcessJsonDocument(string jsonDocument)
    {
        var parser = new JsonDocumentParser(jsonDocument);
        var transaction = _connection.BeginTransaction();

        try
        {
            // Process entities in dependency order
            var sortedEntities = SortEntitiesByDependency(_metadata.Entities);
            var primaryKeyMap = new Dictionary<string, object>();

            foreach (var entity in sortedEntities)
            {
                await ProcessEntity(entity, parser, primaryKeyMap, transaction);
            }

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }

    private async Task ProcessEntity(
        EntityDefinition entity,
        JsonDocumentParser parser,
        Dictionary<string, object> primaryKeyMap,
        IDbTransaction transaction)
    {
        var entityTokens = parser.SelectTokens(entity.JsonPath);

        foreach (var token in entityTokens)
        {
            var parameters = new DynamicParameters();

            // Extract property values
            foreach (var property in entity.Properties)
            {
                if (property.IsAutoGenerated)
                    continue;

                var value = ExtractPropertyValue(token, property, parser);
                parameters.Add(property.ColumnName, value);
            }

            // Add foreign key values
            foreach (var relationship in entity.Relationships)
            {
                var parentKey = primaryKeyMap[relationship.ParentTable];
                parameters.Add(relationship.ForeignKeyColumn, parentKey);
            }

            // Build and execute INSERT statement
            var sql = BuildInsertStatement(entity);
            var insertedId = await _connection.QuerySingleAsync<object>(
                sql, parameters, transaction);

            // Store primary key for child relationships
            var primaryKey = entity.Properties.First(p => p.IsPrimaryKey);
            primaryKeyMap[$"{entity.TableName}"] = insertedId;
        }
    }

    private object ExtractPropertyValue(
        JToken entityToken,
        PropertyDefinition property,
        JsonDocumentParser parser)
    {
        JToken valueToken;

        if (property.IsGlobal)
        {
            // Global properties use absolute path from root
            valueToken = parser.SelectToken(property.JsonPath);
        }
        else
        {
            // Local properties use relative path from entity
            valueToken = entityToken.SelectToken(property.JsonPath);
        }

        return ConvertToSqlType(valueToken, property.DataType);
    }
}
```

#### 3. Dynamic SQL Generation

```csharp
public class SqlGenerator
{
    public string BuildInsertStatement(EntityDefinition entity)
    {
        var columns = entity.Properties
            .Where(p => !p.IsAutoGenerated)
            .Select(p => p.ColumnName)
            .Concat(entity.Relationships.Select(r => r.ForeignKeyColumn));

        var parameters = columns.Select(c => $"@{c}");

        var sql = $@"
            INSERT INTO {entity.TableName} ({string.Join(", ", columns)})
            VALUES ({string.Join(", ", parameters)})";

        // Add RETURNING clause for primary key retrieval
        var primaryKey = entity.Properties.FirstOrDefault(p => p.IsPrimaryKey);
        if (primaryKey != null && !primaryKey.IsAutoGenerated)
        {
            sql += $" RETURNING {primaryKey.ColumnName}";
        }
        else if (primaryKey != null && primaryKey.IsAutoGenerated)
        {
            sql += "; SELECT SCOPE_IDENTITY()"; // SQL Server
            // sql += " RETURNING " + primaryKey.ColumnName; // PostgreSQL
        }

        return sql;
    }

    public string BuildCreateTableStatement(EntityDefinition entity)
    {
        var columns = new List<string>();

        foreach (var property in entity.Properties)
        {
            var columnDef = $"{property.ColumnName} {property.DataType}";

            if (property.IsPrimaryKey)
                columnDef += " PRIMARY KEY";

            columns.Add(columnDef);
        }

        // Add foreign key columns
        foreach (var relationship in entity.Relationships)
        {
            var fkDef = $@"
                {relationship.ForeignKeyColumn} {GetForeignKeyDataType(relationship)},
                CONSTRAINT FK_{entity.TableName}_{relationship.ParentTable}
                FOREIGN KEY ({relationship.ForeignKeyColumn})
                REFERENCES {relationship.ParentTable}({relationship.ParentPrimaryKey})";

            if (relationship.CascadeDelete)
                fkDef += " ON DELETE CASCADE";

            columns.Add(fkDef);
        }

        return $@"
            CREATE TABLE {entity.TableName} (
                {string.Join(",\n                ", columns)}
            )";
    }
}
```

### Error Handling and Validation

```csharp
public class ValidationEngine
{
    public ValidationResult ValidateJsonAgainstMetadata(
        string jsonDocument,
        MetadataSchema metadata)
    {
        var result = new ValidationResult();
        var parser = new JsonDocumentParser(jsonDocument);

        foreach (var entity in metadata.Entities)
        {
            // Verify entity path exists
            var entityToken = parser.SelectToken(entity.JsonPath);
            if (entityToken == null && !entity.IsOptional)
            {
                result.AddError($"Required entity path not found: {entity.JsonPath}");
                continue;
            }

            // Validate required properties
            foreach (var property in entity.Properties.Where(p => p.IsRequired))
            {
                var valueToken = entityToken?.SelectToken(property.JsonPath);
                if (valueToken == null || valueToken.Type == JTokenType.Null)
                {
                    result.AddError($"Required property missing: {property.JsonPath}");
                }
            }

            // Validate data types
            foreach (var property in entity.Properties)
            {
                var valueToken = entityToken?.SelectToken(property.JsonPath);
                if (valueToken != null && !IsValidDataType(valueToken, property.DataType))
                {
                    result.AddError($"Invalid data type for {property.JsonPath}");
                }
            }
        }

        return result;
    }
}
```

---

## Database-Side JSON Flattening

### SQL Server Implementation

#### Stored Procedure for JSON Import

```sql
CREATE OR ALTER PROCEDURE dbo.sp_ImportJsonDocument
    @DocumentType NVARCHAR(100),
    @JsonPayload NVARCHAR(MAX),
    @MetadataJson NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Parse metadata if provided, otherwise use cached metadata
        DECLARE @Metadata TABLE (
            EntityName NVARCHAR(100),
            TableName NVARCHAR(100),
            JsonPath NVARCHAR(500),
            EntityOrder INT
        );

        IF @MetadataJson IS NOT NULL
        BEGIN
            INSERT INTO @Metadata
            SELECT
                EntityName = JSON_VALUE(value, '$.entityName'),
                TableName = JSON_VALUE(value, '$.tableName'),
                JsonPath = JSON_VALUE(value, '$.jsonPath'),
                EntityOrder = JSON_VALUE(value, '$.order')
            FROM OPENJSON(@MetadataJson, '$.entities');
        END
        ELSE
        BEGIN
            -- Load from metadata cache table
            INSERT INTO @Metadata
            SELECT EntityName, TableName, JsonPath, EntityOrder
            FROM dbo.MetadataCache
            WHERE DocumentType = @DocumentType
            ORDER BY EntityOrder;
        END;

        -- Dynamic SQL to process each entity
        DECLARE @EntityCursor CURSOR;
        DECLARE @TableName NVARCHAR(100);
        DECLARE @JsonPath NVARCHAR(500);
        DECLARE @DynamicSQL NVARCHAR(MAX);

        SET @EntityCursor = CURSOR FOR
            SELECT TableName, JsonPath
            FROM @Metadata
            ORDER BY EntityOrder;

        OPEN @EntityCursor;
        FETCH NEXT FROM @EntityCursor INTO @TableName, @JsonPath;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- Build dynamic INSERT statement based on table structure
            SET @DynamicSQL = N'
                INSERT INTO ' + QUOTENAME(@TableName) + '
                SELECT * FROM OPENJSON(@JsonData, ''' + @JsonPath + ''')
                WITH (
                    -- Column mappings generated from metadata
                    ' + dbo.fn_GenerateOpenJsonSchema(@TableName, @DocumentType) + '
                )';

            EXEC sp_executesql @DynamicSQL,
                N'@JsonData NVARCHAR(MAX)',
                @JsonData = @JsonPayload;

            FETCH NEXT FROM @EntityCursor INTO @TableName, @JsonPath;
        END;

        CLOSE @EntityCursor;
        DEALLOCATE @EntityCursor;

        COMMIT TRANSACTION;

        -- Return success with inserted record counts
        SELECT
            'Success' AS Status,
            @@ROWCOUNT AS RecordsInserted;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        THROW;
    END CATCH;
END;
GO
```

#### Helper Function for Schema Generation

```sql
CREATE OR ALTER FUNCTION dbo.fn_GenerateOpenJsonSchema
(
    @TableName NVARCHAR(100),
    @DocumentType NVARCHAR(100)
)
RETURNS NVARCHAR(MAX)
AS
BEGIN
    DECLARE @Schema NVARCHAR(MAX) = '';

    SELECT @Schema = @Schema +
        CASE WHEN @Schema = '' THEN '' ELSE ',' + CHAR(13) + CHAR(10) END +
        '    ' + mc.ColumnName + ' ' + mc.SqlDataType + ' ''' + mc.JsonPath + ''''
    FROM dbo.MetadataColumns mc
    WHERE mc.TableName = @TableName
      AND mc.DocumentType = @DocumentType
    ORDER BY mc.ColumnOrder;

    RETURN @Schema;
END;
GO
```

### PostgreSQL Implementation

#### Function for JSON Import

```sql
-- Create custom types for structured import
CREATE TYPE import_context AS (
    document_type TEXT,
    json_payload JSONB,
    metadata JSONB
);

-- Main import function
CREATE OR REPLACE FUNCTION import_json_document(
    p_document_type TEXT,
    p_json_payload JSONB,
    p_metadata JSONB DEFAULT NULL
)
RETURNS TABLE(status TEXT, records_inserted INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_entity RECORD;
    v_table_name TEXT;
    v_json_path TEXT[];
    v_insert_sql TEXT;
    v_total_inserted INTEGER := 0;
    v_entity_inserted INTEGER;
BEGIN
    -- Start transaction is implicit in PostgreSQL functions

    -- Process entities in dependency order
    FOR v_entity IN
        SELECT
            e->>'tableName' AS table_name,
            e->>'jsonPath' AS json_path,
            e->'properties' AS properties,
            e->'relationships' AS relationships,
            (e->>'order')::INTEGER AS entity_order
        FROM jsonb_array_elements(
            COALESCE(p_metadata->'entities',
                     (SELECT metadata FROM metadata_cache
                      WHERE document_type = p_document_type)->'entities')
        ) AS e
        ORDER BY (e->>'order')::INTEGER
    LOOP
        -- Parse the JSON path into an array
        v_json_path := string_to_array(
            trim(both '$.' from v_entity.json_path), '.'
        );

        -- Build dynamic INSERT statement
        v_insert_sql := format(
            'INSERT INTO %I SELECT * FROM jsonb_populate_recordset(
                null::%I,
                $1#>%L
            )',
            v_entity.table_name,
            v_entity.table_name,
            v_json_path
        );

        -- Execute the INSERT and get row count
        EXECUTE v_insert_sql USING p_json_payload;
        GET DIAGNOSTICS v_entity_inserted = ROW_COUNT;
        v_total_inserted := v_total_inserted + v_entity_inserted;
    END LOOP;

    -- Return success status
    RETURN QUERY
    SELECT 'Success'::TEXT, v_total_inserted;

EXCEPTION
    WHEN OTHERS THEN
        -- Rollback is automatic on exception
        RAISE;
END;
$$;
```

#### Advanced Type Mapping Function

```sql
-- Function to handle complex type conversions
CREATE OR REPLACE FUNCTION convert_json_to_sql_type(
    p_json_value JSONB,
    p_sql_type TEXT
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN CASE
        WHEN p_sql_type ILIKE '%INT%' THEN (p_json_value#>>'{}')::INTEGER::TEXT
        WHEN p_sql_type ILIKE '%NUMERIC%' OR p_sql_type ILIKE '%DECIMAL%'
            THEN (p_json_value#>>'{}')::NUMERIC::TEXT
        WHEN p_sql_type ILIKE '%BOOL%' THEN (p_json_value#>>'{}')::BOOLEAN::TEXT
        WHEN p_sql_type ILIKE '%DATE%' OR p_sql_type ILIKE '%TIME%'
            THEN (p_json_value#>>'{}')::TIMESTAMP::TEXT
        WHEN p_sql_type ILIKE '%JSON%' THEN p_json_value::TEXT
        ELSE p_json_value#>>'{}'
    END;
END;
$$;
```

---

## Relational Schema Design Patterns

### Core Design Principles

#### 1. JSON Structure Mapping

| JSON Pattern | Relational Pattern | Implementation |
|--------------|-------------------|----------------|
| Root Object | Parent Table | Main entity table |
| Nested Object (1:1) | Flattened Columns | Columns with prefix naming |
| Array of Objects (1:N) | Child Table | Separate table with foreign key |
| Array of Primitives | Junction Table or JSON Column | Depends on query requirements |
| Deep Nesting | Multiple Related Tables | Chain of foreign key relationships |

#### 2. Naming Convention Standards

##### Primary Keys
- **Pattern**: `[TableName]Id`
- **Examples**: `OrderId`, `OrderItemId`, `StudentId`
- **Rationale**: Unambiguous identification, simplifies JOIN conditions

##### Foreign Keys
- **Pattern**: Exact match to referenced primary key
- **Examples**: `OrderId` in `OrderItems` table references `Orders.OrderId`
- **Rationale**: Enables automatic JOIN inference, reduces metadata complexity

##### Flattened Object Properties
- **Pattern**: `[ObjectName]_[PropertyName]`
- **Examples**: `CustomerDetails_Name`, `CustomerDetails_Email`, `Address_Street`
- **Rationale**: Preserves hierarchical information, enables automatic JSON reconstruction

##### Arrays and Collections
- **Pattern**: Pluralized child table names
- **Examples**: `Orders` → `OrderItems`, `Students` → `StudentEnrollments`
- **Rationale**: Clear relationship indication, follows standard database conventions

### Schema Examples

#### Example 1: Educational Domain (Ed-Fi Standard)

```sql
-- Student entity (root object)
CREATE TABLE Students (
    StudentId NVARCHAR(50) PRIMARY KEY,
    StudentUniqueId NVARCHAR(100) NOT NULL,
    FirstName NVARCHAR(100) NOT NULL,
    LastSurname NVARCHAR(100) NOT NULL,
    BirthDate DATE NOT NULL,
    BirthCity NVARCHAR(100),
    BirthStateAbbreviation CHAR(2),
    -- Flattened demographic object
    Demographics_HispanicLatinoEthnicity BIT,
    Demographics_Sex NVARCHAR(20),
    -- Flattened contact object
    ContactInfo_Email NVARCHAR(255),
    ContactInfo_MobilePhone NVARCHAR(20),
    CreatedDate DATETIME2 DEFAULT GETUTCDATE(),
    LastModifiedDate DATETIME2 DEFAULT GETUTCDATE()
);

-- Student addresses (array of objects)
CREATE TABLE StudentAddresses (
    StudentAddressId INT IDENTITY(1,1) PRIMARY KEY,
    StudentId NVARCHAR(50) NOT NULL,
    AddressTypeDescriptor NVARCHAR(100) NOT NULL,
    StreetNumberName NVARCHAR(255) NOT NULL,
    City NVARCHAR(100) NOT NULL,
    StateAbbreviation CHAR(2) NOT NULL,
    PostalCode NVARCHAR(20),
    CONSTRAINT FK_StudentAddresses_Students
        FOREIGN KEY (StudentId) REFERENCES Students(StudentId) ON DELETE CASCADE
);

-- Student races (many-to-many relationship)
CREATE TABLE StudentRaces (
    StudentRaceId INT IDENTITY(1,1) PRIMARY KEY,
    StudentId NVARCHAR(50) NOT NULL,
    RaceDescriptor NVARCHAR(100) NOT NULL,
    CONSTRAINT FK_StudentRaces_Students
        FOREIGN KEY (StudentId) REFERENCES Students(StudentId) ON DELETE CASCADE,
    CONSTRAINT UQ_StudentRaces UNIQUE (StudentId, RaceDescriptor)
);

-- Indexes for performance
CREATE INDEX IX_Students_StudentUniqueId ON Students(StudentUniqueId);
CREATE INDEX IX_Students_LastSurname_FirstName ON Students(LastSurname, FirstName);
CREATE INDEX IX_StudentAddresses_StudentId ON StudentAddresses(StudentId);
CREATE INDEX IX_StudentRaces_StudentId ON StudentRaces(StudentId);
```

#### Example 2: Assessment Domain

```sql
-- Assessment entity (root object)
CREATE TABLE Assessments (
    AssessmentId NVARCHAR(50) PRIMARY KEY,
    AssessmentIdentifier NVARCHAR(100) NOT NULL,
    Namespace NVARCHAR(255) NOT NULL,
    Title NVARCHAR(500) NOT NULL,
    Version INT,
    Category NVARCHAR(100),
    -- Flattened period object
    Period_BeginDate DATE,
    Period_EndDate DATE,
    -- Flattened metadata
    Metadata_CreatedBy NVARCHAR(100),
    Metadata_CreatedDate DATETIME2,
    RevisionDate DATE,
    MaxRawScore DECIMAL(10, 2),
    CONSTRAINT UQ_Assessment_Identity UNIQUE (AssessmentIdentifier, Namespace)
);

-- Assessment academic subjects (array)
CREATE TABLE AssessmentAcademicSubjects (
    AssessmentAcademicSubjectId INT IDENTITY(1,1) PRIMARY KEY,
    AssessmentId NVARCHAR(50) NOT NULL,
    AcademicSubjectDescriptor NVARCHAR(100) NOT NULL,
    CONSTRAINT FK_AssessmentAcademicSubjects_Assessments
        FOREIGN KEY (AssessmentId) REFERENCES Assessments(AssessmentId) ON DELETE CASCADE
);

-- Assessment scores (nested array)
CREATE TABLE AssessmentScores (
    AssessmentScoreId INT IDENTITY(1,1) PRIMARY KEY,
    AssessmentId NVARCHAR(50) NOT NULL,
    MinimumScore DECIMAL(10, 2),
    MaximumScore DECIMAL(10, 2),
    ResultDataType NVARCHAR(50),
    -- Flattened reporting method
    ReportingMethod_AssessmentReportingMethodDescriptor NVARCHAR(100),
    CONSTRAINT FK_AssessmentScores_Assessments
        FOREIGN KEY (AssessmentId) REFERENCES Assessments(AssessmentId) ON DELETE CASCADE
);
```

### Handling Complex Scenarios

#### Polymorphic Relationships

When JSON documents contain polymorphic fields (different object types in the same field):

```sql
-- Use a discriminator pattern
CREATE TABLE EducationOrganizations (
    EducationOrganizationId INT PRIMARY KEY,
    OrganizationType NVARCHAR(50) NOT NULL, -- 'School', 'District', 'State'
    Name NVARCHAR(255) NOT NULL,
    -- Common fields
    OperationalStatus NVARCHAR(50),
    WebSite NVARCHAR(500),
    -- Type-specific fields (nullable)
    School_SchoolType NVARCHAR(50),
    School_GradeLevels NVARCHAR(MAX), -- JSON array stored as string
    District_StateOrganizationId INT,
    State_StateAbbreviation CHAR(2)
);
```

#### Recursive Relationships

For self-referential JSON structures:

```sql
-- Organizational hierarchy
CREATE TABLE OrganizationUnits (
    OrganizationUnitId NVARCHAR(50) PRIMARY KEY,
    ParentOrganizationUnitId NVARCHAR(50),
    Name NVARCHAR(255) NOT NULL,
    Level INT NOT NULL,
    Path NVARCHAR(MAX), -- Materialized path for efficient queries
    CONSTRAINT FK_OrganizationUnits_Parent
        FOREIGN KEY (ParentOrganizationUnitId)
        REFERENCES OrganizationUnits(OrganizationUnitId)
);

-- Create hierarchical index
CREATE INDEX IX_OrganizationUnits_Path ON OrganizationUnits(Path);
```

---

## Database-Side JSON Generation

### SQL Server Implementation

#### Basic JSON Generation Pattern

```sql
-- Single entity with nested objects and arrays
CREATE OR ALTER PROCEDURE sp_GetStudentJson
    @StudentId NVARCHAR(50)
AS
BEGIN
    SELECT
        s.StudentId AS 'id',
        s.StudentUniqueId AS 'studentUniqueId',
        s.FirstName AS 'firstName',
        s.LastSurname AS 'lastSurname',
        s.BirthDate AS 'birthDate',
        -- Nested demographics object
        s.Demographics_HispanicLatinoEthnicity AS 'demographics.hispanicLatinoEthnicity',
        s.Demographics_Sex AS 'demographics.sex',
        -- Nested contact info
        s.ContactInfo_Email AS 'contactInfo.email',
        s.ContactInfo_MobilePhone AS 'contactInfo.mobilePhone',
        -- Addresses array (subquery)
        (
            SELECT
                sa.AddressTypeDescriptor AS 'addressTypeDescriptor',
                sa.StreetNumberName AS 'streetNumberName',
                sa.City AS 'city',
                sa.StateAbbreviation AS 'stateAbbreviation',
                sa.PostalCode AS 'postalCode'
            FROM StudentAddresses sa
            WHERE sa.StudentId = s.StudentId
            FOR JSON PATH
        ) AS 'addresses',
        -- Races array (subquery)
        (
            SELECT
                sr.RaceDescriptor AS 'raceDescriptor'
            FROM StudentRaces sr
            WHERE sr.StudentId = s.StudentId
            FOR JSON PATH
        ) AS 'races'
    FROM Students s
    WHERE s.StudentId = @StudentId
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END;
GO
```

#### Advanced Pattern with Multiple Levels

```sql
-- Complex nested structure with multiple relationship levels
CREATE OR ALTER PROCEDURE sp_GetAssessmentHierarchyJson
    @AssessmentId NVARCHAR(50)
AS
BEGIN
    SELECT
        a.AssessmentId AS 'assessmentId',
        a.AssessmentIdentifier AS 'assessmentIdentifier',
        a.Namespace AS 'namespace',
        a.Title AS 'title',
        -- Period object
        JSON_QUERY(
            (SELECT
                a.Period_BeginDate AS 'beginDate',
                a.Period_EndDate AS 'endDate'
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
        ) AS 'period',
        -- Academic subjects array
        JSON_QUERY(
            (SELECT
                aas.AcademicSubjectDescriptor AS 'academicSubjectDescriptor'
            FROM AssessmentAcademicSubjects aas
            WHERE aas.AssessmentId = a.AssessmentId
            FOR JSON PATH)
        ) AS 'academicSubjects',
        -- Scores with nested reporting methods
        JSON_QUERY(
            (SELECT
                ascore.MinimumScore AS 'minimumScore',
                ascore.MaximumScore AS 'maximumScore',
                ascore.ResultDataType AS 'resultDataType',
                JSON_QUERY(
                    (SELECT
                        ascore.ReportingMethod_AssessmentReportingMethodDescriptor AS 'assessmentReportingMethodDescriptor'
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
                ) AS 'reportingMethod'
            FROM AssessmentScores ascore
            WHERE ascore.AssessmentId = a.AssessmentId
            FOR JSON PATH)
        ) AS 'scores'
    FROM Assessments a
    WHERE a.AssessmentId = @AssessmentId
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END;
GO
```

#### Metadata-Driven Dynamic Generation

```sql
CREATE OR ALTER PROCEDURE sp_GenerateJsonDynamically
    @TableName NVARCHAR(100),
    @PrimaryKeyValue NVARCHAR(100),
    @MetadataId INT
AS
BEGIN
    DECLARE @Sql NVARCHAR(MAX);
    DECLARE @SelectClause NVARCHAR(MAX) = '';
    DECLARE @FromClause NVARCHAR(MAX);
    DECLARE @WhereClause NVARCHAR(MAX);

    -- Build SELECT clause from metadata
    SELECT @SelectClause = @SelectClause +
        CASE
            WHEN mc.JsonPath LIKE '%.%' THEN
                mc.ColumnName + ' AS ''' + mc.JsonPath + ''','
            ELSE
                mc.ColumnName + ' AS ''' + mc.JsonPropertyName + ''','
        END
    FROM MetadataColumns mc
    WHERE mc.TableName = @TableName
      AND mc.MetadataId = @MetadataId
    ORDER BY mc.ColumnOrder;

    -- Add subqueries for child relationships
    DECLARE @ChildQueries NVARCHAR(MAX) = '';
    SELECT @ChildQueries = @ChildQueries + '
        (' + cr.SubqueryTemplate + ') AS ''' + cr.JsonPropertyName + ''','
    FROM MetadataChildRelationships cr
    WHERE cr.ParentTable = @TableName
      AND cr.MetadataId = @MetadataId;

    -- Combine into final query
    SET @Sql = '
        SELECT ' + @SelectClause + @ChildQueries + '
        FROM ' + QUOTENAME(@TableName) + '
        WHERE ' + QUOTENAME(@TableName + 'Id') + ' = @KeyValue
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER';

    -- Execute dynamic SQL
    EXEC sp_executesql @Sql, N'@KeyValue NVARCHAR(100)', @KeyValue = @PrimaryKeyValue;
END;
GO
```

### PostgreSQL Implementation

#### Basic JSON Generation Pattern

```sql
-- Function for single entity JSON generation
CREATE OR REPLACE FUNCTION get_student_json(p_student_id TEXT)
RETURNS JSON
LANGUAGE sql
AS $$
    SELECT json_build_object(
        'id', s.student_id,
        'studentUniqueId', s.student_unique_id,
        'firstName', s.first_name,
        'lastSurname', s.last_surname,
        'birthDate', s.birth_date,
        'demographics', json_build_object(
            'hispanicLatinoEthnicity', s.demographics_hispanic_latino_ethnicity,
            'sex', s.demographics_sex
        ),
        'contactInfo', json_build_object(
            'email', s.contact_info_email,
            'mobilePhone', s.contact_info_mobile_phone
        ),
        'addresses', (
            SELECT json_agg(
                json_build_object(
                    'addressTypeDescriptor', sa.address_type_descriptor,
                    'streetNumberName', sa.street_number_name,
                    'city', sa.city,
                    'stateAbbreviation', sa.state_abbreviation,
                    'postalCode', sa.postal_code
                )
            )
            FROM student_addresses sa
            WHERE sa.student_id = s.student_id
        ),
        'races', (
            SELECT json_agg(
                json_build_object(
                    'raceDescriptor', sr.race_descriptor
                )
            )
            FROM student_races sr
            WHERE sr.student_id = s.student_id
        )
    )
    FROM students s
    WHERE s.student_id = p_student_id;
$$;
```

#### Advanced Pattern with JSONB

```sql
-- Using JSONB for better performance and operators
CREATE OR REPLACE FUNCTION get_assessment_hierarchy_jsonb(p_assessment_id TEXT)
RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_result JSONB;
BEGIN
    SELECT jsonb_build_object(
        'assessmentId', a.assessment_id,
        'assessmentIdentifier', a.assessment_identifier,
        'namespace', a.namespace,
        'title', a.title,
        'period', CASE
            WHEN a.period_begin_date IS NOT NULL THEN
                jsonb_build_object(
                    'beginDate', a.period_begin_date,
                    'endDate', a.period_end_date
                )
            ELSE NULL
        END,
        'academicSubjects', (
            SELECT jsonb_agg(
                jsonb_build_object(
                    'academicSubjectDescriptor', aas.academic_subject_descriptor
                )
            )
            FROM assessment_academic_subjects aas
            WHERE aas.assessment_id = a.assessment_id
        ),
        'scores', (
            SELECT jsonb_agg(
                jsonb_build_object(
                    'minimumScore', ascore.minimum_score,
                    'maximumScore', ascore.maximum_score,
                    'resultDataType', ascore.result_data_type,
                    'reportingMethod', jsonb_build_object(
                        'assessmentReportingMethodDescriptor',
                        ascore.reporting_method_assessment_reporting_method_descriptor
                    )
                )
            )
            FROM assessment_scores ascore
            WHERE ascore.assessment_id = a.assessment_id
        )
    ) INTO v_result
    FROM assessments a
    WHERE a.assessment_id = p_assessment_id;

    -- Remove null values for cleaner output
    v_result := jsonb_strip_nulls(v_result);

    RETURN v_result;
END;
$$;
```

#### Metadata-Driven Dynamic Generation

```sql
-- Dynamic JSON generation based on metadata
CREATE OR REPLACE FUNCTION generate_json_dynamically(
    p_table_name TEXT,
    p_primary_key_value TEXT,
    p_metadata_id INTEGER
)
RETURNS JSONB
LANGUAGE plpgsql
AS $$
DECLARE
    v_sql TEXT;
    v_result JSONB;
    v_column_mappings TEXT;
    v_child_queries TEXT;
BEGIN
    -- Build column mappings from metadata
    SELECT string_agg(
        format('''%s'', %I', mc.json_property_name, mc.column_name),
        ', '
    ) INTO v_column_mappings
    FROM metadata_columns mc
    WHERE mc.table_name = p_table_name
      AND mc.metadata_id = p_metadata_id
    ORDER BY mc.column_order;

    -- Build child relationship queries
    SELECT string_agg(
        format(
            '''%s'', (SELECT jsonb_agg(row_to_json(sub.*)) FROM %I sub WHERE sub.%I = main.%I)',
            cr.json_property_name,
            cr.child_table,
            cr.foreign_key_column,
            cr.primary_key_column
        ),
        ', '
    ) INTO v_child_queries
    FROM metadata_child_relationships cr
    WHERE cr.parent_table = p_table_name
      AND cr.metadata_id = p_metadata_id;

    -- Combine into final query
    v_sql := format(
        'SELECT jsonb_build_object(%s%s) FROM %I main WHERE main.%I = $1',
        v_column_mappings,
        CASE WHEN v_child_queries IS NOT NULL
             THEN ', ' || v_child_queries
             ELSE ''
        END,
        p_table_name,
        p_table_name || '_id'
    );

    -- Execute and return result
    EXECUTE v_sql INTO v_result USING p_primary_key_value;

    RETURN jsonb_strip_nulls(v_result);
END;
$$;
```

### Performance Optimization Techniques

#### 1. Indexed JSON Generation Views

```sql
-- SQL Server: Indexed view for frequently accessed JSON
CREATE VIEW vw_StudentJsonIndexed
WITH SCHEMABINDING
AS
SELECT
    s.StudentId,
    CAST(
        (SELECT
            s.StudentId AS 'id',
            s.StudentUniqueId AS 'studentUniqueId',
            -- ... other fields
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)
    AS NVARCHAR(MAX)) AS JsonDocument
FROM dbo.Students s;

CREATE UNIQUE CLUSTERED INDEX IX_vw_StudentJsonIndexed
    ON vw_StudentJsonIndexed(StudentId);
```

#### 2. Materialized JSON Columns

```sql
-- PostgreSQL: Materialized JSONB column with triggers
ALTER TABLE students ADD COLUMN json_document JSONB;

CREATE OR REPLACE FUNCTION update_student_json()
RETURNS TRIGGER AS $$
BEGIN
    NEW.json_document := get_student_json(NEW.student_id)::jsonb;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_student_json
    BEFORE INSERT OR UPDATE ON students
    FOR EACH ROW
    EXECUTE FUNCTION update_student_json();

-- Index the JSONB column for fast queries
CREATE INDEX idx_students_json_document ON students USING gin(json_document);
```

---

## Performance Architecture

### Query Optimization Strategies

#### 1. Pagination Implementation

```csharp
public class PaginationStrategy
{
    public async Task<PagedResult<T>> GetPagedResults<T>(
        string baseQuery,
        int pageNumber,
        int pageSize,
        IDbConnection connection)
    {
        // SQL Server pagination
        var pagedQuery = $@"
            {baseQuery}
            ORDER BY {GetDefaultOrderBy<T>()}
            OFFSET {(pageNumber - 1) * pageSize} ROWS
            FETCH NEXT {pageSize} ROWS ONLY";

        // Get total count for pagination metadata
        var countQuery = $"SELECT COUNT(*) FROM ({baseQuery}) AS CountQuery";

        var multi = await connection.QueryMultipleAsync(
            $"{countQuery}; {pagedQuery}");

        var totalCount = await multi.ReadSingleAsync<int>();
        var results = await multi.ReadAsync<T>();

        return new PagedResult<T>
        {
            Items = results.ToList(),
            TotalCount = totalCount,
            PageNumber = pageNumber,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
        };
    }
}
```

#### 2. Multi-Query Approach for Complex Hierarchies

```csharp
public class MultiQueryStrategy
{
    public async Task<StudentComplete> GetStudentWithDetails(
        string studentId,
        IDbConnection connection)
    {
        var sql = @"
            -- Query 1: Main student record
            SELECT * FROM Students WHERE StudentId = @StudentId;

            -- Query 2: Addresses
            SELECT * FROM StudentAddresses WHERE StudentId = @StudentId;

            -- Query 3: Races
            SELECT * FROM StudentRaces WHERE StudentId = @StudentId;

            -- Query 4: Enrollments with school info
            SELECT se.*, s.SchoolName
            FROM StudentEnrollments se
            JOIN Schools s ON se.SchoolId = s.SchoolId
            WHERE se.StudentId = @StudentId;";

        using var multi = await connection.QueryMultipleAsync(
            sql, new { StudentId = studentId });

        var student = await multi.ReadSingleOrDefaultAsync<Student>();
        if (student != null)
        {
            student.Addresses = (await multi.ReadAsync<StudentAddress>()).ToList();
            student.Races = (await multi.ReadAsync<StudentRace>()).ToList();
            student.Enrollments = (await multi.ReadAsync<StudentEnrollment>()).ToList();
        }

        return student;
    }
}
```

#### 3. Batch Processing for Bulk Operations

```csharp
public class BatchProcessor
{
    private const int BatchSize = 1000;

    public async Task ProcessLargeJsonArray(
        JArray jsonArray,
        MetadataSchema metadata,
        IDbConnection connection)
    {
        var batches = jsonArray
            .Select((item, index) => new { item, index })
            .GroupBy(x => x.index / BatchSize)
            .Select(g => g.Select(x => x.item).ToList());

        foreach (var batch in batches)
        {
            using var transaction = connection.BeginTransaction();
            try
            {
                // Process batch in parallel where possible
                var tasks = batch.Select(item =>
                    ProcessSingleDocument(item, metadata, connection, transaction));

                await Task.WhenAll(tasks);
                transaction.Commit();
            }
            catch
            {
                transaction.Rollback();
                throw;
            }
        }
    }
}
```

### Caching Strategies

#### 1. Metadata Caching

```csharp
public class MetadataCache
{
    private readonly IMemoryCache _cache;
    private readonly IDbConnection _connection;

    public async Task<MetadataSchema> GetMetadata(string documentType)
    {
        return await _cache.GetOrCreateAsync(
            $"metadata_{documentType}",
            async entry =>
            {
                entry.SlidingExpiration = TimeSpan.FromHours(1);
                entry.Priority = CacheItemPriority.High;

                var metadata = await _connection.QuerySingleAsync<string>(
                    "SELECT MetadataJson FROM MetadataDefinitions WHERE DocumentType = @DocumentType",
                    new { DocumentType = documentType });

                return JsonSerializer.Deserialize<MetadataSchema>(metadata);
            });
    }
}
```

#### 2. Query Plan Caching

```sql
-- SQL Server: Plan guide for frequently used dynamic queries
EXEC sp_create_plan_guide
    @name = N'StudentJsonGuide',
    @stmt = N'SELECT ... FOR JSON PATH',
    @type = N'SQL',
    @module_or_batch = NULL,
    @params = N'@StudentId NVARCHAR(50)',
    @hints = N'OPTION (OPTIMIZE FOR (@StudentId = ''SAMPLE_ID''))';
```

### Index Strategy

#### Core Indexing Principles

```sql
-- Primary key indexes (automatic)
-- Already created with PRIMARY KEY constraints

-- Foreign key indexes (critical for JOINs)
CREATE INDEX IX_StudentAddresses_StudentId
    ON StudentAddresses(StudentId)
    INCLUDE (AddressTypeDescriptor, StreetNumberName, City);

-- Covering indexes for JSON generation queries
CREATE INDEX IX_Students_JsonGeneration
    ON Students(StudentId)
    INCLUDE (StudentUniqueId, FirstName, LastSurname,
             Demographics_HispanicLatinoEthnicity, Demographics_Sex);

-- Filtered indexes for common queries
CREATE INDEX IX_Students_Active
    ON Students(StudentId)
    INCLUDE (StudentUniqueId, FirstName, LastSurname)
    WHERE IsActive = 1;

-- Composite indexes for search patterns
CREATE INDEX IX_Students_NameSearch
    ON Students(LastSurname, FirstName)
    INCLUDE (StudentId, StudentUniqueId);
```

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)

#### Week 1-2: Infrastructure Setup
- Set up development and testing environments
- Configure SQL Server and PostgreSQL instances
- Establish CI/CD pipelines
- Create initial project structure

#### Week 3-4: Core Components
- Implement metadata schema parser
- Create JSON document parser using Newtonsoft.Json
- Develop SQL generator for basic operations
- Implement database connection management with Dapper

### Phase 2: Application-Side Implementation (Weeks 5-8)

#### Week 5-6: Write Path
- Implement JSON-to-relational flattening engine
- Create validation framework
- Develop error handling and retry logic
- Build transaction management

#### Week 7-8: Read Path
- Implement relational-to-JSON reconstruction
- Create pagination system
- Develop multi-query strategy
- Build result caching layer

### Phase 3: Database-Side Implementation (Weeks 9-12)

#### Week 9-10: SQL Server
- Create stored procedures for JSON import
- Implement FOR JSON generation procedures
- Develop metadata-driven dynamic SQL generation
- Create helper functions and utilities

#### Week 11-12: PostgreSQL
- Create PL/pgSQL functions for JSON import
- Implement json_build_object generation functions
- Develop JSONB optimization strategies
- Create triggers for materialized views

### Phase 4: Performance Optimization (Weeks 13-16)

#### Week 13-14: Query Optimization
- Implement comprehensive indexing strategy
- Create query plan guides
- Develop batch processing system
- Optimize pagination queries

#### Week 15-16: Caching and Monitoring
- Implement metadata caching
- Create query result caching
- Develop performance monitoring
- Build diagnostic tools

### Phase 5: Integration and Testing (Weeks 17-20)

#### Week 17-18: Integration
- Integrate with Ed-Fi DMS API
- Implement authentication and authorization
- Create API endpoints
- Develop client SDKs

#### Week 19-20: Testing
- Comprehensive unit testing
- Integration testing
- Performance testing
- Load testing

### Phase 6: Documentation and Deployment (Weeks 21-24)

#### Week 21-22: Documentation
- Technical documentation
- API documentation
- Operations guide
- Troubleshooting guide

#### Week 23-24: Deployment
- Production deployment preparation
- Migration scripts
- Rollback procedures
- Go-live support

---

## Technical Appendices

### Appendix A: Database Type Mappings

#### SQL Server to JSON Type Mappings

| SQL Server Type | JSON Type | C# Type | Notes |
|----------------|-----------|---------|-------|
| INT, BIGINT | number | int, long | Direct mapping |
| DECIMAL, NUMERIC | number | decimal | Precision preserved |
| FLOAT, REAL | number | double, float | May lose precision |
| NVARCHAR, VARCHAR | string | string | UTF-8 encoding |
| DATE | string | DateTime | ISO 8601 format |
| DATETIME2 | string | DateTime | ISO 8601 with time |
| BIT | boolean | bool | Direct mapping |
| UNIQUEIDENTIFIER | string | Guid | Standard GUID format |
| VARBINARY | string | byte[] | Base64 encoded |

#### PostgreSQL to JSON Type Mappings

| PostgreSQL Type | JSON Type | C# Type | Notes |
|----------------|-----------|---------|-------|
| INTEGER, BIGINT | number | int, long | Direct mapping |
| NUMERIC, DECIMAL | number | decimal | Arbitrary precision |
| REAL, DOUBLE PRECISION | number | float, double | IEEE 754 |
| TEXT, VARCHAR | string | string | UTF-8 encoding |
| DATE | string | DateTime | ISO 8601 format |
| TIMESTAMP | string | DateTime | With timezone info |
| BOOLEAN | boolean | bool | Direct mapping |
| UUID | string | Guid | Standard UUID format |
| BYTEA | string | byte[] | Base64 encoded |
| JSONB | object/array | JObject/JArray | Native JSON |

### Appendix B: JSONPath Reference

#### Supported JSONPath Expressions

| Expression | Description | Example |
|-----------|-------------|---------|
| `$` | Root object | `$` |
| `.property` | Child property | `$.order` |
| `..property` | Recursive descent | `$..price` |
| `[n]` | Array index | `$.items[0]` |
| `[*]` | All array elements | `$.items[*]` |
| `.property.subproperty` | Nested properties | `$.order.customer.name` |
| `['property']` | Bracket notation | `$['order']['customer']` |

### Appendix C: Error Codes and Handling

#### Application Error Codes

| Code | Category | Description | Resolution |
|------|----------|-------------|------------|
| 1001 | Validation | Missing required field | Check JSON against metadata |
| 1002 | Validation | Invalid data type | Verify type conversions |
| 1003 | Validation | Foreign key violation | Ensure parent records exist |
| 2001 | Processing | JSON parsing error | Validate JSON syntax |
| 2002 | Processing | Metadata not found | Check metadata configuration |
| 2003 | Processing | Database connection failed | Verify connection string |
| 3001 | Performance | Query timeout | Optimize query or increase timeout |
| 3002 | Performance | Memory limit exceeded | Implement pagination |

### Appendix D: Monitoring Queries

#### SQL Server Monitoring

```sql
-- Monitor JSON generation performance
SELECT
    qt.query_sql_text,
    q.query_id,
    p.plan_id,
    rs.avg_duration / 1000000.0 AS avg_duration_sec,
    rs.avg_cpu_time / 1000000.0 AS avg_cpu_sec,
    rs.avg_logical_io_reads,
    rs.execution_count
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE '%FOR JSON%'
ORDER BY rs.avg_duration DESC;

-- Check index usage
SELECT
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE database_id = DB_ID()
ORDER BY s.user_seeks + s.user_scans + s.user_lookups DESC;
```

#### PostgreSQL Monitoring

```sql
-- Monitor query performance
SELECT
    query,
    calls,
    total_time / 1000 AS total_time_sec,
    mean_time / 1000 AS mean_time_sec,
    max_time / 1000 AS max_time_sec,
    rows
FROM pg_stat_statements
WHERE query LIKE '%json_build_object%'
ORDER BY mean_time DESC
LIMIT 20;

-- Check index efficiency
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;
```

### Appendix E: Sample Configuration Files

#### Application Configuration (appsettings.json)

```json
{
  "DatabaseSettings": {
    "Provider": "SqlServer",
    "ConnectionString": "Server=localhost;Database=EdFiDMS;Trusted_Connection=true;",
    "CommandTimeout": 30,
    "EnableQueryLogging": true
  },
  "MetadataSettings": {
    "CacheEnabled": true,
    "CacheDurationMinutes": 60,
    "MetadataPath": "./metadata",
    "ValidateOnLoad": true
  },
  "PerformanceSettings": {
    "DefaultPageSize": 100,
    "MaxPageSize": 1000,
    "BatchSize": 500,
    "EnableParallelProcessing": true,
    "MaxDegreeOfParallelism": 4
  },
  "JsonSettings": {
    "MaxDepth": 32,
    "PropertyNameCaseInsensitive": true,
    "AllowTrailingCommas": true,
    "WriteIndented": false
  }
}
```

#### Metadata Validation Schema (schema.json)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["schemaVersion", "documentType", "entities"],
  "properties": {
    "schemaVersion": {
      "type": "string",
      "pattern": "^\\d+\\.\\d+$"
    },
    "documentType": {
      "type": "string",
      "minLength": 1
    },
    "entities": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["tableName", "jsonPath", "properties"],
        "properties": {
          "tableName": {
            "type": "string",
            "pattern": "^[A-Za-z][A-Za-z0-9_]*$"
          },
          "jsonPath": {
            "type": "string",
            "pattern": "^\\$"
          },
          "properties": {
            "type": "array",
            "minItems": 1,
            "items": {
              "type": "object",
              "required": ["columnName", "dataType", "jsonPath"],
              "properties": {
                "columnName": {
                  "type": "string",
                  "pattern": "^[A-Za-z][A-Za-z0-9_]*$"
                },
                "dataType": {
                  "type": "string"
                },
                "jsonPath": {
                  "type": "string"
                },
                "isPrimaryKey": {
                  "type": "boolean"
                },
                "isRequired": {
                  "type": "boolean"
                },
                "isAutoGenerated": {
                  "type": "boolean"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Conclusion

This design provides a comprehensive, metadata-driven solution for bidirectional JSON-relational transformations in the Ed-Fi Data Management Service. The architecture leverages modern database JSON capabilities while maintaining the flexibility to adapt to evolving data structures without code changes.

The key innovation is the complete elimination of code generation in favor of runtime interpretation of metadata, combined with database-side processing for optimal performance. This approach is particularly well-suited to the Ed-Fi ecosystem's requirement for handling diverse educational data structures while maintaining consistency and performance.

The implementation roadmap provides a clear path forward, with each phase building upon the previous to create a robust, scalable, and maintainable system that can serve as the foundation for the next generation of Ed-Fi data management capabilities.
