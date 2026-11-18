PBI 2: Design QE Test Cases for Party Domain: Bronze to Silver

Suggested effort: 2

Description
As a Quality Engineer, I want to design QE test cases for the Party Domain Bronze to Silver layer based on the Data Contracts, data model, profiling results and rule mapping, so that data quality, integrity and completeness can be validated before development is completed.

Scope of work

Validate primary key and business key uniqueness.
Validate NULL, missing and duplicate Party records.
Validate schema, data types and mandatory attributes.
Validate completeness and record count reconciliation between Bronze and Silver.
Validate invalid-record handling, including quarantine routing and exception scenarios.
Include positive, negative, boundary and edge-case scenarios.

Acceptance criteria

QE test cases are documented for applicable Party Domain rules at the Bronze to Silver layer.

Scenarios cover uniqueness, NULL handling, duplicate detection, schema validation and completeness.

Positive, negative, boundary and exception scenarios are included.

Invalid Party records and quarantine handling are covered, including that good records are not quarantined and bad records do not reach Silver.

Each test case is traceable to a Party Domain data contract rule, table and column where applicable.

Additional scenarios beyond developer unit tests are identified and marked separately.

Test cases are documented in a QE test case format agreed with the team.

Dependencies: PBI 1 (rule mapping); Party Domain Data Contract; data profiling results; QE test case template or format.

Deliverables: Bronze to Silver QE Test Case Document.
