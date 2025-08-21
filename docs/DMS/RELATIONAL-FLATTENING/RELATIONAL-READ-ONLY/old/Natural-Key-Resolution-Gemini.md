# Design: Resolving Natural Keys with Views
This document proposes a design for creating database views on top of the flattened relational tables described in the "Ed-Fi Data Management Service: Simplified Relational Flattening Design." The goal of these views is to "resolve" the surrogate key references, presenting a denormalized picture where the natural keys of all related entities are directly included.

## Core Principle: Progressive Joining
The fundamental principle is to create views that progressively join from a child or referencing table back to its parents or referenced tables. By using the surrogate key foreign key relationships for the joins, we can pull in the natural key columns from each parent table. This provides users with a more intuitive, "natural key" view of the data without forcing them to write complex joins themselves.

Views will be created for each significant entity, especially those that are frequently queried.

## Example 1: Resolving StudentSchoolAssociation
The StudentSchoolAssociation table is a central entity that references Student and School. Its own natural key is a composite of these references plus an EntryDate.

### Base Tables (Natural Keys identified):

  - Student: StudentUniqueId
  - School: SchoolId
  - StudentSchoolAssociation: Student_Id (FK), School_Id (FK), EntryDate

### Proposed View: vw_StudentSchoolAssociation
This view will join StudentSchoolAssociation with Student and School to replace the surrogate keys (Student_Id, School_Id) with their corresponding natural keys (StudentUniqueId, SchoolId).

```sql
CREATE VIEW vw_StudentSchoolAssociation AS
SELECT
    -- Natural Keys from referenced tables
    s.StudentUniqueId,
    sch.SchoolId,

    -- Natural Key component from the base table
    ssa.EntryDate,

    -- All other data columns from StudentSchoolAssociation
    ssa.EnrollmentTypeDescriptor,

    -- Include surrogate keys for potential joins to child views
    ssa.Id AS StudentSchoolAssociationId,
    s.Id AS StudentId,
    sch.Id AS SchoolId,

    -- Document identifiers for traceability
    ssa.DocumentUuid,
    ssa.DocumentPartitionKey
FROM
    StudentSchoolAssociation ssa
JOIN
    Student s ON ssa.Student_Id = s.Id
JOIN
    School sch ON ssa.School_Id = sch.Id;
```

### Benefits of this View:

  - Users can query by StudentUniqueId and SchoolId directly.
  - The view presents a business-centric view of the association.
  - The underlying surrogate key joins are abstracted away.

## Example 2: Resolving Nested Collections

This example addresses child tables, such as arrays of objects or references within a root entity.

### StudentOtherName (Child of Student)
The StudentOtherName table's unique identity includes its parent's surrogate key (Student_Id) and OtherNameTypeDescriptor. The view will resolve Student_Id to StudentUniqueId.

### Proposed View: vw_StudentOtherName

```sql
CREATE VIEW vw_StudentOtherName AS
SELECT
    -- Resolved Natural Key from the parent Student table
    s.StudentUniqueId,

    -- Natural Key components from the StudentOtherName table itself
    son.OtherNameTypeDescriptor,
    son.LastSurname,

    -- Include surrogate keys for reference
    son.Id AS StudentOtherNameId,
    son.Student_Id AS StudentId,

    -- Document identifiers
    son.DocumentUuid,
    son.DocumentPartitionKey
FROM
    StudentOtherName son
JOIN
    Student s ON son.Student_Id = s.Id;
```

### StudentSchoolAssociationAlternativeGraduationPlan (References GraduationPlan)
This table is a child of StudentSchoolAssociation and also references GraduationPlan. The view needs to resolve both foreign keys. We will join to our previously defined vw_StudentSchoolAssociation to simplify the logic.

### Proposed View: vw_StudentSchoolAssociationAlternativeGraduationPlan

```sql
CREATE VIEW vw_StudentSchoolAssociationAlternativeGraduationPlan AS
SELECT
    -- Resolved Natural Keys from the parent vw_StudentSchoolAssociation
    v_ssa.StudentUniqueId,
    v_ssa.SchoolId,
    v_ssa.EntryDate,

    -- Resolved Natural Key from the referenced GraduationPlan table
    gp.GraduationPlanTypeDescriptor,

    -- Include surrogate keys for reference
    ssaagp.Id AS StudentSchoolAssociationAlternativeGraduationPlanId,
    ssaagp.StudentSchoolAssociation_Id,
    ssaagp.AlternativeGraduationPlan_Id AS GraduationPlanId,

    -- Document identifiers
    ssaagp.DocumentUuid,
    ssaagp.DocumentPartitionKey
FROM
    StudentSchoolAssociationAlternativeGraduationPlan ssaagp
JOIN
    -- Join to the view to easily get the parent's full natural key
    vw_StudentSchoolAssociation v_ssa ON ssaagp.StudentSchoolAssociation_Id = v_ssa.StudentSchoolAssociationId
JOIN
    GraduationPlan gp ON ssaagp.AlternativeGraduationPlan_Id = gp.Id;
```

