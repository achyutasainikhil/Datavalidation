# ═══════════════════════════════════════════════════════════════════════════════════════════
# Open Data Contract Standard (ODCS) v3.1.0
# Party Core Raw Vault - Silver Layer Data Contract
# Source: bronze.rtmi_individual, bronze.iris_customer  ->  Target: Microsoft Fabric Lakehouse lh_silver
# ═══════════════════════════════════════════════════════════════════════════════════════════

# ───────────────────────────────────────────────────────────────────────────────
# 1. FUNDAMENTALS
# ───────────────────────────────────────────────────────────────────────────────
apiVersion: v3.1.0
kind: DataContract

id: dc-party-party-core-silver-001
name: Party Core silver raw vault data contract
version: 0.3.0
status: draft
domain: party
dataProduct: party-core
tenant: TruStage

description:
  purpose: >-
    Governs the bronze to silver load of Party Core. It defines the raw vault and error mart tables the
    Party Core load writes from bronze.rtmi_individual and bronze.iris_customer, where each column comes
    from, and the checks every load is held to.
  limitations: >-
    Covers the Party Core crawl build: 26 raw vault tables in schema party_raw_vault and one error mart
    table in schema party_ops, both in lakehouse lh_silver. Every satellite boundary is provisional until
    the second profiling pass measures its rate of change, which is blocking question 5 of the Party Core
    design. The business vault and gold are covered by their own data contracts. Decisions D26 and D65
    are open, and the severities here follow their proposals.
  usage: >-
    The Party Core load validates each load into lh_silver against this data contract and writes every
    result to the data quality log. Only Party Core's own load and delivery runs read these tables. A
    column's transformSourceObjects names the bronze column it is loaded from, and its transformDescription
    says how.

# ───────────────────────────────────────────────────────────────────────────────
# 2. SCHEMA
# ───────────────────────────────────────────────────────────────────────────────
schema:
- id: hub_party_obj
  name: hub_party
  physicalName: party_raw_vault.hub_party
  physicalType: table
  businessName: Hub party
  description: >-
    This hub is the party spine. It holds one row per distinct anchoring business key ever seen, from
    every system that presents a party key, and the load only inserts into it.
  dataGranularityDescription: One row per distinct anchoring business key ever seen
  properties:
  - id: hub_party__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      SHA-256 hash key. Primary key. Pre-image: sha256(lower(trim(party_business_key_domain_code)) ||
      '|' || lower(trim(party_business_key)))
    classification: Neither
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: hub_party__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: hub_party__party_business_key_prop
    name: party_business_key
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      RTMI: The RTMI member number, the anchoring MDM identifier of an RTMI-anchored party. Iris: The
      Iris customer identifier, the anchoring key of an Iris-anchored party.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IRIS_ID
    - bronze.rtmi_individual.RTMI_MEMBER_NUMBER
    transformDescription: >-
      RTMI: Cast to STRING, which is the target type. The value is otherwise unchanged. party_business_key_domain_code
      beside it is the RTMI MDM domain constant, and party_hk is the hash over the pair. Iris: Loaded
      as supplied. The Iris customer identifier is the business key of the party this feed anchors, and
      party_business_key_domain_code beside it is the Iris customer domain.
    quality:
    - id: hub_party__party_business_key_rule_1
      type: text
      description: >-
        RTMI: A duplicate member number within a file is reported, and decision D23 says what the load
        does with it.
      severity: warning
  - id: hub_party__party_business_key_domain_code_prop
    name: party_business_key_domain_code
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to Reference Data. Business key collision code naming the issuing authority: IRIS_MDM, RTMI_MDM,
      NAIC_COMPANY, IRS_FEIN, CUIC_ORG_CONTRACT, NIPR_NPN, CMS_NPI, DNB_DUNS, GLEIF_LEI, FRB_RSSD, FDIC_CERT,
      plus one value per partner mastering regime
    classification: Neither
    transformDescription: >-
      RTMI: One constant for every row this feed writes: the configured value for the RTMI MDM individual
      domain. The value has to be agreed in the load configuration list decision D18 holds before the
      first load. Iris: One constant for every row this feed writes: the configured value for the Iris
      customer domain. The value has to be agreed in the load configuration list decision D18 holds before
      the first load.
  - id: hub_party__load_date_prop
    name: load_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: When the key was first presented to the platform
    classification: Neither
    transformDescription: >-
      RTMI: When the key was first presented to the platform Iris: When the key was first presented to
      the platform
  - id: hub_party__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to the System Registry. The system that first presented the key
    classification: Neither
    transformDescription: >-
      RTMI: The System Registry code for this feed, one constant for every row the table holds. It is
      not a key column, because the table name carries the source. Iris: The System Registry code for
      this feed, one constant for every row the table holds. It is not a key column, because the table
      name carries the source.
  quality:
  - id: hub_party_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: Each party_hk appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk
      FROM lh_silver.party_raw_vault.hub_party
      GROUP BY party_hk
      HAVING COUNT(*) > 1
  - id: hub_party_business_key_unique
    type: custom
    engine: sql
    name: Unique business key
    description: >-
      Each party_business_key, party_business_key_domain_code combination appears on one row. This is
      load test 1 in the load specification. Returns each duplicated combination.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_business_key, party_business_key_domain_code
      FROM lh_silver.party_raw_vault.hub_party
      GROUP BY party_business_key, party_business_key_domain_code
      HAVING COUNT(*) > 1
  - id: hub_party_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, party_business_key, party_business_key_domain_code,
      load_date, record_source. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk
      FROM lh_silver.party_raw_vault.hub_party
      WHERE party_hk IS NULL
         OR party_business_key IS NULL
         OR party_business_key_domain_code IS NULL
         OR load_date IS NULL
         OR record_source IS NULL
  - id: hub_party_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk is 64 lowercase hexadecimal characters. This is load test 5 in the load
      specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk
      FROM lh_silver.party_raw_vault.hub_party
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
  - id: hub_party_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.hub_party
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: hub_party_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.hub_party AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
- id: hub_party_identifier_obj
  name: hub_party_identifier
  physicalName: party_raw_vault.hub_party_identifier
  physicalType: table
  businessName: Hub party identifier
  description: >-
    This hub holds every identifier that refers to a party as a thing in its own right, separate from
    the fact of its attachment, which is entity 5.
  dataGranularityDescription: One row per identifier under an issuing authority and namespace
  properties:
  - id: hub_party_identifier__party_identifier_hk_prop
    name: party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      SHA-256 hash key. Primary key. Pre-image: sha256(lower(trim(issuing_authority_code)) || '|' || lower(trim(key_namespace))
      || '|' || lower(trim(identifier_type_code)) || '|' || lower(trim(identifier_value)))
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: hub_party_identifier__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: hub_party_identifier__identifier_type_code_prop
    name: identifier_type_code
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to Reference Data. MDM_INDIVIDUAL_ID, MPI_GUID, POLICY_NUMBER, APPLICATION_NUMBER, EMAIL_HASH,
      PHONE_HASH, SSN_HASH, TIN_HASH, CRM_CONTACT_ID, SSO_SUB, BILLING_ACCOUNT_ID, MEMBER_ID, NAIC_COMPANY_CODE,
      NAIC_GROUP_CODE, FEIN, LEI, DUNS, NCUA_CHARTER_NUMBER, RSSD_ID, FDIC_CERT, CRD_NUMBER, SEC_FILE_NUMBER,
      CIK, NPN, NPI, CUIC_ORG_CONTRACT_NUMBER, FEC_COMMITTEE_ID, RTMI_HOUSEHOLD_ID, MERLIN_HOUSEHOLD_ID
    classification: Neither
    transformDescription: >-
      RTMI: One configured constant per load path: the anchoring identifier type on one occurrence and
      the SSN identifier type on the other. The value has to be agreed in the load configuration list
      decision D18 holds before the first load. Iris: One configured constant per load path: the anchoring
      identifier type on one occurrence and the SSN identifier type on the other. The value has to be
      agreed in the load configuration list decision D18 holds before the first load.
  - id: hub_party_identifier__identifier_value_prop
    name: identifier_value
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      RTMI, anchoring MDM identifier occurrence: The RTMI member number held as an identifier in its own
      right. RTMI, sSN occurrence/Iris, sSN occurrence: The tokenized social security number held as an
      identifier of the party. Iris, iris customer identifier occurrence: The Iris customer identifier
      held as an identifier in its own right.
    transformSourceObjects:
    - bronze.iris_customer.IRIS_ID
    - bronze.iris_customer.SSN_TOKEN
    - bronze.rtmi_individual.RTMI_MEMBER_NUMBER
    - bronze.rtmi_individual.SSN_TOKEN
    transformDescription: >-
      RTMI, anchoring MDM identifier occurrence: Cast to STRING. One hub row carrying the MDM individual
      identifier type, RTMI as issuing authority, and the RTMI MDM key namespace. That row joins hub_party
      through lnk_party_anchor_identifier as well as through lnk_party_identifier. RTMI, sSN occurrence:
      Loaded as supplied. One hub row per distinct SSN token, carrying the SSN identifier type, its issuing
      authority and the SSN token namespace. The token is already tokenized, so is_tokenized_flag carries
      TRUE on this occurrence and the attachment reaches the party through lnk_party_identifier only.
      Iris, iris customer identifier occurrence: Loaded as supplied. One hub row carrying the Iris customer
      identifier type, Iris as issuing authority and the Iris customer key namespace. Iris, sSN occurrence:
      Loaded as supplied. One hub row per distinct SSN token. It is the column the MPI cross-reference
      pass uses to find that an Iris customer and an RTMI member are one person.
    quality:
    - id: hub_party_identifier__identifier_value_rule_1
      type: text
      description: 'RTMI: Reported where blank on a row carrying other identifying data.'
      severity: warning
    - id: hub_party_identifier__identifier_value_rule_2
      type: text
      description: 'Iris: Reported where blank.'
      severity: warning
  - id: hub_party_identifier__issuing_authority_code_prop
    name: issuing_authority_code
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to Reference Data. The authority that issued the value: IRIS, RTMI, MPI, MERLIN, CUIC, NAIC,
      IRS, GLEIF, DNB, NCUA, FRB, FDIC, FINRA, SEC, NIPR, CMS, FEC. Part of the key, because one identifier
      type is issued by two authorities
    classification: Neither
    transformDescription: >-
      RTMI: One configured constant per load path, naming the authority behind each identifier. The value
      has to be agreed in the load configuration list decision D18 holds before the first load. Iris:
      One configured constant per load path, naming the authority behind each identifier. The value has
      to be agreed in the load configuration list decision D18 holds before the first load.
  - id: hub_party_identifier__key_namespace_prop
    name: key_namespace
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      Scope qualifier where one identifier type originates from more than one instance of a system. Stored
      as the empty string for a globally scoped identifier, because it sits inside the hash key
    classification: Neither
    transformDescription: >-
      RTMI: One configured constant per load path. The token namespace matters, because the same person
      tokenized under two vendors gives two values and therefore two hub rows. The value has to be agreed
      in the load configuration list decision D18 holds before the first load. Iris: One configured constant
      per load path. The token namespace matters, because the same person tokenized under two vendors
      gives two values and therefore two hub rows. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: hub_party_identifier__load_date_prop
    name: load_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: When the identifier was first presented
    classification: Neither
    transformDescription: >-
      RTMI: When the identifier was first presented Iris: When the identifier was first presented
  - id: hub_party_identifier__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to the System Registry. The system that first presented it
    classification: Neither
    transformDescription: >-
      RTMI: The System Registry code for this feed, one constant for every row the table holds. It is
      not a key column, because the table name carries the source. Iris: The System Registry code for
      this feed, one constant for every row the table holds. It is not a key column, because the table
      name carries the source.
  quality:
  - id: hub_party_identifier_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: Each party_identifier_hk appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_identifier_hk
      FROM lh_silver.party_raw_vault.hub_party_identifier
      GROUP BY party_identifier_hk
      HAVING COUNT(*) > 1
  - id: hub_party_identifier_business_key_unique
    type: custom
    engine: sql
    name: Unique business key
    description: >-
      Each identifier_type_code, identifier_value, issuing_authority_code, key_namespace combination appears
      on one row. Returns each duplicated combination.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT identifier_type_code, identifier_value, issuing_authority_code, key_namespace
      FROM lh_silver.party_raw_vault.hub_party_identifier
      GROUP BY identifier_type_code, identifier_value, issuing_authority_code, key_namespace
      HAVING COUNT(*) > 1
  - id: hub_party_identifier_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_identifier_hk, identifier_type_code, identifier_value,
      issuing_authority_code, key_namespace, load_date, record_source. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_identifier_hk
      FROM lh_silver.party_raw_vault.hub_party_identifier
      WHERE party_identifier_hk IS NULL
         OR identifier_type_code IS NULL
         OR identifier_value IS NULL
         OR issuing_authority_code IS NULL
         OR key_namespace IS NULL
         OR load_date IS NULL
         OR record_source IS NULL
  - id: hub_party_identifier_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_identifier_hk is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_identifier_hk
      FROM lh_silver.party_raw_vault.hub_party_identifier
      WHERE NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
  - id: hub_party_identifier_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_identifier_hk is the zero key. This is load test 6 in
      the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.hub_party_identifier
      WHERE party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: hub_party_identifier_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.hub_party_identifier AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
