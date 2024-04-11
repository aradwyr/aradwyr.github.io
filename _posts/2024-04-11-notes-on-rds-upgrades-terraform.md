---
title: "Notes on: RDS Upgrades in Terraform"
---

This post presents a simple, straightforward pathway to regularly upgrading key infrastructure via terraform. The focus of this guide will be on upgrading AWS RDS Postgresql instances but very similar approaches would also work for other cloud platforms and database engines. The main benefits of using terraform are transparency across engineering teams as well as terraform prevents accidents from premature upgrades, i.e. when the pre-requisite steps have not been yet addressed. Goes without saying, the following steps should be done during off hours for production instances to avoid any and all disruption, making use of the resource's set maintenance window to ensure there's no active connections during this change. 

## Pre-requisites
1. Apply pending OS updates on the instance(s) (if any)
    - This setting is not yet available in Terraform so will have to either be manually managed or with routine maintenance scripts. 
2. Ensure the instance class is supported in the desired engine version
    - For example, in PG13 t2 generation is not supported.
    - Using the [cli](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-orderable-db-instance-options.html), check if the new RDS engine version and available classes are compatible with existing config

    ```shell
    aws rds describe-orderable-db-instance-options --engine postgres --engine-version 13.8 | jq '.OrderableDBInstanceOptions[].DBInstanceClass'
    ```

3. Ensure the backup retention period is > 0 (under `Maintenance & Backups`); RDS takes 2 snapshots during the upgrade process pre- and post-upgrade. 
4. Managing PG Extensions
    - Connect to the db and see all currently installed with `\dx`
    - If `postgis` is installed, you will only be able to upgrade one major version at a given time. Before the upgrade, run `ALTER EXTENSION postgis UPDATE;`
        - You may run into and be prompted to run: 
            ```
            SELECT postgis_extensions_upgrade();
            ```
    - If `chkpass`, `tsearch2` extensions are installed then run: 
        - Pre-upgrade: `DROP EXTENSION <name>;`
        - Post-upgrade: `DROP EXTENSION <name>;` 

> [!WARNING]  
> Earlier AWS postgis documentation used to suggest dropping the postgis extension via `DROP EXTENSION example_extension CASCADE;` Please do not do this. 

5. Check for CDC replication slots (if any) and drop only just before the upgrade, else the upgrade will not succeed. 

```sql
select * from pg_replication_slots;

select pg_drop_replication_slot('debezium');
```

You might see `ERROR: replication slot "debezium" is active for PID <NNNNN>`, then run the following only just before the upgrade: 

```sql
select pg_terminate_backend(NNNNN); select pg_drop_replication_slot('debezium');

## Terraform Process
First you'll need to have the following parameters applied to state before you attempt the upgrade. 
```tf
allow_major_version_upgrade = true
apply_immediately = false
```

Only once the above is applied then you can adjust engine versions and parameter groups on any resources.  
```tf
engine_version = "13"
parameter_group_name = "custom-postgres-13"
```

> [!TIP]
> If the database in question has a primary and replica/reader then be sure to only upgrade the primary instance. Else, the upgrade will not succeed. Nonetheless, the reader will still need the `allow_major_version_upgrade = true` and `deletion_protection = false` are set on the replica(s). 

> [!NOTE]
> If the upgrade fails in terraform, check RDS logs for the specific reason, it's usually extension-related.

## Post-upgrade
```sql
vacuumdb --analyze-only --verbose --echo --job=100 $DB_CONN_STR
```

If upgrading to PG13, you can and should opt to take advantage of newly available index storage optimizations, which requires a reindex of those btree indexes.

```sql
psql <postgres://user:pass@endpoint:5432/db_name>

reindex database concurrently <db_name>;
```
> [!WARNING]  
> For an 86GB db the reindex took ~6 hours and the connection must remain open during the reindex process. 

## References
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.html 
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.PostGIS.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance 

> [!TIP]
> JQ tangent/invaluable resource: https://www.devtoolsdaily.com/jq_playground/ 

## Feedback & Contributions
Feel free to reach out at aradwyr@gmail.com