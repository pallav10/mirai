# 0005 — HAPI FHIR for the interoperability and ingest layer

- **Status:** Accepted
- **Date:** 2026-08-29
- **Specs:** [09-data-model](../specs/09-data-model.md), [11-integrations](../specs/11-integrations.md), [12-api-and-interoperability](../specs/12-api-and-interoperability.md)

## Context

The data model is FHIR-aligned, ingest of the existing client's patient data arrives via the REST/FHIR path, and the platform's stated evolution is toward a full interoperability surface (HL7 v2 breadth, ABDM/national federation). With a real client's data landing on day one, FHIR validation, resource handling, and terminology cannot be hand-rolled.

## Decision

**HAPI FHIR** (Java) is the FHIR engine, embedded in the integrations service (ADR 0004):

- HAPI's parser/validator handles all inbound FHIR resources — structural validation, profile validation against Atlas profiles, terminology checks — before anything enters the processing pipeline.
- Atlas's internal canonical model remains its own (spec 09); HAPI is the boundary translator, not the system of record. Mapping FHIR ↔ canonical model lives in one place, in the integrations service.
- Outbound FHIR (the API's interoperability surface, spec 12) is generated through the same library, one direction of the same mapping.

## Consequences

- This decision forces the JVM service tier (ADR 0003); the two stand or fall together.
- Atlas FHIR profiles (which fields, which extensions, which code systems) become a maintained artifact with the same review discipline as the API contract.
- Adding HL7 v2 later is an adapter in front of the same pipeline, as the architecture spec's evolution criteria require.

## Alternatives rejected

- **Hand-rolled FHIR handling (any language):** FHIR's surface (resource types, profiles, terminology, versioning) is years of edge cases; rebuilding it is undifferentiated risk with real patient data in the pipeline.
- **Medplum / TypeScript FHIR stacks:** credible open-source work, but adopting one as the ingest engine couples the platform's compliance-critical boundary to a far smaller ecosystem than HAPI's two decades of production use.
- **Standalone FHIR server (HAPI JPA server) as the record store:** Atlas's consent/QR/tenancy model demands its own canonical store; a FHIR-native store would bend the core model around the exchange format.