- id: lnk_party_identifier_obj
  name: lnk_party_identifier
  physicalName: party_raw_vault.lnk_party_identifier
  physicalType: table
  businessName: Link party identifier
  description: >-
    This link holds one row per party and identifier pair. It is the crosswalk edge, and the construct
    the Consumer Key Ring identifier view projects from.
  dataGranularityDescription: One row per party and identifier pair
  properties:
  - id: lnk_party_identifier__lnk_party_identifier_hk_prop
    name: lnk_party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      SHA-256 hash key. Primary key. Pre-image: sha256(party_hk || '|' || party_identifier_hk)
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_identifier__lnk_party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_identifier__party_hk_prop
    name: party_hk
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to Hub Party
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_identifier__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_identifier__party_identifier_hk_prop
    name: party_identifier_hk
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to Hub Party Identifier
    relationships:
    - type: foreignKey
      to: hub_party_identifier.party_identifier_hk
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_identifier__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_identifier__load_date_prop
    name: load_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: When the attachment first reached the platform
    classification: Neither
    transformDescription: >-
      RTMI: When the attachment first reached the platform Iris: When the attachment first reached the
      platform
  - id: lnk_party_identifier__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to the System Registry
    classification: Neither
    transformDescription: >-
      RTMI: The System Registry code for this feed, one constant for every row the table holds. It is
      not a key column, because the table name carries the source. Iris: The System Registry code for
      this feed, one constant for every row the table holds. It is not a key column, because the table
      name carries the source.
  quality:
  - id: lnk_party_identifier_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: Each lnk_party_identifier_hk appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT lnk_party_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_identifier
      GROUP BY lnk_party_identifier_hk
      HAVING COUNT(*) > 1
  - id: lnk_party_identifier_business_key_unique
    type: custom
    engine: sql
    name: Unique business key
    description: >-
      Each party_hk, party_identifier_hk combination appears on one row. Returns each duplicated combination.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, party_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_identifier
      GROUP BY party_hk, party_identifier_hk
      HAVING COUNT(*) > 1
  - id: lnk_party_identifier_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: lnk_party_identifier_hk, party_hk, party_identifier_hk,
      load_date, record_source. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT lnk_party_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_identifier
      WHERE lnk_party_identifier_hk IS NULL
         OR party_hk IS NULL
         OR party_identifier_hk IS NULL
         OR load_date IS NULL
         OR record_source IS NULL
  - id: lnk_party_identifier_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in lnk_party_identifier_hk, party_hk, party_identifier_hk is 64 lowercase hexadecimal
      characters. This is load test 5 in the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT lnk_party_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_identifier
      WHERE NOT lnk_party_identifier_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
  - id: lnk_party_identifier_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose lnk_party_identifier_hk is the zero key. This is load test
      6 in the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.lnk_party_identifier
      WHERE lnk_party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: lnk_party_identifier_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.lnk_party_identifier AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.lnk_party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
- id: lnk_party_anchor_identifier_obj
  name: lnk_party_anchor_identifier
  physicalName: party_raw_vault.lnk_party_anchor_identifier
  physicalType: table
  businessName: Link party anchor identifier
  description: >-
    This link holds one row per party: the identifier its hub key hashes. The load path that anchors the
    party writes it, from the identifier its load configuration names.
  dataGranularityDescription: One row per party per identifier that has ever been its anchor
  properties:
  - id: lnk_party_anchor_identifier__lnk_party_anchor_identifier_hk_prop
    name: lnk_party_anchor_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      SHA-256 hash key. Primary key. Pre-image: sha256(party_hk || '|' || party_identifier_hk)
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_anchor_identifier__lnk_party_anchor_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_anchor_identifier__party_hk_prop
    name: party_hk
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to Hub Party
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_anchor_identifier__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_anchor_identifier__party_identifier_hk_prop
    name: party_identifier_hk
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to Hub Party Identifier
    relationships:
    - type: foreignKey
      to: hub_party_identifier.party_identifier_hk
    transformDescription: >-
      RTMI: Computed at load from the business key of its construct, or carried unchanged from the parent.
      SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section of the load
      specification. Iris: Computed at load from the business key of its construct, or carried unchanged
      from the parent. SHA-256 over a pipe-delimited pre-image, to the convention in the Hash keys section
      of the load specification.
    quality:
    - id: lnk_party_anchor_identifier__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: lnk_party_anchor_identifier__load_date_prop
    name: load_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: When the anchor designation reached the platform
    classification: Neither
    transformDescription: >-
      RTMI: When the anchor designation reached the platform Iris: When the anchor designation reached
      the platform
  - id: lnk_party_anchor_identifier__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to the System Registry
    classification: Neither
    transformDescription: >-
      RTMI: The System Registry code for this feed, one constant for every row the table holds. It is
      not a key column, because the table name carries the source. Iris: The System Registry code for
      this feed, one constant for every row the table holds. It is not a key column, because the table
      name carries the source.
  quality:
  - id: lnk_party_anchor_identifier_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each lnk_party_anchor_identifier_hk appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT lnk_party_anchor_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier
      GROUP BY lnk_party_anchor_identifier_hk
      HAVING COUNT(*) > 1
  - id: lnk_party_anchor_identifier_business_key_unique
    type: custom
    engine: sql
    name: Unique business key
    description: >-
      Each party_hk, party_identifier_hk combination appears on one row. Returns each duplicated combination.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, party_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier
      GROUP BY party_hk, party_identifier_hk
      HAVING COUNT(*) > 1
  - id: lnk_party_anchor_identifier_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: lnk_party_anchor_identifier_hk, party_hk, party_identifier_hk,
      load_date, record_source. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT lnk_party_anchor_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier
      WHERE lnk_party_anchor_identifier_hk IS NULL
         OR party_hk IS NULL
         OR party_identifier_hk IS NULL
         OR load_date IS NULL
         OR record_source IS NULL
  - id: lnk_party_anchor_identifier_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in lnk_party_anchor_identifier_hk, party_hk, party_identifier_hk is 64 lowercase hexadecimal
      characters. This is load test 5 in the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT lnk_party_anchor_identifier_hk
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier
      WHERE NOT lnk_party_anchor_identifier_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
  - id: lnk_party_anchor_identifier_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose lnk_party_anchor_identifier_hk is the zero key. This is load
      test 6 in the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier
      WHERE lnk_party_anchor_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: lnk_party_anchor_identifier_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.lnk_party_anchor_identifier AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.lnk_party_anchor_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
- id: sat_party_identity_rtmi_obj
  name: sat_party_identity_rtmi
  physicalName: party_raw_vault.sat_party_identity_rtmi
  physicalType: table
  businessName: Party identity satellite (RTMI)
  description: >-
    This satellite carries the near-immutable identity attributes. The party type never changes and the
    first established date moves only when an earlier relationship is discovered, so grouping them with
    status would rewrite them on every status change. Split axis: producer: near-immutable identity, written
    by the delivering feed. This copy holds the RTMI rows only. Column names and datatypes are the ones
    RTMI delivers. PROVISIONAL: the rate-of- change boundary stays unmeasured until the second profiling
    pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_identity_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_identity_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_identity_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_identity_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_identity_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_identity_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_identity_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_identity_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_identity_rtmi__party_type_code_prop
    name: party_type_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. PERSON or ORGANIZATION. The primary routing discriminator
    classification: Neither
    transformDescription: >-
      One constant for every row this feed writes: the configured value for a person. The value has to
      be agreed in the load configuration list decision D18 holds before the first load.
  - id: sat_party_identity_rtmi__origin_timestamp_prop
    name: origin_timestamp
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: Earliest date TruStage had any relationship with this party.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      Loaded as supplied, as the TIMESTAMP RTMI sends, so the time of day RTMI sent is kept.
  quality:
  - id: sat_party_identity_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_identity_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_identity_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_identity_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_identity_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_identity_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_identity_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_identity_rtmi__origin_timestamp_not_after_load_date
    type: custom
    engine: sql
    name: origin_timestamp not later than the load date
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_rtmi
      WHERE origin_timestamp > load_date
- id: sat_party_identity_iris_obj
  name: sat_party_identity_iris
  physicalName: party_raw_vault.sat_party_identity_iris
  physicalType: table
  businessName: Party identity satellite (Iris)
  description: >-
    This satellite carries the near-immutable identity attributes. The party type never changes and the
    first established date moves only when an earlier relationship is discovered, so grouping them with
    status would rewrite them on every status change. Split axis: producer: near-immutable identity, written
    by the delivering feed. This copy holds the Iris rows only. Column names and datatypes are the ones
    Iris delivers. PROVISIONAL: the rate-of- change boundary stays unmeasured until the second profiling
    pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_identity_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_identity_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_identity_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_identity_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_identity_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_identity_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_identity_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_identity_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_identity_iris__party_type_code_prop
    name: party_type_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. PERSON or ORGANIZATION. The primary routing discriminator
    classification: Neither
    transformDescription: >-
      One constant for every row this feed writes: the configured value for a person. The value has to
      be agreed in the load configuration list decision D18 holds before the first load.
  - id: sat_party_identity_iris__start_date_prop
    name: start_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: The date the Iris customer record came into effect.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.StartDate
    transformDescription: >-
      Loaded as supplied, as the TIMESTAMP Iris sends. The RTMI feed writes this column from a member
      origin and this feed writes it from a source record effectivity, which are different kinds of date.
  quality:
  - id: sat_party_identity_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_identity_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_identity_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE hash_diff IS NULL
  - id: sat_party_identity_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_identity_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_identity_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_identity_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_identity_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_identity_iris__start_date_not_after_load_date
    type: custom
    engine: sql
    name: start_date not later than the load date
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identity_iris
      WHERE start_date > load_date
