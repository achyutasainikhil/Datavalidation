Title: Party Core Data Quality Rules: Review New Rules

As a Data Product Team Member, I want to review the new Party Core data quality rules and get answers to the open design questions from Bob McIlvride, so that the contract rules are correct, the load design matches the data, and the rules can be tested and enforced.

Context: PBI 3586168 added 116 warning-level data quality rules to the Party Core silver contract (v0.5.0). Profiling of the RTMI and Iris bronze tables raised questions that only the contract owner can answer. Until they are answered, some rules cannot be finalised and the load design may be wrong for some data.


1. A review session is held with Bob McIlvride covering the 116 new rules in contract v0.5.0 and the Party Core DQ Work Log.
2. Decision D71 is answered: Iris sends two address rows for one customer in one delivery with no address type (20,705 cases). The agreed key or selection rule is recorded in the contract.
3. Decision D72 is answered: whether the RTMI loads read only open rows (EndDate 9999 or later). Bronze holds 705,704 closed rows beside 78,683,030 open rows.
4. Decision D73 is answered: what the 4,202,506 thirteen-digit RTMI member numbers are, and whether they can collide with Iris customer identifiers.
5. Decision D74 is answered: the SSN token format and range, including why 17 Iris values start with 1, 2, 5 or 6.
6. Severity decisions D26 and D65 are answered: which rules stay warning and which become critical.
7. Contract text errors are confirmed: StartDate described as null on all Iris rows (it is populated), ADDRESS_VALIDATION_DATE empty on every RTMI row, and SUFFIX_NAME holding 198 distinct values in a 5-character column.
8. Objects named in the contract but missing from it are listed, with a plan to add them.
9. Dev silver tables and access for running the new rules are agreed, including Contributor access to lh_bronze.
10. Answers are recorded in the Questions tab of the work log and the contract is updated or flagged for update.
