## Root Entity Tables

### StudentSchoolAssociation Table

```sql
CREATE TABLE StudentSchoolAssociation (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),               -- Surrogate PK
    DocumentUuid UUID NOT NULL,                        -- Link to Document table
    DocumentPartitionKey TINYINT NOT NULL,             -- Partition key from Document
    Student_Id BIGINT NOT NULL,                        -- FK to Student table (also part of identity)
    School_Id BIGINT NOT NULL,                         -- FK to School table (also part of identity)
    EntryDate DATE NOT NULL,                           -- Natural key component (part of identity)
    EnrollmentTypeDescriptor VARCHAR(100),             -- Example data field
    CONSTRAINT FK_StudentSchoolAssociation_Document    -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT FK_StudentSchoolAssociation_Student     -- Reference to Student table
        FOREIGN KEY (Student_Id) REFERENCES Student(Id),
    CONSTRAINT FK_StudentSchoolAssociation_School      -- Reference to School table
        FOREIGN KEY (School_Id) REFERENCES School(Id),
    CONSTRAINT UQ_StudentSchoolAssociation_Identity    -- Natural key uniqueness
        UNIQUE (Student_Id, School_Id, EntryDate)
);
```

### Student Table

```sql
-- Simple root table
CREATE TABLE Student (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),          -- Surrogate PK
    DocumentUuid UUID NOT NULL,                   -- Link to Document table
    DocumentPartitionKey TINYINT NOT NULL,        -- Partition key from Document
    StudentUniqueId VARCHAR(100) NOT NULL,        -- Natural key
    LastSurname VARCHAR(100) NOT NULL,            -- Example data field
    CONSTRAINT FK_Student_Document                -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_Student_StudentUniqueId UNIQUE (StudentUniqueId)  -- Natural key uniqueness
);
```

### School Table

```sql
-- Simple root table
CREATE TABLE School (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                -- Surrogate PK
    DocumentUuid UUID NOT NULL,                         -- Link to Document table
    DocumentPartitionKey TINYINT NOT NULL,              -- Partition key from Document
    SchoolId BIGINT NOT NULL,                           -- Natural key
    NameOfInstitution VARCHAR(255) NOT NULL,            -- Example data field
    CONSTRAINT FK_School_Document                       -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_School_SchoolId UNIQUE (SchoolId)     -- Natural key uniqueness
);
```

### GraduationPlan Table

```sql
-- Simple root table
CREATE TABLE GraduationPlan (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                -- Surrogate PK
    DocumentUuid UUID NOT NULL,                         -- Link to Document table
    DocumentPartitionKey TINYINT NOT NULL,              -- Partition key from Document
    GraduationPlanTypeDescriptor VARCHAR(100) NOT NULL, -- Natural key component (part of identity)
    CONSTRAINT FK_GraduationPlan_Document               -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_GraduationPlan_Identity               -- Natural key uniqueness
        UNIQUE (GraduationPlanTypeDescriptor)
);
```

## Child Entity Tables

### StudentOtherName Table (array of Common on Student)

```sql
CREATE TABLE StudentOtherName (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),            -- Surrogate PK
    Student_Id BIGINT NOT NULL,                     -- FK to root
    DocumentUuid UUID NOT NULL,                     -- Link to Document (same as root table)
    DocumentPartitionKey TINYINT NOT NULL,          -- Partition key from Document (same as root table)
    OtherNameTypeDescriptor VARCHAR(100) NOT NULL,  -- Part of identity for sub-table
    LastSurname VARCHAR(100) NOT NULL,              -- Example data field
    CONSTRAINT FK_StudentOtherName_Student          -- Back to root table
        FOREIGN KEY (Student_Id) REFERENCES Student(Id) ON DELETE CASCADE,
    CONSTRAINT FK_StudentOtherName_Document         -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_StudentOtherName_Type              -- Uniqueness on FK to parent + array identity
        UNIQUE (Student_Id, OtherNameTypeDescriptor)
);
```

### StudentSchoolAssociationEducationPlan Table (array of single value)

```sql
CREATE TABLE StudentSchoolAssociationEducationPlan (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                    -- Surrogate PK
    StudentSchoolAssociation_Id BIGINT NOT NULL,            -- FK to root
    DocumentUuid UUID NOT NULL,                             -- Link to Document
    DocumentPartitionKey TINYINT NOT NULL,                  -- Partition key from Document
    EducationPlanDescriptor VARCHAR(100) NOT NULL,          -- The single value
    CONSTRAINT FK_SSAEducationPlan_StudentSchoolAssociation -- Back to root table
        FOREIGN KEY (StudentSchoolAssociation_Id)
        REFERENCES StudentSchoolAssociation(Id) ON DELETE CASCADE,
    CONSTRAINT FK_SSAEducationPlan_Document                 -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_SSAEducationPlan_Type                     -- Uniqueness on FK to root + value
        UNIQUE (StudentSchoolAssociation_Id, EducationPlanDescriptor)
);
```

### StudentSchoolAssociationAlternativeGraduationPlan Table (array of references)

```sql
CREATE TABLE StudentSchoolAssociationAlternativeGraduationPlan (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                    -- Surrogate PK
    StudentSchoolAssociation_Id BIGINT NOT NULL,            -- FK to root
    DocumentUuid UUID NOT NULL,                             -- Link to Document
    DocumentPartitionKey TINYINT NOT NULL,                  -- Partition key from Document
    AlternativeGraduationPlan_Id BIGINT NOT NULL,           -- FK to GraduationPlan root table
    CONSTRAINT FK_SSAAltGradPlan_StudentSchoolAssociation   -- Back to root table
        FOREIGN KEY (StudentSchoolAssociation_Id)
        REFERENCES StudentSchoolAssociation(Id) ON DELETE CASCADE,
    CONSTRAINT FK_SSAAltGradPlan_AltGraduationPlan          -- Reference to GraduationPlan
        FOREIGN KEY (AlternativeGraduationPlan_Id)
        REFERENCES GraduationPlan(Id),
    CONSTRAINT FK_SSAAltGradPlan_Document                   -- Back to parent Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE
    CONSTRAINT UQ_SAAltGradPlan_AltGraduationPlan           -- Uniqueness on FK to root + reference
        UNIQUE (StudentSchoolAssociation_Id, AlternativeGraduationPlan_Id)
);
```