- id: sat_party_status_rtmi_obj
  name: sat_party_status_rtmi
  physicalName: party_raw_vault.sat_party_status_rtmi
  physicalType: table
  businessName: Party status satellite (RTMI)
  description: >-
    This satellite carries the lifecycle status as delivered. Deceased, dissolved, and inactive all land
    here, and the two columns move far more often than anything on entity 19. Split axis: producer: delivered
    lifecycle status. This copy holds the RTMI rows only. Column names and datatypes are the ones RTMI
    delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured until the second profiling pass
    runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_status_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_status_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_status_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_status_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_status_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_status_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_status_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_status_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_status_rtmi__deceased_indicator_prop
    name: deceased_indicator
    logicalType: string
    physicalType: STRING
    required: false
    description: Status as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.DECEASED_INDICATOR
    transformDescription: >-
      Loaded as supplied. This column holds the status as the source represents it, so the RTMI indicator
      lands here unchanged and bsat_party_conformed_status conforms it.
    quality:
    - id: sat_party_status_rtmi__deceased_indicator_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_status_rtmi__date_of_death_prop
    name: date_of_death
    logicalType: date
    physicalType: DATE
    required: false
    description: Date the status took effect in the source.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.DATE_OF_DEATH
    transformDescription: >-
      Loaded as supplied. A sat_party_status row is written when the feed carries a status payload.
  quality:
  - id: sat_party_status_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_status_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_status_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_status_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_status_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_status_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_status_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_status_rtmi__date_of_death_not_after_load_date
    type: custom
    engine: sql
    name: date_of_death not later than the load date or date_of_death
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_status_rtmi
      WHERE date_of_death > load_date
