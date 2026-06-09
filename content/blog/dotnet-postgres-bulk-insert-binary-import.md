+++
title = "Bulk Inserts in PostgreSQL with Npgsql Binary Import"
date = "2026-06-09"
description = "How to use Npgsql's BeginBinaryImportAsync for high-performance bulk inserts in a .NET Clean Architecture project — including JSONB, temp table bulk updates, and why COPY beats INSERT for volume."
tags = ["dotnet", "dapper", "postgresql", "backend", "clean-architecture"]
+++

This is part of the [Clean Architecture series](/blog/dotnet-clean-architecture-scaffolding/). The [previous post](/blog/dotnet-clean-architecture-dapper-setup/) covered Dapper setup and `IDbConnection` registration. This post covers a pattern used heavily in SampleProject for high-volume writes: PostgreSQL's `COPY` protocol via Npgsql's binary import API.

---

## Why Not Just INSERT?

When you need to write hundreds or thousands of rows, `INSERT` per row is slow:

```csharp
// slow — one round trip per row
foreach (var voucher in journalVouchers)
{
    await _dbConnection.ExecuteAsync(
        "INSERT INTO journal_vouchers (...) VALUES (...)", voucher);
}
```

Each call is a round trip: send SQL, wait for ack, send next. For 500 rows, that's 500 round trips.

PostgreSQL's `COPY` protocol streams all rows in one operation. Npgsql exposes this as `BeginBinaryImportAsync`. Binary format skips text parsing on the server side — faster than even `COPY ... FROM STDIN (FORMAT TEXT)`.

---

## The Core Pattern

All binary imports follow the same four-step structure:

```csharp
using var writer = await ((NpgsqlConnection)_dbConnection)
    .BeginBinaryImportAsync(
        "COPY journal_vouchers (id, voucher_date_ad, voucher_no, type_id, fiscal_year, branch_code, amount) FROM STDIN (FORMAT BINARY)"
    );

foreach (var item in journalVouchers)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(item.Id,            NpgsqlTypes.NpgsqlDbType.Uuid);
    await writer.WriteAsync(item.VoucherDateAD, NpgsqlTypes.NpgsqlDbType.Date);
    await writer.WriteAsync(item.VoucherNo,     NpgsqlTypes.NpgsqlDbType.Text);
    await writer.WriteAsync(item.TypeId,        NpgsqlTypes.NpgsqlDbType.Uuid);
    await writer.WriteAsync(item.FiscalYear,    NpgsqlTypes.NpgsqlDbType.Text);
    await writer.WriteAsync(item.BranchCode,    NpgsqlTypes.NpgsqlDbType.Text);
    await writer.WriteAsync(item.Amount,        NpgsqlTypes.NpgsqlDbType.Numeric);
}

await writer.CompleteAsync();
```

Steps:
1. `BeginBinaryImportAsync` — opens a `COPY` stream for the named table and column list
2. `StartRowAsync` — signals the start of a new row
3. `WriteAsync(value, NpgsqlDbType)` — writes one column value, called once per column in order
4. `CompleteAsync` — finalises and flushes the stream to PostgreSQL

**Column order must match** the column list in the `COPY` statement exactly. Mismatch causes a silent data corruption or a type error.

---

## Why the Cast to NpgsqlConnection

The repository injects `IDbConnection`, not `NpgsqlConnection`:

```csharp
public class JournalVoucherRepository : IJournalVoucherRepository
{
    private readonly IDbConnection _dbConnection;

    public JournalVoucherRepository(IDbConnection dbConnection)
    {
        _dbConnection = dbConnection;
    }
}
```

`BeginBinaryImportAsync` is an Npgsql-specific method — it doesn't exist on `IDbConnection`. The cast is safe because `PersistenceServiceRegistration` always registers an `NpgsqlConnection`:

```csharp
services.AddScoped<IDbConnection>(db =>
{
    var connection = new NpgsqlConnection(connectionString);
    connection.Open();
    return connection;
});
```

---

## NpgsqlDbType Mapping

Always specify `NpgsqlDbType` explicitly. Npgsql can infer types for simple cases, but explicit types are faster and avoid ambiguity — especially for `Uuid`, `Numeric`, and `Jsonb`.

| C# type | NpgsqlDbType |
|---|---|
| `Guid` | `NpgsqlDbType.Uuid` |
| `string` | `NpgsqlDbType.Text` |
| `decimal` | `NpgsqlDbType.Numeric` |
| `DateOnly` | `NpgsqlDbType.Date` |
| `DateTime` | `NpgsqlDbType.Timestamp` |
| `bool` | `NpgsqlDbType.Boolean` |
| `int` | `NpgsqlDbType.Integer` |
| `long` | `NpgsqlDbType.Bigint` |
| `string` (JSON) | `NpgsqlDbType.Jsonb` |

---

## JSONB Columns

Binary import supports `jsonb` directly. From the delete log repository:

```csharp
using var writer = await ((NpgsqlConnection)_dbConnection)
    .BeginBinaryImportAsync(
        "COPY delete_log (id, table_name, deleted_data, reason, deleted_by_id, deleted_by_fullname) FROM STDIN (FORMAT BINARY)"
    );

foreach (var item in deleteLogs)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(item.Id,                NpgsqlTypes.NpgsqlDbType.Uuid);
    await writer.WriteAsync(item.TableName,         NpgsqlTypes.NpgsqlDbType.Text);
    await writer.WriteAsync(item.DeletedData,       NpgsqlTypes.NpgsqlDbType.Jsonb);  // JSON string → jsonb
    await writer.WriteAsync(item.Reason,            NpgsqlTypes.NpgsqlDbType.Text);
    await writer.WriteAsync(item.DeletedById,       NpgsqlTypes.NpgsqlDbType.Uuid);
    await writer.WriteAsync(item.DeletedByFullname, NpgsqlTypes.NpgsqlDbType.Text);
}

await writer.CompleteAsync();
```

Pass the JSON as a plain `string` — Npgsql handles the serialisation.

---

## Sync vs Async

Npgsql provides both sync and async variants. Use async in web contexts to avoid blocking threads:

```csharp
// sync — blocks the thread, use only in background jobs or migration scripts
using var writer = ((NpgsqlConnection)_dbConnection).BeginBinaryImport(
    "COPY journal_vouchers (...) FROM STDIN (FORMAT BINARY)"
);
foreach (var item in items)
{
    writer.StartRow();
    writer.Write(item.Id, NpgsqlTypes.NpgsqlDbType.Uuid);
    // ...
}
writer.Complete();

// async — use in controllers and handlers
using var writer = await ((NpgsqlConnection)_dbConnection).BeginBinaryImportAsync(
    "COPY journal_vouchers (...) FROM STDIN (FORMAT BINARY)"
);
foreach (var item in items)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(item.Id, NpgsqlTypes.NpgsqlDbType.Uuid);
    // ...
}
await writer.CompleteAsync();
```

---

## Bulk Update via Temp Table

`COPY` only supports inserts. For bulk updates, the pattern is: bulk-insert into a temp table, then `UPDATE ... FROM` the temp table.

```csharp
// 1. Create temp table for the batch
await _dbConnection.ExecuteAsync(
    "CREATE TEMPORARY TABLE temp_sbt (id UUID, sell_bill_id UUID, transaction_number TEXT, effective_rate NUMERIC)"
);

// 2. Bulk-copy data into the temp table
using (var writer = await ((NpgsqlConnection)_dbConnection)
    .BeginBinaryImportAsync(
        "COPY temp_sbt (id, sell_bill_id, transaction_number, effective_rate) FROM STDIN BINARY"
    ))
{
    foreach (var t in sellBillTransactions)
    {
        await writer.StartRowAsync();
        await writer.WriteAsync(t.Id,                NpgsqlDbType.Uuid);
        await writer.WriteAsync(t.SellBillId,        NpgsqlDbType.Uuid);
        await writer.WriteAsync(t.TransactionNumber, NpgsqlDbType.Text);
        await writer.WriteAsync(t.EffectiveRate,     NpgsqlDbType.Numeric);
    }
    await writer.CompleteAsync();
}

// 3. Join temp table back to the real table and update
await _dbConnection.ExecuteAsync(@"
    UPDATE sell_bill_transactions sbt
    SET
        is_billed       = true,
        sell_bill_id    = temp_sbt.sell_bill_id,
        effective_rate  = temp_sbt.effective_rate,
        co_amount       = cm_05.closeout_amount,
        cgt             = cm_05.cgt
    FROM temp_sbt
    JOIN cm_05 ON temp_sbt.transaction_number = cm_05.transaction_number
    WHERE sbt.id = temp_sbt.id
");

// 4. Clean up
await _dbConnection.ExecuteAsync("DROP TABLE temp_sbt");
```

This pattern produces two round trips regardless of batch size: one `COPY` stream and one `UPDATE`. For 5000 rows that would otherwise need 5000 `UPDATE` statements, the difference is significant.

Temporary tables in PostgreSQL are session-scoped — they disappear when the connection closes. The explicit `DROP` at the end is a safety measure in case the same connection is reused.

---

## Transactions with Binary Import

Binary import respects open transactions. Wrap the whole operation in the `UnitOfWork` from the previous post to make the bulk write atomic:

```csharp
_unitOfWork.Begin();

await ImportJournalVouchersAsync(vouchers);
await ImportJournalVoucherTransactionsAsync(transactions);

_unitOfWork.Commit();
```

If either import fails mid-stream (connection drop, type error), `Commit()` is never called and the transaction rolls back. No partial writes.

---

## Common Mistakes

**Mismatched column count** — calling `WriteAsync` fewer or more times than there are columns in the `COPY` statement throws at `CompleteAsync`, not during writes.

**Wrong NpgsqlDbType** — passing `NpgsqlDbType.Integer` for a `UUID` column fails at the server. Match types carefully.

**Not calling CompleteAsync** — if you return early or throw without calling `CompleteAsync`, the `COPY` is aborted and no rows are written. The `using` block handles disposal but not completion.

**FORMAT BINARY vs BINARY shorthand** — both work:
```sql
COPY table (...) FROM STDIN (FORMAT BINARY)  -- standard
COPY table (...) FROM STDIN BINARY           -- older shorthand
```

---

## Summary

Binary import via `BeginBinaryImportAsync` is Npgsql's fastest path for bulk inserts. The pattern is:
1. Cast `IDbConnection` to `NpgsqlConnection`
2. Open a `COPY` stream with explicit column list
3. Loop: `StartRowAsync` → `WriteAsync` × N columns
4. `CompleteAsync` to flush

For bulk updates, stage into a temp table via binary import, then run a single `UPDATE ... FROM`.
