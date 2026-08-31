# WHERE ile HAVING arasındaki fark nereden geliyor?

> PostgreSQL **20devel** ağacı. Satır numaraları bu sürüme aittir.

## 30 saniyelik özet

İkisi de gramerde **aynı şey**: `WHERE a_expr` ve `HAVING a_expr`. Fark, parser'ın
ifadeyi hangi *bağlamda* (`ParseExprKind`) dönüştürdüğü ve sonucu `Query` ağacının
hangi *alanına* koyduğuyla başlar. Geri kalan her şey — aggregate izni, GROUP BY
kolon kontrolü, planda hangi düğüme `Filter` olarak asılacağı — bu iki karardan türer.

```
  SELECT ... FROM t WHERE p1 GROUP BY g HAVING p2

  gram.y            where_clause          having_clause
                    (WHERE a_expr)        (HAVING a_expr)     <-- fark YOK
                          |                     |
  analyze.c        EXPR_KIND_WHERE       EXPR_KIND_HAVING     <-- fark BURADA basliyor
                          |                     |
                   aggregate YASAK       aggregate SERBEST
                          |                     |
  Query agaci      jointree->quals       havingQual           <-- iki ayri alan
                          |                     |
  planner.c               |<--------------------|  (aggregate yoksa HAVING WHERE'e tasinir)
                          |                     |
  plan              Seq Scan Filter        Agg Filter
                          |                     |
  executor          satir basina          grup basina
                    ExecQual              project_aggregates()
```

Üç cümlelik model:

- **WHERE** aggregate göremez, çünkü satırlar henüz gruplanmamıştır; kaynak satır üstünde çalışır.
- **HAVING** aggregate görebilir, çünkü `Agg` düğümünün *üstünde*, grup sonucu üzerinde çalışır.
- Ama HAVING içinde aggregate **yoksa** planner onu sessizce WHERE'e taşır — yani "HAVING her zaman sonra çalışır" ifadesi plan seviyesinde doğru değildir.

## Kaynak haritası

| Dosya | Rolü |
|---|---|
| [../src/backend/parser/gram.y](../src/backend/parser/gram.y) | `where_clause` ve `having_clause` kuralları — ikisi de `a_expr` |
| [../src/include/parser/parse_node.h](../src/include/parser/parse_node.h) | `EXPR_KIND_WHERE` / `EXPR_KIND_HAVING` enum'u |
| [../src/backend/parser/analyze.c](../src/backend/parser/analyze.c) | `transformSelectStmt` — iki cümleciği ayrı alanlara yazar |
| [../src/backend/parser/parse_agg.c](../src/backend/parser/parse_agg.c) | Bağlama göre aggregate/window izni; `parseCheckAggregates` |
| [../src/include/nodes/parsenodes.h](../src/include/nodes/parsenodes.h) | `Query.jointree->quals` ve `Query.havingQual` |
| [../src/backend/optimizer/plan/planner.c](../src/backend/optimizer/plan/planner.c) | HAVING → WHERE taşıma (pushdown) |
| [../src/backend/optimizer/plan/initsplan.c](../src/backend/optimizer/plan/initsplan.c) | WHERE qual'ının `baserestrictinfo`'ya dağıtılması |
| [../src/backend/optimizer/plan/createplan.c](../src/backend/optimizer/plan/createplan.c) | Qual'ların `Seq Scan` / `Agg` düğümlerine asılması |
| [../src/backend/executor/nodeAgg.c](../src/backend/executor/nodeAgg.c) | HAVING'in grup başına değerlendirilmesi |

---

## 1. Gramerde fark yok

İki kural birbirinin kopyası. Parser bir tanesine ayrıcalık tanımaz:

```c
where_clause:
			WHERE a_expr							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NULL; }
		;
```
[../src/backend/parser/gram.y:14955-14958](../src/backend/parser/gram.y#L14955-L14958)

```c
having_clause:
			HAVING a_expr							{ $$ = $2; }
			| /*EMPTY*/								{ $$ = NULL; }
		;
```
[../src/backend/parser/gram.y:14338-14341](../src/backend/parser/gram.y#L14338-L14341)

Yani `WHERE count(*) > 3` **sözdizimsel olarak geçerlidir**. Hata gramerde değil,
bir sonraki aşamada verilir.

## 2. Fark burada doğuyor: `ParseExprKind`

`transformSelectStmt` iki cümleciği de **aynı fonksiyona** verir; tek fark
üçüncü argüman:

```c
	/* transform WHERE */
	qual = transformWhereClause(pstate, stmt->whereClause,
								EXPR_KIND_WHERE, "WHERE");

	/* initial processing of HAVING clause is much like WHERE clause */
	qry->havingQual = transformWhereClause(pstate, stmt->havingClause,
										   EXPR_KIND_HAVING, "HAVING");
```
[../src/backend/parser/analyze.c:1792-1798](../src/backend/parser/analyze.c#L1792-L1798)

Kod yorumunun kendisi de söylüyor: *"initial processing of HAVING clause is much
like WHERE clause"*. İkisi de bool'a zorlanır, ikisi de aynı ifade dönüşümünden geçer.

Enum tanımı:

```c
	EXPR_KIND_WHERE,			/* WHERE */
	EXPR_KIND_HAVING,			/* HAVING */
```
[../src/include/parser/parse_node.h:46-47](../src/include/parser/parse_node.h#L46-L47)

Bu değer `pstate->p_expr_kind` alanına konur ve ifade ağacında aşağı doğru taşınır.
Aggregate, window function, alt sorgu gibi her özel yapı bu değere bakarak
"burada olabilir miyim?" diye sorar.

## 3. Aggregate izni: tek `switch`, iki farklı `case`

`check_agglevels_and_constraints`, bir `Aggref` veya `GroupingFunc` gördüğünde
bulunduğu bağlama bakar:

```c
		case EXPR_KIND_WHERE:
			errkind = true;
			break;
...
		case EXPR_KIND_HAVING:
			/* okay */
			break;
```
[../src/backend/parser/parse_agg.c:407-419](../src/backend/parser/parse_agg.c#L407-L419)

`errkind = true` ise sonda genel hata basılır:

```c
	if (errkind)
	{
		if (isAgg)
			/* translator: %s is name of a SQL construct, eg GROUP BY */
			err = _("aggregate functions are not allowed in %s");
...
		ereport(ERROR,
				(errcode(ERRCODE_GROUPING_ERROR),
				 errmsg_internal(err,
								 ParseExprKindName(pstate->p_expr_kind)),
```
[../src/backend/parser/parse_agg.c:618-630](../src/backend/parser/parse_agg.c#L618-L630)

Bilinen mesaj tam olarak buradan çıkıyor — regression testinde de görülüyor:

```
select ten, sum(distinct four) from onek a
group by ten
having exists (select 1 from onek b
               where sum(distinct a.four + b.four) = b.four);
ERROR:  aggregate functions are not allowed in WHERE
```
[../src/test/regress/expected/aggregates.out:802-808](../src/test/regress/expected/aggregates.out#L802-L808)

**Not:** window function için ikisi de yasaktır. `check_windowfunc_call` içinde
`EXPR_KIND_WHERE` de `EXPR_KIND_HAVING` de `errkind = true` yapar:

[../src/backend/parser/parse_agg.c:947-955](../src/backend/parser/parse_agg.c#L947-L955)

Yani `HAVING row_number() over () > 1` da hata verir. HAVING'in ayrıcalığı sadece
aggregate'e özeldir, "gruplama sonrası her şey serbest" demek değildir.

## 4. `Query` ağacında iki ayrı alan

WHERE, jointree'nin qual'ı olur:

```c
	qry->jointree = makeFromExpr(pstate->p_joinlist, qual);
```
[../src/backend/parser/analyze.c:1864](../src/backend/parser/analyze.c#L1864)

HAVING ise kendi alanında durur:

```c
	Node	   *havingQual;		/* qualifications applied to groups */
```
[../src/include/nodes/parsenodes.h:226](../src/include/nodes/parsenodes.h#L226)

Yorumdaki fark tek kelimelik ama belirleyici: WHERE **satırlara**, HAVING
**gruplara** uygulanır. Ayrıca `hasAggs` bayrağının tanımı da HAVING'i kapsar:

```c
	/* has aggregates in tlist or havingQual */
	bool		hasAggs pg_node_attr(query_jumble_ignore);
```
[../src/include/nodes/parsenodes.h:154-155](../src/include/nodes/parsenodes.h#L154-L155)

## 5. GROUP BY kolon kontrolü HAVING'i de kapsar

`parseCheckAggregates` hem targetlist'i hem HAVING'i aynı iki fonksiyondan geçirir:

```c
	clause = (Node *) qry->havingQual;
	finalize_grouping_exprs(clause, pstate, qry,
							groupClauses, hasJoinRTEs,
							have_non_var_grouping);
	if (hasJoinRTEs)
		clause = flatten_join_alias_for_parser(qry, clause, 0);
	qry->havingQual =
		substitute_grouped_columns(clause, pstate, qry,
								   groupClauses, groupClauseCommonVars,
								   gset_common,
								   have_non_var_grouping,
								   &func_grouped_rels);
```
[../src/backend/parser/parse_agg.c:1334-1345](../src/backend/parser/parse_agg.c#L1334-L1345)

Sonuç: HAVING içinde gruplanmamış bir kolona dokunursanız
*"column ... must appear in the GROUP BY clause"* alırsınız. WHERE bu kontrolden
geçmez — orada zaten her satır tek başınadır. Bu, HAVING'in **ikinci** kısıtıdır:
aggregate serbest ama çıplak kolon yasak; WHERE'de tam tersi.

`substitute_grouped_columns` ayrıca HAVING'deki gruplanmış Var'ları `RTE_GROUP`
RTE'sine bakan Var'larla değiştirir — planner'da işe yarayacak bir işaretleme.

## 6. Sürpriz: planner HAVING'i çoğu zaman WHERE'e taşır

`subquery_planner` içindeki bu döngü, notun en önemli parçası:

```c
	newHaving = NIL;
	havingIdx = 0;
	foreach(l, (List *) parse->havingQual)
	{
		Node	   *havingclause = (Node *) lfirst(l);

		if (contain_agg_clause(havingclause) ||
			contain_volatile_functions(havingclause) ||
			contain_subplans(havingclause) ||
			bms_is_member(havingIdx, havingPushdownConflicts) ||
			(parse->groupClause && parse->groupingSets &&
			 bms_is_member(root->group_rtindex, pull_varnos(root, havingclause))))
		{
			/* keep it in HAVING */
			newHaving = lappend(newHaving, havingclause);
		}
```
[../src/backend/optimizer/plan/planner.c:1325-1340](../src/backend/optimizer/plan/planner.c#L1325-L1340)

Üç davranış var:

1. **HAVING'de kalır** — aggregate, volatile fonksiyon, subplan, opfamily/collation
   çakışması veya grouping set tarafından NULL'lanabilen kolon içeriyorsa.
2. **WHERE'e taşınır** (kopya değil, taşıma) — GROUP BY var ve boş grouping set yoksa:

```c
		else if (parse->groupClause &&
				 (parse->groupingSets == NIL ||
				  (List *) linitial(parse->groupingSets) != NIL))
		{
			/* There is GROUP BY, but no empty grouping set */
			Node	   *whereclause;

			/* Preprocess the HAVING clause fully */
			whereclause = preprocess_expression(root, havingclause,
												EXPRKIND_QUAL);
			/* ... and move it to WHERE */
			parse->jointree->quals = (Node *)
				list_concat((List *) parse->jointree->quals,
							(List *) whereclause);
		}
```
[../src/backend/optimizer/plan/planner.c:1341-1356](../src/backend/optimizer/plan/planner.c#L1341-L1356)

3. **Her ikisine birden kopyalanır** — boş grouping set varsa (örtük olarak da olabilir).
   Sebep yorumda: boş grup, hiç satır gelmese bile bir çıktı satırı üretir; o satırın
   elenmesi için qual HAVING'de de kalmalıdır:
   [../src/backend/optimizer/plan/planner.c:1357-1370](../src/backend/optimizer/plan/planner.c#L1357-L1370)

Taşımanın gerekçesi kod yorumunda açık:

> *"Otherwise, we move or copy the HAVING clause into WHERE, in hopes of
> eliminating tuples before aggregation instead of after."*

[../src/backend/optimizer/plan/planner.c:1298-1301](../src/backend/optimizer/plan/planner.c#L1298-L1301)

Yani **anlamsal** olarak HAVING gruplama sonrasıdır; **fiziksel** olarak aggregate
içermeyen bir HAVING gruplama öncesine indirilir. `WHERE x > 1` ile
`GROUP BY x HAVING x > 1` aynı planı üretir.

Aynı yorumda bir heuristik de var: subplan içeren cümlecikler pahalı sayılıp grup
başına bir kez çalışsınlar diye HAVING'de bırakılır.

### 6.1 20devel'e özgü fren: `find_having_conflicts`

Bu ağaçta pushdown'a yeni bir koruma eklenmiş. GROUP BY'ın eşitlik operatörüyle
HAVING'in kullandığı operatör farklı opfamily/collation'a aitse taşıma yapılmaz:

```c
/*
 * find_having_conflicts
 *	  Identify HAVING clauses that must not be moved to WHERE because they
 *	  apply a different equivalence relation than GROUP BY.  Pushing such a
 *	  clause to WHERE would filter individual rows before grouping happens,
 *	  eliminating rows that GROUP BY would have merged into a single group
 *	  and thereby changing aggregate results.
 */
```
[../src/backend/optimizer/plan/planner.c:1565-1590](../src/backend/optimizer/plan/planner.c#L1565-L1590)

Fonksiyon, HAVING listesindeki çakışan cümleciklerin indekslerini bir `Bitmapset`
olarak döndürür; yukarıdaki döngüde `bms_is_member(havingIdx, havingPushdownConflicts)`
kontrolü bunu okur. `flatten_group_exprs`'ten **önce** çağrılması şart, çünkü
GROUP Var'ları (RTE_GROUP'a bakan Var'lar) grup collation'ını ve eqop'unu taşır.

## 7. Plan düğümüne asılma: `Filter` nereye yazılır

**WHERE** → `distribute_quals_to_rels` → ilgili base rel'in `baserestrictinfo`'su
([../src/backend/optimizer/plan/initsplan.c:3478](../src/backend/optimizer/plan/initsplan.c#L3478))
→ tarama düğümünün qual'ı:

```c
	/* Sort clauses into best execution order */
	scan_clauses = order_qual_clauses(root, scan_clauses);

	/* Reduce RestrictInfo list to bare expressions; ignore pseudoconstants */
	scan_clauses = extract_actual_clauses(scan_clauses, false);
...
	scan_plan = make_seqscan(tlist,
							 scan_clauses,
							 scan_relid);
```
[../src/backend/optimizer/plan/createplan.c:2772-2787](../src/backend/optimizer/plan/createplan.c#L2772-L2787)

**HAVING** → `AggPath->qual` → `Agg` düğümünün qual'ı:

```c
	quals = order_qual_clauses(root, best_path->qual);

	plan = make_agg(tlist, quals,
					best_path->aggstrategy,
```
[../src/backend/optimizer/plan/createplan.c:2178-2182](../src/backend/optimizer/plan/createplan.c#L2178-L2182)

EXPLAIN çıktısında ikisi de `Filter:` diye görünür — ama **hangi düğümün altında**
göründüğü farkı ele verir.

## 8. Çalışma zamanı

WHERE, tarama düğümünde satır başına `ExecQual` ile değerlendirilir. HAVING ise
grup projeksiyonundan hemen önce:

```c
static TupleTableSlot *
project_aggregates(AggState *aggstate)
{
	ExprContext *econtext = aggstate->ss.ps.ps_ExprContext;

	/*
	 * Check the qual (HAVING clause); if the group does not match, ignore it.
	 */
	if (ExecQual(aggstate->ss.ps.qual, econtext))
	{
		...
		return ExecProject(aggstate->ss.ps.ps_ProjInfo);
	}
	else
		InstrCountFiltered1(aggstate, 1);

	return NULL;
}
```
[../src/backend/executor/nodeAgg.c:1371-1388](../src/backend/executor/nodeAgg.c#L1371-L1388)

Qual'ın init'i:

```c
	aggstate->ss.ps.qual =
		ExecInitQual(node->plan.qual, (PlanState *) aggstate);
```
[../src/backend/executor/nodeAgg.c:3473-3474](../src/backend/executor/nodeAgg.c#L3473-L3474)

Bu satırın üstündeki yorum, HAVING'deki `Aggref`'lerin nasıl bulunduğunu anlatır:
targetlist'tekiler `ExecAssignProjectionInfo` sırasında, qual'dakiler burada.
İkisi de `aggstate->aggs` listesine girer — yani `HAVING sum(x) > 10` içindeki
`sum(x)`, targetlist'te hiç görünmese bile hesaplanır.

Elenen grup sayısı `InstrCountFiltered1` ile sayılır; `EXPLAIN ANALYZE` çıktısındaki
`Rows Removed by Filter` satırı buradan gelir.

## 9. Kanıt: regression test çıktıları

### HAVING WHERE'e taşındı

```
-- the clause can be pushed down to WHERE
explain (costs off)
select a, count(*) from t_having group by a having a = row(1.0)::avg_rec;
               QUERY PLAN
----------------------------------------
 GroupAggregate
   ->  Seq Scan on t_having
         Filter: (a = '(1.0)'::avg_rec)
```
[../src/test/regress/expected/aggregates.out:1757-1765](../src/test/regress/expected/aggregates.out#L1757-L1765)

`Filter` **taramanın** altında. HAVING yazdınız, WHERE çalıştı.

### HAVING'de kaldı (opfamily çakışması)

```
-- the clause must stay in HAVING
explain (costs off)
select a, count(*) from t_having group by a having a *= row(1.0)::avg_rec;
            QUERY PLAN
-----------------------------------
 HashAggregate
   Group Key: a
   Filter: (a *= '(1.0)'::avg_rec)
   ->  Seq Scan on t_having
```
[../src/test/regress/expected/aggregates.out:1724-1733](../src/test/regress/expected/aggregates.out#L1724-L1733)

`Filter` **HashAggregate'in** altında. Aynı sorgu şekli, farklı operatör, farklı plan.

### Tek HAVING ikiye bölündü (grouping sets)

```
select a, b, count(*) from gstest2 group by grouping sets ((a, b), (a)) having a > 1 and b > 1;
           QUERY PLAN
---------------------------------
 GroupAggregate
   Group Key: a, b
   Group Key: a
   Filter: (b > 1)
   ->  Sort
         Sort Key: a, b
         ->  Seq Scan on gstest2
               Filter: (a > 1)
```
[../src/test/regress/expected/groupingsets.out:1028-1040](../src/test/regress/expected/groupingsets.out#L1028-L1040)

`HAVING a > 1 AND b > 1` ikiye ayrıldı: `a` her grouping set'te mevcut → WHERE'e indi.
`b`, `(a)` set'inde NULL'lanıyor → HAVING'de kaldı. Bu tam olarak
`bms_is_member(root->group_rtindex, pull_varnos(...))` kontrolünün sonucudur.

### Kopyalama (boş grouping set)

```
-- test pushdown of degenerate HAVING clause
select count(*) from gstest2 group by grouping sets (()) having false;
            QUERY PLAN
-----------------------------------
 Aggregate
   Group Key: ()
   Filter: false
   ->  Result
         Replaces: Scan on gstest2
         One-Time Filter: false
```
[../src/test/regress/expected/groupingsets.out:1067-1077](../src/test/regress/expected/groupingsets.out#L1067-L1077)

Aynı `false` iki yerde birden: `Filter: false` (Agg'de kaldı) ve
`One-Time Filter: false` (WHERE'e kopyalandı). Üçüncü davranış dalı budur.

## 10. Karşılaştırma tablosu

| | WHERE | HAVING |
|---|---|---|
| Gramer | `WHERE a_expr` | `HAVING a_expr` — aynı |
| `ParseExprKind` | `EXPR_KIND_WHERE` | `EXPR_KIND_HAVING` |
| Aggregate | yasak (`errkind = true`) | serbest |
| Window function | yasak | yasak |
| Gruplanmamış kolon | serbest | yasak (`parseCheckAggregates`) |
| `Query` alanı | `jointree->quals` | `havingQual` |
| GROUP BY gerektirir mi | hayır | hayır (o zaman tek grup) |
| Planner'da taşınır mı | — | aggregate yoksa WHERE'e taşınır/kopyalanır |
| Plan düğümü | `Seq Scan` / `Index Scan` qual'ı | `Agg` qual'ı |
| Değerlendirme birimi | satır | grup |
| Index kullanabilir mi | evet | hayır (WHERE'e taşınmadıysa) |

## Tek sayfalık özet

```
                    SELECT ... WHERE p1 GROUP BY g HAVING p2
                                    |
   +----------------------------- gram.y ------------------------------+
   |  where_clause: WHERE a_expr        having_clause: HAVING a_expr   |
   |  (gram.y:14955)                    (gram.y:14338)                 |
   +--------------------|---------------------------|------------------+
                        |                           |
   +---------------- analyze.c:1792-1798 --------------------------+
   |  transformWhereClause(..., EXPR_KIND_WHERE)                   |
   |  transformWhereClause(..., EXPR_KIND_HAVING)                  |
   +--------------------|---------------------------|-------------+
                        |                           |
              parse_agg.c:407              parse_agg.c:417
              errkind = true               /* okay */
              "aggregate functions         + parse_agg.c:1334
               are not allowed in            GROUP BY kolon kontrolu
               WHERE"
                        |                           |
   Query:      jointree->quals              havingQual
               (analyze.c:1864)             (parsenodes.h:226)
                        |                           |
                        |                  planner.c:1325-1375
                        |                           |
                        |     +---------------------+---------------------+
                        |     |                     |                     |
                        |  aggregate/          GROUP BY var,          bos grouping
                        |  volatile/           bos set yok            set var
                        |  subplan/            -> WHERE'e             -> ikisine
                        |  opfamily cakismasi     TASINIR                KOPYALANIR
                        |     |                     |                     |
                        |  HAVING'de kalir          |                     |
                        |     |                     |                     |
                        |<----+---------------------+---------------------+
                        |                           |
        initsplan.c:3478 baserestrictinfo    createplan.c:2178 AggPath->qual
                        |                           |
        createplan.c:2785 make_seqscan       createplan.c:2180 make_agg
                        |                           |
   EXPLAIN:     Seq Scan on t                Agg
                  Filter: p1                   Filter: p2
                        |                           |
   Executor:    satir basina ExecQual        nodeAgg.c:1378
                                             project_aggregates()
                                             grup basina ExecQual
```

## Hata ayıklama

Bir HAVING'in gerçekten nerede çalıştığını görmek için:

```sql
EXPLAIN (COSTS OFF)
SELECT a, count(*) FROM t GROUP BY a HAVING a > 1;
--  Filter taramanin altindaysa -> WHERE'e tasinmis
--  Filter Agg'in altindaysa    -> HAVING'de kalmis

EXPLAIN (ANALYZE, COSTS OFF)
SELECT a, count(*) FROM t GROUP BY a HAVING count(*) > 1;
--  Agg satirindaki "Rows Removed by Filter" = elenen GRUP sayisi
--  (nodeAgg.c:1385, InstrCountFiltered1)
```

Ayrıştırılmış ağacı doğrudan görmek için sunucuyu `debug_print_parse = on` ile
çalıştırıp log'da `jointree->quals` ile `havingQual` alanlarını karşılaştırın.
