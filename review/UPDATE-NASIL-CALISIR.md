# Bir UPDATE komutu PostgreSQL içinde nasıl çalışır?

Örnek sorgu — dosya boyunca bunu takip ediyoruz:

```sql
UPDATE users SET name = 'ali' WHERE id = 42;
```

Kaynak ağacı: PostgreSQL **20devel** ([meson.build](../meson.build)). Satır numaraları sürüme bağlıdır.

## 30 saniyelik özet

PostgreSQL'de UPDATE **yerinde değiştirme değildir**. Eski satır ölü işaretlenir, yeni bir satır yazılır:

```
                       ESKİ TUPLE                YENİ TUPLE
                    ┌──────────────┐          ┌──────────────┐
  UPDATE öncesi:    │ xmin=100     │          │              │
                    │ xmax=0       │          │   (yok)      │
                    │ t_ctid=kendi │          │              │
                    └──────────────┘          └──────────────┘

                    ┌──────────────┐          ┌──────────────┐
  UPDATE sonrası:   │ xmin=100     │  t_ctid  │ xmin=500     │
                    │ xmax=500  ───┼─────────►│ xmax=0       │
                    │ HOT_UPDATED  │          │ HEAP_UPDATED │
                    └──────────────┘          └──────────────┘
                       ↑ ölü olacak              ↑ canlı sürüm
```

Boru hattı:

```
[1] preprocess_targetlist   plana gizli "ctid" junk kolonu ekler
[2] ExecModifyTable         alt plandan satır çeker, junk ctid'yi okur
[3] table_tuple_fetch_...   ctid ile ESKİ tuple'ı diskten getirir
[4] ExecGetUpdateNewTuple   eski tuple + değişen kolonlar → YENİ tuple
[5] ExecUpdatePrologue      BEFORE ROW trigger'ları
[6] table_tuple_update      → heap_update():
        ├─ hangi kolonlar değişti?      (HOT / kilit modu kararı)
        ├─ eşzamanlı yazan var mı?      (bekle, EPQ)
        ├─ TOAST gerekiyor mu?
        ├─ yeni tuple aynı sayfaya sığar mı?
        ├─ eski tuple: xmax = benim xid, t_ctid = yeni tuple
        ├─ yeni tuple: sayfaya yaz
        └─ WAL kaydı (XLOG_HEAP_UPDATE / XLOG_HEAP_HOT_UPDATE)
[7] ExecUpdateEpilogue      index girdileri + AFTER ROW trigger'ları
```

Üç madde yeter:

- **MVCC:** eski sürüm silinmez, sadece `xmax` alanına silen transaction yazılır. Eski sürümü kimse göremez hale gelince VACUUM/pruning temizler.
- **HOT (Heap Only Tuple):** yeni sürüm **aynı sayfaya** sığıyorsa ve **hiçbir indexlenmiş kolon** değişmediyse, index'e hiç dokunulmaz. Bu tek optimizasyon UPDATE maliyetinin çoğunu belirler.
- **Zincir:** eski tuple'ın `t_ctid` alanı yeni sürümü işaret eder. Index hâlâ eski satır işaretçisine bakar; okuyucu zinciri takip eder.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/optimizer/prep/preptlist.c](../src/backend/optimizer/prep/preptlist.c) | Hedef listeyi hazırlar; `update_colnos` çıkarır |
| [../src/backend/optimizer/util/appendinfo.c](../src/backend/optimizer/util/appendinfo.c) | Satır kimliği (`ctid`, `wholerow`) junk kolonlarını ekler |
| [../src/backend/executor/nodeModifyTable.c](../src/backend/executor/nodeModifyTable.c) | `ExecModifyTable`, `ExecUpdate` — UPDATE'in executor tarafı |
| [../src/backend/executor/execIndexing.c](../src/backend/executor/execIndexing.c) | Index girdilerinin eklenmesi, `indexUnchanged` ipucu |
| [../src/backend/executor/execMain.c](../src/backend/executor/execMain.c) | `EvalPlanQual` — eşzamanlı UPDATE çakışması çözümü |
| [../src/backend/access/heap/heapam_handler.c](../src/backend/access/heap/heapam_handler.c) | Table AM köprüsü: `heapam_tuple_update` |
| [../src/backend/access/heap/heapam.c](../src/backend/access/heap/heapam.c) | **`heap_update`** — işin gerçekten yapıldığı yer |
| [../src/backend/access/heap/heapam_indexscan.c](../src/backend/access/heap/heapam_indexscan.c) | `heap_hot_search_buffer` — HOT zincirini takip eder |
| [../src/backend/access/heap/pruneheap.c](../src/backend/access/heap/pruneheap.c) | Sayfa içi budama, redirect line pointer'lar |
| [../src/backend/access/heap/README.HOT](../src/backend/access/heap/README.HOT) | HOT tasarım belgesi (okunması şart) |
| [../src/include/access/htup_details.h](../src/include/access/htup_details.h) | Tuple başlığı, `t_ctid`, infomask bitleri |
| [../src/include/access/heapam_xlog.h](../src/include/access/heapam_xlog.h) | `xl_heap_update` WAL kayıt biçimi |

---

## 1. Planlayıcı: "hangi satırı güncelleyeceğim?"

`SELECT`'ten farklı olarak UPDATE'in bir hedef satırı bulması gerekir. Bunu planlayıcı, plana **junk (gizli) kolon** ekleyerek çözer.

