# Journey 242 — Ontology Update & Element Governance

## Overview
**Duration**: 9–11 minutes  
**Primary Personas**: Eve (Ontology Steward), Bob (Reviewer)  
**Supporting Personas**: Automation Bot, Modeling Team  
**Complexity**: High

**Business Value**: Ensures ontology extensions follow governance—importing a new ArchiMate catalog, validating relationship cardinality, approving `OntologyAssociation`, and propagating updates downstream.

---

## Prerequisites
- Existing `OntologyCatalogue` version `archimate-3.1` approved.
- Proposed catalogue update `archimate-3.2` with new `capability_serves_value_stream` relationship (cardinality `one_to_many`).
- Modeling tool export available for import.
- Ontology governance board defined in `GovernanceOwnerGroup` (“Ontology Council”).

---

## Journey Steps

### Phase 1: Import Proposed Catalogue (Eve)
1. Eve uploads the new ontology package via Steward console.
2. System stages `OntologyCatalogue` version `archimate-3.2` with status `draft`.
3. Automated validation checks relationship schemas and compares to existing elements.

**Expected Outcome**
- Draft catalogue stored with metadata (source, checksum).
- Validation warnings recorded for impacted relationships.
- `GovernanceNotification` issued to Ontology Council (`notification_type=decision_request`).

### Phase 2: Council Review & Cardinality Enforcement (Bob)
1. Bob inspects staged relationships, focusing on `capability_serves_value_stream`.
2. Bob approves catalogue; governance engine updates `OntologyCatalogue.status=approved`.
3. Existing `RepositoryElementRelationship` rows validated; any that violate new cardinality flagged.

**Expected Outcome**
- Catalogue version `archimate-3.2` approved.
- Relationship cardinality enforcement job highlights offending relationships (if any) and queues remediation tasks.
- `GovernanceRecord` captures approval meeting notes.

**Test Assertion**
```typescript
const catalogue = await db.findOne('ontology_catalogues', {
  graph_id: enterpriseRepo.id,
  ontology_source: 'archimate',
  version: '3.2'
});
expect(catalogue.status).toBe('approved');

const relationships = await db.find('graph_element_relationships', {
  relationship_type: 'capability_serves_value_stream'
});
expect(relationships.every(rel => rel.cardinality === 'one_to_many')).toBe(true);
```

### Phase 3: Create Cross-Ontology Association (Modeling Team)
1. Modeling team links a capability element to a BPMN process using new semantics.
2. System stores `OntologyAssociation` (`association_type=realises`) with justification and authority reference.
3. Governance bot checks association against catalogue; status `approved` automatically because template matches.

**Expected Outcome**
- `OntologyAssociation` recorded with governance status `approved`.
- `TraceLink` / knowledge graph updated with new cross-ontology connection.
- Notifications sent to impacted transformation owners.

### Phase 4: Propagate Updates & Telemetry
1. Repository sync job publishes updated ontology structures to knowledge graph and search index.
2. Telemetry captures catalogue version adoption in `RepositoryStatisticSnapshot` and `GovernanceMetricSnapshot` (ontology coverage KPI).
3. Impact analysis report lists elements affected by new relationships.

**Expected Outcome**
- Sync job `status=succeeded` with counts of ontology entities updated.
- KPI trend shows ontology coverage improvement.
- Impact report stored as `AssetVersion` (model documentation).

---

## Telemetry & KPIs
- Ontology coverage percentage.
- Time from catalogue import to approval.
- Number of element relationships audited.
- Cross-ontology association count.

---

## FR/NFR Coverage
- **FR-18**: Ontology and relationship management.
- **FR-20**: Governance audit trail for ontology changes.
- **NFR-Obs-01**: Telemetry visibility into ontology adoption.
- **NFR-Compliance-02**: Cardinality enforcement.

---

## Related Journeys
- **Journey 233**: Initiative planning uses updated ontologies.
- **Journey 235**: Landscape views leverage new relationships.
- **Journey 237**: Bulk import may rely on refreshed ontology definitions.

---

## Revision History
- **2025-02-XX**: Initial draft covering ontology stewardship workflows.