- id: sat_party_name_pii_rtmi_obj
  name: sat_party_name_pii_rtmi
  physicalName: party_raw_vault.sat_party_name_pii_rtmi
  physicalType: table
  businessName: Party name PII satellite (RTMI)
  description: >-
    This satellite carries the person name parts. The split lets column-level security and masking policies
    apply at table grain without masking a credit union's legal name. Split axis: PII classification,
    split from the non-PII columns of sat_party_name. This copy holds the RTMI rows only. Column names
    and datatypes are the ones RTMI delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured
    until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_name_pii_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_name_pii_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_name_pii_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_name_pii_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_name_pii_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_name_pii_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_name_pii_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_name_pii_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_name_pii_rtmi__prefix_prop
    name: prefix
    logicalType: string
    physicalType: STRING
    required: false
    description: Mr, Ms, Dr. Empty for organizations
    classification: PII-Direct
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_name_pii_rtmi__first_name_prop
    name: first_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Given name as supplied on the RTMI member file.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.FIRST_NAME
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
    quality:
    - id: sat_party_name_pii_rtmi__first_name_rule_1
      type: text
      description: Reported where blank.
      severity: warning
  - id: sat_party_name_pii_rtmi__middle_name_prop
    name: middle_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Middle name or initial as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.MIDDLE_NAME
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_name_pii_rtmi__last_name_prop
    name: last_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Family name as supplied on the RTMI member file.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.LAST_NAME
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
    quality:
    - id: sat_party_name_pii_rtmi__last_name_rule_1
      type: text
      description: Reported where blank.
      severity: warning
  - id: sat_party_name_pii_rtmi__suffix_name_prop
    name: suffix_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Name suffix as RTMI supplies it, carrying the suffix alone.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.SUFFIX_NAME
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_name_pii_rtmi__generation_prop
    name: generation
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      Generation as RTMI supplies it, held apart from the suffix so neither value is destroyed by the
      other.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.GENERATION
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  quality:
  - id: sat_party_name_pii_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_name_pii_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_name_pii_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_name_pii_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_name_pii_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_name_pii_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_name_pii_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_name_pii_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_name_pii_iris_obj
  name: sat_party_name_pii_iris
  physicalName: party_raw_vault.sat_party_name_pii_iris
  physicalType: table
  businessName: Party name PII satellite (Iris)
  description: >-
    This satellite carries the person name parts. The split lets column-level security and masking policies
    apply at table grain without masking a credit union's legal name. Split axis: PII classification,
    split from the non-PII columns of sat_party_name. This copy holds the Iris rows only. Column names
    and datatypes are the ones Iris delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured
    until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_name_pii_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_name_pii_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_name_pii_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_name_pii_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_name_pii_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_name_pii_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_name_pii_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_name_pii_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_name_pii_iris__prefix_prop
    name: prefix
    logicalType: string
    physicalType: STRING
    required: false
    description: Mr, Ms, Dr. Empty for organizations
    classification: PII-Direct
    transformDescription: No Iris candidate. bronze.iris_customer carries no column for it.
  - id: sat_party_name_pii_iris__first_name_prop
    name: first_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Given name as supplied on the Iris customer record.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.FIRST_NAME
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
    quality:
    - id: sat_party_name_pii_iris__first_name_rule_1
      type: text
      description: Reported where blank.
      severity: warning
  - id: sat_party_name_pii_iris__middle_name_prop
    name: middle_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Middle name or initial as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.MIDDLE_NAME
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_name_pii_iris__last_name_prop
    name: last_name
    logicalType: string
    physicalType: STRING
    required: false
    description: Family name as supplied on the Iris customer record.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.LAST_NAME
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
    quality:
    - id: sat_party_name_pii_iris__last_name_rule_1
      type: text
      description: Reported where blank.
      severity: warning
  - id: sat_party_name_pii_iris__generation_name_prop
    name: generation_name
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      Generation as supplied. The Iris column names the generation, so it lands on the generation column
      rather than on suffix, which now holds the suffix alone under decision D7.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.GENERATION_NAME
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  quality:
  - id: sat_party_name_pii_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_name_pii_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_name_pii_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      WHERE hash_diff IS NULL
  - id: sat_party_name_pii_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_name_pii_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_name_pii_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_name_pii_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_name_pii_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_address_rtmi_obj
  name: sat_party_address_rtmi
  physicalName: party_raw_vault.sat_party_address_rtmi
  physicalType: table
  businessName: Party address satellite (RTMI)
  description: >-
    This satellite carries the delivered address except the street lines and the geocode, which carry
    a different classification and sit on entity 26, and it carries the household identifier the source
    delivers with the address. Split axis: subject. This copy holds the RTMI rows only. Column names and
    datatypes are the ones RTMI delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured until
    the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_address_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_address_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_address_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_address_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_address_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_address_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_address_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_address_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_address_rtmi__address_type_prop
    name: address_type
    logicalType: string
    physicalType: STRING
    required: false
    description: Address type as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_TYPE
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. Payload,
      because RTMI delivers one address for each member in a delivery, so a changed type writes a new
      version of the address. Where RTMI sends no type the column is null, as decision D16 leaves country_code
      null.
    quality:
    - id: sat_party_address_rtmi__address_type_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__is_primary_flag_prop
    name: is_primary_flag
    logicalType: boolean
    physicalType: BOOLEAN
    required: false
    description: The source's assertion of the primary address
    classification: Neither
    transformDescription: >-
      Left null in the raw vault. Which occurrence is primary, and how confident an attachment is, are
      survivorship and resolution judgments the source does not make. The business vault asserts them
      from ref_party_source_priority.
  - id: sat_party_address_rtmi__city_prop
    name: city
    logicalType: string
    physicalType: STRING
    required: false
    description: City or locality as supplied.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.CITY
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_rtmi__state_prop
    name: state
    logicalType: string
    physicalType: STRING
    required: false
    description: State or province of the address.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.STATE
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_address_rtmi__state_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__zip_code_prop
    name: zip_code
    logicalType: string
    physicalType: STRING
    required: false
    description: Postal code of the address as supplied.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ZIP_CODE
    transformDescription: >-
      Loaded as supplied and stored untrimmed, including a plus four extension where RTMI sends one inside
      the same column. Splitting the extension out is a soft rule and belongs in the business vault.
  - id: sat_party_address_rtmi__postal_code_extension_prop
    name: postal_code_extension
    logicalType: string
    physicalType: STRING
    required: false
    description: Four-digit extension where supplied
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_address_rtmi__county_prop
    name: county
    logicalType: string
    physicalType: STRING
    required: false
    description: County of the address as supplied.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.COUNTY
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_address_rtmi__county_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__country_code_prop
    name: country_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. ISO country
    classification: Neither
    transformDescription: >-
      Left null, under decision D16. Neither feed sends a country, and writing a configured default would
      put an assertion in the raw vault that no source made. State and postal constructs already imply
      the country for every row in scope, and a load that needs the value explicitly reads it from the
      business vault.
  - id: sat_party_address_rtmi__address_effective_date_prop
    name: address_effective_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When this address took effect
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_address_rtmi__address_end_date_prop
    name: address_end_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When this address stopped being used
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_address_rtmi__originating_system_code_prop
    name: originating_system_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to the System Registry. The system that produced the address, where the feed identifies it
    classification: Neither
    transformDescription: >-
      One constant: the configured value for RTMI as the system the row originated in. The value has to
      be agreed in the load configuration list decision D18 holds before the first load.
  - id: sat_party_address_rtmi__address_cleansed_flag_prop
    name: address_cleansed_flag
    logicalType: string
    physicalType: STRING
    required: false
    description: The cleansing verdict RTMI delivers, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_CLEANSED_FLAG
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. RTMI computed
      the verdict upstream and delivered it, so it is a source claim and loads raw.
    quality:
    - id: sat_party_address_rtmi__address_cleansed_flag_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__address_deliverable_flag_prop
    name: address_deliverable_flag
    logicalType: string
    physicalType: STRING
    required: false
    description: The deliverability verdict RTMI delivers, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_DELIVERABLE_FLAG
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. Delivered
      by RTMI rather than computed here.
    quality:
    - id: sat_party_address_rtmi__address_deliverable_flag_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__address_validation_date_prop
    name: address_validation_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When RTMI states it last validated the address.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_VALIDATION_DATE
    transformDescription: >-
      Loaded as supplied. The source column is already typed as a date, so the target type is lossless.
  - id: sat_party_address_rtmi__ncoa_action_prop
    name: ncoa_action
    logicalType: string
    physicalType: STRING
    required: false
    description: The change-of-address action RTMI delivers, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.NCOA_ACTION
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. The National
      Change of Address result is delivered by RTMI, not derived here.
    quality:
    - id: sat_party_address_rtmi__ncoa_action_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__ncoa_delivery_prop
    name: ncoa_delivery
    logicalType: string
    physicalType: STRING
    required: false
    description: The change-of-address delivery result RTMI delivers, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.NCOA_DELIVERY
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. Delivered
      by RTMI, not derived here.
    quality:
    - id: sat_party_address_rtmi__ncoa_delivery_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__ncoa_move_type_prop
    name: ncoa_move_type
    logicalType: string
    physicalType: STRING
    required: false
    description: The change-of-address move type RTMI delivers, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.NCOA_MOVE_TYPE
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite. Delivered
      by RTMI, not derived here.
    quality:
    - id: sat_party_address_rtmi__ncoa_move_type_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_rtmi__rtmi_household_number_prop
    name: rtmi_household_number
    logicalType: string
    physicalType: STRING
    required: false
    description: Household identifier as RTMI delivers it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.RTMI_HOUSEHOLD_NUMBER
    transformDescription: >-
      Loaded as supplied. The household identifier as RTMI delivers it on the address of the individual
      extract, so it sits on the address satellite. The same number is the business key of hub_party_grouping,
      which keys on the household number as RTMI issues it.
  quality:
  - id: sat_party_address_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_address_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_address_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_address_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_address_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_address_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_address_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_address_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_address_iris_obj
  name: sat_party_address_iris
  physicalName: party_raw_vault.sat_party_address_iris
  physicalType: table
  businessName: Party address satellite (Iris)
  description: >-
    This satellite carries the delivered address except the street lines and the geocode, which carry
    a different classification and sit on entity 26, and it carries the household identifier the source
    delivers with the address. Split axis: subject. This copy holds the Iris rows only. Column names and
    datatypes are the ones Iris delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured until
    the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_address_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_address_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_address_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_address_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_address_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_address_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_address_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_address_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_address_iris__is_primary_flag_prop
    name: is_primary_flag
    logicalType: boolean
    physicalType: BOOLEAN
    required: false
    description: The source's assertion of the primary address
    classification: Neither
    transformDescription: >-
      Left null in the raw vault. Which occurrence is primary, and how confident an attachment is, are
      survivorship and resolution judgments the source does not make. The business vault asserts them
      from ref_party_source_priority.
  - id: sat_party_address_iris__city_prop
    name: city
    logicalType: string
    physicalType: STRING
    required: false
    description: City or locality as supplied.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.CITY
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_iris__state_prop
    name: state
    logicalType: string
    physicalType: STRING
    required: false
    description: State or province of the address.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.STATE
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the Iris value as Iris sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_address_iris__state_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_iris__zip_code_prop
    name: zip_code
    logicalType: string
    physicalType: STRING
    required: false
    description: Postal code of the address as supplied.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.ZIP_CODE
    transformDescription: >-
      Loaded as supplied and stored untrimmed, including a plus four extension where Iris sends one inside
      the same column.
  - id: sat_party_address_iris__postal_code_extension_prop
    name: postal_code_extension
    logicalType: string
    physicalType: STRING
    required: false
    description: Four-digit extension where supplied
    classification: Neither
    transformDescription: No Iris candidate. bronze.iris_customer carries no column for it.
  - id: sat_party_address_iris__country_code_prop
    name: country_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. ISO country
    classification: Neither
    transformDescription: >-
      Left null, under decision D16. Neither feed sends a country, and writing a configured default would
      put an assertion in the raw vault that no source made. State and postal constructs already imply
      the country for every row in scope, and a load that needs the value explicitly reads it from the
      business vault.
  - id: sat_party_address_iris__address_effective_date_prop
    name: address_effective_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When this address took effect
    classification: Neither
    transformDescription: No Iris candidate. bronze.iris_customer carries no column for it.
  - id: sat_party_address_iris__address_end_date_prop
    name: address_end_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When this address stopped being used
    classification: Neither
    transformDescription: No Iris candidate. bronze.iris_customer carries no column for it.
  - id: sat_party_address_iris__originating_system_code_prop
    name: originating_system_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to the System Registry. The system that produced the address, where the feed identifies it
    classification: Neither
    transformDescription: >-
      One constant: the configured value for Iris as the system the row originated in. The value has to
      be agreed in the load configuration list decision D18 holds before the first load.
  quality:
  - id: sat_party_address_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_address_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_address_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      WHERE hash_diff IS NULL
  - id: sat_party_address_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_address_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_address_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_address_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_address_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_address_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_address_pii_rtmi_obj
  name: sat_party_address_pii_rtmi
  physicalName: party_raw_vault.sat_party_address_pii_rtmi
  physicalType: table
  businessName: Party address PII satellite (RTMI)
  description: >-
    This satellite carries the four street lines and the delivered geocode, which are the PII-Direct part
    of an address. Split axis: PII classification, split from the non-PII columns of sat_party_address.
    This copy holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL:
    the rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_address_pii_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_address_pii_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_address_pii_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_address_pii_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_address_pii_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_address_pii_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_address_pii_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_address_pii_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_address_pii_rtmi__address_line_1_prop
    name: address_line_1
    logicalType: string
    physicalType: STRING
    required: false
    description: Primary delivery line as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_LINE_1
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_rtmi__address_line_2_prop
    name: address_line_2
    logicalType: string
    physicalType: STRING
    required: false
    description: Secondary unit or suite line as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_LINE_2
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_rtmi__address_line_3_prop
    name: address_line_3
    logicalType: string
    physicalType: STRING
    required: false
    description: Third delivery line as supplied. The column exists under decision D31.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.ADDRESS_LINE_3
    transformDescription: >-
      Stored as RTMI sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_rtmi__latitude_prop
    name: latitude
    logicalType: string
    physicalType: STRING
    required: false
    description: The geocode latitude RTMI delivers.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.LATITUDE
    transformDescription: >-
      Loaded as supplied and stored untrimmed, held as text. RTMI types the column as text in bronze,
      so a decimal target would reject a value that is not numeric and a raw vault does not reject. The
      typed pair is a business vault computation.
    quality:
    - id: sat_party_address_pii_rtmi__latitude_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_address_pii_rtmi__longitude_prop
    name: longitude
    logicalType: string
    physicalType: STRING
    required: false
    description: The geocode longitude RTMI delivers.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.LONGITUDE
    transformDescription: Loaded as supplied and stored untrimmed, held as text. See latitude.
    quality:
    - id: sat_party_address_pii_rtmi__longitude_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  quality:
  - id: sat_party_address_pii_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_address_pii_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_address_pii_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_address_pii_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_address_pii_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_address_pii_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_address_pii_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_address_pii_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_address_pii_iris_obj
  name: sat_party_address_pii_iris
  physicalName: party_raw_vault.sat_party_address_pii_iris
  physicalType: table
  businessName: Party address PII satellite (Iris)
  description: >-
    This satellite carries the four street lines and the delivered geocode, which are the PII-Direct part
    of an address. Split axis: PII classification, split from the non-PII columns of sat_party_address.
    This copy holds the Iris rows only. Column names and datatypes are the ones Iris delivers. PROVISIONAL:
    the rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_address_pii_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_address_pii_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_address_pii_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_address_pii_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_address_pii_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_address_pii_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_address_pii_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_address_pii_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_address_pii_iris__line1_prop
    name: line1
    logicalType: string
    physicalType: STRING
    required: false
    description: Primary delivery line as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.LINE1
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_iris__line2_prop
    name: line2
    logicalType: string
    physicalType: STRING
    required: false
    description: Secondary unit or suite line as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.LINE2
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_iris__line3_prop
    name: line3
    logicalType: string
    physicalType: STRING
    required: false
    description: Third delivery line as supplied. The column exists under decision D31.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.LINE3
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  - id: sat_party_address_pii_iris__line4_prop
    name: line4
    logicalType: string
    physicalType: STRING
    required: false
    description: Fourth delivery line as supplied. The column exists under decision D31.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.LINE4
    transformDescription: >-
      Stored as Iris sent it, including any leading or trailing whitespace. Trimming happens inside the
      hash pre-image and never on the stored value, so a padded value resolves to the hub row its key
      already has while the column still reports what arrived. Internal spacing, punctuation and case
      are held as delivered. Part of the hash_diff payload, so a corrected value opens a new row and the
      previous claim survives.
  quality:
  - id: sat_party_address_pii_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_address_pii_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_address_pii_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      WHERE hash_diff IS NULL
  - id: sat_party_address_pii_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_address_pii_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_address_pii_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_address_pii_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_address_pii_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: msat_party_contact_point_rtmi_obj
  name: msat_party_contact_point_rtmi
  physicalName: party_raw_vault.msat_party_contact_point_rtmi
  physicalType: table
  businessName: Party contact point satellite (RTMI)
  description: >-
    This satellite carries phone, email, and digital handle values with their channel and dates. RTMI
    delivers a home and a business telephone on one row, so the RTMI instance is multi-active. Split axis:
    subject. This copy holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers.
    PROVISIONAL: the rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and contact_point_type_code and load_date.
  properties:
  - id: msat_party_contact_point_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: msat_party_contact_point_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: msat_party_contact_point_rtmi__contact_point_type_code_prop
    name: contact_point_type_code
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to Reference Data. HOME_PHONE, BUSINESS_PHONE, MOBILE_PHONE, FAX, PERSONAL_EMAIL, WORK_EMAIL,
      PORTAL_HANDLE. Written from the load configuration list decision D18 holds, one value for each occurrence
      the load path separates, because no feed sends a contact point type. Part of the primary key
    classification: Neither
    transformDescription: >-
      One configured constant per load path: the home telephone value on one row and the business telephone
      value on the other. The value has to be agreed in the load configuration list decision D18 holds
      before the first load.
  - id: msat_party_contact_point_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: msat_party_contact_point_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 3
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: msat_party_contact_point_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: msat_party_contact_point_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: msat_party_contact_point_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: msat_party_contact_point_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: msat_party_contact_point_rtmi__contact_point_channel_code_prop
    name: contact_point_channel_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. Phone, email, digital handle
    classification: Neither
    transformDescription: >-
      The configured value for a telephone channel. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: msat_party_contact_point_rtmi__is_primary_flag_prop
    name: is_primary_flag
    logicalType: boolean
    physicalType: BOOLEAN
    required: false
    description: The source's assertion of the primary contact point for the channel
    classification: Neither
    transformDescription: >-
      Left null in the raw vault. Which occurrence is primary, and how confident an attachment is, are
      survivorship and resolution judgments the source does not make. The business vault asserts them
      from ref_party_source_priority.
  - id: msat_party_contact_point_rtmi__contact_point_value_prop
    name: contact_point_value
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      RTMI, home telephone occurrence: Home telephone number as supplied. RTMI, business telephone occurrence:
      Business telephone number as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.BUSINESS_TELEPHONE_NUMBER
    - bronze.rtmi_individual.HOME_TELEPHONE_NUMBER
    transformDescription: >-
      RTMI, home telephone occurrence: Loaded as supplied and stored untrimmed, with the formatting characters
      RTMI sent left in place. The row carries the home telephone contact point type. RTMI, business telephone
      occurrence: Loaded as supplied and stored untrimmed, with the formatting characters RTMI sent left
      in place. The row carries the business telephone contact point type.
    quality:
    - id: msat_party_contact_point_rtmi__contact_point_value_rule_1
      type: text
      description: Reported where blank.
      severity: warning
  - id: msat_party_contact_point_rtmi__contact_point_effective_date_prop
    name: contact_point_effective_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When the contact point took effect
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: msat_party_contact_point_rtmi__contact_point_end_date_prop
    name: contact_point_end_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When it stopped being used
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  quality:
  - id: msat_party_contact_point_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, contact_point_type_code, load_date combination appears on one row. Returns each duplicated
      key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, contact_point_type_code, load_date
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      GROUP BY party_hk, contact_point_type_code, load_date
      HAVING COUNT(*) > 1
  - id: msat_party_contact_point_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, contact_point_type_code, record_source, load_date,
      applied_date. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, contact_point_type_code, load_date
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      WHERE party_hk IS NULL
         OR contact_point_type_code IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: msat_party_contact_point_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, contact_point_type_code, load_date
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      WHERE hash_diff IS NULL
  - id: msat_party_contact_point_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, contact_point_type_code, load_date
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: msat_party_contact_point_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: msat_party_contact_point_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: msat_party_contact_point_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.msat_party_contact_point_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_classification_rtmi_obj
  name: sat_party_classification_rtmi
  physicalName: party_raw_vault.sat_party_classification_rtmi
  physicalType: table
  businessName: Party classification satellite (RTMI)
  description: >-
    This satellite carries multi-scheme classification as delivered: industry codes from a data provider,
    asset tier from the regulator, and market segment from internal marketing. Split axis: subject. This
    copy holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL:
    the rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_classification_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_classification_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_classification_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_classification_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_classification_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_classification_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_classification_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_classification_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_classification_rtmi__classification_scheme_code_prop
    name: classification_scheme_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. The scheme: NAICS, SIC, ORGANIZATION_CATEGORY, CU_ASSET_TIER, MARKET_SEGMENT,
      LIFECYCLE_SEGMENT, DIVERSITY_CLASSIFICATION, FIELD_OF_MEMBERSHIP_TYPE. Delivered as it stands, no
      source twin. Payload in this instance, because RTMI delivers one occurrence for each parent in a
      delivery
    classification: Neither
    transformDescription: >-
      The configured value for the wealth band scheme. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: sat_party_classification_rtmi__classification_effective_date_prop
    name: classification_effective_date
    logicalType: date
    physicalType: DATE
    required: false
    description: >-
      When the classification took effect. Payload in this instance, because RTMI delivers one occurrence
      for each parent in a delivery
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_classification_rtmi__wealth_code_prop
    name: wealth_code
    logicalType: string
    physicalType: STRING
    required: false
    description: The wealth band value as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.WEALTH_CODE
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_classification_rtmi__wealth_code_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_classification_rtmi__classification_description_prop
    name: classification_description
    logicalType: string
    physicalType: STRING
    required: false
    description: The description the source supplies with the value
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_classification_rtmi__scheme_authority_code_prop
    name: scheme_authority_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. The authority that owns the scheme: CENSUS, NCUA, INTERNAL, VENDOR
    classification: Neither
    transformDescription: >-
      The configured authority that owns the scheme. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: sat_party_classification_rtmi__is_primary_in_scheme_flag_prop
    name: is_primary_in_scheme_flag
    logicalType: boolean
    physicalType: BOOLEAN
    required: false
    description: The source's assertion of the primary value within the scheme
    classification: Neither
    transformDescription: >-
      Left null in the raw vault. Which occurrence is primary, and how confident an attachment is, are
      survivorship and resolution judgments the source does not make. The business vault asserts them
      from ref_party_source_priority.
  - id: sat_party_classification_rtmi__assertion_method_code_prop
    name: assertion_method_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. How the classification was asserted: DECLARED, DERIVED_FROM_TRANSACTION, DERIVED_FROM_RELATIONSHIP,
      VENDOR_APPENDED
    classification: Neither
    transformDescription: >-
      The configured value for a delivered assertion. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: sat_party_classification_rtmi__classification_end_date_prop
    name: classification_end_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When the classification ended
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  quality:
  - id: sat_party_classification_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_classification_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_classification_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_classification_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_classification_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_classification_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_classification_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_classification_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_consent_rtmi_obj
  name: sat_party_consent_rtmi
  physicalName: party_raw_vault.sat_party_consent_rtmi
  physicalType: table
  businessName: Party consent satellite (RTMI)
  description: >-
    This satellite carries consent and preference decisions with their capture evidence. A consent decision
    is a compliance fact, so it stays in one table with its evidence pointer. Split axis: subject. This
    copy holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL:
    the rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_consent_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_consent_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_consent_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_consent_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_consent_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_consent_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_consent_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_consent_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_consent_rtmi__consent_type_code_prop
    name: consent_type_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. The kind of decision: MARKETING_CONSENT, MARKETING_SUPPRESSION, DO_NOT_CALL,
      DO_NOT_EMAIL, DO_NOT_MAIL, AFFILIATE_SHARING_RESTRICTION, CCPA_OPT_OUT, CCPA_DELETE_REQUEST, PURPOSE_LIMITATION,
      PREFERRED_CHANNEL, PREFERRED_LANGUAGE. Payload in this instance, because RTMI delivers one occurrence
      for each parent in a delivery
    classification: Neither
    transformDescription: >-
      The configured value for a marketing suppression. The value has to be agreed in the load configuration
      list decision D18 holds before the first load.
  - id: sat_party_consent_rtmi__consent_scope_code_prop
    name: consent_scope_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. What the decision covers: ALL, CHANNEL, PRODUCT_LINE, PURPOSE, LEGAL_ENTITY.
      Payload in this instance, because RTMI delivers one occurrence for each parent in a delivery
    classification: Neither
    transformDescription: >-
      The configured value for all channels. The feed names no scope. The value has to be agreed in the
      load configuration list decision D18 holds before the first load.
  - id: sat_party_consent_rtmi__opt_out_date_prop
    name: opt_out_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When the consent decision took effect.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.OPT_OUT_DATE
    transformDescription: >-
      Loaded as supplied, from the rows where it carries a value, under decision D64. A row with no opt-out
      date carries no opt-out decision, so it writes no consent row.
  - id: sat_party_consent_rtmi__consent_scope_value_prop
    name: consent_scope_value
    logicalType: string
    physicalType: STRING
    required: false
    description: The scoped value the consent applies to, such as a channel or a purpose
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_consent_rtmi__consent_decision_code_prop
    name: consent_decision_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. GRANTED, DENIED, WITHDRAWN, NOT_PROVIDED
    classification: Neither
    transformDescription: >-
      The configured value for a withdrawn decision. The feed carries an opt out and no grant. The value
      has to be agreed in the load configuration list decision D18 holds before the first load.
  - id: sat_party_consent_rtmi__opt_out_reason_prop
    name: opt_out_reason
    logicalType: string
    physicalType: STRING
    required: false
    description: The reason RTMI gives for the opt-out decision, as RTMI represents it.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.OPT_OUT_REASON
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_consent_rtmi__opt_out_reason_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_consent_rtmi__consent_capture_method_code_prop
    name: consent_capture_method_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. How the decision was captured: APPLICATION, WEB_FORM, PHONE, WRITTEN, REGULATORY_DEFAULT,
      INHERITED
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_consent_rtmi__consent_capture_evidence_uri_prop
    name: consent_capture_evidence_uri
    logicalType: string
    physicalType: STRING
    required: false
    description: Pointer to the captured evidence
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_consent_rtmi__consent_end_date_prop
    name: consent_end_date
    logicalType: date
    physicalType: DATE
    required: false
    description: When the decision stopped applying
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  quality:
  - id: sat_party_consent_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_consent_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_consent_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_consent_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_consent_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_consent_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_consent_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_consent_rtmi__opt_out_date_not_after_load_date
    type: custom
    engine: sql
    name: opt_out_date not later than the load date
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_consent_rtmi
      WHERE opt_out_date > load_date
