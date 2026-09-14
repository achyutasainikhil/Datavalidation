5.4 Fabric to Snowflake Sharing Validation Queries (Executed from Snowflake)

Overview & Objective
Validates the return path of the bi-directional data loop. This step executes directly within Snowflake to verify that curated key-ring datasets (key_ring_entity and key_ring_identifier) generated in Microsoft Fabric (lh_Silver) are accessible, synchronized, and queryable through Snowflake shared consumer views (shared_key_ring_entity and shared_key_ring_identifier).

Snowflake Return-Path Validation Script
