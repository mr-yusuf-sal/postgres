# Bir SELECT komutu PostgreSQL içinde nasıl çalışır?

Örnek sorgu — dosya boyunca bunu takip ediyoruz:

```sql
SELECT name FROM users WHERE id > 10;
```

## 30 saniyelik özet

```
İstemci → soket
   ↓
[1] PostgresMain          soketten 'Q' mesajını okur
[2] exec_simple_query     tüm boru hattını sürer
[3] raw_parser            metin → SelectStmt   (sözdizimi)
[4] parse_analyze         SelectStmt → Query   (anlam: OID, tip, sütun)
[5] QueryRewrite          view/rule genişletme → Query listesi
[6] planner               Query → PlannedStmt  (maliyet tabanlı seçim)
[7] PortalStart/PortalRun portal + QueryDesc kurulumu
[8] ExecutorStart         Plan ağacı → PlanState ağacı
[9] ExecutePlan           satır satır çekme döngüsü
[10] printtup             her satır → DataRow mesajı → soket
```

Her satır bu ağaçtan **bir kez** yukarı çekilir. Toplu değil, tek tek. Bu "volcano / pull model" mimarisidir.

---

## 1. Mesajın alınması — `PostgresMain`

[src/backend/tcop/postgres.c:4364](../src/backend/tcop/postgres.c#L4364)

Backend süreci sonsuz bir döngüde istemciden protokol mesajı bekler. Basit sorgu protokolünde mesaj tipi `'Q'` (`PqMsg_Query`):

[src/backend/tcop/postgres.c:4930-4946](../src/backend/tcop/postgres.c#L4930-L4946)

```c
case PqMsg_Query:
    {
        const char *query_string;

        SetCurrentStatementStartTimestamp();

        query_string = pq_getmsgstring(&input_message);
        pq_getmsgend(&input_message);

        if (am_walsender)
        {
            if (!exec_replication_command(query_string))
                exec_simple_query(query_string);
        }
        else
            exec_simple_query(query_string);
```

Sorgu artık düz bir C string. Buradan sonrası tek fonksiyonun içinde.

> Not: `PQprepare` / JDBC gibi istemciler **extended protocol** kullanır (`Parse`/`Bind`/`Execute` = `'P'`/`'B'`/`'E'`). Aynı aşamalar geçerlidir, sadece parçalara ayrılmıştır: `exec_parse_message`, `exec_bind_message`, `exec_execute_message`.

---

## 2. Orkestrasyon — `exec_simple_query`

[src/backend/tcop/postgres.c:1030](../src/backend/tcop/postgres.c#L1030)

Tüm hattı sürdüren fonksiyon. Sıralı olarak yaptıkları:

### 2.1 Transaction ve bellek bağlamı

```c
start_xact_command();                              /* :1064 */
drop_unnamed_stmt();                               /* :1072 */
oldcontext = MemoryContextSwitchTo(MessageContext);/* :1077 */
```

`MessageContext`, tek bir istemci mesajının ömrü boyunca yaşayan bellek arenasıdır. Parse ağacı ve plan buraya ayrılır; mesaj bitince tek hamlede serbest bırakılır (`palloc`/`MemoryContextReset` modeli).

### 2.2 Ham ayrıştırma

```c
parsetree_list = pg_parse_query(query_string);     /* :1083 */
```

Bu adım **transaction abort durumunda bile güvenlidir**, çünkü hiçbir katalog erişimi yapmaz — sadece sözdizimi.

### 2.3 Snapshot alma

[src/backend/tcop/postgres.c:1184-1188](../src/backend/tcop/postgres.c#L1184-L1188)

```c
if (analyze_requires_snapshot(parsetree))
{
    PushActiveSnapshot(GetTransactionSnapshot());
    snapshot_set = true;
}
```

MVCC görünürlüğü için bir snapshot gerekir. Analiz/planlama katalogları okuyacağı için burada alınır.

### 2.4 Analiz + rewrite + plan

[src/backend/tcop/postgres.c:1213-1217](../src/backend/tcop/postgres.c#L1213-L1217)

```c
querytree_list = pg_analyze_and_rewrite_fixedparams(parsetree, query_string,
                                                    NULL, 0, NULL);

plantree_list = pg_plan_queries(querytree_list, query_string,
                                CURSOR_OPT_PARALLEL_OK, NULL);
```

`CURSOR_OPT_PARALLEL_OK` — paralel plan üretilmesine izin verildiğini söyleyen bayrak.

### 2.5 Portal kurulumu

[src/backend/tcop/postgres.c:1239-1258](../src/backend/tcop/postgres.c#L1239-L1258)

```c
portal = CreatePortal("", true, true);
portal->visible = false;

PortalDefineQuery(portal, NULL, query_string, commandTag, plantree_list, NULL);

PortalStart(portal, NULL, 0, InvalidSnapshot);
```

**Portal**, çalışan bir sorgunun durumunu tutan nesnedir — cursor'ın backend içindeki karşılığı. `SELECT` için de açılır, çünkü `FETCH`/`LIMIT` semantiği aynı makineyle çalışır.

### 2.6 Çıkış hedefi

[src/backend/tcop/postgres.c:1285-1287](../src/backend/tcop/postgres.c#L1285-L1287)

```c
receiver = CreateDestReceiver(dest);
if (dest == DestRemote)
    SetRemoteDestReceiverParams(receiver, portal);
```

`DestReceiver`, "satırı nereye yazayım" sorusunun cevabıdır. Soyut arayüz sayesinde aynı executor kodu istemciye, `COPY`'ye, `CREATE TABLE AS` hedefine veya tuplestore'a yazabilir.

---

## 3. Sözdizimi ayrıştırma — `raw_parser`

[src/backend/parser/parser.c:42](../src/backend/parser/parser.c#L42)

`flex` (`scan.l`) + `bison` (`gram.y`) ikilisi. İlgili gramer kuralları:

| Kural | Konum |
|---|---|
| `SelectStmt` | [gram.y:13639](../src/backend/parser/gram.y#L13639) |
| `select_with_parens` | [gram.y:13643](../src/backend/parser/gram.y#L13643) |
| `select_no_parens` | [gram.y:13659](../src/backend/parser/gram.y#L13659) |
| `simple_select` | [gram.y:13751](../src/backend/parser/gram.y#L13751) |

Çıktı bir `SelectStmt` düğümüdür — ham, çözümlenmemiş. Bu aşamada:

- `users` sadece bir isim; tablo olup olmadığı **bilinmez**
- `name` sadece bir `ColumnRef`; hangi tabloya ait olduğu **bilinmez**
- `id > 10` bir `A_Expr`; hangi `>` operatörü olduğu **bilinmez**

Kasıtlı olarak böyle: veritabanına dokunmadan sözdizimi hatası verilebilsin diye.

---

## 4. Anlamsal analiz — `transformSelectStmt`

[src/backend/parser/analyze.c:1742](../src/backend/parser/analyze.c#L1742)

Giriş: `SelectStmt` (ham). Çıkış: `Query` (çözümlenmiş).

Sıra **önemlidir** ve koddaki sırayla aynıdır:

```c
qry->commandType = CMD_SELECT;                          /* :1749 */

/* 1. WITH (CTE) — her şeyden bağımsız, en önce */
qry->cteList = transformWithClause(pstate, stmt->withClause);   /* :1755 */

/* 2. FROM — range table'ı doldurur, sonrasında isim çözümleme mümkün olur */
transformFromClause(pstate, stmt->fromClause);                  /* :1774 */

/* 3. SELECT hedef listesi */
qry->targetList = transformTargetList(pstate, stmt->targetList,
                                      EXPR_KIND_SELECT_TARGET); /* :1777 */

/* 4. WHERE */
qual = transformWhereClause(pstate, stmt->whereClause,
                            EXPR_KIND_WHERE, "WHERE");          /* :1792 */

/* 5. HAVING — WHERE ile aynı mekanizma */
qry->havingQual = transformWhereClause(pstate, stmt->havingClause,
                                       EXPR_KIND_HAVING, "HAVING"); /* :1796 */

/* 6. ORDER BY önce (GROUP BY ve DISTINCT sonucuna ihtiyaç duyar) */
qry->sortClause  = transformSortClause(...);                    /* :1805 */
qry->groupClause = transformGroupClause(...);                   /* :1811 */
/* 7. DISTINCT / DISTINCT ON */                                 /* :1820-1842 */
/* 8. LIMIT / OFFSET */                                         /* :1845-1851 */
/* 9. WINDOW — tüm window fonksiyonları görüldükten sonra */    /* :1854 */

qry->rtable   = pstate->p_rtable;                               /* :1862 */
qry->jointree = makeFromExpr(pstate->p_joinlist, qual);         /* :1864 */

assign_query_collations(pstate, qry);                           /* :1877 */

if (pstate->p_hasAggs || qry->groupClause || qry->groupingSets || qry->havingQual)
    parseCheckAggregates(pstate, qry);                          /* :1881 */

return qry;
```

**FROM'un neden ilk olduğu:** `transformFromClause` range table'ı (`pstate->p_rtable`) doldurur. Ondan sonra `name` sütunu `users` RTE'sine `Var(varno=1, varattno=2)` olarak bağlanabilir. Sıra tersine çevrilse isim çözümleme imkânsız olurdu.

Bu aşamada olan diğer şeyler:

- **Tablo kilidi**: `users` üzerinde `AccessShareLock` alınır
- **Yetki kontrolü**: `rteperminfos` doldurulur
- **Operatör çözümleme**: `id > 10` → `OpExpr(opno = int4gt OID)`
- **Tip zorlama**: gerekiyorsa implicit cast düğümleri eklenir

Örneğimizin sonucu, kabaca:

```
Query
 ├─ commandType: CMD_SELECT
 ├─ rtable:      [ RangeTblEntry(relid = users OID, rellockmode = AccessShareLock) ]
 ├─ targetList:  [ TargetEntry(resno=1, resname="name", expr = Var(1,2)) ]
 └─ jointree:    FromExpr
                  ├─ fromlist: [ RangeTblRef(1) ]
                  └─ quals:    OpExpr(int4gt, Var(1,1), Const(10))
```

---

## 5. Yeniden yazma — `pg_rewrite_query`

[src/backend/tcop/postgres.c:816](../src/backend/tcop/postgres.c#L816) → `QueryRewrite` ([src/backend/rewrite/rewriteHandler.c](../src/backend/rewrite/rewriteHandler.c))

Rule sistemi. `users` bir **view** olsaydı burada tanımıyla değiştirilirdi. Row-level security (RLS) politikaları da bu aşamada `WHERE` koşulu olarak eklenir.

Bir `Query` girer, **bir liste** `Query` çıkar — bir rule birden fazla sorgu üretebilir.

Düz tablo üzerindeki basit `SELECT` için bu adım pratikte pas geçilir.

---

## 6. Planlama — `standard_planner`

[src/backend/optimizer/plan/planner.c:346](../src/backend/optimizer/plan/planner.c#L346)

En karmaşık aşama. `Query` → `PlannedStmt`.

### 6.1 Paralellik uygunluğu

[planner.c:413-429](../src/backend/optimizer/plan/planner.c#L413-L429)

```c
if ((cursorOptions & CURSOR_OPT_PARALLEL_OK) != 0 &&
    IsUnderPostmaster &&
    parse->commandType == CMD_SELECT &&
    !parse->hasModifyingCTE &&
    max_parallel_workers_per_gather > 0 &&
    !IsParallelWorker())
{
    glob->maxParallelHazard = max_parallel_hazard(parse);
    glob->parallelModeOK = (glob->maxParallelHazard != PROPARALLEL_UNSAFE);
}
```

Ucuz kontroller geçerse sorgu ağacı taranıp `PROPARALLEL_UNSAFE` fonksiyon aranır.

### 6.2 `tuple_fraction` — kaç satır gerçekten okunacak?

[planner.c:451-477](../src/backend/optimizer/plan/planner.c#L451-L477)

```c
if (cursorOptions & CURSOR_OPT_FAST_PLAN)
    tuple_fraction = cursor_tuple_fraction;   /* cursor: muhtemelen hepsi değil */
else
    tuple_fraction = 0.0;                     /* varsayılan: hepsi lazım */
```

Bu değer maliyet karşılaştırmasını doğrudan değiştirir. `0.0` ise `total_cost`, kesirli ise `startup_cost` ağırlık kazanır. Bir cursor için "ilk satırı hızlı ver" planı daha iyidir.

### 6.3 `subquery_planner` → `grouping_planner`

[planner.c:770](../src/backend/optimizer/plan/planner.c#L770) → [planner.c:1706](../src/backend/optimizer/plan/planner.c#L1706)

`grouping_planner` içindeki ana boru hattı (satır numaraları gerçek çağrı yerleri):

```c
current_rel = query_planner(root, standard_qp_callback, &qp_extra);  /* :1926 */

apply_scanjoin_target_to_paths(root, current_rel, ...);              /* :2045 */

if (have_grouping)
    current_rel = create_grouping_paths(root, ...);                  /* :2071 */
if (activeWindows)
    current_rel = create_window_paths(root, ...);                    /* :2089 */
if (parse->distinctClause)
    current_rel = create_distinct_paths(root, ...);                  /* :2109 */
if (parse->sortClause)
    current_rel = create_ordered_paths(root, ...);                   /* :2124 */

final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);             /* :2140 */
```

Her aşama bir **upper relation** üretir ve bir sonrakinin girdisi olur. Bu, SQL'in mantıksal değerlendirme sırasıdır:

```
FROM/JOIN → WHERE → GROUP BY/HAVING → WINDOW → DISTINCT → ORDER BY → LIMIT
```

### 6.4 Taban ilişki yolları — `make_one_rel`

[src/backend/optimizer/path/allpaths.c:183](../src/backend/optimizer/path/allpaths.c#L183)

```
make_one_rel
 ├─ set_base_rel_pathlists      allpaths.c:384   — her tablo için erişim yolları
 └─ make_rel_from_joinlist      allpaths.c:3847  — join sırası araması
```

`users` için üretilen aday `Path` nesneleri:

- `SeqPath` — tam tablo taraması
- `IndexPath` — `id` üzerinde indeks varsa (`id > 10` indekslenebilir bir koşul)
- `BitmapHeapPath` — orta seçicilikte
- `Partial*Path` + `GatherPath` — paralellik açıksa

Her `Path` üzerinde `startup_cost`, `total_cost`, `rows`, `width` alanları taşınır. `add_path()` baskın olmayanları eler (Pareto dominansı: hem startup hem total maliyette yenilen yol atılır).

`make_rel_from_joinlist` join sırasını arar: az tablo varsa dinamik programlama (`standard_join_search`), `geqo_threshold`'u aşarsa genetik algoritma (`geqo`).

### 6.5 `Path` → `Plan` dönüşümü — `create_plan`

[src/backend/optimizer/plan/createplan.c:345](../src/backend/optimizer/plan/createplan.c#L345) → [create_plan_recurse:396](../src/backend/optimizer/plan/createplan.c#L396)

Planlayıcı ucuz `Path` düğümleriyle **arama yapar**, sonra kazananı ağır `Plan` düğümlerine **çevirir**. `Path` küçüktür (maliyet + kimlik), `Plan` yürütülebilir olandır (target list, qual ifadeleri, alt ağaç).

### 6.6 Son rötuşlar

[planner.c:545-549](../src/backend/optimizer/plan/planner.c#L545-L549) — scrollable cursor ise geriye tarama için `Material` düğümü eklenir:

```c
if (cursorOptions & CURSOR_OPT_SCROLL)
{
    if (!ExecSupportsBackwardScan(top_plan))
        top_plan = materialize_finished_plan(top_plan);
}
```

[planner.c:643](../src/backend/optimizer/plan/planner.c#L643) — `set_plan_references`:

```c
top_plan = set_plan_references(root, top_plan);
```

Bu adım kritik. Yaptıkları:

- `Var` düğümlerini range table indekslerinden **slot offset**'lerine çevirir (`OUTER_VAR`, `INNER_VAR`)
- Alt planları düzleştirir
- Nihai `finalrtable`'ı kurar

Yürütmede isim aramasını sıfırlar — executor artık sadece dizi indeksi kullanır.

[planner.c:655](../src/backend/optimizer/plan/planner.c#L655) — `PlannedStmt` üretilir ve döner.

Örneğimizin planı (`EXPLAIN` çıktısıyla aynı yapı):

```
Seq Scan on users
  Filter: (id > 10)
  Output: name
```

---

## 7. Portal ve QueryDesc — `PortalStart` / `PortalRun`

[src/backend/tcop/pquery.c:430](../src/backend/tcop/pquery.c#L430) — `PortalStart`:

`PlannedStmt`'i sarmalayan bir `QueryDesc` oluşturur, strateji belirler (`PORTAL_ONE_SELECT`, `PORTAL_MULTI_QUERY`, …), `ExecutorStart` çağırır, çıktı `TupleDesc`'ini hesaplar.

[src/backend/tcop/pquery.c:681](../src/backend/tcop/pquery.c#L681) — `PortalRun` → [pquery.c:860](../src/backend/tcop/pquery.c#L860) `PortalRunSelect`:

[pquery.c:912-920](../src/backend/tcop/pquery.c#L912-L920)

```c
if (portal->holdStore)
    nprocessed = RunFromStore(portal, direction, (uint64) count, dest);
else
{
    PushActiveSnapshot(queryDesc->snapshot);
    ExecutorRun(queryDesc, direction, (uint64) count);
    nprocessed = queryDesc->estate->es_processed;
    PopActiveSnapshot();
}
```

`count` = kaç satır isteniyor. Basit `SELECT`'te `FETCH_ALL` → executor'a `0` olarak geçer, yani "sınır yok".

`portal->atEnd` / `portal->atStart` bayrakları cursor konumunu tutar. [pquery.c:922-929](../src/backend/tcop/pquery.c#L922-L929)

---

## 8. Executor kurulumu — `ExecutorStart` / `InitPlan`

[src/backend/executor/execMain.c:124](../src/backend/executor/execMain.c#L124) → [standard_ExecutorStart:143](../src/backend/executor/execMain.c#L143) → [InitPlan:847](../src/backend/executor/execMain.c#L847)

`InitPlan` yaptıkları:

1. `EState` kurar — snapshot, range table, per-query bellek bağlamı
2. Range table'daki ilişkileri açar
3. **`ExecInitNode`** ile `Plan` ağacını `PlanState` ağacına çevirir

[src/backend/executor/execProcnode.c:142](../src/backend/executor/execProcnode.c#L142) — `ExecInitNode` büyük bir `switch` ile düğüm tipine göre dallanır: `T_SeqScan` → `ExecInitSeqScan`, `T_HashJoin` → `ExecInitHashJoin`, …

**Ayrım önemli:**

| | `Plan` | `PlanState` |
|---|---|---|
| Ne | Salt-okunur reçete | Çalışma anı durumu |
| Ömür | Plan cache'te tekrar kullanılabilir | Tek yürütmeye ait |
| İçerik | Node tipi, qual, targetlist | Tuple slot, scan descriptor, sayaçlar |

[src/backend/executor/nodeSeqscan.c:220](../src/backend/executor/nodeSeqscan.c#L220) — `ExecInitSeqScan` ilişkiyi açar, tuple slot ayırır ve **hangi yürütme fonksiyonunun kullanılacağını seçer**: `ExecSeqScan`, `ExecSeqScanWithQual`, `ExecSeqScanWithProject`, … Bu, çalışma anında dal tahmin maliyetini kaldıran bir optimizasyondur.

---

## 9. Yürütme döngüsü — `ExecutePlan`

[src/backend/executor/execMain.c:1722](../src/backend/executor/execMain.c#L1722)

Kalbi burası:

[execMain.c:1765-1824](../src/backend/executor/execMain.c#L1765-L1824)

```c
for (;;)
{
    /* Satır başına ifade bağlamını sıfırla — bellek sızıntısını önler */
    ResetPerTupleExprContext(estate);

    /* Plan ağacından BİR satır çek */
    slot = ExecProcNode(planstate);

    /* NULL = veri bitti */
    if (TupIsNull(slot))
        break;

    /* Junk sütunları (ör. ctid) at */
    if (estate->es_junkFilter != NULL)
        slot = ExecFilterJunk(estate->es_junkFilter, slot);

    /* İstemciye gönder */
    if (sendTuples)
    {
        if (!dest->receiveSlot(slot, dest))
            break;                    /* bağlantı kapandı */
    }

    if (operation == CMD_SELECT)
        (estate->es_processed)++;

    /* LIMIT / FETCH sayacı */
    current_tuple_count++;
    if (numberTuples && numberTuples == current_tuple_count)
        break;
}
```

Dikkat: **satırlar biriktirilmez.** Her satır çekilir, gönderilir, unutulur. 100 milyon satırlık bir `SELECT` sabit bellekte çalışır (plan `Sort`/`Hash` gibi bloklayıcı düğüm içermediği sürece).

`ExecProcNode` bir makrodur; `PlanState->ExecProcNode` fonksiyon işaretçisini çağırır. İlk çağrıda [ExecProcNodeFirst:448](../src/backend/executor/execProcnode.c#L448) devreye girip gerçek fonksiyonu bağlar (instrumentation kancası için).

---

## 10. Bir satırın gerçekten okunması

`ExecProcNode(planstate)` çağrısı örneğimizde `ExecSeqScanWithQual`'a düşer. Zincir:

```
ExecSeqScanWithQual        nodeSeqscan.c:139
  └─ ExecScanExtended      execScan.h:161      (pg_always_inline)
       ├─ ExecScanFetch    execScan.h:~135  → accessMtd
       │    └─ SeqNext     nodeSeqscan.c:52
       │         └─ table_scan_getnextslot   (tableam soyutlaması)
       │              └─ heap_getnextslot    heapam.c:1475
       │                   └─ heapgettup_pagemode / heapgettup
       └─ ExecQual         (id > 10 değerlendirmesi)
```

### 10.1 `SeqNext` — tarama tanımlayıcısını tembel açar

[src/backend/executor/nodeSeqscan.c:52-93](../src/backend/executor/nodeSeqscan.c#L52-L93)

```c
scandesc = node->ss.ss_currentScanDesc;
estate   = node->ss.ps.state;
direction = estate->es_direction;
slot     = node->ss.ss_ScanTupleSlot;

if (scandesc == NULL)
{
    uint32 flags = SO_NONE;

    if (ScanRelIsReadOnly(&node->ss))
        flags |= SO_HINT_REL_READ_ONLY;

    scandesc = table_beginscan(node->ss.ss_currentRelation,
                               estate->es_snapshot,
                               0, NULL, flags);
    node->ss.ss_currentScanDesc = scandesc;
}

if (table_scan_getnextslot(scandesc, direction, slot))
    return slot;
return NULL;
```

Tarama **ilk satır istendiğinde** başlatılır, düğüm init edildiğinde değil. Hiç çalıştırılmayan alt ağaçlar (ör. `NestLoop` içinde erken çıkış) hiç I/O yapmaz.

`estate->es_snapshot` buraya taşınır — MVCC görünürlüğü bu noktada devreye girer.

### 10.2 `heap_getnextslot` — heap katmanı

[src/backend/access/heap/heapam.c:1475-1502](../src/backend/access/heap/heapam.c#L1475-L1502)

```c
if (sscan->rs_flags & SO_ALLOW_PAGEMODE)
    heapgettup_pagemode(scan, direction, sscan->rs_nkeys, sscan->rs_key);
else
    heapgettup(scan, direction, sscan->rs_nkeys, sscan->rs_key);

if (scan->rs_ctup.t_data == NULL)
{
    ExecClearTuple(slot);
    return false;                 /* tarama bitti */
}

pgstat_count_heap_getnext(scan->rs_base.rs_rd);

ExecStoreBufferHeapTuple(&scan->rs_ctup, slot, scan->rs_cbuf);
return true;
```

- **`heapgettup_pagemode`** — bir sayfayı bir kez kilitler, görünür tüm satır işaretçilerini toplar, sonra kilidi bırakıp tek tek dağıtır. Varsayılan ve hızlı yol.
- **`heapgettup`** — satır başına buffer kilidi tutar. Sadece bazı özel durumlarda.
- **`ExecStoreBufferHeapTuple`** — satır **kopyalanmaz**. Slot buffer'a bir referans (pin) tutar. Kopyalamayı önleyen kritik optimizasyon.

MVCC görünürlük kontrolü (`HeapTupleSatisfiesVisibility`) bu katmanın içinde, `heapgettup*` fonksiyonlarında yapılır. Snapshot'ınıza göre silinmiş/gelecekteki satırlar burada elenir.

### 10.3 `ExecScanExtended` — filtre ve projeksiyon

[src/include/executor/execScan.h:161-253](../src/include/executor/execScan.h#L161-L253)

```c
/* Ne qual ne projeksiyon varsa: tüm masrafı atla */
if (!qual && !projInfo)
{
    ResetExprContext(econtext);
    return ExecScanFetch(node, epqstate, accessMtd, recheckMtd);
}

ResetExprContext(econtext);

for (;;)
{
    TupleTableSlot *slot;

    slot = ExecScanFetch(node, epqstate, accessMtd, recheckMtd);

    if (TupIsNull(slot))              /* veri bitti */
    {
        if (projInfo)
            return ExecClearTuple(projInfo->pi_state.resultslot);
        else
            return slot;
    }

    econtext->ecxt_scantuple = slot;  /* ifadelerin göreceği satır */

    if (qual == NULL || ExecQual(qual, econtext))
    {
        if (projInfo)
            return ExecProject(projInfo);   /* SELECT listesini uygula */
        else
            return slot;
    }
    else
        InstrCountFiltered1(node, 1);       /* EXPLAIN "Rows Removed by Filter" */

    ResetExprContext(econtext);       /* elenen satırın belleğini bırak */
}
```

Buradaki üç ayrıntı:

1. **`pg_always_inline`** — fonksiyon inline edilir, `qual`/`projInfo` derleme zamanında `NULL` sabiti geçildiğinde derleyici o dalları tamamen siler. Bu yüzden [nodeSeqscan.c:127-132](../src/backend/executor/nodeSeqscan.c#L127-L132) sabit `NULL` geçirir.
2. **`ExecQual`** — JIT açıksa (`jit_above_cost`) bu ifade LLVM ile makine koduna derlenmiş olabilir. Değilse `execExprInterp.c` içindeki bytecode yorumlayıcısı çalışır.
3. **Döngü** — qual'i geçemeyen satır için dışarı çıkılmaz, içeride tekrar denenir. Yani `ExecScanExtended` her zaman ya kabul edilmiş bir satır ya da EOF döndürür.

Bizim sorgumuzda:
- `qual` = `id > 10` → `ExecQual`
- `projInfo` = `name` sütununu seçen projeksiyon → `ExecProject`

---

## 11. Satırın istemciye yazılması — `printtup`

`ExecutePlan` içindeki `dest->receiveSlot(slot, dest)` çağrısı buraya iner.

[src/backend/access/common/printtup.c:305-383](../src/backend/access/common/printtup.c#L305-L383)

```c
/* Slot'taki tüm sütunları aç (lazy deforming'i tamamla) */
slot_getallattrs(slot);

oldcontext = MemoryContextSwitchTo(myState->tmpcontext);

pq_beginmessage_reuse(buf, PqMsg_DataRow);   /* 'D' mesajı */
pq_sendint16(buf, natts);

for (i = 0; i < natts; ++i)
{
    PrinttupAttrInfo *thisState = myState->myinfo + i;
    Datum attr = slot->tts_values[i];

    if (slot->tts_isnull[i])
    {
        pq_sendint32(buf, -1);       /* NULL = uzunluk -1 */
        continue;
    }

    if (thisState->format == 0)
    {
        /* Metin formatı: tip çıktı fonksiyonu (ör. int4out, textout) */
        char *outputstr = OutputFunctionCall(&thisState->finfo, attr);
        pq_sendcountedtext(buf, outputstr, strlen(outputstr));
    }
    else
    {
        /* İkili format: tip send fonksiyonu (ör. int4send) */
        bytea *outputbytes = SendFunctionCall(&thisState->finfo, attr);
        pq_sendint32(buf, VARSIZE(outputbytes) - VARHDRSZ);
        pq_sendbytes(buf, VARDATA(outputbytes), VARSIZE(outputbytes) - VARHDRSZ);
    }
}

pq_endmessage_reuse(buf);

MemoryContextSwitchTo(oldcontext);
MemoryContextReset(myState->tmpcontext);     /* satır belleğini serbest bırak */

return true;
```

Kilit noktalar:

- **`tmpcontext` reset'i**: her satırdan sonra. `int4out` gibi çıktı fonksiyonları `palloc` yapar; bu olmadan milyon satırlık sorgu belleği tüketirdi.
- **`return true`**: `false` dönerse `ExecutePlan` döngüyü kırar — "istemci gitti" sinyali.
- **`format`**: `printtup_startup` ([printtup.c:112](../src/backend/access/common/printtup.c#L112)) `RowDescription` (`'T'`) mesajını gönderir ve sütun formatlarını sabitler.

Ardından `ExecutorRun` dönüşünde `dest->rShutdown` çağrılır, `exec_simple_query` `CommandComplete` (`'C'`) gönderir, `PostgresMain` döngüsü `ReadyForQuery` (`'Z'`) ile istemciye sırayı devreder.

---

## Tam çağrı zinciri — tek bakışta

```
PostgresMain                              tcop/postgres.c:4364
└─ exec_simple_query                      tcop/postgres.c:1030
   ├─ pg_parse_query                      tcop/postgres.c:617
   │  └─ raw_parser                       parser/parser.c:42
   │     └─ base_yyparse                  parser/gram.y  (simple_select :13751)
   │
   ├─ pg_analyze_and_rewrite_fixedparams  tcop/postgres.c:683
   │  ├─ parse_analyze_fixedparams        parser/analyze.c:128
   │  │  └─ transformTopLevelStmt         parser/analyze.c:272
   │  │     └─ transformStmt              parser/analyze.c:335
   │  │        └─ transformSelectStmt     parser/analyze.c:1742
   │  └─ pg_rewrite_query                 tcop/postgres.c:816
   │     └─ QueryRewrite                  rewrite/rewriteHandler.c
   │
   ├─ pg_plan_queries                     tcop/postgres.c:988
   │  └─ pg_plan_query                    tcop/postgres.c:900
   │     └─ planner                       optimizer/plan/planner.c:328
   │        └─ standard_planner           optimizer/plan/planner.c:346
   │           ├─ subquery_planner        optimizer/plan/planner.c:770
   │           │  └─ grouping_planner     optimizer/plan/planner.c:1706
   │           │     ├─ query_planner     optimizer/plan/planmain.c:54
   │           │     │  └─ make_one_rel   optimizer/path/allpaths.c:183
   │           │     │     ├─ set_base_rel_pathlists    allpaths.c:384
   │           │     │     └─ make_rel_from_joinlist    allpaths.c:3847
   │           │     ├─ create_grouping_paths           planner.c:4052
   │           │     ├─ create_window_paths             planner.c:4798
   │           │     ├─ create_distinct_paths           planner.c:5055
   │           │     └─ create_ordered_paths            planner.c:5573
   │           ├─ create_plan              optimizer/plan/createplan.c:345
   │           └─ set_plan_references      optimizer/plan/setrefs.c
   │
   ├─ PortalStart                          tcop/pquery.c:430
   │  └─ ExecutorStart                     executor/execMain.c:124
   │     └─ standard_ExecutorStart         executor/execMain.c:143
   │        └─ InitPlan                    executor/execMain.c:847
   │           └─ ExecInitNode             executor/execProcnode.c:142
   │              └─ ExecInitSeqScan       executor/nodeSeqscan.c:220
   │
   └─ PortalRun                            tcop/pquery.c:681
      └─ PortalRunSelect                   tcop/pquery.c:860
         └─ ExecutorRun                    executor/execMain.c:308
            └─ standard_ExecutorRun        executor/execMain.c:318
               └─ ExecutePlan              executor/execMain.c:1722
                  ├─ ExecProcNode  ────────┐  (döngü, satır başına 1 kez)
                  │  └─ ExecSeqScan*       │  executor/nodeSeqscan.c:119/139/…
                  │     └─ ExecScanExtended│  include/executor/execScan.h:161
                  │        ├─ SeqNext      │  executor/nodeSeqscan.c:52
                  │        │  └─ heap_getnextslot   access/heap/heapam.c:1475
                  │        ├─ ExecQual     │  (WHERE)
                  │        └─ ExecProject  │  (SELECT listesi)
                  └─ dest->receiveSlot ────┘
                     └─ printtup           access/common/printtup.c:305
```

---

## Akılda kalması gereken 5 şey

1. **Dört ayrı ağaç var**: `SelectStmt` (sözdizimi) → `Query` (anlam) → `Plan` (reçete) → `PlanState` (çalışma durumu). Karıştırmayın.
2. **Pull model**: üst düğüm alt düğümden satır ister. `ExecProcNode` her çağrıda **tek** satır döner.
3. **Path ucuz, Plan pahalı**: planlayıcı yüzlerce `Path` üretip karşılaştırır, sadece kazananı `Plan`'a çevirir.
4. **Bellek bağlamları**: `MessageContext` (mesaj ömrü), `es_query_cxt` (sorgu ömrü), per-tuple context (satır ömrü, her satırda reset).
5. **Satırlar biriktirilmez**: taranıp filtrelenip anında sokete yazılır. Bloklayıcı düğümler (`Sort`, `Hash`, `Materialize`) bu kuralın istisnasıdır.

---

## Kendiniz izlemek için

```bash
# 1. Planı görün
psql -c "EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT name FROM users WHERE id > 10;"

# 2. Parse ağacını loglayın (postgresql.conf)
#    debug_print_parse = on
#    debug_print_rewritten = on
#    debug_print_plan = on
#    debug_pretty_print = on

# 3. gdb ile kırılma noktası
gdb -p $(psql -Atc "SELECT pg_backend_pid()")
(gdb) break exec_simple_query
(gdb) break ExecutePlan
(gdb) break SeqNext
(gdb) continue
```
