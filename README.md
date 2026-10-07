# sql-walkthrough

An agent skill for explaining batch / SQL / aggregation logic to someone who asks
*"why is this value what it is — is it safe to run again — does it overwrite?"*

Instead of prose, the answer is always one bundle:

1. the source with **real line numbers** from the deployed ref
2. the **real rows** those lines read (≤ 5 rows, anonymised ids)
3. per-line **①②③ arithmetic** on those rows
4. a **results table** where only the input changes
5. a **counter-example**: the one hypothetical condition that would have made it unsafe, executed

Born from a production incident where the proof was a *negative* ("there is no date cut in the SUM") —
which never lands in prose, but lands instantly when `154 / 154 / 50` sit side by side.

## Install

```bash
npx skills add cskwork/sql-walkthrough -g -a claude-code codex opencode -s '*' -y
```

or clone and symlink `skills/sql-walkthrough` into your agent's skills directory.

## Contents

| path | what |
|---|---|
| `skills/sql-walkthrough/SKILL.md` | the method: when, output shape, procedure, pitfalls |
| `skills/sql-walkthrough/scripts/extract_block.py` | pull a MyBatis statement or one CTE from any git ref **with original line numbers**; `--scan` lists date/range predicates, `--sql` emits runnable SQL |
| `skills/sql-walkthrough/references/method.md` | query patterns for finding a sample entity, dumping raw rows, running the block three times, whole-population diff, write test |

## Example

```
$ python3 scripts/extract_block.py my-batch origin/prod src/main/resources/mapper/StatsMapper.xml upsertDailyStats time_total
 1738  time_total AS (/* study time */
 1739      SELECT log.ENTITY_ID, ROUND(SUM(TIMESTAMPDIFF(SECOND, BEGIN_DT, IFNULL(END_DT, ...))) / 60) AS MINUTES
 1740      FROM time_log log
 1741          JOIN access_list sel ON sel.entity_id = log.ENTITY_ID
 1742      WHERE sel.day = #{inputDay} AND BEGIN_DT < END_DT
 1743      GROUP BY log.ENTITY_ID)
```

| input | MINUTES | meaning |
|---|---|---|
| 2026-08-06 | 154 | records after 8/6 are included |
| 2026-08-07 | 154 | different input, same value |
| 2026-08-06 + hypothetical `AND log.BEGIN_DT < '2026-08-07'` | 50 | what a date cut would have done |

Domain-specific companions (real table names, environment runners) belong in a private skill that
references this one; nothing environment-specific lives here.

## License

MIT
