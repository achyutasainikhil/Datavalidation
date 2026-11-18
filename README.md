Design QE Test Cases for Party Domain: Silver to Gold, Data Manager Consultation, Traceability and Dev Review

Suggested effort: 3, since the consultation adds work.

Description
As a Quality Engineer, I want to design QE test cases for the Party Domain Silver to Gold layer, consult the Party data managers for business input, finalize traceability and review the scenarios with the dev team, so that Party data can be validated end to end across all layers before development is completed.

Scope of work

Validate data integrity, consistency and transformation accuracy from Silver to Gold.

Validate business rules and critical business attributes.

Validate cross-table consistency across the Party data products.

Meet with the Party data managers to understand the business rules and critical attributes, and get input on the test cases and scenarios. Agree how QE will work with them going forward.

Identify volume and performance test scenarios where applicable.

Finalize the coverage and gap analysis across all layers.

Review the QE scenarios with the dev team in colo.

Acceptance criteria

QE test cases are documented for applicable Party Domain rules at the Silver to Gold layer.

Scenarios cover data integrity, completeness, consistency and business rules.

Positive, negative, boundary and exception scenarios are included.

Party data managers are identified and a consultation meeting is held on the business rules and test scenarios.

Business input from the data managers is captured and reflected in the test cases, and the way of working with them is agreed.

Test cases are traceable to Party Domain data contract rules, tables and columns where applicable.

A test coverage and gap analysis across Bronze, Silver and Gold is documented.

Additional QE scenarios beyond developer unit tests are identified, including any performance scenarios.

QE scenarios are reviewed with the dev team in colo and feedback is captured.

Dependencies: PBIs 1 and 2; names and availability of the Party data managers (Christine said she would give you the names); Party Domain Data Model and source-to-target mappings; dev team availability for the review.

Deliverables: Silver to Gold QE Test Case Document; notes from the data manager consultation; Test Coverage and Gap Analysis; additional QE validation scenarios.