- id: sat_party_person_rtmi_obj
  name: sat_party_person_rtmi
  physicalName: party_raw_vault.sat_party_person_rtmi
  physicalType: table
  businessName: Person satellite (RTMI)
  description: >-
    This satellite carries the person subject as one table. RTMI delivers the demographics in one row,
    and no feed in scope sends the occupation or the language preference. Split axis: subject. This copy
    holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL: the
    rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_person_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_person_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_person_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_person_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_person_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_person_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_person_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_person_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_person_rtmi__gender_prop
    name: gender
    logicalType: string
    physicalType: STRING
    required: false
    description: Gender as recorded on the RTMI member file.
    classification: PII-Indirect
    transformSourceObjects:
    - bronze.rtmi_individual.GENDER
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_person_rtmi__gender_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_person_rtmi__marital_status_prop
    name: marital_status
    logicalType: string
    physicalType: STRING
    required: false
    description: Marital status as recorded on the RTMI member file.
    classification: PII-Indirect
    transformSourceObjects:
    - bronze.rtmi_individual.MARITAL_STATUS
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_person_rtmi__marital_status_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  - id: sat_party_person_rtmi__birth_year_prop
    name: birth_year
    logicalType: integer
    physicalType: INT
    required: false
    description: Year alone, for analysis that does not need the full date
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_person_rtmi__occupation_description_prop
    name: occupation_description
    logicalType: string
    physicalType: STRING
    required: false
    description: Occupation as the source states it
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_person_rtmi__occupation_class_code_prop
    name: occupation_class_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. Underwriting occupation class
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_person_rtmi__preferred_language_code_prop
    name: preferred_language_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. Preferred correspondence language
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_person_rtmi__death_verification_source_code_prop
    name: death_verification_source_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. Source-Reported, Death Master File, Claim-Confirmed, Obituary-Match, Unverified.
      Its own code set, with no conformed twin
    classification: Neither
    transformDescription: No RTMI candidate. bronze.rtmi_individual carries no column for it.
  - id: sat_party_person_rtmi__dob_source_indicator_prop
    name: dob_source_indicator
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      How RTMI states the birth date was established. A provenance code, which does not disclose the date,
      so it sits here rather than on the PII satellite.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.DOB_SOURCE_INDICATOR
    transformDescription: >-
      Loaded as supplied and stored untrimmed. The raw vault holds the RTMI value as RTMI sent it, which
      is what lets it answer what the source sent on a given date. Conforming it is a business rule and
      belongs in the business vault, which reads this column through its conformed satellite.
    quality:
    - id: sat_party_person_rtmi__dob_source_indicator_rule_1
      type: text
      description: >-
        Recorded as supplied. Not validated against a code set at load, and a value the source has not
        sent before is loaded and reported rather than rejected.
      severity: warning
  quality:
  - id: sat_party_person_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_person_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_person_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_person_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_person_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_person_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_person_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_person_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: sat_party_person_pii_rtmi_obj
  name: sat_party_person_pii_rtmi
  physicalName: party_raw_vault.sat_party_person_pii_rtmi
  physicalType: table
  businessName: Person PII satellite (RTMI)
  description: >-
    This satellite carries the two person dates, held apart so column-level security and masking policies
    apply at table grain. Split axis: PII classification, paired with sat_party_person. This copy holds
    the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL: the rate-of-change
    boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_person_pii_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_person_pii_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_person_pii_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_person_pii_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_person_pii_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: sat_party_person_pii_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: sat_party_person_pii_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_person_pii_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_person_pii_rtmi__date_of_birth_prop
    name: date_of_birth
    logicalType: date
    physicalType: DATE
    required: false
    description: The birth date of the person as supplied.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.DATE_OF_BIRTH
    transformDescription: >-
      Loaded as supplied. Part of the hash_diff payload, so a corrected birth date opens a new row and
      the previous claim survives.
  - id: sat_party_person_pii_rtmi__date_of_death_prop
    name: date_of_death
    logicalType: date
    physicalType: DATE
    required: false
    description: Date of death where a contributor reports one.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.rtmi_individual.DATE_OF_DEATH
    transformDescription: >-
      Loaded as supplied. The same column also lands on sat_party_status_rtmi.party_status_effective_date,
      which is the date the status took effect rather than the date of the event. The two are different
      columns holding one value, because the only status RTMI sends is death. D50 records why that is
      not duplication.
  quality:
  - id: sat_party_person_pii_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_person_pii_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_person_pii_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE hash_diff IS NULL
  - id: sat_party_person_pii_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_person_pii_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_person_pii_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_person_pii_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_person_pii_rtmi__date_of_birth_not_after_load_date
    type: custom
    engine: sql
    name: date_of_birth not later than the load date or date_of_death
    description: >-
      Reported where later than the load date, and where later than DATE_OF_DEATH. Neither rejects the
      row. Returns each offending row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE date_of_birth > load_date
         OR date_of_birth > date_of_death
  - id: sat_party_person_pii_rtmi__date_of_death_not_after_load_date
    type: custom
    engine: sql
    name: date_of_death not later than the load date or date_of_death
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_rtmi
      WHERE date_of_death > load_date
