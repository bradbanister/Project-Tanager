# StudentSchoolAssociation Entity Relationship Diagrams

## Overview
This document provides Entity Relationship Diagrams (ERDs) for the flattened table structures, separated into two main groupings:
1. StudentSchoolAssociation and its relationships
2. StudentEducationOrganizationAssociation with multi-level sub-tables

## Diagram 1: StudentSchoolAssociation Grouping

```mermaid
erDiagram
    Document {
        UUID DocumentUuid PK
        TINYINT DocumentPartitionKey PK
        JSONB EdfiDoc
        BIGINT Id
    }
    
    Student {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        VARCHAR StudentUniqueId UK
        VARCHAR LastSurname
    }
    
    School {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT SchoolId UK
        VARCHAR NameOfInstitution
    }
    
    GraduationPlan {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        VARCHAR GraduationPlanTypeDescriptor UK
    }
    
    StudentSchoolAssociation {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT Student_Id FK
        BIGINT School_Id FK
        DATE EntryDate
        VARCHAR EnrollmentTypeDescriptor
    }
    
    StudentOtherName {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT Student_Id FK
        VARCHAR OtherNameTypeDescriptor
        VARCHAR LastSurname
    }
    
    StudentSchoolAssociationEducationPlan {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT StudentSchoolAssociation_Id FK
        VARCHAR EducationPlanDescriptor
    }
    
    StudentSchoolAssociationAlternativeGraduationPlan {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT StudentSchoolAssociation_Id FK
        BIGINT AlternativeGraduationPlan_Id FK
    }
    
    %% Document relationships - all tables link back to Document
    Document ||--o{ Student : "source of truth"
    Document ||--o{ School : "source of truth"
    Document ||--o{ GraduationPlan : "source of truth"
    Document ||--o{ StudentSchoolAssociation : "source of truth"
    Document ||--o{ StudentOtherName : "source of truth"
    Document ||--o{ StudentSchoolAssociationEducationPlan : "source of truth"
    Document ||--o{ StudentSchoolAssociationAlternativeGraduationPlan : "source of truth"
    
    %% Cross-resource relationships (managed by Reference table)
    Student ||--o{ StudentSchoolAssociation : "enrolls in"
    School ||--o{ StudentSchoolAssociation : "enrolls"
    GraduationPlan ||--o{ StudentSchoolAssociationAlternativeGraduationPlan : "alternative plan"
    
    %% Intra-resource relationships (direct FKs)
    Student ||--o{ StudentOtherName : "has other names"
    StudentSchoolAssociation ||--o{ StudentSchoolAssociationEducationPlan : "has education plans"
    StudentSchoolAssociation ||--o{ StudentSchoolAssociationAlternativeGraduationPlan : "has alternative graduation plans"
```

## Diagram 2: StudentEducationOrganizationAssociation Multi-Level Grouping

```mermaid
erDiagram
    Document {
        UUID DocumentUuid PK
        TINYINT DocumentPartitionKey PK
        JSONB EdfiDoc
        BIGINT Id
    }
    
    Student {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        VARCHAR StudentUniqueId UK
    }
    
    EducationOrganization {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT EducationOrganizationId UK
    }
    
    StudentEducationOrganizationAssociation {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT Student_Id FK
        BIGINT EducationOrganization_Id FK
        VARCHAR ProfileThumbnail
        VARCHAR LoginId
        BIT HispanicLatinoEthnicity
    }
    
    StudentEducationOrganizationAssociationAddress {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT StudentEducationOrganizationAssociation_Id FK
        VARCHAR AddressTypeDescriptor
        VARCHAR StreetNumberName
        VARCHAR City
        VARCHAR PostalCode
    }
    
    StudentEducationOrganizationAssociationAddressPeriod {
        BIGINT Id PK
        UUID DocumentUuid FK
        TINYINT DocumentPartitionKey FK
        BIGINT StudentEducationOrganizationAssociationAddress_Id FK
        DATE BeginDate
        DATE EndDate
    }
    
    %% Document relationships
    Document ||--o{ Student : "source of truth"
    Document ||--o{ EducationOrganization : "source of truth"
    Document ||--o{ StudentEducationOrganizationAssociation : "source of truth"
    Document ||--o{ StudentEducationOrganizationAssociationAddress : "source of truth"
    Document ||--o{ StudentEducationOrganizationAssociationAddressPeriod : "source of truth"
    
    %% Cross-resource relationships
    Student ||--o{ StudentEducationOrganizationAssociation : "associates with"
    EducationOrganization ||--o{ StudentEducationOrganizationAssociation : "has association"
    
    %% Multi-level hierarchy (intra-resource)
    StudentEducationOrganizationAssociation ||--o{ StudentEducationOrganizationAssociationAddress : "has addresses"
    StudentEducationOrganizationAssociationAddress ||--o{ StudentEducationOrganizationAssociationAddressPeriod : "has periods"
```

## Relationship Annotations

### 1. Document Table Relationships
**Pattern**: Every flattened table maintains a foreign key back to the Document table
- **Purpose**: Maintains traceability and source of truth
- **Constraint**: `ON DELETE CASCADE` ensures flattened data is removed when Document is deleted
- **Keys**: Composite FK on (DocumentUuid, DocumentPartitionKey)

### 2. Cross-Resource Relationships
**Pattern**: References between different root resources (e.g., Student → School)
- **Management**: Reference validation handled by existing Reference table
- **Foreign Keys**: Surrogate keys stored but not enforced as FKs in flattened tables
- **Examples**:
  - StudentSchoolAssociation → Student
  - StudentSchoolAssociation → School
  - StudentSchoolAssociation → GraduationPlan

