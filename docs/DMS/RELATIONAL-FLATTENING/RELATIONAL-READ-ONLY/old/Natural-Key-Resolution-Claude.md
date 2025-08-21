## View Design for Natural Key Resolution

### Core Design Principles for Views

1. **Complete Natural Key Exposure**: Every surrogate key reference is expanded to show all natural key columns from the referenced table
2. **Naming Convention**: Natural key columns are prefixed with the referenced table name for clarity
3. **Hierarchical Resolution**: Views resolve keys at all levels - from root references down through nested collections
4. **Query-Friendly**: Views maintain the same cardinality as base tables but with enhanced readability

### View Examples

#### StudentSchoolAssociation_View

This view resolves the Student and School surrogate keys to their natural keys:

```sql
CREATE VIEW StudentSchoolAssociation_View AS
SELECT
    ssa.Id,
    ssa.DocumentUuid,
    ssa.DocumentPartitionKey,

    -- Resolved Student natural key
    ssa.Student_Id,
    s.StudentUniqueId AS Student_StudentUniqueId,

    -- Resolved School natural key
    ssa.School_Id,
    sch.SchoolId AS School_SchoolId,

    -- Original StudentSchoolAssociation fields
    ssa.EntryDate,
    ssa.EnrollmentTypeDescriptor
FROM StudentSchoolAssociation ssa
INNER JOIN Student s ON ssa.Student_Id = s.Id
INNER JOIN School sch ON ssa.School_Id = sch.Id;
```

#### StudentOtherName_View

This view resolves the Student surrogate key to its natural key:

```sql
CREATE VIEW StudentOtherName_View AS
SELECT
    son.Id,
    son.DocumentUuid,
    son.DocumentPartitionKey,

    -- Resolved Student natural key
    son.Student_Id,
    s.StudentUniqueId AS Student_StudentUniqueId,

    -- StudentOtherName fields
    son.OtherNameTypeDescriptor,
    son.LastSurname
FROM StudentOtherName son
INNER JOIN Student s ON son.Student_Id = s.Id;
```

#### StudentSchoolAssociationEducationPlan_View

This view resolves the StudentSchoolAssociation surrogate key to its complete natural key (which includes resolved Student and School keys):

```sql
CREATE VIEW StudentSchoolAssociationEducationPlan_View AS
SELECT
    ssaep.Id,
    ssaep.DocumentUuid,
    ssaep.DocumentPartitionKey,

    -- Resolved StudentSchoolAssociation natural keys (composite)
    ssaep.StudentSchoolAssociation_Id,
    s.StudentUniqueId AS StudentSchoolAssociation_Student_StudentUniqueId,
    sch.SchoolId AS StudentSchoolAssociation_School_SchoolId,
    ssa.EntryDate AS StudentSchoolAssociation_EntryDate,

    -- StudentSchoolAssociationEducationPlan fields
    ssaep.EducationPlanDescriptor
FROM StudentSchoolAssociationEducationPlan ssaep
INNER JOIN StudentSchoolAssociation ssa ON ssaep.StudentSchoolAssociation_Id = ssa.Id
INNER JOIN Student s ON ssa.Student_Id = s.Id
INNER JOIN School sch ON ssa.School_Id = sch.Id;
```

#### StudentSchoolAssociationAlternativeGraduationPlan_View

This view resolves both the StudentSchoolAssociation and GraduationPlan surrogate keys:

```sql
CREATE VIEW StudentSchoolAssociationAlternativeGraduationPlan_View AS
SELECT
    ssaagp.Id,
    ssaagp.DocumentUuid,
    ssaagp.DocumentPartitionKey,

    -- Resolved StudentSchoolAssociation natural keys
    ssaagp.StudentSchoolAssociation_Id,
    s.StudentUniqueId AS StudentSchoolAssociation_Student_StudentUniqueId,
    sch.SchoolId AS StudentSchoolAssociation_School_SchoolId,
    ssa.EntryDate AS StudentSchoolAssociation_EntryDate,

    -- Resolved GraduationPlan natural key
    ssaagp.AlternativeGraduationPlan_Id,
    gp.GraduationPlanTypeDescriptor AS AlternativeGraduationPlan_GraduationPlanTypeDescriptor

FROM StudentSchoolAssociationAlternativeGraduationPlan ssaagp
INNER JOIN StudentSchoolAssociation ssa ON ssaagp.StudentSchoolAssociation_Id = ssa.Id
INNER JOIN Student s ON ssa.Student_Id = s.Id
INNER JOIN School sch ON ssa.School_Id = sch.Id
INNER JOIN GraduationPlan gp ON ssaagp.AlternativeGraduationPlan_Id = gp.Id;
```

### Multi-Level Example: StudentEducationOrganizationAssociation

For the multi-level hierarchy, we need views that resolve keys through multiple levels:

#### StudentEducationOrganizationAssociationAddress_View