- id: sat_party_person_pii_iris_obj
  name: sat_party_person_pii_iris
  physicalName: party_raw_vault.sat_party_person_pii_iris
  physicalType: table
  businessName: Person PII satellite (Iris)
  description: >-
    This satellite carries the two person dates, held apart so column-level security and masking policies
    apply at table grain. Split axis: PII classification, paired with sat_party_person. This copy holds
    the Iris rows only. Column names and datatypes are the ones Iris delivers. PROVISIONAL: the rate-of-change
    boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: sat_party_person_pii_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_person_pii_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_person_pii_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_person_pii_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_person_pii_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_person_pii_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_person_pii_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_person_pii_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_person_pii_iris__birth_date_prop
    name: birth_date
    logicalType: date
    physicalType: DATE
    required: false
    description: The birth date of the person as supplied on the Iris customer record.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.BIRTH_DATE
    transformDescription: >-
      Loaded as supplied. Part of the hash_diff payload, so a corrected birth date opens a new row.
  quality:
  - id: sat_party_person_pii_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_person_pii_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date, applied_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_person_pii_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE hash_diff IS NULL
  - id: sat_party_person_pii_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk, hash_diff is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_person_pii_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_person_pii_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_person_pii_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: sat_party_person_pii_iris__birth_date_not_after_load_date
    type: custom
    engine: sql
    name: birth_date not later than the load date
    description: >-
      Reported where later than the load date. The row is loaded, not rejected. Returns each offending
      row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_person_pii_iris
      WHERE birth_date > load_date
- id: sat_party_identifier_iris_obj
  name: sat_party_identifier_iris
  physicalName: party_raw_vault.sat_party_identifier_iris
  physicalType: table
  businessName: Party identifier satellite (Iris)
  description: >-
    This satellite carries facts about the identifier itself, which stay true whichever party holds it.
    A merge moves the attachment and leaves this satellite untouched. Split axis: subject. This copy holds
    the Iris rows only. Column names and datatypes are the ones Iris delivers. PROVISIONAL: the rate-of-change
    boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_identifier_hk and load_date.
  properties:
  - id: sat_party_identifier_iris__party_identifier_hk_prop
    name: party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party_identifier. Part of the primary key
    relationships:
    - type: foreignKey
      to: hub_party_identifier.party_identifier_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: sat_party_identifier_iris__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: sat_party_identifier_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: sat_party_identifier_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: sat_party_identifier_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: sat_party_identifier_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: sat_party_identifier_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      PII-Direct, because its payload holds a PII-Direct column. SHA-256 over the satellite payload as
      delivered, for delta detection. Never a key
    classification: PII-Direct
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: sat_party_identifier_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: sat_party_identifier_iris__identifier_category_code_prop
    name: identifier_category_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to Reference Data. Party-Assigned, Registry-Issued, Business-Object, Digital, Household
    classification: Neither
    transformDescription: >-
      One configured constant per load path. The value has to be agreed in the load configuration list
      decision D18 holds before the first load.
  - id: sat_party_identifier_iris__is_tokenized_flag_prop
    name: is_tokenized_flag
    logicalType: boolean
    physicalType: BOOLEAN
    required: false
    description: TRUE where the value is a hash or token requiring vault resolution
    classification: Neither
    transformDescription: >-
      FALSE on the anchoring identifier occurrence and TRUE on the SSN occurrence, which arrives already
      tokenized.
  - id: sat_party_identifier_iris__identifier_masked_value_prop
    name: identifier_masked_value
    logicalType: string
    physicalType: STRING
    required: false
    description: The last four digits of the social security number, as Iris sends them.
    classification: PII-Direct
    transformSourceObjects:
    - bronze.iris_customer.SSN_LAST_FOUR
    transformDescription: >-
      Loaded as supplied on the SSN occurrence, beside the token that carries the value itself. It mints
      no hub row, because a masked rendering identifies no single party.
    quality:
    - id: sat_party_identifier_iris__identifier_masked_value_rule_1
      type: text
      description: Reported where present on a row carrying no SSN_TOKEN.
      severity: warning
  - id: sat_party_identifier_iris__issuing_jurisdiction_code_prop
    name: issuing_jurisdiction_code
    logicalType: string
    physicalType: STRING
    required: false
    description: FK to Reference Data. Jurisdiction of the issuing authority where one applies
    classification: Neither
    transformDescription: No Iris candidate. bronze.iris_customer carries no column for it.
  - id: sat_party_identifier_iris__originating_system_code_prop
    name: originating_system_code
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      FK to the System Registry. The system that produced the identifier, where the feed identifies it
    classification: Neither
    transformDescription: >-
      One constant: the configured value for Iris as the system the row originated in. The value has to
      be agreed in the load configuration list decision D18 holds before the first load.
  quality:
  - id: sat_party_identifier_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_identifier_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      GROUP BY party_identifier_hk, load_date
      HAVING COUNT(*) > 1
  - id: sat_party_identifier_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_identifier_hk, record_source, load_date, applied_date.
      Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      WHERE party_identifier_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: sat_party_identifier_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      WHERE hash_diff IS NULL
  - id: sat_party_identifier_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_identifier_hk, hash_diff is 64 lowercase hexadecimal characters. This is load
      test 5 in the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      WHERE NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: sat_party_identifier_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_identifier_hk is the zero key. This is load test 6 in
      the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      WHERE party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: sat_party_identifier_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: sat_party_identifier_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.sat_party_identifier_iris
      WHERE party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: esat_lnk_party_identifier_rtmi_obj
  name: esat_lnk_party_identifier_rtmi
  physicalName: party_raw_vault.esat_lnk_party_identifier_rtmi
  physicalType: table
  businessName: Identifier attachment effectivity satellite (RTMI)
  description: >-
    This satellite dates each attachment of an identifier to a party with the opens and closes the source
    delivers. Split axis: subject. This copy holds the RTMI rows only. Column names and datatypes are
    the ones RTMI delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured until the second
    profiling pass runs.
  dataGranularityDescription: One row per lnk_party_identifier_hk and load_date.
  properties:
  - id: esat_lnk_party_identifier_rtmi__lnk_party_identifier_hk_prop
    name: lnk_party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to lnk_party_identifier. Part of the primary key
    relationships:
    - type: foreignKey
      to: lnk_party_identifier.lnk_party_identifier_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: esat_lnk_party_identifier_rtmi__lnk_party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: esat_lnk_party_identifier_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: esat_lnk_party_identifier_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: esat_lnk_party_identifier_rtmi__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      COALESCE(ORIGIN_TIMESTAMP, File_Date). RTMI states when it applied the change on all but 0.02 percent
      of rows, and the delivery states its own date for the rest. File_Date returned one value per delivery
      across 18 deliveries.
    quality:
    - id: esat_lnk_party_identifier_rtmi__applied_date_rule_1
      type: text
      description: >-
        Where ORIGIN_TIMESTAMP is empty the delivery date stands, and the vault load date is never written
        here.
  - id: esat_lnk_party_identifier_rtmi__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: esat_lnk_party_identifier_rtmi__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: esat_lnk_party_identifier_rtmi__origin_timestamp_prop
    name: origin_timestamp
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: When the attachment opened.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.ORIGIN_TIMESTAMP
    transformDescription: >-
      Loaded as supplied. This is the only place the delivered attachment timestamp is stored, because
      the link it attaches to carries no dependent child key. Decision D55 settles that this satellite
      holds the delivered attachment timeline.
  - id: esat_lnk_party_identifier_rtmi__end_timestamp_prop
    name: end_timestamp
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: >-
      When it closed. Null while open. Filled by the load: null on an opening row, and the batch load
      date on the closing row an RTMI DELTA_TYPE D writes
    classification: Neither
    transformDescription: >-
      Null on the opening row, which is what makes the row open. A DELTA_TYPE delete writes a closing
      row stamped with the load timestamp of that batch.
  quality:
  - id: esat_lnk_party_identifier_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each lnk_party_identifier_hk, load_date combination appears on one row. Returns each duplicated
      key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      GROUP BY lnk_party_identifier_hk, load_date
      HAVING COUNT(*) > 1
  - id: esat_lnk_party_identifier_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: lnk_party_identifier_hk, record_source, load_date, applied_date.
      Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      WHERE lnk_party_identifier_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: esat_lnk_party_identifier_rtmi_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      WHERE hash_diff IS NULL
  - id: esat_lnk_party_identifier_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in lnk_party_identifier_hk, hash_diff is 64 lowercase hexadecimal characters. This is
      load test 5 in the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      WHERE NOT lnk_party_identifier_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: esat_lnk_party_identifier_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose lnk_party_identifier_hk is the zero key. This is load test
      6 in the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      WHERE lnk_party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: esat_lnk_party_identifier_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.lnk_party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: esat_lnk_party_identifier_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_rtmi
      WHERE lnk_party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: esat_lnk_party_identifier_iris_obj
  name: esat_lnk_party_identifier_iris
  physicalName: party_raw_vault.esat_lnk_party_identifier_iris
  physicalType: table
  businessName: Identifier attachment effectivity satellite (Iris)
  description: >-
    This satellite dates each attachment of an identifier to a party with the opens and closes the source
    delivers. Split axis: subject. This copy holds the Iris rows only. Column names and datatypes are
    the ones Iris delivers. PROVISIONAL: the rate-of-change boundary stays unmeasured until the second
    profiling pass runs.
  dataGranularityDescription: One row per lnk_party_identifier_hk and load_date.
  properties:
  - id: esat_lnk_party_identifier_iris__lnk_party_identifier_hk_prop
    name: lnk_party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to lnk_party_identifier. Part of the primary key
    relationships:
    - type: foreignKey
      to: lnk_party_identifier.lnk_party_identifier_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: esat_lnk_party_identifier_iris__lnk_party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: esat_lnk_party_identifier_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: esat_lnk_party_identifier_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: esat_lnk_party_identifier_iris__applied_date_prop
    name: applied_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      When the source states the change was applied. The row timestamp the feed sends, or the timestamp
      of the delivery that carried it where the row has none. Never the load date, and never in the hash_diff,
      because it dates the assertion rather than the values
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.IngestedTime
    transformDescription: >-
      IngestedTime, the timestamp of the delivery that carried the row. Iris sends no row-level change
      timestamp: StartDate returned a null on all 22,610,668 rows of the profiling run. Decision D60,
      which is open, decides whether IngestedTime is the Iris extract stamp or the platform ingestion
      stamp.
    quality:
    - id: esat_lnk_party_identifier_iris__applied_date_rule_1
      type: text
      description: One value per delivery until Iris sends a row timestamp.
  - id: esat_lnk_party_identifier_iris__hash_diff_prop
    name: hash_diff
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      SHA-256 over the satellite payload as delivered, for delta detection. Never a key
    classification: Neither
    transformDescription: >-
      Computed at load over the payload columns of this construct, in the order the construct declares
      them. A change in any of them opens a new row.
    quality:
    - id: esat_lnk_party_identifier_iris__hash_diff_rule_1
      type: text
      description: Recomputed on every load and compared with the current row.
  - id: esat_lnk_party_identifier_iris__start_date_prop
    name: start_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: When the attachment opened.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.StartDate
    transformDescription: >-
      Loaded as supplied. This is the only place the delivered attachment timestamp is stored, because
      the link it attaches to carries no dependent child key. Decision D55 settles that this satellite
      holds the delivered attachment timeline.
  - id: esat_lnk_party_identifier_iris__end_date_prop
    name: end_date
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: false
    description: When the attachment closed.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.EndDate
    transformDescription: >-
      Loaded as supplied. Iris states when the source record stopped applying, so this feed closes an
      attachment from the data rather than from a delta instruction.
  quality:
  - id: esat_lnk_party_identifier_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each lnk_party_identifier_hk, load_date combination appears on one row. Returns each duplicated
      key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      GROUP BY lnk_party_identifier_hk, load_date
      HAVING COUNT(*) > 1
  - id: esat_lnk_party_identifier_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: lnk_party_identifier_hk, record_source, load_date, applied_date.
      Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE lnk_party_identifier_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
         OR applied_date IS NULL
  - id: esat_lnk_party_identifier_iris_mapping_not_null
    type: custom
    engine: sql
    name: Columns the mapping requires not null
    description: >-
      The mapping requires a value in hash_diff on every row, and the DDL declares them nullable. Returns
      each offending row's key.
    severity: warning
    dimension: completeness
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE hash_diff IS NULL
  - id: esat_lnk_party_identifier_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in lnk_party_identifier_hk, hash_diff is 64 lowercase hexadecimal characters. This is
      load test 5 in the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE NOT lnk_party_identifier_hk RLIKE '^[0-9a-f]{64}$'
         OR NOT hash_diff RLIKE '^[0-9a-f]{64}$'
  - id: esat_lnk_party_identifier_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose lnk_party_identifier_hk is the zero key. This is load test
      6 in the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE lnk_party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: esat_lnk_party_identifier_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.lnk_party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: esat_lnk_party_identifier_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE lnk_party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
  - id: esat_lnk_party_identifier_iris__end_date_not_before_start
    type: custom
    engine: sql
    name: end_date not earlier than start_date
    description: >-
      Reported where earlier than StartDate, which lands in start_date. The row is loaded, not rejected.
      Returns each offending row's key.
    severity: warning
    dimension: accuracy
    implementation: |
      SELECT lnk_party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.esat_lnk_party_identifier_iris
      WHERE end_date < start_date
