---
layout: post
title: "Migrating PySpark to dbt: How We Cut Transformation Runtime by 70%"
tags:
- Data Engineering
- dbt
- PySpark
- AWS
- Redshift
---

<p style='text-align: justify;'>At Lingoda we had a marketing attribution model that lived entirely in PySpark. It worked — until it didn't. Long runtimes, opaque logic buried in Python functions, no test coverage, and every change requiring someone who knew the history of the code. When we decided to migrate it to dbt + SQL running on Redshift, we knew the migration itself was just the start. The real challenge was proving the new version produced identical results.</p>

<h4>Why dbt over PySpark for transformations</h4>

<p style='text-align: justify;'>PySpark made sense when we were on a Hadoop-style architecture and needed distributed compute for raw data processing. Once Redshift became our primary warehouse, running PySpark transformations meant spinning up a cluster, paying for it, and maintaining a separate execution environment — just to produce tables that Redshift could have computed with SQL. The operational overhead was constant and the benefits minimal for this workload.</p>

<p style='text-align: justify;'>dbt gave us what PySpark couldn't: version-controlled, documented, testable transformation logic that lives directly in the warehouse. Every model is a SELECT statement. Every column can have a description. Every critical field can have a test. And the lineage is automatic.</p>

<h4>The migration approach</h4>

<p style='text-align: justify;'>The attribution model touched several upstream sources and produced a handful of downstream metrics used in Finance and Marketing dashboards. We couldn't just rewrite and switch over — we needed confidence that nothing changed.</p>

<p style='text-align: justify;'>The approach was a reconciliation framework: run both versions in parallel for a period, then compare outputs row by row. We defined metric parity thresholds (zero tolerance for financial figures, small epsilon for attribution percentages due to floating-point handling differences) and built a comparison job that flagged any divergence automatically.</p>

<p style='text-align: justify;'>A few specific things that needed careful handling:</p>

<ul>
  <li><strong>Session windowing</strong> — PySpark's window functions and Redshift SQL windows behave slightly differently at partition boundaries with ties. We had to explicitly define ORDER BY tiebreakers to make results deterministic on both sides.</li>
  <li><strong>Null handling</strong> — PySpark's default null propagation in aggregations differs from SQL in edge cases. A few COALESCE calls in the dbt model that weren't in the original PySpark surfaced this.</li>
  <li><strong>Incremental strategy</strong> — The PySpark job was a full recompute every run. The dbt model uses an incremental strategy with a merge key on session ID and date. Getting the first incremental run to match the full historical output required a carefully constructed backfill.</li>
</ul>

<h4>The result</h4>

<p style='text-align: justify;'>After several weeks of parallel running with zero divergence, we cut over. The transformation runtime dropped by 70% — from something that took the better part of an hour to one that completes in under 20 minutes, with no separate cluster to manage. The model now has column-level documentation, dbt tests on key fields, and lineage that traces back to raw sources automatically.</p>

<p style='text-align: justify;'>More importantly, the next person who needs to change the attribution logic can read a SQL SELECT statement instead of untangling Python closures. That might be the biggest win of all.</p>
