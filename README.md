
# Comm-Log Send Reconciliation


## 1. Reconciliation Bridge

| Step | Description | Result | Reason |
|---:|---|---:|---|
| 0 | Naive count of all campaign sends | **30** | Starting point: all communication-log rows in scope |
| 1 | Filter for successful deliveries | **26** | `delivery_status = 900` represents successfully delivered sends |
| 2 | Exclude non-reportable campaigns | **22** | Campaign `9004` is `approval_awaiting`, so its 4 sends are not eligible for reporting |
| 3 | Investigate retry chain `9001 → 9002 → 9003` | **13 attempts → 10 customers** | Multiple attempts to the same customer within a retry chain represent one underlying communication |
| 4 | Investigate retry chain `9201 → 9202` | **6 attempts → 5 customers** | Retry attempts are counted once per distinct customer within the underlying communication |
| 5 | Check standalone campaign `9101` | **7 attempts → 7 events** | A standalone campaign counts each send as a separate event, even when the same customer appears more than once |
| **Final** | **Finance `target_base`** | **22** | Reconciles with Finance's reported value |

> **Note:** The initial delivered + campaign-status filtering also produces 22. The retry analysis was used to validate that this result is consistent with the underlying `target_base` definition.

---

## 2. Final SQL Query

The final query resolves campaign retry chains and applies the appropriate counting rule:

- **Standalone campaign:** every send counts as a separate event.
- **Retry chain:** each distinct customer counts once across the chain.
- Only campaigns eligible for reporting are included.

```sql
WITH RECURSIVE campaign_chain AS (

    SELECT
        id AS campaign_id,
        id AS root_id,
        parent_id
    FROM campaign
    WHERE merchant_id = 501

    UNION ALL

    SELECT
        cc.campaign_id,
        c.id AS root_id,
        c.parent_id
    FROM campaign_chain cc
    JOIN campaign c
        ON cc.parent_id = c.id
),

resolved_campaigns AS (

    SELECT
        campaign_id,
        root_id
    FROM campaign_chain
    WHERE parent_id IS NULL
),

eligible_sends AS (

    SELECT
        cl.id AS send_id,
        cl.customer_id,
        cl.communication_id,
        rc.root_id
    FROM communication_log cl
    JOIN resolved_campaigns rc
        ON cl.communication_id = rc.campaign_id
    JOIN campaign c
        ON cl.communication_id = c.id
    WHERE cl.merchant_id = 501
      AND cl.communication_type = '2'
      AND cl.sent_time >= '2026-10-01'
      AND cl.sent_time < '2026-11-01'
      AND c.creation_status IN (
          'approved',
          'aborted',
          'resumed',
          'stopped'
      )
      AND c.processing_status = 'processed'
),

root_summary AS (

    SELECT
        root_id,
        COUNT(DISTINCT communication_id) AS campaign_count
    FROM eligible_sends
    GROUP BY root_id
)

SELECT
    SUM(
        CASE
            WHEN rs.campaign_count = 1 THEN
                (
                    SELECT COUNT(*)
                    FROM eligible_sends e
                    WHERE e.root_id = rs.root_id
                )
            ELSE
                (
                    SELECT COUNT(DISTINCT e.customer_id)
                    FROM eligible_sends e
                    WHERE e.root_id = rs.root_id
                )
        END
    ) AS target_base
FROM root_summary rs;
```


**Final result: `22`**


## 3. What Surprised Me?

One thing that surprised me was that duplicate customers cannot simply be removed using `COUNT(DISTINCT customer_id)` across the entire dataset. Within a retry chain, multiple attempts to the same customer represent the same underlying communication and should count only once. However, in the standalone campaign `9101`, the same customer appears in two separate sends, and both are valid events.

I also found that campaign `9004` already had communication-log records even though its creation status was `approval_awaiting. This showed that the existence of a send record does not necessarily mean that the campaign is eligible for reporting.

---

## 4. Tools Used

I used **Python with SQLite (`sqlite3`)** to connect to and run SQL queries against the provided `comm_log.db` database.