- id: rts_party_rtmi_obj
  name: rts_party_rtmi
  physicalName: party_raw_vault.rts_party_rtmi
  physicalType: table
  businessName: Party record tracking satellite (RTMI)
  description: >-
    This satellite records which loads a source presented a party in. It writes one row per party per
    load whether or not anything changed, so the last load a source presented a key in is a maximum over
    `load_date`. Split axis: producer: the delivering feed, whose presentations this records. This copy
    holds the RTMI rows only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL: the
    rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: rts_party_rtmi__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: rts_party_rtmi__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: rts_party_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: rts_party_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: rts_party_rtmi__delta_type_prop
    name: delta_type
    logicalType: string
    physicalType: STRING
    required: false
    description: The change indicator RTMI delivers with the member record.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.DELTA_TYPE
    transformDescription: >-
      Loaded as supplied, on every row RTMI presents. The record tracking satellite writes a row per load
      whether or not anything changed, so this column records what RTMI said about the row each time,
      including a delete.
    quality:
    - id: rts_party_rtmi__delta_type_rule_1
      type: text
      description: Reported where outside the delivered vocabulary.
      severity: warning
  quality:
  - id: rts_party_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_rtmi
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: rts_party_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date. Returns each offending
      row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_rtmi
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
  - id: rts_party_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk is 64 lowercase hexadecimal characters. This is load test 5 in the load
      specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_rtmi
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
  - id: rts_party_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.rts_party_rtmi
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: rts_party_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.rts_party_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: rts_party_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.rts_party_rtmi
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: rts_party_iris_obj
  name: rts_party_iris
  physicalName: party_raw_vault.rts_party_iris
  physicalType: table
  businessName: Party record tracking satellite (Iris)
  description: >-
    This satellite records which loads a source presented a party in. It writes one row per party per
    load whether or not anything changed, so the last load a source presented a key in is a maximum over
    `load_date`. Split axis: producer: the delivering feed, whose presentations this records. This copy
    holds the Iris rows only. Column names and datatypes are the ones Iris delivers. PROVISIONAL: the
    rate-of-change boundary stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_hk and load_date.
  properties:
  - id: rts_party_iris__party_hk_prop
    name: party_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party. Part of the primary key
    classification: Neither
    relationships:
    - type: foreignKey
      to: hub_party.party_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: rts_party_iris__party_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: rts_party_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: rts_party_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: rts_party_iris__event_type_prop
    name: event_type
    logicalType: string
    physicalType: STRING
    required: false
    description: The change indicator Iris delivers with the customer record.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.Event_type
    transformDescription: >-
      Loaded as supplied, on every row Iris presents. The record tracking satellite writes a row per load
      whether or not anything changed, so this column records what Iris said about the row each time.
    quality:
    - id: rts_party_iris__event_type_rule_1
      type: text
      description: Reported where outside the delivered vocabulary.
      severity: warning
  quality:
  - id: rts_party_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_iris
      GROUP BY party_hk, load_date
      HAVING COUNT(*) > 1
  - id: rts_party_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_hk, record_source, load_date. Returns each offending
      row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_iris
      WHERE party_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
  - id: rts_party_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_hk is 64 lowercase hexadecimal characters. This is load test 5 in the load
      specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_iris
      WHERE NOT party_hk RLIKE '^[0-9a-f]{64}$'
  - id: rts_party_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_hk is the zero key. This is load test 6 in the load
      specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.rts_party_iris
      WHERE party_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: rts_party_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.rts_party_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: rts_party_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.rts_party_iris
      WHERE party_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: rts_party_identifier_rtmi_obj
  name: rts_party_identifier_rtmi
  physicalName: party_raw_vault.rts_party_identifier_rtmi
  physicalType: table
  businessName: Party identifier record tracking satellite (RTMI)
  description: >-
    This satellite records which loads a source presented an identifier in, at the identifier grain. Split
    axis: producer: the delivering feed, whose presentations this records. This copy holds the RTMI rows
    only. Column names and datatypes are the ones RTMI delivers. PROVISIONAL: the rate-of-change boundary
    stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_identifier_hk and load_date.
  properties:
  - id: rts_party_identifier_rtmi__party_identifier_hk_prop
    name: party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party_identifier. Part of the primary key
    relationships:
    - type: foreignKey
      to: hub_party_identifier.party_identifier_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: rts_party_identifier_rtmi__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: rts_party_identifier_rtmi__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: rts_party_identifier_rtmi__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: rts_party_identifier_rtmi__delta_type_prop
    name: delta_type
    logicalType: string
    physicalType: STRING
    required: false
    description: The change indicator RTMI delivers with the member record.
    classification: Neither
    transformSourceObjects:
    - bronze.rtmi_individual.DELTA_TYPE
    transformDescription: Loaded as supplied, at the identifier grain.
    quality:
    - id: rts_party_identifier_rtmi__delta_type_rule_1
      type: text
      description: Reported where outside the delivered vocabulary.
      severity: warning
  quality:
  - id: rts_party_identifier_rtmi_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_identifier_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi
      GROUP BY party_identifier_hk, load_date
      HAVING COUNT(*) > 1
  - id: rts_party_identifier_rtmi_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_identifier_hk, record_source, load_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi
      WHERE party_identifier_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
  - id: rts_party_identifier_rtmi_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_identifier_hk is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi
      WHERE NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
  - id: rts_party_identifier_rtmi_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_identifier_hk is the zero key. This is load test 6 in
      the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi
      WHERE party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: rts_party_identifier_rtmi_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: rts_party_identifier_rtmi_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the RTMI load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.rts_party_identifier_rtmi
      WHERE party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: rts_party_identifier_iris_obj
  name: rts_party_identifier_iris
  physicalName: party_raw_vault.rts_party_identifier_iris
  physicalType: table
  businessName: Party identifier record tracking satellite (Iris)
  description: >-
    This satellite records which loads a source presented an identifier in, at the identifier grain. Split
    axis: producer: the delivering feed, whose presentations this records. This copy holds the Iris rows
    only. Column names and datatypes are the ones Iris delivers. PROVISIONAL: the rate-of-change boundary
    stays unmeasured until the second profiling pass runs.
  dataGranularityDescription: One row per party_identifier_hk and load_date.
  properties:
  - id: rts_party_identifier_iris__party_identifier_hk_prop
    name: party_identifier_hk
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to hub_party_identifier. Part of the primary key
    relationships:
    - type: foreignKey
      to: hub_party_identifier.party_identifier_hk
    transformDescription: >-
      Computed at load from the business key of its construct, or carried unchanged from the parent. SHA-256
      over a pipe-delimited pre-image, to the convention in the Hash keys section of the load specification.
    quality:
    - id: rts_party_identifier_iris__party_identifier_hk_rule_1
      type: text
      description: Deterministic for the same inputs across every load.
  - id: rts_party_identifier_iris__record_source_prop
    name: record_source
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      FK to the System Registry. The system that delivered this row. Constant within this table, which
      carries the source in its name, so it is not part of the key
    classification: Neither
    transformDescription: >-
      The System Registry code for this feed, one constant for every row the table holds. It is not a
      key column, because the table name carries the source.
  - id: rts_party_identifier_iris__load_date_prop
    name: load_date
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: >-
      The load date of the batch that delivered the row, the same on a rerun and on a rebuild from bronze.
      System-generated, gapless, never sourced
    classification: Neither
  - id: rts_party_identifier_iris__event_type_prop
    name: event_type
    logicalType: string
    physicalType: STRING
    required: false
    description: The change indicator Iris delivers with the customer record.
    classification: Neither
    transformSourceObjects:
    - bronze.iris_customer.Event_type
    transformDescription: Loaded as supplied, at the identifier grain.
    quality:
    - id: rts_party_identifier_iris__event_type_rule_1
      type: text
      description: Reported where outside the delivered vocabulary.
      severity: warning
  quality:
  - id: rts_party_identifier_iris_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each party_identifier_hk, load_date combination appears on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris
      GROUP BY party_identifier_hk, load_date
      HAVING COUNT(*) > 1
  - id: rts_party_identifier_iris_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: party_identifier_hk, record_source, load_date. Returns
      each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris
      WHERE party_identifier_hk IS NULL
         OR record_source IS NULL
         OR load_date IS NULL
  - id: rts_party_identifier_iris_hash_format
    type: custom
    engine: sql
    name: Hash columns are 64 lowercase hex characters
    description: >-
      Every value in party_identifier_hk is 64 lowercase hexadecimal characters. This is load test 5 in
      the load specification. Returns each offending row's key.
    severity: critical
    dimension: conformity
    implementation: |
      SELECT party_identifier_hk, load_date
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris
      WHERE NOT party_identifier_hk RLIKE '^[0-9a-f]{64}$'
  - id: rts_party_identifier_iris_one_ghost_row
    type: custom
    engine: sql
    name: One zero-key row
    description: >-
      The table holds exactly one row whose party_identifier_hk is the zero key. This is load test 6 in
      the load specification. Returns the count when it is not one.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT COUNT(*) AS zero_key_rows
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris
      WHERE party_identifier_hk = '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(*) <> 1
  - id: rts_party_identifier_iris_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL
        AND t.party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
  - id: rts_party_identifier_iris_record_source_single_value
    type: custom
    engine: sql
    name: One record_source in a source table
    description: >-
      Every row the Iris load writes carries one record_source value, the zero-key row aside. Returns
      the count of distinct values when it is more than one.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT COUNT(DISTINCT record_source) AS record_source_values
      FROM lh_silver.party_raw_vault.rts_party_identifier_iris
      WHERE party_identifier_hk <> '0000000000000000000000000000000000000000000000000000000000000000'
      HAVING COUNT(DISTINCT record_source) > 1