```sql
CREATE VIEW StudentEducationOrganizationAssociationAddress_View AS
SELECT
    seoaa.Id,
    seoaa.DocumentUuid,
    seoaa.DocumentPartitionKey,

    -- Resolved StudentEducationOrganizationAssociation natural keys
    seoaa.StudentEducationOrganizationAssociation_Id,
    s.StudentUniqueId AS SEOA_Student_StudentUniqueId,
    eo.EducationOrganizationId AS SEOA_EducationOrganization_Id,

    -- Address fields (these form the natural key at this level)
    seoaa.AddressTypeDescriptor,
    seoaa.StreetNumberName,
    seoaa.City,
    seoaa.PostalCode
FROM StudentEducationOrganizationAssociationAddress seoaa
INNER JOIN StudentEducationOrganizationAssociation seoa
    ON seoaa.StudentEducationOrganizationAssociation_Id = seoa.Id
INNER JOIN Student s ON seoa.Student_Id = s.Id
INNER JOIN EducationOrganization eo ON seoa.EducationOrganization_Id = eo.Id;
```

#### StudentEducationOrganizationAssociationAddressPeriod_View

This second-level view resolves keys through the entire hierarchy:

```sql
CREATE VIEW StudentEducationOrganizationAssociationAddressPeriod_View AS
SELECT
    seoaap.Id,
    seoaap.DocumentUuid,
    seoaap.DocumentPartitionKey,

    -- Resolved Address natural keys (includes parent SEOA natural keys)
    seoaap.StudentEducationOrganizationAssociationAddress_Id,
    s.StudentUniqueId AS SEOA_Student_StudentUniqueId,
    eo.EducationOrganizationId AS SEOA_EducationOrganization_Id,
    seoaa.AddressTypeDescriptor AS Address_AddressTypeDescriptor,
    seoaa.StreetNumberName AS Address_StreetNumberName,

    -- Period fields
    seoaap.BeginDate,
    seoaap.EndDate
FROM StudentEducationOrganizationAssociationAddressPeriod seoaap
INNER JOIN StudentEducationOrganizationAssociationAddress seoaa
    ON seoaap.StudentEducationOrganizationAssociationAddress_Id = seoaa.Id
INNER JOIN StudentEducationOrganizationAssociation seoa
    ON seoaa.StudentEducationOrganizationAssociation_Id = seoa.Id
INNER JOIN Student s ON seoa.Student_Id = s.Id
INNER JOIN EducationOrganization eo ON seoa.EducationOrganization_Id = eo.Id;
```

### Generalized Pattern for View Generation

```typescript
function generateNaturalKeyView(entity: Entity): string {
    const viewName = `${entity.name}_View`;
    const selections = [];
    const joins = [];

    // Always include base columns
    selections.push('base.Id', 'base.DocumentUuid', 'base.DocumentPartitionKey');

    // For each foreign key reference
    entity.foreignKeys.forEach(fk => {
        // Include the surrogate key
        selections.push(`base.${fk.columnName}`);

        // Resolve to natural keys
        const referencedEntity = getEntity(fk.referencedTable);
        const alias = fk.referencedTable.toLowerCase();

        // Add join
        joins.push(`INNER JOIN ${fk.referencedTable} ${alias}
                    ON base.${fk.columnName} = ${alias}.Id`);

        // Add natural key columns with prefix
        referencedEntity.naturalKeys.forEach(nk => {
            const columnAlias = `${fk.columnName.replace('_Id', '')}_${nk.name}`;
            selections.push(`${alias}.${nk.name} AS ${columnAlias}`);
        });

        // Recursively handle parent natural keys if referenced entity has FKs
        if (referencedEntity.foreignKeys.length > 0) {
            // Recursively resolve parent natural keys
            resolveParentNaturalKeys(referencedEntity, selections, joins, fk.columnName.replace('_Id', ''));
        }
    });

    // Add entity's own fields
    entity.fields.forEach(field => {
        selections.push(`base.${field.name}`);
    });

    return `
        CREATE VIEW ${viewName} AS
        SELECT ${selections.join(',\n    ')}
        FROM ${entity.name} base
        ${joins.join('\n')};
    `;
}
```

### Benefits of This View Design

1. **Query Simplicity**: Users can query using business identifiers without knowing surrogate keys
2. **Self-Documenting**: Column names clearly indicate the source of each natural key
3. **Hierarchical Clarity**: Multi-level relationships are fully resolved
4. **Join Elimination**: Common queries no longer need complex joins
5. **Backwards Compatible**: Original surrogate key columns are retained for performance-critical queries

### Example Usage

Instead of writing complex joins:
```sql
-- Without views (complex)
SELECT *
FROM StudentSchoolAssociationEducationPlan ssaep
JOIN StudentSchoolAssociation ssa ON ssaep.StudentSchoolAssociation_Id = ssa.Id
JOIN Student s ON ssa.Student_Id = s.Id
JOIN School sch ON ssa.School_Id = sch.Id
WHERE s.StudentUniqueId = 'S123456'
  AND sch.SchoolId = 987
  AND ssa.EntryDate = '2024-09-01';
```

Users can simply query the view:
```sql
-- With views (simple)
SELECT *
FROM StudentSchoolAssociationEducationPlan_View
WHERE StudentSchoolAssociation_Student_StudentUniqueId = 'S123456'
  AND StudentSchoolAssociation_School_SchoolId = 987
  AND StudentSchoolAssociation_EntryDate = '2024-09-01';
```

This view design provides the relational access that customers expect while maintaining the performance benefits of surrogate keys in the underlying tables.