## Multi-level Child tables

Given a `StudentEducationOrganizationAssociation` root table:

### First-Level Sub-Table: Address

```sql
-- First-level child table: Address collection
CREATE TABLE StudentEducationOrganizationAssociationAddress (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                           -- Surrogate PK
    StudentEducationOrganizationAssociation_Id BIGINT NOT NULL,    -- FK to root
    DocumentUuid UUID NOT NULL,                                    -- Link to Document
    DocumentPartitionKey TINYINT NOT NULL,                         -- Partition key from Document
    AddressTypeDescriptor VARCHAR(100) NOT NULL,                   -- Part of identity
    StreetNumberName VARCHAR(150) NOT NULL,                        -- Part of identity
    CONSTRAINT FK_SEOAAddress_StudentEducationOrganizationAssociation
        FOREIGN KEY (StudentEducationOrganizationAssociation_Id)
        REFERENCES StudentEducationOrganizationAssociation(Id) ON DELETE CASCADE,
    CONSTRAINT FK_SEOAAddress_Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_SEOAAddress_Identity
        UNIQUE (StudentEducationOrganizationAssociation_Id, AddressTypeDescriptor, StreetNumberName)
);
```

### Second-Level Sub-Table: AddressPeriod

```sql
-- Second-level child table: Period collection within Address
CREATE TABLE StudentEducationOrganizationAssociationAddressPeriod (
    Id BIGINT PRIMARY KEY IDENTITY(1,1),                               -- Surrogate PK
    StudentEducationOrganizationAssociationAddress_Id BIGINT NOT NULL, -- FK to parent Address table (not root)
    DocumentUuid UUID NOT NULL,                                        -- Link to Document
    DocumentPartitionKey TINYINT NOT NULL,                             -- Partition key from Document
    BeginDate DATE NOT NULL,                                           -- Part of identity
    EndDate DATE,                                                      -- Optional field
    CONSTRAINT FK_SEOAAddressPeriod_SEOAAddress
        FOREIGN KEY (StudentEducationOrganizationAssociationAddress_Id)
        REFERENCES StudentEducationOrganizationAssociationAddress(Id) ON DELETE CASCADE,
    CONSTRAINT FK_SEOAAddressPeriod_Document
        FOREIGN KEY (DocumentUuid, DocumentPartitionKey)
        REFERENCES Document(DocumentUuid, DocumentPartitionKey) ON DELETE CASCADE,
    CONSTRAINT UQ_SEOAAddressPeriod_Identity
        UNIQUE (StudentEducationOrganizationAssociationAddress_Id, BeginDate)
);
```




## Relationship Examples

### Intra-Resource Relationships (Foreign Keys Within Same Resource)

Within a single resource, we use surrogate-key foreign keys:
- StudentOtherName → Student (via Student_Id)
- StudentSchoolAssociationEducationPlan → StudentSchoolAssociation (via StudentSchoolAssociation_Id)
- SchoolGradeLevel → School (via School_Id)
- GraduationPlanCreditsBySubject → GraduationPlan (via GraduationPlan_Id)

These foreign keys ensure referential integrity within the resource boundary and cascade deletes appropriately.

### Cross-Resource Relationships (Foreign Keys, but Managed by Reference Table)

The relationships between different root resources (e.g., Student to School, StudentSchoolAssociation to Student/School) are managed through the existing Reference table mechanism, though there are surrogate foreign keys in the flattened tables.

Example: A StudentSchoolAssociation references both a Student and a School:
- The `Student_Id` and `School_Id` columns store the surrogate keys
- These are populated during flattening by looking up the corresponding records
- The Reference table continues to handle the actual reference validation
- If a referenced Student or School is attempted to be deleted, the Reference table constraints will prevent the deletion


## Index Strategy

```sql
-- Document relationship indexes, primary means of finding flattened tables for a document
CREATE INDEX IX_StudentSchoolAssociation_DocumentUuid
    ON StudentSchoolAssociation(DocumentUuid, DocumentPartitionKey);
CREATE INDEX IX_Student_DocumentUuid
    ON Student(DocumentUuid, DocumentPartitionKey);

-- Parent-child relationship indexes for efficient joins
CREATE INDEX IX_StudentOtherName_Student_Id
    ON StudentOtherName(Student_Id);
CREATE INDEX IX_SSAAltGradPlan_StudentSchoolAssociation_Id
    ON StudentSchoolAssociationAlternativeGraduationPlan(StudentSchoolAssociation_Id);
```

## Notes on Design Decisions

1. **Surrogate Keys Everywhere**: All tables use BIGINT IDENTITY surrogate keys for simplicity and performance, no natural keys
2. **Natural Key Uniqueness**: Root entities have UNIQUE constraints on their natural keys. Child tables add the root table surrogate key to the constraint
3. **Prefixed Foreign Key Columns**: FK columns are prefixed with underscores (e.g., `Student_Id`) to avoid naming collisions with fields from the JSON
4. **DocumentUuid/DocumentPartitionKey**: Every table maintains a link back to the parent Document for traceability
5. **Collection Identity**: Child tables use composite uniqueness constraints to prevent duplicate entries