### 3. Intra-Resource Relationships (Same Resource Hierarchy)
**Pattern**: Parent-child relationships within the same resource boundary
- **Foreign Keys**: Enforced with CASCADE DELETE
- **Examples**:
  - Student → StudentOtherName (one-to-many)
  - StudentSchoolAssociation → StudentSchoolAssociationEducationPlan (one-to-many)
  - StudentEducationOrganizationAssociation → StudentEducationOrganizationAssociationAddress (one-to-many)

### 4. Multi-Level Sub-Table Relationships
**Pattern**: Second-level child tables reference their immediate parent, not the root
- **Hierarchy**: Root → First-Level Sub-Table → Second-Level Sub-Table
- **Example Chain**:
  - StudentEducationOrganizationAssociation (root)
  - → StudentEducationOrganizationAssociationAddress (level 1)
  - → StudentEducationOrganizationAssociationAddressPeriod (level 2)

## Simplified View: StudentSchoolAssociation Focus

```mermaid
erDiagram
    StudentSchoolAssociation {
        BIGINT Id PK
        BIGINT Student_Id FK "Reference to Student"
        BIGINT School_Id FK "Reference to School"
        DATE EntryDate "Part of natural key"
    }
    
    Student {
        BIGINT Id PK
        VARCHAR StudentUniqueId UK "Natural key"
    }
    
    School {
        BIGINT Id PK
        BIGINT SchoolId UK "Natural key"
    }
    
    GraduationPlan {
        BIGINT Id PK
        VARCHAR GraduationPlanTypeDescriptor "Part of natural key"
    }
    
    StudentSchoolAssociationEducationPlan {
        BIGINT StudentSchoolAssociation_Id FK
        VARCHAR EducationPlanDescriptor "The value"
    }
    
    StudentSchoolAssociationAlternativeGraduationPlan {
        BIGINT StudentSchoolAssociation_Id FK
        BIGINT AlternativeGraduationPlan_Id FK "Reference"
    }
    
    Student ||--o{ StudentSchoolAssociation : "enrolled in"
    School ||--o{ StudentSchoolAssociation : "enrolls"
    StudentSchoolAssociation ||--o{ StudentSchoolAssociationEducationPlan : "has plans"
    StudentSchoolAssociation ||--o{ StudentSchoolAssociationAlternativeGraduationPlan : "has alt plans"
    GraduationPlan ||--o{ StudentSchoolAssociationAlternativeGraduationPlan : "referenced by"
```

## Key Design Principles Illustrated

### 1. Surrogate Keys Everywhere
- All tables use `BIGINT IDENTITY(1,1)` as primary key
- Natural keys become unique constraints, not primary keys

### 2. Foreign Key Naming Convention
- FK columns prefixed with parent table name + "_Id"
- Example: `Student_Id`, `School_Id`, `StudentSchoolAssociation_Id`

### 3. Uniqueness Constraints
- Root tables: Natural key fields
- Child tables: Parent FK + identifying fields
- Second-level: Immediate parent FK + identifying fields

### 4. Cascade Behavior
```
Document deletion → cascades to all flattened tables
Root table deletion → cascades to child tables
First-level deletion → cascades to second-level tables
```

### 5. Reference Table Integration
- Cross-resource relationships validated by Reference table
- Flattened tables store surrogate keys for query performance
- Reference table prevents orphaned references

## Multi-Level Hierarchy Example

```mermaid
graph TD
    D[Document] --> SEOA[StudentEducationOrganizationAssociation]
    SEOA --> SEOA_Addr[Address - Level 1]
    SEOA_Addr --> SEOA_AddrPer[AddressPeriod - Level 2]
    SEOA --> SEOA_Char[StudentCharacteristic - Level 1]
    SEOA_Char --> SEOA_CharPer[CharacteristicPeriod - Level 2]
    SEOA --> SEOA_Dis[Disability - Level 1]
    SEOA_Dis --> SEOA_DisDes[DisabilityDesignation - Level 2]
    
    style D fill:#f9f,stroke:#333,stroke-width:4px
    style SEOA fill:#bbf,stroke:#333,stroke-width:2px
    style SEOA_Addr fill:#bfb,stroke:#333,stroke-width:1px
    style SEOA_AddrPer fill:#ffb,stroke:#333,stroke-width:1px
```

## Index Strategy Visualization

```mermaid
graph LR
    subgraph "Primary Indexes"
        PK1[Id - Clustered PK]
    end
    
    subgraph "Document Lookups"
        IX1[DocumentUuid + DocumentPartitionKey]
    end
    
    subgraph "Parent-Child Navigation"
        IX2[Parent_Id FK indexes]
    end
    
    subgraph "Natural Key Lookups"
        IX3[StudentUniqueId]
        IX4[SchoolId]
    end
    
    subgraph "Query Optimization"
        IX5[Commonly filtered fields]
        IX6[Date range queries]
    end
```

## Notes on ERD Interpretation

1. **Cardinality Notation**:
   - `||` = one (mandatory)
   - `o{` = zero or many
   - Example: `Student ||--o{ StudentOtherName` means "one Student has zero or many OtherNames"

2. **Foreign Key Types**:
   - **Solid lines**: Enforced foreign keys (intra-resource)
   - **Conceptual relationships**: Cross-resource references (managed by Reference table)

3. **Cascade Rules**:
   - All relationships from Document use CASCADE DELETE
   - Intra-resource relationships use CASCADE DELETE
   - Cross-resource relationships rely on Reference table constraints

4. **Multi-Level Considerations**:
   - Each level only knows about its immediate parent
   - Deletion cascades flow through all levels
   - Query reconstruction requires joining through each level