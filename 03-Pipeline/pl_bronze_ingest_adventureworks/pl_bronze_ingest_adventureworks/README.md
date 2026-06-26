# Bronze Ingestion Pipeline — `pl_bronze_ingest_adventureworks`

Fabric Data Factory pipeline that ingests AdventureWorks data from an on-premises SQL Server instance into the Bronze layer of a Microsoft Fabric medallion architecture lakehouse.

## What it does

- A `SetVariable` activity (`tableList`) holds an array of 18 source tables spanning the `Sales`, `Production`, `Purchasing`, `HumanResources`, and `Person` schemas.
- A sequential `ForEach` loop iterates over that list, running one generic `Copy` activity per table. Each table lands in the `lh_adventureworks_bronze` lakehouse under the `dbo` schema, named as `<SourceSchema>_<TableName>` (e.g. `Sales.SalesOrderHeader` → `dbo.Sales_SalesOrderHeader`).
- The on-premises SQL Server connection runs through the `lc_on_prem_gateway` on-premises data gateway, authenticated with a dedicated `db_datareader` service account scoped to read-only access — no broader permissions than the pipeline actually needs.

## Why two tables get special-cased

Two source tables use SQL Server types that Fabric's Copy activity can't map directly into a Lakehouse table:

| Table | Column | Type | Handling |
|---|---|---|---|
| `HumanResources.Employee` | `OrganizationNode` | `hierarchyid` | Cast to text via `.ToString()` in the source SQL query |
| `Person.Address` | `SpatialLocation` | `geography` | Cast to text via `.ToString()` in the source SQL query |

Rather than letting either of these break the whole `ForEach` loop, they're pulled out into their own standalone `Copy_employee` and `Copy_address` activities with explicit, hand-written `SELECT` statements that cast the unsupported column before it ever reaches the sink. The generic loop stays generic; the exceptions are handled explicitly instead of being discovered as a runtime failure.

This is the most deliberate piece of engineering judgment in the pipeline: not every table in a source database can be treated identically just because they share a connection and a sink, and a Bronze layer should be designed with room for those exceptions from the start rather than bolted on after something breaks.

## Source

On-premises SQL Server, `AdventureWorks2025` database, accessed via the `lc_on_prem_gateway` gateway.

## Destination

`lh_adventureworks_bronze` Lakehouse, `dbo` schema, `Tables` root folder.

## Files in this folder

- `pl_bronze_ingest_adventureworks.json` — the exported pipeline definition (ARM deployment-template format), containing the full activity logic described above.
- `manifest.json` — export metadata: author, required linked service types, and a rendered canvas preview. Included as-is to reflect the actual Fabric "Export pipeline template" output.