## Example 3: Resolving Multi-Level Nesting
For deeply nested structures like StudentEducationOrganizationAssociationAddressPeriod, we can create a chain of views or use a single, more complex view with multiple joins. Chaining views is often more manageable.

### Base Tables Hierarchy:

  - StudentEducationOrganizationAssociation (Root)
  - StudentEducationOrganizationAssociationAddress (Child of #1)
  - StudentEducationOrganizationAssociationAddressPeriod (Child of #2)

### Step 1: Create View for the First-Level Child (vw_SEOAAddress)
First, create a view for the Address table that resolves its parent's natural key. Let's assume the natural key for StudentEducationOrganizationAssociation is a combination of Student_Id and EducationOrganization_Id.

```sql
-- First, a view for the root to resolve its own references (Student, EdOrg)
CREATE VIEW vw_StudentEducationOrganizationAssociation AS
SELECT
    s.StudentUniqueId,
    eo.EducationOrganizationId, -- Assuming an EducationOrganization table
    seoa.Id AS StudentEducationOrganizationAssociationId,
    seoa.LoginId,
    seoa.HispanicLatinoEthnicity
    -- etc.
FROM
    StudentEducationOrganizationAssociation seoa
JOIN
    Student s ON seoa.Student_Id = s.Id
JOIN
    EducationOrganization eo ON seoa.EducationOrganization_Id = eo.Id;


-- Now, the view for the Address child table
CREATE VIEW vw_StudentEducationOrganizationAssociationAddress AS
SELECT
    -- Resolved natural key from the parent view
    v_seoa.StudentUniqueId,
    v_seoa.EducationOrganizationId,

    -- Natural key components from the Address table
    seoa_addr.AddressTypeDescriptor,
    seoa_addr.StreetNumberName,
    seoa_addr.City,
    seoa_addr.PostalCode,

    -- Surrogate key for joining from the next level down
    seoa_addr.Id AS StudentEducationOrganizationAssociationAddressId,
    seoa_addr.StudentEducationOrganizationAssociation_Id
FROM
    StudentEducationOrganizationAssociationAddress seoa_addr
JOIN
    vw_StudentEducationOrganizationAssociation v_seoa
    ON seoa_addr.StudentEducationOrganizationAssociation_Id = v_seoa.StudentEducationOrganizationAssociationId;
```

### Step 2: Create View for the Second-Level Child (vw_SEOAAddressPeriod)
Now, create the view for the Period table, which joins to the vw_SEOAAddress view created above to pull in the full, resolved natural key from the root.

```sql
CREATE VIEW vw_StudentEducationOrganizationAssociationAddressPeriod AS
SELECT
    -- Resolved natural keys from the parent Address view (which includes root keys)
    v_seoa_addr.StudentUniqueId,
    v_seoa_addr.EducationOrganizationId,
    v_seoa_addr.AddressTypeDescriptor,
    v_seoa_addr.StreetNumberName,

    -- Natural key from the Period table itself
    seoa_period.BeginDate,

    -- Other data columns
    seoa_period.EndDate,

    -- Surrogate keys for reference
    seoa_period.Id as StudentEducationOrganizationAssociationAddressPeriodId,
    seoa_period.StudentEducationOrganizationAssociationAddress_Id
FROM
    StudentEducationOrganizationAssociationAddressPeriod seoa_period
JOIN
    vw_StudentEducationOrganizationAssociationAddress v_seoa_addr
    ON seoa_period.StudentEducationOrganizationAssociationAddress_Id = v_seoa_addr.StudentEducationOrganizationAssociationAddressId;
```

## Naming Convention for Views
A simple and clear naming convention is proposed:

  - Prefix all views with vw_.
  - The rest of the view name should match the name of the primary base table it represents (e.g., StudentSchoolAssociation -> vw_StudentSchoolAssociation).

This makes it easy for users to discover and understand the purpose of each view.
