# Ordre d'exécution SQL

L'ordre dans lequel tu **écris** une requête SQL n'est pas l'ordre dans lequel elle est **exécutée**.

## Ordre logique d'exécution

| Étape | Clause |
|-------|--------|
| 1 | `FROM` / `JOIN` |
| 2 | `WHERE` |
| 3 | `GROUP BY` |
| 4 | `HAVING` |
| 5 | Fonctions window (`LAG`, `RANK`, `ROW_NUMBER`…) |
| 6 | `SELECT` |
| 7 | `DISTINCT` |
| 8 | `ORDER BY` |
| 9 | `LIMIT` / `OFFSET` |

## Conséquence pratique clé : WHERE vs fonctions window

Le `WHERE` s'exécute **avant** les fonctions window. Donc si tu filtres et calcules un `LAG` dans la **même requête**, le `LAG` ne voit que les lignes qui ont passé le filtre.

```sql
-- ❌ Problématique : le LAG ne voit que les lignes filtrées
select
  day,
  lag(day) over (partition by ID_player order by day) as prev_ts
from connections
where day >= '2026-04-30'  -- filtre appliqué avant le LAG
```

```sql
-- ✅ Correct : séparer en deux CTEs
with all_rows as (
  select
    day,
    lag(day) over (partition by ID_player order by day) as prev_ts  -- LAG voit tout
  from connections
),
filtered as (
  select * from all_rows
  where day >= '2026-04-30'  -- filtre appliqué après, prev_ts est déjà calculé
)
select * from filtered
```

## Alternative : QUALIFY (BigQuery, Snowflake)

`QUALIFY` est évalué **après** les fonctions window, comme un `HAVING` pour les window functions. Utile pour éviter une CTE supplémentaire.

```sql
select
  day,
  lag(day) over (partition by ID_player order by day) as prev_ts
from connections
qualify day = reg_ts  -- filtre sur le résultat du LAG, pas avant
```

## À retenir

> Si tu as besoin de filtrer **sur le résultat** d'une fonction window, tu dois soit passer par une CTE/sous-requête, soit utiliser `QUALIFY`.
