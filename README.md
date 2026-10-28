1. The Party Core silver data contract (ODCS v3.1.0) is extended with executable data quality rules for the RTMI and Iris loads. Contract version 0.5.0 adds 116 rules to the existing 281 (397 in total).
2. The rules cover uniqueness, foreign key integrity (orphan checks), hash key recomputation, blank and null handling, accepted value sets for code columns, format checks (ZIP, latitude, longitude, member number, SSN token shape), plausible date ranges, and cross-column consistency.
3. Scope follows Christine McWilliams' direction (14 Oct): hard data quality rules, not business rules.
4. Rules follow the contract's existing convention: id, name, description, severity, dimension, and SQL that returns offending rows (zero rows = pass). All new rules are severity "warning" pending decisions D26 and D65.
5. Rules were designed from profiling of the bronze tables (rtmi_individual and iris_customer) in Fabric. Findings are documented in the Party Core DQ Work Log.
6. Contract changelog (v0.5.0) and open decisions D71 to D74 are recorded in the contract.
7. Open questions for the data product owner and the vault design owner are documented, and the remaining work is tracked in a follow-on PBI.