- id: err_party_load_reject_obj
  name: err_party_load_reject
  physicalName: party_ops.err_party_load_reject
  physicalType: table
  businessName: Party load reject error mart
  description: >-
    The table holds one row for each source row a hard rule refused, with the row as delivered, so the
    load can run again once the rule is corrected.
  dataGranularityDescription: One row per refused source row per delivery
  properties:
  - id: err_party_load_reject__record_source_prop
    name: record_source
    primaryKey: true
    primaryKeyPosition: 1
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to the System Registry. The system that delivered the row
    classification: Neither
    transformDescription: >-
      The System Registry code for the feed that delivered the refused row. Part of the key here, because
      one error mart holds every feed.
  - id: err_party_load_reject__source_object_name_prop
    name: source_object_name
    primaryKey: true
    primaryKeyPosition: 2
    logicalType: string
    physicalType: STRING
    required: true
    description: The bronze object the row was read from, schema qualified
    classification: Neither
    transformDescription: The bronze object the refused row was read from, schema qualified.
  - id: err_party_load_reject__batch_identifier_prop
    name: batch_identifier
    primaryKey: true
    primaryKeyPosition: 3
    logicalType: string
    physicalType: STRING
    required: true
    description: The delivery the row arrived in, as the delivering system names it
    classification: Neither
    transformDescription: >-
      RTMI, rows this feed delivered that a hard rule refused: The delivery the refused row arrived in,
      as RTMI names it. Iris, rows this feed delivered that a hard rule refused: The delivery the refused
      row arrived in, as Iris names it.
  - id: err_party_load_reject__source_row_number_prop
    name: source_row_number
    primaryKey: true
    primaryKeyPosition: 4
    logicalType: integer
    physicalType: BIGINT
    required: true
    description: The row's ordinal position within that delivery
    classification: Neither
    transformDescription: >-
      The ordinal position of the refused row inside that delivery. With the three columns before it this
      reproduces the same key on a rerun, so a reload cannot create a second entry for one source row.
  - id: err_party_load_reject__reject_timestamp_prop
    name: reject_timestamp
    logicalType: timestamp
    physicalType: TIMESTAMP
    required: true
    description: When the load refused the row
    classification: Neither
    transformDescription: The moment the load refused the row. Its own timeline, not a vault load_date.
  - id: err_party_load_reject__reject_rule_code_prop
    name: reject_rule_code
    logicalType: string
    physicalType: STRING
    required: true
    description: FK to Reference Data. The hard rule that refused the row
    classification: Neither
    transformDescription: The hard rule that refused the row, from the Reference Data code set.
  - id: err_party_load_reject__reject_detail_prop
    name: reject_detail
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      What failed: the column and the class of failure. The value stays in `rejected_row`
    classification: Neither
    transformDescription: >-
      What failed, naming the column and the class of failure. The delivered value is not repeated here,
      because it is already in rejected_row and this text reaches logs and alerts.
  - id: err_party_load_reject__target_construct_name_prop
    name: target_construct_name
    logicalType: string
    physicalType: STRING
    required: false
    description: >-
      The construct the row was headed for. Empty where the load refused the row before it resolved a
      target
    classification: Neither
    transformDescription: >-
      The construct the row was headed for. Empty where the load refused the row before it resolved a
      target.
  - id: err_party_load_reject__rejected_row_prop
    name: rejected_row
    logicalType: string
    physicalType: STRING
    required: true
    description: >-
      PII-Direct. The delivered row as structured JSON, every column exactly as the source sent it
    classification: PII-Direct
    transformDescription: >-
      RTMI, rows this feed delivered that a hard rule refused: The whole delivered row as structured JSON,
      every column exactly as RTMI sent it. A reload needs the complete row, which is why this column
      carries the strictest PII classification in the table. Iris, rows this feed delivered that a hard
      rule refused: The whole delivered row as structured JSON, every column exactly as Iris sent it.
      A reload needs the complete row, which is why this column carries the strictest PII classification
      in the table.
  quality:
  - id: err_party_load_reject_primary_key_unique
    type: custom
    engine: sql
    name: Unique primary key
    description: >-
      Each record_source, source_object_name, batch_identifier, source_row_number combination appears
      on one row. Returns each duplicated key.
    severity: critical
    dimension: uniqueness
    implementation: |
      SELECT record_source, source_object_name, batch_identifier, source_row_number
      FROM lh_silver.party_ops.err_party_load_reject
      GROUP BY record_source, source_object_name, batch_identifier, source_row_number
      HAVING COUNT(*) > 1
  - id: err_party_load_reject_required_not_null
    type: custom
    engine: sql
    name: Required columns not null
    description: >-
      These columns hold a value on every row: record_source, source_object_name, batch_identifier, source_row_number,
      reject_timestamp, reject_rule_code, rejected_row. Returns each offending row's key.
    severity: critical
    dimension: completeness
    implementation: |
      SELECT record_source, source_object_name, batch_identifier, source_row_number
      FROM lh_silver.party_ops.err_party_load_reject
      WHERE record_source IS NULL
         OR source_object_name IS NULL
         OR batch_identifier IS NULL
         OR source_row_number IS NULL
         OR reject_timestamp IS NULL
         OR reject_rule_code IS NULL
         OR rejected_row IS NULL
  - id: err_party_load_reject_record_source_registered
    type: custom
    engine: sql
    name: record_source is registered
    description: >-
      Every record_source resolves to a System Registry registration, the zero-key row aside. This is
      load test 8 in the load specification. Returns each unregistered value.
    severity: warning
    dimension: consistency
    implementation: |
      SELECT DISTINCT t.record_source
      FROM lh_silver.party_ops.err_party_load_reject AS t
      LEFT JOIN lh_gold.reference.registered_system AS r
        ON r.system_code = t.record_source
      WHERE r.system_code IS NULL

# ───────────────────────────────────────────────────────────────────────────────
# 3. SUPPORT & COMMUNICATION CHANNELS
# ───────────────────────────────────────────────────────────────────────────────
support:
- id: production_incidents
  channel: data-products-distribution
  tool: email
  scope: announcements
  url: mailto:ds-dataproducts@trustage.com
  description: Email distribution list for product incident notifications

# ───────────────────────────────────────────────────────────────────────────────
# 4. INFRASTRUCTURE & SERVERS
# ───────────────────────────────────────────────────────────────────────────────
servers:
- server: spark-server-party-raw-vault
  type: databricks
  description: >-
    Microsoft Fabric lakehouse lh_silver, schema party_raw_vault. The databricks server type is the house
    pattern for a Fabric lakehouse.
  host: fabric-local
  catalog: lh_silver
  schema: party_raw_vault
- server: spark-server-party-ops
  type: databricks
  description: >-
    Microsoft Fabric lakehouse lh_silver, schema party_ops, which holds the error mart. Each table's physicalName
    carries its schema.
  host: fabric-local
  catalog: lh_silver
  schema: party_ops

# ───────────────────────────────────────────────────────────────────────────────
# 5. CUSTOM PROPERTIES
# ───────────────────────────────────────────────────────────────────────────────
customProperties:
- property: sourceTables
  value:
  - bronze.rtmi_individual
  - bronze.iris_customer
- property: mappingWorkbooks
  value:
  - rtmi_individual_vault_mapping_v36.xlsx
  - iris_customer_vault_mapping_v33.xlsx
- property: targetDdl
  value: slv-party-core-vault.ddl.sql
- property: sourceColumnsNotLoaded
  value:
  - sourceColumn: bronze.rtmi_individual.ACQUISITION_DATE
    disposition: Not mapped under the proposal
    detail: >-
      The competing candidate for first_established_date, where the proposal takes ORIGIN_TIMESTAMP. It
      has no other target in crawl.
  - sourceColumn: bronze.rtmi_individual.File_Date
    disposition: Not carried into the vault
    detail: >-
      Not carried into the vault. Decision D1 is closed: record_source carries the System Registry code
      for the delivering system, so the bronze file identity is not carried and no satellite key depends
      on it. Batch lineage, if it is wanted later, is a separate column rather than an overload of record_source,
      and adding one changes no existing key.
  - sourceColumn: bronze.rtmi_individual.IngestedTime
    disposition: Not carried into the vault
    detail: >-
      The time the platform ingested the RTMI delivery. RTMI states its own change timestamp in ORIGIN_TIMESTAMP
      and its delivery date in File_Date, and the applied date takes those two under decision D59, so
      no construct in crawl reads IngestedTime.
  - sourceColumn: bronze.rtmi_individual.FileName
    disposition: Not carried into the vault
    detail: >-
      Not carried into the vault. Decision D1 is closed: record_source carries the System Registry code
      for the delivering system, so the bronze file identity is not carried and no satellite key depends
      on it. Batch lineage, if it is wanted later, is a separate column rather than an overload of record_source,
      and adding one changes no existing key. File_Date and FileName stand together.
  - sourceColumn: bronze.iris_customer.Event_id
    disposition: Not mapped
    detail: >-
      The identifier Iris gives the event that caused the row. Decision D34, which is open, asks whether
      Event_type carries a business event vocabulary. If it does, Event_id loads into sat_party_resolution_event_detail.event_id,
      which decision D33 gives its own column, and until then no event row exists to carry it.
- property: severityConvention
  value:
    critical: >-
      A key, completeness, or hash rule from load tests 1 to 7 in the load specification. Decision D65
      proposes that a failure stops a release.
    warning: >-
      An observation. Decision D26 proposes that every mapping rule records an observation and loads the
      row, and decision D65 places load test 8 there.
    text: >-
      A text rule restates a load rule from the mapping that no query can test on the loaded table. It
      carries a severity only where the rule reports something.
- property: openDecisions
  value:
  - decision: D2
    question: ORIGIN_TIMESTAMP, ACQUISITION_DATE, or neither?
    columnsCiting: 2
  - decision: D8
    question: Clear, or tokenized?
    columnsCiting: 2
  - decision: D9
    question: One rule for both feeds, or a rule per feed?
    columnsCiting: 2
  - decision: D18
    question: Approve the list, or amend it?
    columnsCiting: 18
  - decision: D26
    question: Report only, or reject on the named rules?
    columnsCiting: 8
  - decision: D28
    question: Leave them null, or set defaults?
    columnsCiting: 6
  - decision: D29
    question: Anchor its own party, or attach as an identifier?
    columnsCiting: 2
  - decision: D34
    question: Load behavior, or a business event type?
    columnsCiting: 2
  - decision: D42
    question: Leave the extension null, or split at load?
    columnsCiting: 2
  - decision: D45
    question: Overlap threshold, or a new grouping on every change?
    columnsCiting: 1
  - decision: D51
    question: Add a disposition and a rule class to the data contract standard?
    columnsCiting: 0
  - decision: D59
    question: Add applied_date and a subsequence to the key, or keep stopping the load?
    columnsCiting: 18
  - decision: D60
    question: The Iris extract, or the platform ingestion?
    columnsCiting: 7
  - decision: D61
    question: Delete the rows, or destroy a key per person?
    columnsCiting: 0
  - decision: D62
    question: party_ops on the business key, or somewhere else?
    columnsCiting: 0
  - decision: D63
    question: A random token, or one derived from the identifier?
    columnsCiting: 0
  - decision: D65
    question: Run them as proposed?
    columnsCiting: 0
  - decision: D70
    question: BINARY(32) after the tenant test, or hex text?
    columnsCiting: 0
- property: openItems
  value: Party_Core_Silver_Data_Contract_Issues_v0.3.xlsx lists every open item.
- property: lifecycle
  value:
    changelog:
    - version: 0.1.0
      date: '2026-10-21'
      description: >-
        Initial draft, built from rtmi_individual_vault_mapping_v36.xlsx, iris_customer_vault_mapping_v33.xlsx,
        and slv-party-core-vault.ddl.sql.
    - version: 0.2.0
      date: '2026-10-21'
      description: >-
        Removed the per-column custom properties populationClass, transformRule, sourceOfTruth, mappingStatus,
        and decisionRefs. The mapping workbooks named in mappingWorkbooks hold them.
    - version: 0.3.0
      date: '2026-10-21'
      description: >-
        Removed the table custom properties constructKind, writtenBy, and satelliteBoundary, and the column
        custom property classificationPending. The table names, the table descriptions, and the issue
        log carry the same facts.

contractCreatedTs: '2026-10-21T20:26:33Z'