[../src/backend/optimizer/prep/preptlist.c:12-15](../src/backend/optimizer/prep/preptlist.c#L12-L15)

```c
 * For UPDATE and DELETE queries, the targetlist must also contain "junk"
 * tlist entries needed to allow the executor to identify the rows to be
 * updated or deleted; for example, the ctid of a heap row.  (The planner
 * adds these; they're not in what we receive from the parser/rewriter.)
```

İki iş yapılır. Önce **hangi kolonların hedef olduğu** ayrı bir listeye çıkarılır:

[../src/backend/optimizer/prep/preptlist.c:110-111](../src/backend/optimizer/prep/preptlist.c#L110-L111)

```c
	else if (command_type == CMD_UPDATE)
		root->update_colnos = extract_update_targetlist_colnos(tlist);
```

Sonra satır kimliği eklenir:

[../src/backend/optimizer/prep/preptlist.c:121-128](../src/backend/optimizer/prep/preptlist.c#L121-L128)

```c
	if ((command_type == CMD_UPDATE || command_type == CMD_DELETE ||
		 command_type == CMD_MERGE) &&
		!target_rte->inh)
	{
		/* row-identity logic expects to add stuff to processed_tlist */
		root->processed_tlist = tlist;
		add_row_identity_columns(root, result_relation,
								 target_rte, target_relation);
```

Normal bir tabloda bu kimlik `ctid` sistem kolonudur:

[../src/backend/optimizer/util/appendinfo.c:969-983](../src/backend/optimizer/util/appendinfo.c#L969-L983)

```c
	if (relkind == RELKIND_RELATION ||
		relkind == RELKIND_MATVIEW ||
		relkind == RELKIND_PARTITIONED_TABLE)
	{
		/*
		 * Emit CTID so that executor can find the row to merge, update or
		 * delete.
		 */
		var = makeVar(rtindex,
					  SelfItemPointerAttributeNumber,
					  TIDOID,
					  -1,
					  InvalidOid,
					  0);
		add_row_identity_var(root, var, rtindex, "ctid");
	}
```

Foreign table'da `ctid` yoktur; FDW kendi kimliğini ekler, ayrıca UPDATE için `wholerow` istenir ([appendinfo.c:1011-1022](../src/backend/optimizer/util/appendinfo.c#L1011-L1022)).

Sonuç: alt planın ürettiği her satırda **sadece değişen kolonların yeni değerleri** + gizli bir `ctid` bulunur. Değişmeyen kolonlar planda yoktur.

> Bu, önemli bir tasarım kararıdır: alt plan tabloya yeniden dokunmaz; eski satırı executor `ctid` ile kendisi getirir.

---

## 2. Executor döngüsü — `ExecModifyTable`

[../src/backend/executor/nodeModifyTable.c:4635](../src/backend/executor/nodeModifyTable.c#L4635)

Her tur alt plandan bir satır çeker, önce junk `ctid`'yi ayıklar:

[../src/backend/executor/nodeModifyTable.c:4864-4900](../src/backend/executor/nodeModifyTable.c#L4864-L4900)

```c
				datum = ExecGetJunkAttribute(slot,
											 resultRelInfo->ri_RowIdAttNo,
											 &isNull);
				...
					elog(ERROR, "ctid is NULL");
				}

				tupleid = (ItemPointer) DatumGetPointer(datum);
				tuple_ctid = *tupleid;	/* be sure we don't free ctid!! */
				tupleid = &tuple_ctid;
```

Sonra `CMD_UPDATE` dalı. Burada **eski tuple diskten yeniden okunur**:

[../src/backend/executor/nodeModifyTable.c:4984-5022](../src/backend/executor/nodeModifyTable.c#L4984-L5022)

```c
			case CMD_UPDATE:
				tuplock = false;

				/* Initialize projection info if first time for this table */
				if (unlikely(!resultRelInfo->ri_projectNewInfoValid))
					ExecInitUpdateProjection(node, resultRelInfo);

				/*
				 * Make the new tuple by combining plan's output tuple with
				 * the old tuple being updated.
				 */
				oldSlot = resultRelInfo->ri_oldTupleSlot;
				...
					if (!table_tuple_fetch_row_version(relation, tupleid,
													   SnapshotAny,
													   oldSlot))
						elog(ERROR, "failed to fetch tuple being updated");
				}
				slot = ExecGetUpdateNewTuple(resultRelInfo, context.planSlot,
											 oldSlot);

				/* Now apply the update. */
				slot = ExecUpdate(&context, resultRelInfo, tupleid, oldtuple,
								  oldSlot, slot, node->canSetTag);
```

`SnapshotAny` kullanılması dikkat çekicidir: satırın hangi sürümünü göreceğimize zaten `ctid` karar verdi, görünürlük testine gerek yok.

### Yeni tuple nasıl kuruluyor?

[../src/backend/executor/nodeModifyTable.c:717-732](../src/backend/executor/nodeModifyTable.c#L717-L732)

```c
 * UPDATE always needs a projection, because (1) there's always some junk
 * attrs, and (2) we may need to merge values of not-updated columns from
 * the old tuple into the final tuple.  In UPDATE, the tuple arriving from
 * the subplan contains only new values for the changed columns, plus row
 * identity info in the junk attrs.
```

Projeksiyon iki girdi alır — plan çıktısı ve eski tuple:

[../src/backend/executor/nodeModifyTable.c:836-851](../src/backend/executor/nodeModifyTable.c#L836-L851)

```c
ExecGetUpdateNewTuple(ResultRelInfo *relinfo,
					  TupleTableSlot *planSlot,
					  TupleTableSlot *oldSlot)
{
	ProjectionInfo *newProj = relinfo->ri_projectNew;
	ExprContext *econtext;
	...
	econtext = newProj->pi_exprContext;
	econtext->ecxt_outertuple = planSlot;
	econtext->ecxt_scantuple = oldSlot;
	return ExecProject(newProj);
}
```

`ExecBuildUpdateProjection` bu ikisini `update_colnos` listesine göre birleştirir ([nodeModifyTable.c:773-780](../src/backend/executor/nodeModifyTable.c#L773-L780)). Yani **her UPDATE tam bir satır yazar** — sadece bir kolonu değiştirseniz bile.

---

## 3. `ExecUpdate` — üç aşama

[../src/backend/executor/nodeModifyTable.c:2759](../src/backend/executor/nodeModifyTable.c#L2759)

### 3.1 Prologue — BEFORE ROW trigger'ları

[../src/backend/executor/nodeModifyTable.c:2395-2411](../src/backend/executor/nodeModifyTable.c#L2395-L2411)

```c
	if (resultRelationDesc->rd_rel->relhasindex &&
		resultRelInfo->ri_IndexRelationDescs == NULL)
		ExecOpenIndices(resultRelInfo, false);

	/* BEFORE ROW UPDATE triggers */
	if (resultRelInfo->ri_TrigDesc &&
		resultRelInfo->ri_TrigDesc->trig_update_before_row)
	{
		/* Flush any pending inserts, so rows are visible to the triggers */
		if (context->estate->es_insert_pending_result_relations != NIL)
			ExecPendingInserts(context->estate);

		return ExecBRUpdateTriggers(...);
```

BEFORE trigger `NULL` döndürürse UPDATE o satır için iptal edilir.

### 3.2 Act — kısıt kontrolleri, sonra heap

[../src/backend/executor/nodeModifyTable.c:2477-2503](../src/backend/executor/nodeModifyTable.c#L2477-L2503) sırasıyla: `GENERATED` kolonları hesapla → partition kısıtını kontrol et → RLS `WITH CHECK` politikaları. Partition kısıtı düşerse satır başka partition'a taşınır (bkz. bölüm 8).

Sonra normal kısıtlar ve asıl çağrı:

[../src/backend/executor/nodeModifyTable.c:2579-2598](../src/backend/executor/nodeModifyTable.c#L2579-L2598)

```c
	if (resultRelationDesc->rd_att->constr)
		ExecConstraints(resultRelInfo, slot, estate);
	...
	result = table_tuple_update(resultRelationDesc, tupleid, slot,
								estate->es_output_cid,
								0,
								estate->es_snapshot,
								estate->es_crosscheck_snapshot,
								true /* wait for commit */ ,
								&context->tmfd, &updateCxt->lockmode,
								&updateCxt->updateIndexes);
```

`table_tuple_update` Table AM arayüzüdür; heap için [heapam_handler.c:218](../src/backend/access/heap/heapam_handler.c#L218) → `heap_update`.

### 3.3 Epilogue — index'ler ve AFTER trigger'ları

[../src/backend/executor/nodeModifyTable.c:2618-2642](../src/backend/executor/nodeModifyTable.c#L2618-L2642)

```c
	/* insert index entries for tuple if necessary */
	if (resultRelInfo->ri_NumIndices > 0 && (updateCxt->updateIndexes != TU_None))
	{
		uint32		flags = EIIT_IS_UPDATE;

		if (updateCxt->updateIndexes == TU_Summarizing)
			flags |= EIIT_ONLY_SUMMARIZING;
		recheckIndexes = ExecInsertIndexTuples(resultRelInfo, context->estate,
											   flags, slot, NIL,
											   NULL);
	}
	...
	/* AFTER ROW UPDATE Triggers */
	ExecARUpdateTriggers(...);
```

Dikkat: **eski index girdileri silinmez.** Index'te iki girdi kalır, biri ölü satıra bakar. Temizlik VACUUM'a bırakılır. Bu, UPDATE'in index'ler açısından "ekle-ve-unut" olmasının nedenidir.

Son olarak sayaç ve `RETURNING`:

[../src/backend/executor/nodeModifyTable.c:3003-3013](../src/backend/executor/nodeModifyTable.c#L3003-L3013)

```c
	if (canSetTag)
		(estate->es_processed)++;

	ExecUpdateEpilogue(context, &updateCxt, resultRelInfo, tupleid, oldtuple,
					   slot);

	/* Process RETURNING if present */
	if (resultRelInfo->ri_projectReturning)
		return ExecProcessReturning(context, resultRelInfo, false,
									oldSlot, slot, context->planSlot);
```

---

## 4. `heap_update` — işin kalbi

[../src/backend/access/heap/heapam.c:3267](../src/backend/access/heap/heapam.c#L3267)

Yaklaşık 1100 satırlık bu fonksiyon UPDATE'in tüm zor kararlarını verir. Sırayla.

### 4.1 Hangi kolonlar değişti?

Buffer kilidi **alınmadan önce** dört bitmap toplanır (relcache'ten, kilitli iken syscache'e gitmek deadlock riski taşır):

[../src/backend/access/heap/heapam.c:3356-3367](../src/backend/access/heap/heapam.c#L3356-L3367)

```c
	hot_attrs = RelationGetIndexAttrBitmap(relation,
										   INDEX_ATTR_BITMAP_HOT_BLOCKING);
	sum_attrs = RelationGetIndexAttrBitmap(relation,
										   INDEX_ATTR_BITMAP_SUMMARIZED);
	key_attrs = RelationGetIndexAttrBitmap(relation, INDEX_ATTR_BITMAP_KEY);
	id_attrs = RelationGetIndexAttrBitmap(relation,
										  INDEX_ATTR_BITMAP_IDENTITY_KEY);
```

Anlamları [../src/backend/utils/cache/relcache.c:5290-5297](../src/backend/utils/cache/relcache.c#L5290-L5297):

| Bitmap | İçerik | Neyi etkiler |
|---|---|---|
| `KEY` | Partial olmayan unique index kolonları | Satır kilidi modu (FK eşzamanlılığı) |
| `HOT_BLOCKING` | HOT'u engelleyen index kolonları (btree vb.) | HOT olup olmayacağı |
| `SUMMARIZED` | Özetleyici index kolonları (BRIN) | HOT'ta bile index güncellemesi |
| `IDENTITY_KEY` | Replica identity kolonları | Logical decoding'e ne yazılacağı |

Karşılaştırma [heapam.c:3452](../src/backend/access/heap/heapam.c#L3452) → `HeapDetermineColumnsInfo` ([tanım: heapam.c:4549](../src/backend/access/heap/heapam.c#L4549)). İlginç bir ayrıntı: değer **byte-eşitliği** ile değil `heap_attr_equals` ile karşılaştırılır ve `SET x = x` yazsanız bile değişmemiş sayılır.

### 4.2 Satır kilidi modu — key mi değil mi?

[../src/backend/access/heap/heapam.c:3456-3488](../src/backend/access/heap/heapam.c#L3456-L3488)

```c
	/*
	 * If we're not updating any "key" column, we can grab a weaker lock type.
	 * This allows for more concurrency when we are running simultaneously
	 * with foreign key checks.
	 */
	if (!bms_overlap(modified_attrs, key_attrs))
	{
		*lockmode = LockTupleNoKeyExclusive;
		mxact_status = MultiXactStatusNoKeyUpdate;
		key_intact = true;
		...
	}
	else
	{
		*lockmode = LockTupleExclusive;
		mxact_status = MultiXactStatusUpdate;
		key_intact = false;
	}
```

Bu, 9.3'te gelen "`SELECT FOR KEY SHARE`" iyileştirmesidir: FK kontrolü yapan bir transaction ile aynı satırın key olmayan kolonunu güncelleyen transaction **birbirini beklemez**. Detay için bkz. [KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md).

### 4.3 Eşzamanlılık — "bu satırı başkası mı yazıyor?"

[../src/backend/access/heap/heapam.c:3498-3513](../src/backend/access/heap/heapam.c#L3498-L3513)

```c
l2:
	checked_lockers = false;
	locker_remains = false;
	result = HeapTupleSatisfiesUpdate(&oldtup, cid, buffer);
	...
	else if (result == TM_BeingModified && wait)
```

`l2` etiketine geri dönüşler bu fonksiyonun karakteristiğidir: durum değiştiyse baştan kontrol.

Bekleme kararı üç duruma ayrılır:

1. **MultiXact** ([:3558](../src/backend/access/heap/heapam.c#L3558)) — birden çok kilitleyici var; `MultiXactIdWait` ile beklenir, kalan kilitleyiciler yeni tuple'a taşınır.
2. **Kilitleyici benim** ([:3631](../src/backend/access/heap/heapam.c#L3631)) — bekleme yok.
3. **Sadece key-share kilidi var ve key değişmiyor** ([:3641](../src/backend/access/heap/heapam.c#L3641)) — bekleme yok, kilit korunur.
4. Aksi halde ([:3652](../src/backend/access/heap/heapam.c#L3652)) — önce tuple kilidi alınır (sıraya girmek için), sonra `XactLockTableWait`.

Bekleme bitince sonuç belirlenir:

[../src/backend/access/heap/heapam.c:3682-3687](../src/backend/access/heap/heapam.c#L3682-L3687)

```c
		if (can_continue)
			result = TM_Ok;
		else if (!ItemPointerEquals(&oldtup.t_self, &oldtup.t_data->t_ctid))
			result = TM_Updated;
		else
			result = TM_Deleted;
```

`t_ctid` kendini göstermiyorsa satır güncellenmiş (`TM_Updated`), gösteriyorsa silinmiş (`TM_Deleted`). Bu ayrım executor'da EPQ'ya gidip gitmeyeceğimizi belirler.

### 4.4 Yeni tuple'ın başlığı

[../src/backend/access/heap/heapam.c:3802-3812](../src/backend/access/heap/heapam.c#L3802-L3812)

```c
	newtup->t_data->t_infomask &= ~(HEAP_XACT_MASK);
	newtup->t_data->t_infomask2 &= ~(HEAP2_XACT_MASK);
	HeapTupleHeaderSetXmin(newtup->t_data, xid);
	HeapTupleHeaderSetCmin(newtup->t_data, cid);
	newtup->t_data->t_infomask |= HEAP_UPDATED | infomask_new_tuple;
	newtup->t_data->t_infomask2 |= infomask2_new_tuple;
	HeapTupleHeaderSetXmax(newtup->t_data, xmax_new_tuple);
```

`HEAP_UPDATED` bayrağı "bu satır bir UPDATE sonucu doğdu" der ([htup_details.h:210](../src/include/access/htup_details.h#L210)). Eski satırdaki key-share kilitleyiciler varsa yeni tuple'ın `xmax`'ına taşınır — kilitler sürüm değişince kaybolmaz.

### 4.5 TOAST ve sayfa seçimi

[../src/backend/access/heap/heapam.c:3844-3848](../src/backend/access/heap/heapam.c#L3844-L3848)

```c
	pagefree = PageGetHeapFreeSpace(page);

	newtupsize = MAXALIGN(newtup->t_len);

	if (need_toast || newtupsize > pagefree)
```

Bu dala girilirse buffer kilidi bırakılmak zorundadır (TOAST yazmak / yeni sayfa açmak I/O gerektirir). Bırakmadan önce eski tuple **geçici olarak kilitli işaretlenir** ve bu geçici değişiklik bile WAL'a yazılır:

[../src/backend/access/heap/heapam.c:3855-3865](../src/backend/access/heap/heapam.c#L3855-L3865)

```c
		/*
		 * To prevent concurrent sessions from updating the tuple, we have to
		 * temporarily mark it locked, while we release the page-level lock.
		 *
		 * To satisfy the rule that any xid potentially appearing in a buffer
		 * written out to disk, we unfortunately have to WAL log this
		 * temporary modification.  We can reuse xl_heap_lock for this
		 * purpose.  If we crash/error before following through with the
		 * actual update, xmax will be of an aborted transaction, allowing
		 * other sessions to proceed.
		 */
```

Sonra yeni sayfa aranır. İki sayfayı birden kilitlemek deadlock riski taşıdığı için kural nettir:

[../src/backend/access/heap/heapam.c:3979-3986](../src/backend/access/heap/heapam.c#L3979-L3986)

```c
		 * What's more, if we need to get a new page, we will need to acquire
		 * buffer locks on both old and new pages.  To avoid deadlock against
		 * some other backend trying to get the same two locks in the other
		 * order, we must be consistent about the order we get the locks in.
		 * We use the rule "lock the lower-numbered page of the relation
		 * first".
```

### 4.6 HOT kararı

[../src/backend/access/heap/heapam.c:4063-4089](../src/backend/access/heap/heapam.c#L4063-L4089)

```c
	if (newbuf == buffer)
	{
		/*
		 * Since the new tuple is going into the same page, we might be able
		 * to do a HOT update.  Check if any of the index columns have been
		 * changed.
		 */
		if (!bms_overlap(modified_attrs, hot_attrs))
		{
			use_hot_update = true;
			...
			if (bms_overlap(modified_attrs, sum_attrs))
				summarized_update = true;
		}
	}
	else
	{
		/* Set a hint that the old page could use prune/defrag */
		PageSetFull(page);
	}
```

İki koşul, ikisi de zorunlu:

1. Yeni tuple **aynı sayfaya** sığdı (`newbuf == buffer`).
2. Değişen kolonların hiçbiri HOT-blocking index'lerde geçmiyor.

Sığmadıysa sayfaya "doldu" ipucu bırakılır — bir sonraki tarama budamayı tetikleyecektir.

### 4.7 Kritik bölüm — asıl yazma

[../src/backend/access/heap/heapam.c:4167-4214](../src/backend/access/heap/heapam.c#L4167-L4214)

```c
	START_CRIT_SECTION();
	...
	PageSetPrunable(page, xid);
	if (newbuf != buffer)
		PageSetPrunable(newpage, xid);

	if (use_hot_update)
	{
		/* Mark the old tuple as HOT-updated */
		HeapTupleSetHotUpdated(&oldtup);
		/* And mark the new tuple as heap-only */
		HeapTupleSetHeapOnly(heaptup);
		/* Mark the caller's copy too, in case different from heaptup */
		HeapTupleSetHeapOnly(newtup);
	}
	...
	RelationPutHeapTuple(relation, newbuf, heaptup, false); /* insert new tuple */

	/* Clear obsolete visibility flags, possibly set by ourselves above... */
	oldtup.t_data->t_infomask &= ~(HEAP_XMAX_BITS | HEAP_MOVED);
	oldtup.t_data->t_infomask2 &= ~HEAP_KEYS_UPDATED;
	/* ... and store info about transaction updating this tuple */
	HeapTupleHeaderSetXmax(oldtup.t_data, xmax_old_tuple);
	oldtup.t_data->t_infomask |= infomask_old_tuple;
	oldtup.t_data->t_infomask2 |= infomask2_old_tuple;
	HeapTupleHeaderSetCmax(oldtup.t_data, cid, iscombo);

	/* record address of new tuple in t_ctid of old one */
	oldtup.t_data->t_ctid = heaptup->t_self;
```

Tüm UPDATE üç satıra iner: **yeni tuple'ı yaz, eski tuple'ın xmax'ını doldur, eski tuple'ın t_ctid'sini yeniye çevir.** Bu üç işlem tek kritik bölümde, tek WAL kaydıyla atomiktir.

Ayrıca visibility map bitleri temizlenir ([:4221-4252](../src/backend/access/heap/heapam.c#L4221-L4252)) — sayfa artık "hepsi görünür" değildir.

### 4.8 WAL

[../src/backend/access/heap/heapam.c:4273-4281](../src/backend/access/heap/heapam.c#L4273-L4281) → `log_heap_update` ([:9015](../src/backend/access/heap/heapam.c#L9015)).

Kayıt tipi HOT'a göre değişir:

[../src/backend/access/heap/heapam.c:9041](../src/backend/access/heap/heapam.c#L9041)

```c
		info = XLOG_HEAP_HOT_UPDATE;
```

Eski ve yeni tuple aynı sayfadaysa WAL boyutu **prefix/suffix sıkıştırmasıyla** küçültülür:

[../src/backend/access/heap/heapam.c:9074-9094](../src/backend/access/heap/heapam.c#L9074-L9094)

```c
		for (prefixlen = 0; prefixlen < Min(oldlen, newlen); prefixlen++)
		{
			if (newp[prefixlen] != oldp[prefixlen])
				break;
		}
		...
		if (prefixlen < 3)
			prefixlen = 0;

		/* Same for suffix */
		for (suffixlen = 0; suffixlen < Min(oldlen, newlen) - prefixlen; suffixlen++)
		{
			if (newp[newlen - suffixlen - 1] != oldp[oldlen - suffixlen - 1])
				break;
		}
		if (suffixlen < 3)
			suffixlen = 0;
```

Yani geniş bir tablonun tek kolonunu güncellerseniz WAL'a sadece değişen orta parça yazılır. Bayraklar `XLH_UPDATE_PREFIX_FROM_OLD` / `XLH_UPDATE_SUFFIX_FROM_OLD` ([heapam_xlog.h:91-92](../src/include/access/heapam_xlog.h#L91-L92)).

Logical replication için ayrıca eski satırın replica identity kolonları yazılır ([heapam.c:4098-4101](../src/backend/access/heap/heapam.c#L4098-L4101), `XLH_UPDATE_CONTAINS_OLD_KEY`).

### 4.9 Dönüş — index'lere ne yapılacak?

[../src/backend/access/heap/heapam.c:4342-4356](../src/backend/access/heap/heapam.c#L4342-L4356)

```c
	if (use_hot_update)
	{
		if (summarized_update)
			*update_indexes = TU_Summarizing;
		else
			*update_indexes = TU_None;
	}
	else
		*update_indexes = TU_All;
```

Üç sonuç: hiç index'e dokunma / sadece özetleyici index'ler / hepsi.

---

## 5. HOT — neden bu kadar önemli?

[../src/backend/access/heap/README.HOT](../src/backend/access/heap/README.HOT) belgesindeki şema:

```
	Index points to 1
	lp [1]  [2]

	[111111111]->[2222222222]
```

Tuple 1 `HEAP_HOT_UPDATED`, tuple 2 `HEAP_ONLY_TUPLE`. Index sadece 1'e bakar.

Bayraklar [../src/include/access/htup_details.h:295-296](../src/include/access/htup_details.h#L295-L296):

```c
#define HEAP_HOT_UPDATED		0x4000	/* tuple was HOT-updated */
#define HEAP_ONLY_TUPLE			0x8000	/* this is heap-only tuple */
```

### 5.1 Okuyucu zinciri nasıl takip eder?

[../src/backend/access/heap/heapam_indexscan.c:90](../src/backend/access/heap/heapam_indexscan.c#L90) — `heap_hot_search_buffer`.

Redirect line pointer takibi:

[../src/backend/access/heap/heapam_indexscan.c:128-140](../src/backend/access/heap/heapam_indexscan.c#L128-L140)

```c
		/* check for unused, dead, or redirected items */
		if (!ItemIdIsNormal(lp))
		{
			/* We should only see a redirect at start of chain */
			if (ItemIdIsRedirected(lp) && at_chain_start)
			{
				/* Follow the redirect */
				offnum = ItemIdGetRedirect(lp);
				at_chain_start = false;
				continue;
			}
			/* else must be end of chain */
			break;
		}
```

Zincirin bütünlüğü `xmin`/`xmax` eşleşmesiyle doğrulanır:

[../src/backend/access/heap/heapam_indexscan.c:159-166](../src/backend/access/heap/heapam_indexscan.c#L159-L166)

```c
		/*
		 * The xmin should match the previous xmax value, else chain is
		 * broken.
		 */
		if (TransactionIdIsValid(prev_xmax) &&
			!TransactionIdEquals(prev_xmax,
								 HeapTupleHeaderGetXmin(heapTuple->t_data)))
			break;
```

Zincir tek sayfa içinde kaldığı için bu takip **ek disk erişimi getirmez**. HOT'un tek sayfa kısıtının nedeni budur.

### 5.2 Temizlik — sayfa içi budama

Tam VACUUM beklenmeden, normal tarama sırasında sayfa budanabilir:

[../src/backend/access/heap/pruneheap.c:272](../src/backend/access/heap/pruneheap.c#L272) — `heap_page_prune_opt`. Tetikleme eşiği:

[../src/backend/access/heap/pruneheap.c:307-323](../src/backend/access/heap/pruneheap.c#L307-L323)

```c
	/*
	 * We prune when a previous UPDATE failed to find enough space on the page
	 * for a new tuple version, or when free space falls below the relation's
	 * fill-factor target (but not less than 10%).
	 */
	minfree = RelationGetTargetPageFreeSpace(relation,
											 HEAP_DEFAULT_FILLFACTOR);
	minfree = Max(minfree, BLCKSZ / 10);

	if (PageIsFull(page) || PageGetHeapFreeSpace(page) < minfree)
```

Budama tam ölü HOT tuple'ları siler ama **zincir başındaki line pointer'ı silmez** — index hâlâ ona bakıyor. Onun yerine "redirect" işaretçisine çevrilir ([pruneheap.c:1506](../src/backend/access/heap/pruneheap.c#L1506) `heap_prune_chain`, [:1732](../src/backend/access/heap/pruneheap.c#L1732) `heap_prune_record_redirect`).

Bu yüzden **HOT UPDATE'ler VACUUM'a ihtiyaç duymadan alan geri kazanır.** Aynı sayfada `fillfactor` boşluğu bittiği anda HOT zinciri kopar ve tablo şişmeye başlar — `fillfactor` ayarının yoğun UPDATE alan tablolarda önemli olmasının sebebi.

---

## 6. Index bakımı

HOT değilse `ExecInsertIndexTuples` her index'e yeni bir girdi ekler. İki optimizasyon var.

**Sadece özetleyici index'ler:**

[../src/backend/executor/execIndexing.c:373-377](../src/backend/executor/execIndexing.c#L373-L377)

```c
		/*
		 * Skip processing of non-summarizing indexes if we only update
		 * summarizing indexes
		 */
		if ((flags & EIIT_ONLY_SUMMARIZING) && !indexInfo->ii_Summarizing)
			continue;
```

**`indexUnchanged` ipucu (bottom-up index deletion):**

[../src/backend/executor/execIndexing.c:441-456](../src/backend/executor/execIndexing.c#L441-L456)

```c
		indexUnchanged = ((flags & EIIT_IS_UPDATE) &&
						  index_unchanged_by_update(...));
		...
						 indexUnchanged,	/* UPDATE without logical change? */
```

Karar mantığı [../src/backend/executor/execIndexing.c:1018](../src/backend/executor/execIndexing.c#L1018) `index_unchanged_by_update`: index'in key kolonlarından hiçbiri güncellenmediyse `true`. Bu ipucu btree'ye "bu sayfadaki girdilerin çoğu muhtemelen ölü sürümler, temizlemeyi dene" der ve sayfa bölünmesini önler.

---

## 7. Eşzamanlı UPDATE — EvalPlanQual

İki transaction aynı satırı güncellerse ne olur? `heap_update` `TM_Updated` döner ve executor devreye girer:

[../src/backend/executor/nodeModifyTable.c:2894-2953](../src/backend/executor/nodeModifyTable.c#L2894-L2953)

```c
			case TM_Updated:
				{
					...
					if (IsolationUsesXactSnapshot())
						ereport(ERROR,
								(errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),
								 errmsg("could not serialize access due to concurrent update")));
					...
					result = table_tuple_lock(resultRelationDesc, tupleid,
											  estate->es_snapshot,
											  inputslot, estate->es_output_cid,
											  updateCxt.lockmode, LockWaitBlock,
											  TUPLE_LOCK_FLAG_FIND_LAST_VERSION,
											  &context->tmfd);
					...
							epqslot = EvalPlanQual(context->epqstate, ...);
							if (TupIsNull(epqslot))
								/* Tuple not passing quals anymore, exiting... */
								return NULL;
							...
							slot = ExecGetUpdateNewTuple(resultRelInfo,
														 epqslot, oldSlot);
							goto redo_act;
```

İzolasyon seviyesine göre iki farklı davranış:

| İzolasyon | Davranış |
|---|---|
| REPEATABLE READ / SERIALIZABLE | Hata: `could not serialize access due to concurrent update` |
| READ COMMITTED | EPQ: satırın **en son sürümü** alınır, `WHERE` yeniden değerlendirilir |

`EvalPlanQual` mantığı [../src/backend/executor/execMain.c:2715](../src/backend/executor/execMain.c#L2715):

```c
 * This tests whether the tuple in inputslot still matches the relevant
 * quals. For that result to be useful, typically the input tuple has to be
 * last row version (otherwise the result isn't particularly useful) and
 * locked (otherwise the result might be out of date).
```

Yani READ COMMITTED'da UPDATE **kendi snapshot'ının dışına çıkar** — güncel sürümü görür. `WHERE` hâlâ tutuyorsa `redo_act` etiketine dönülüp yeniden denenir; tutmuyorsa satır atlanır. Bu, PostgreSQL'in READ COMMITTED semantiğinin en şaşırtıcı köşesidir.

Diğer sonuç kodları:

| `TM_Result` | Anlamı | Tepki |
|---|---|---|
| `TM_Ok` | Başarılı | Devam |
| `TM_SelfModified` | Aynı komut zaten güncelledi | Sessizce atla veya hata ([:2857](../src/backend/executor/nodeModifyTable.c#L2857)) |
| `TM_Updated` | Başkası güncelledi | EPQ veya serialization hatası |
| `TM_Deleted` | Başkası sildi | Satırı atla ([:2989](../src/backend/executor/nodeModifyTable.c#L2989)) |
| `TM_BeingModified` | Devam eden yazma | `heap_update` içinde beklenir |

> `TM_SelfModified`, join'li UPDATE'lerde aynı hedef satıra birden çok kaynak satır eşleştiğinde görülür: **ilk eşleşme uygulanır, gerisi yok sayılır.**

---

## 8. Partition sınırını aşan UPDATE

Partition key güncellenirse satır fiziksel olarak yer değiştirmelidir. PostgreSQL bunu DELETE + INSERT olarak yapar:

[../src/backend/executor/nodeModifyTable.c:2515-2527](../src/backend/executor/nodeModifyTable.c#L2515-L2527)

```c
		/*
		 * ExecCrossPartitionUpdate will first DELETE the row from the
		 * partition it's currently in and then insert it back into the root
		 * table, which will re-route it to the correct partition.  However,
		 * if the tuple has been concurrently updated, a retry is needed.
		 */
		if (ExecCrossPartitionUpdate(context, resultRelInfo, ...))
```

Sonuçları:

- Eski partition'daki tuple'ın `t_ctid`'si özel bir "moved partitions" değeri alır ([htup_details.h:473-479](../src/include/access/htup_details.h#L473-L479)).
- Kaynak partition'da BEFORE DELETE, hedefte BEFORE INSERT trigger'ları çalışır — **UPDATE trigger'ları değil**.
- FK kontrolü için kök tablonun AFTER UPDATE trigger'ları ayrıca kuyruğa alınır ([nodeModifyTable.c:2666](../src/backend/executor/nodeModifyTable.c#L2666)).
- Kök olmayan bir ata doğrudan FK ile referanslanıyorsa hata verilir ([:2713-2720](../src/backend/executor/nodeModifyTable.c#L2713-L2720)).

---

## 9. Tek sayfalık özet

```
UPDATE users SET name='ali' WHERE id=42;

 PLANLAYICI
 ├─ preprocess_targetlist        update_colnos = {2}
 └─ add_row_identity_columns     tlist += junk "ctid"
                                    │
 EXECUTOR (nodeModifyTable.c)       ▼
 ├─ ExecModifyTable ── alt plan → (name='ali', ctid=(5,3))
 │   ├─ table_tuple_fetch_row_version(ctid, SnapshotAny) → ESKİ tuple
 │   └─ ExecGetUpdateNewTuple(plan ⨝ eski) ─────────────► YENİ tuple
 ├─ ExecUpdatePrologue           BEFORE ROW trigger
 ├─ ExecUpdateAct
 │   ├─ GENERATED / partition check / RLS / ExecConstraints
 │   └─ table_tuple_update ──────────────────┐
 └─ ExecUpdateEpilogue                       │
     ├─ ExecInsertIndexTuples (TU_All ise)   │
     └─ AFTER ROW trigger                    │
                                             ▼
 HEAP (heapam.c: heap_update)
 ├─ 1. hangi kolonlar değişti?      HeapDetermineColumnsInfo
 │      ├─ key_attrs   ile kesişim → LockTupleExclusive mi NoKeyExclusive mi
 │      ├─ hot_attrs   ile kesişim → HOT olabilir mi
 │      └─ sum_attrs   ile kesişim → BRIN güncellenecek mi
 ├─ 2. HeapTupleSatisfiesUpdate     eşzamanlı yazan var mı? (l2 döngüsü)
 │      └─ varsa bekle → TM_Ok / TM_Updated / TM_Deleted
 ├─ 3. TOAST gerekiyor mu / sayfaya sığıyor mu?
 │      └─ sığmıyorsa: eski tuple'ı geçici kilitle, WAL'a yaz,
 │         buffer kilidini bırak, yeni sayfa bul (düşük blok önce)
 ├─ 4. HOT kararı: aynı sayfa && index kolonu değişmedi
 ├─ 5. KRİTİK BÖLÜM
 │      ├─ RelationPutHeapTuple(yeni)
 │      ├─ eski.xmax = xid,  eski.cmax = cid
 │      ├─ eski.t_ctid = yeni.t_self          ← zincir kuruldu
 │      ├─ HOT ise: eski|=HOT_UPDATED, yeni|=HEAP_ONLY_TUPLE
 │      ├─ PageSetPrunable, visibility map temizle
 │      └─ log_heap_update → XLOG_HEAP_(HOT_)UPDATE + prefix/suffix sıkıştırma
 └─ 6. *update_indexes = TU_None | TU_Summarizing | TU_All

 SONRASI
 ├─ Okuyucular: index → line pointer → heap_hot_search_buffer → zinciri izle
 ├─ Budama:     heap_page_prune_opt → ölü tuple'lar gider, redirect lp kalır
 └─ VACUUM:     index girdilerini ve zincir başlarını temizler
```

---

## 10. İzleme ve hata ayıklama

**HOT oranı** — bu tablonun UPDATE maliyetini belirleyen tek sayı:

```sql
SELECT relname,
       n_tup_upd,
       n_tup_hot_upd,
       n_tup_newpage_upd,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC;
```

Sayaçların kaynağı [../src/backend/access/heap/heapam.c:4330](../src/backend/access/heap/heapam.c#L4330):

```c
	pgstat_count_heap_update(relation, use_hot_update, newbuf != buffer);
```

`n_tup_newpage_upd` yüksekse yeni sürümler başka sayfaya taşınıyor demektir → `fillfactor` düşürmeyi düşünün:

```sql
ALTER TABLE users SET (fillfactor = 70);
VACUUM FULL users;   -- yeni fillfactor mevcut sayfalara ancak yeniden yazınca uygulanır
```

**Bir satırın sürüm zincirini görmek:**

```sql
SELECT ctid, xmin, xmax, id, name FROM users WHERE id = 42;
```

**Sayfa düzeyinde ne olduğunu görmek** (`pageinspect` eklentisi):

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_off, lp_flags, t_xmin, t_xmax, t_ctid,
       (t_infomask2 & 16384) <> 0 AS hot_updated,   -- HEAP_HOT_UPDATED
       (t_infomask2 & 32768) <> 0 AS heap_only      -- HEAP_ONLY_TUPLE
FROM heap_page_items(get_raw_page('users', 0));
```

`lp_flags = 2` redirect line pointer, `3` ölü işaretçi demektir.

**Şişmeyi görmek:**

```sql
SELECT relname, n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

**Kilit beklemesini görmek** — bir UPDATE takıldıysa:

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type IN ('Lock', 'LWLock');
```

`wait_event = 'transactionid'` ise başka bir transaction'ın aynı satırı yazması bekleniyordur ([heapam.c:3661](../src/backend/access/heap/heapam.c#L3661) `XactLockTableWait`). Kilit tarafı için bkz. [KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md).

---

## Pratik sonuçlar

1. **UPDATE bir INSERT'tür.** Tablo her UPDATE'te büyüme eğilimindedir; küçülme VACUUM'a bağlıdır.
2. **Bir kolonu güncellemek tüm satırı yeniden yazar.** Geniş satırlarda maliyet kolon sayısıyla değil satır boyutuyla ölçeklenir (WAL tarafında prefix/suffix sıkıştırması bunu kısmen telafi eder).
3. **Index sayısı UPDATE maliyetini doğrudan belirler** — ama sadece HOT kaçırıldığında. Nadiren güncellenen kolonlarda index tutmak ucuzdur.
4. **`fillfactor`** yoğun UPDATE alan tablolarda HOT oranını doğrudan etkiler.
5. **READ COMMITTED'da UPDATE snapshot'ının dışına çıkar** (EPQ). Kritik iş mantığında `SELECT ... FOR UPDATE` ya da REPEATABLE READ tercih edin.
6. **Partition key'i güncellemek DELETE+INSERT'tür** — UPDATE trigger'larınız çalışmaz.
