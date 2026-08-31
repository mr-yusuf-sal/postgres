# Bir INSERT ve bir DELETE komutu PostgreSQL içinde nasıl çalışır?

Örnek sorgular — dosya boyunca bu ikisini takip ediyoruz:

```sql
INSERT INTO users (id, name) VALUES (42, 'ali');
DELETE FROM users WHERE id = 42;
```

Kaynak ağacı: PostgreSQL **20devel** ([meson.build](../meson.build) `version: '20devel'`). Satır numaraları sürüme bağlıdır.

Kardeş not: [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md). UPDATE = DELETE + INSERT olduğu için burada anlatılan her şey oraya da uygulanır; tekrar etmemek için ortak konularda o dosyaya link verilmiştir.

## 30 saniyelik özet

INSERT sayfaya **yeni bir tuple ekler**; DELETE **hiçbir şey silmez**, sadece eski tuple'ın `xmax` alanına kendi transaction id'sini yazar.

```
                       INSERT                        DELETE
                  ┌──────────────┐             ┌──────────────┐
  öncesi:         │              │             │ xmin=500     │
                  │    (yok)     │             │ xmax=0       │
                  │              │             │ t_ctid=kendi │
                  └──────────────┘             └──────────────┘

                  ┌──────────────┐             ┌──────────────┐
  sonrası:        │ xmin=500     │             │ xmin=500     │
                  │ xmax=0       │             │ xmax=700     │  ← tek değişiklik
                  │ t_ctid=kendi │             │ t_ctid=kendi │  ← DEĞİŞMEZ
                  │ XMAX_INVALID │             │ HeapTupleHeaderClearHotUpdated
                  └──────────────┘             └──────────────┘
                        ↑                            ↑
                  yeni satır doğdu            satır hâlâ diskte, ölü işaretli
```

Boru hattı:

```
INSERT                                    DELETE
[1] ExecModifyTable  CMD_INSERT           [1] ExecModifyTable  CMD_DELETE
[2] ExecGetInsertNewTuple                 [2] junk "ctid" oku
[3] ExecInsert                            [3] ExecDelete
     ├─ partition routing                      ├─ ExecDeletePrologue  (BEFORE ROW)
     ├─ BEFORE ROW trigger                     ├─ ExecDeleteAct → heap_delete()
     ├─ GENERATED / RLS / ExecConstraints      │     ├─ eşzamanlı yazan var mı?
     ├─ ON CONFLICT ise → speculative          │     ├─ xmax = benim xid
     ├─ table_tuple_insert → heap_insert()     │     └─ WAL (XLOG_HEAP_DELETE)
     │     ├─ heap_prepare_insert (TOAST!)     ├─ index'e DOKUNULMAZ
     │     ├─ RelationGetBufferForTuple (FSM)  └─ ExecDeleteEpilogue   (AFTER ROW)
     │     ├─ RelationPutHeapTuple             [4] RETURNING (eski satır)
     │     └─ WAL (XLOG_HEAP_INSERT)
     ├─ ExecInsertIndexTuples  ← unique kontrolü burada
     └─ AFTER ROW trigger
[4] RETURNING (yeni satır)
[5] AfterTriggerEndQuery  ← ertelenmiş kuyruk
```

Altı madde yeter:

- **INSERT'te asıl zorluk "nereye yazacağım?"** sorusudur. Cevabı Free Space Map verir, ama FSM **yaklaşık** bilgi tutar; bulunan sayfa dolu çıkabilir, döngüye girilir.
- **DELETE'te asıl zorluk hiç yoktur** — tek bir alan yazılır. Zorluk eşzamanlılıktadır: aynı satırı başkası yazıyorsa beklemek gerekir.
- **Index girdileri INSERT'te eklenir, DELETE'te silinmez.** Silme işi VACUUM'undur. Bu yüzden DELETE, INSERT'ten çok daha ucuzdur.
- **Unique kısıt heap'te değil index AM'inde uygulanır.** `ExecInsertIndexTuples` sadece "kontrol et" bayrağı geçer; atomik kontrolü btree yapar.
- **`ON CONFLICT` speculative insertion demektir**: tuple önce `t_ctid` alanına bir *token* konarak yazılır, index'lere girilir, sonra ya onaylanır ya geri alınır (`heap_abort_speculative`).
- **`t_ctid` DELETE'te kendini gösterir.** UPDATE'te ileri zincir kurar; farkın kaynağı budur ve `TM_Updated`/`TM_Deleted` ayrımı tam olarak buna bakar.

---

## Kaynak haritası

| Dosya | Ne yapar |
|---|---|
| [../src/backend/executor/nodeModifyTable.c](../src/backend/executor/nodeModifyTable.c) | `ExecModifyTable`, `ExecInsert`, `ExecDelete` — executor tarafı |
| [../src/backend/executor/execIndexing.c](../src/backend/executor/execIndexing.c) | `ExecInsertIndexTuples`, `ExecCheckIndexConstraints`, speculative insertion tasarımı |
| [../src/backend/executor/execMain.c](../src/backend/executor/execMain.c) | `ExecConstraints` — NOT NULL ve CHECK kısıtları |
| [../src/backend/commands/trigger.c](../src/backend/commands/trigger.c) | Trigger makinesi, ertelenmiş AFTER kuyruğu |
| [../src/backend/access/heap/heapam.c](../src/backend/access/heap/heapam.c) | **`heap_insert`, `heap_delete`, `heap_multi_insert`, `heap_abort_speculative`** |
| [../src/backend/access/heap/hio.c](../src/backend/access/heap/hio.c) | `RelationGetBufferForTuple` — hangi sayfaya yazılacağı |
| [../src/backend/storage/freespace/freespace.c](../src/backend/storage/freespace/freespace.c) | Free Space Map arayüzü |
| [../src/backend/storage/freespace/README](../src/backend/storage/freespace/README) | FSM ağaç tasarımı (okunması şart) |
| [../src/backend/access/nbtree/nbtinsert.c](../src/backend/access/nbtree/nbtinsert.c) | `_bt_check_unique` — unique kısıtın gerçekten uygulandığı yer |
| [../src/backend/commands/copyfrom.c](../src/backend/commands/copyfrom.c) | `COPY` ve multi-insert tampon yönetimi |
| [../src/include/access/htup_details.h](../src/include/access/htup_details.h) | Tuple başlığı, `HEAP_XMAX_*` infomask bitleri |

---

## 1. Executor döngüsü — iki dal

Her iki komut da aynı düğümden geçer: `ExecModifyTable` ([../src/backend/executor/nodeModifyTable.c:4635](../src/backend/executor/nodeModifyTable.c#L4635)).

### 1.1 `CMD_INSERT` dalı

[../src/backend/executor/nodeModifyTable.c:4975-4982](../src/backend/executor/nodeModifyTable.c#L4975-L4982)

```c
			case CMD_INSERT:
				/* Initialize projection info if first time for this table */
				if (unlikely(!resultRelInfo->ri_projectNewInfoValid))
					ExecInitInsertProjection(node, resultRelInfo);
				slot = ExecGetInsertNewTuple(resultRelInfo, context.planSlot);
				slot = ExecInsert(&context, resultRelInfo, slot,
								  node->canSetTag, NULL, NULL);
				break;
```

UPDATE'ten en büyük farkı burada görülür: **hiçbir junk kolon okunmaz, diskten hiçbir şey getirilmez.** Alt planın ürettiği satır zaten tam satırdır.

`ExecInitInsertProjection` ([:664](../src/backend/executor/nodeModifyTable.c#L664)) çoğu zaman projeksiyon bile kurmaz:

[../src/backend/executor/nodeModifyTable.c:674-697](../src/backend/executor/nodeModifyTable.c#L674-L697)

```c
	/* Extract non-junk columns of the subplan's result tlist. */
	foreach(l, subplan->targetlist)
	{
		TargetEntry *tle = (TargetEntry *) lfirst(l);

		if (!tle->resjunk)
			insertTargetList = lappend(insertTargetList, tle);
		else
			need_projection = true;
	}
	...
	/* Build ProjectionInfo if needed (it probably isn't). */
	if (need_projection)
```

Yorumdaki "it probably isn't" ciddidir: junk kolon yoksa INSERT'te projeksiyon maliyeti sıfırdır. UPDATE'te ise **her zaman** projeksiyon vardır (bkz. [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm 2).

### 1.2 `CMD_DELETE` dalı

[../src/backend/executor/nodeModifyTable.c:5028-5031](../src/backend/executor/nodeModifyTable.c#L5028-L5031)

```c
			case CMD_DELETE:
				slot = ExecDelete(&context, resultRelInfo, tupleid, oldtuple,
								  true, false, node->canSetTag, NULL, NULL, NULL);
				break;
```

`tupleid` yukarıda, junk `ctid` kolonundan okunmuştur — planlayıcı bunu UPDATE'teki ile birebir aynı yoldan ekler (`add_row_identity_columns`, [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm 1). DELETE'in UPDATE'ten farkı: **eski tuple'ı diskten getirmez.** `ctid` yeterlidir; satırın içeriği ancak `RETURNING` ya da trigger varsa okunur.

### 1.3 Döngü bittiğinde

[../src/backend/executor/nodeModifyTable.c:5050-5062](../src/backend/executor/nodeModifyTable.c#L5050-L5062)

```c
	/*
	 * Insert remaining tuples for batch insert.
	 */
	if (estate->es_insert_pending_result_relations != NIL)
		ExecPendingInserts(estate);

	/*
	 * We're done, but fire AFTER STATEMENT triggers before exiting.
	 */
	fireASTriggers(node);

	node->mt_done = true;
```

---

## 2. `ExecInsert` — dokuz aşama

[../src/backend/executor/nodeModifyTable.c:874](../src/backend/executor/nodeModifyTable.c#L874)

Fonksiyon uzundur ama sırası nettir.

### 2.1 Partition routing ve index açma

[../src/backend/executor/nodeModifyTable.c:897-917](../src/backend/executor/nodeModifyTable.c#L897-L917)

```c
	if (proute)
	{
		ResultRelInfo *partRelInfo;

		slot = ExecPrepareTupleRouting(mtstate, estate, proute,
									   resultRelInfo, slot,
									   &partRelInfo);
		resultRelInfo = partRelInfo;
	}

	ExecMaterializeSlot(slot);
	...
	if (resultRelationDesc->rd_rel->relhasindex &&
		resultRelInfo->ri_IndexRelationDescs == NULL)
		ExecOpenIndices(resultRelInfo, onconflict != ONCONFLICT_NONE);
```

`ExecOpenIndices`'in ikinci argümanı `speculative` bayrağıdır: `ON CONFLICT` varsa index'ler ek bilgiyle açılır (arbiter index seçimi için).

### 2.2 BEFORE ROW trigger

[../src/backend/executor/nodeModifyTable.c:920-937](../src/backend/executor/nodeModifyTable.c#L920-L937)

```c
	/*
	 * BEFORE ROW INSERT Triggers.
	 *
	 * Note: We fire BEFORE ROW TRIGGERS for every attempted insertion in an
	 * INSERT ... ON CONFLICT statement.  We cannot check for constraint
	 * violations before firing these triggers, because they can change the
	 * values to insert.  Also, they can run arbitrary user-defined code with
	 * side-effects that we can't cancel by just not inserting the tuple.
	 */
	if (resultRelInfo->ri_TrigDesc &&
		resultRelInfo->ri_TrigDesc->trig_insert_before_row)
	{
		/* Flush any pending inserts, so rows are visible to the triggers */
		if (estate->es_insert_pending_result_relations != NIL)
			ExecPendingInserts(estate);

		if (!ExecBRInsertTriggers(estate, resultRelInfo, slot))
			return NULL;		/* "do nothing" */
	}
```

İki ayrıntı önemli:

- Trigger `NULL` döndürürse satır **sessizce** atlanır; hata yoktur, `es_processed` artmaz.
- `ON CONFLICT DO NOTHING` yazsanız bile BEFORE trigger **her denemede** çalışır. Trigger içinde sayaç artırıyorsanız bu sizi yanıltır.

### 2.3 GENERATED, RLS, kısıtlar

[../src/backend/executor/nodeModifyTable.c:1084-1129](../src/backend/executor/nodeModifyTable.c#L1084-L1129) sırasıyla:

1. `ExecComputeStoredGenerated(..., CMD_INSERT)` — `GENERATED ... STORED` kolonları hesaplanır.
2. `ExecWithCheckOptions(WCO_RLS_INSERT_CHECK, ...)` — RLS `WITH CHECK` politikaları.
3. `ExecConstraints(resultRelInfo, slot, estate)` — NOT NULL + CHECK.
4. `ExecPartitionCheck(...)` — partition kısıtı (routing ile gelindiyse çoğu zaman atlanır).

`ExecConstraints` ([../src/backend/executor/execMain.c:2046](../src/backend/executor/execMain.c#L2046)) iki iş yapar:

[../src/backend/executor/execMain.c:2063-2074](../src/backend/executor/execMain.c#L2063-L2074)

```c
	if (constr->has_not_null)
	{
		for (AttrNumber attnum = 1; attnum <= tupdesc->natts; attnum++)
		{
			Form_pg_attribute att = TupleDescAttr(tupdesc, attnum - 1);

			if (att->attnotnull && att->attgenerated == ATTRIBUTE_GENERATED_VIRTUAL)
				notnull_virtual_attrs = lappend_int(notnull_virtual_attrs, attnum);
			else if (att->attnotnull && slot_attisnull(slot, attnum))
				ReportNotNullViolationError(resultRelInfo, slot, estate, attnum);
		}
	}
```

sonra `relchecks > 0` ise `ExecRelCheck` ([:1844](../src/backend/executor/execMain.c#L1844)).

> Dikkat: `ExecConstraints` **partition kısıtını kontrol etmez** ve **unique kısıtı bilmez.** Unique tamamen index AM'inin işidir (bölüm 4). Yorum bunu açıkça söyler: "The partition constraint is *NOT* checked." ([:2039](../src/backend/executor/execMain.c#L2039))

### 2.4 Normal INSERT yolu

`ON CONFLICT` yoksa iki satırlık iştir:

[../src/backend/executor/nodeModifyTable.c:1273-1286](../src/backend/executor/nodeModifyTable.c#L1273-L1286)

```c
		else
		{
			/* insert the tuple normally */
			table_tuple_insert(resultRelationDesc, slot,
							   estate->es_output_cid,
							   0, NULL);

			/* insert index entries for tuple */
			if (resultRelInfo->ri_NumIndices > 0)
				recheckIndexes = ExecInsertIndexTuples(resultRelInfo, estate,
													   0, slot, NIL,
													   NULL);
		}
```

`table_tuple_insert` Table AM arayüzüdür; heap için `heap_insert`.

### 2.5 AFTER ROW trigger ve RETURNING

[../src/backend/executor/nodeModifyTable.c:1288-1321](../src/backend/executor/nodeModifyTable.c#L1288-L1321)

```c
	if (canSetTag)
		(estate->es_processed)++;
	...
	/* AFTER ROW INSERT Triggers */
	ExecARInsertTriggers(estate, resultRelInfo, slot, recheckIndexes,
						 ar_insert_trig_tcs);
```

`recheckIndexes` listesi buraya taşınır: ertelenmiş (`DEFERRABLE`) unique/exclusion kısıtları olan index'lerin OID'leri. Bunlar sonradan yeniden kontrol edilecek olay olarak kuyruğa girer.

`RETURNING` ise [../src/backend/executor/nodeModifyTable.c:1383-1384](../src/backend/executor/nodeModifyTable.c#L1383-L1384):

```c
		result = ExecProcessReturning(context, resultRelInfo, false,
									  oldSlot, slot, planSlot);
```

`ExecProcessReturning` ([:318](../src/backend/executor/nodeModifyTable.c#L318)) OLD/NEW ayrımını bayraklarla yapar:

[../src/backend/executor/nodeModifyTable.c:330-343](../src/backend/executor/nodeModifyTable.c#L330-L343)

```c
	if (isDelete)
	{
		/* return old tuple by default */
		if (oldSlot)
			econtext->ecxt_scantuple = oldSlot;
	}
	else
	{
		/* return new tuple by default */
		if (newSlot)
			econtext->ecxt_scantuple = newSlot;
	}
	econtext->ecxt_outertuple = planSlot;
```

Yani `RETURNING *` INSERT'te yeni satırı, DELETE'te eski satırı verir; `RETURNING OLD.*` / `NEW.*` yazıldığında ise `ecxt_oldtuple` / `ecxt_newtuple` ayrı ayrı doldurulur ve olmayan taraf için `EEO_FLAG_OLD_IS_NULL` / `EEO_FLAG_NEW_IS_NULL` bayrağı konur ([:363-374](../src/backend/executor/nodeModifyTable.c#L363-L374)).

---

## 3. `heap_insert` — sayfaya yazma

[../src/backend/access/heap/heapam.c:2005](../src/backend/access/heap/heapam.c#L2005)

### 3.1 Başlık ve TOAST — `heap_prepare_insert`

[../src/backend/access/heap/heapam.c:2028](../src/backend/access/heap/heapam.c#L2028)

```c
	heaptup = heap_prepare_insert(relation, tup, xid, cid, options);
```

Fonksiyonun kendisi ([:2228](../src/backend/access/heap/heapam.c#L2228)) tuple başlığını doldurur:

[../src/backend/access/heap/heapam.c:2242-2251](../src/backend/access/heap/heapam.c#L2242-L2251)

```c
	tup->t_data->t_infomask &= ~(HEAP_XACT_MASK);
	tup->t_data->t_infomask2 &= ~(HEAP2_XACT_MASK);
	tup->t_data->t_infomask |= HEAP_XMAX_INVALID;
	HeapTupleHeaderSetXmin(tup->t_data, xid);
	if (options & HEAP_INSERT_FROZEN)
		HeapTupleHeaderSetXminFrozen(tup->t_data);

	HeapTupleHeaderSetCmin(tup->t_data, cid);
	HeapTupleHeaderSetXmax(tup->t_data, 0); /* for cleanliness */
	tup->t_tableOid = RelationGetRelid(relation);
```

Yeni tuple'ın `xmax`'ı sıfırdır ve `HEAP_XMAX_INVALID` bayrağı konur. UPDATE'in ürettiği yeni tuple'dan tek farkı `HEAP_UPDATED` bayrağının olmamasıdır.

Hemen sonra TOAST kararı:

[../src/backend/access/heap/heapam.c:2253-2267](../src/backend/access/heap/heapam.c#L2253-L2267)

```c
	if (relation->rd_rel->relkind != RELKIND_RELATION &&
		relation->rd_rel->relkind != RELKIND_MATVIEW)
	{
		/* toast table entries should never be recursively toasted */
		Assert(!HeapTupleHasExternal(tup));
		return tup;
	}
	else if (HeapTupleHasExternal(tup) || tup->t_len > TOAST_TUPLE_THRESHOLD)
		return heap_toast_insert_or_update(relation, tup, NULL, options);
	else
		return tup;
```

Bu tek `if` TOAST'ın giriş kapısıdır — ayrıntısı [TOAST-NASIL-CALISIR.md](TOAST-NASIL-CALISIR.md).

### 3.2 `HEAP_INSERT_*` bayrakları

[../src/include/access/heapam.h:36-39](../src/include/access/heapam.h#L36-L39)

```c
#define HEAP_INSERT_SKIP_FSM	TABLE_INSERT_SKIP_FSM
#define HEAP_INSERT_FROZEN		TABLE_INSERT_FROZEN
#define HEAP_INSERT_NO_LOGICAL	TABLE_INSERT_NO_LOGICAL
#define HEAP_INSERT_SPECULATIVE 0x0010
```

| Bayrak | Anlamı | Kim kullanır |
|---|---|---|
| `SKIP_FSM` | FSM'e hiç bakma, doğrudan sona ekle | Bu transaction'da yaratılmış tablolar (`COPY`, `CREATE TABLE AS`) |
| `FROZEN` | `xmin`'i donmuş yaz — satır herkese görünür | `COPY FROM` + `FREEZE` |
| `NO_LOGICAL` | Logical decoding'e yazma | Geçici/görünmez tablolar |
| `SPECULATIVE` | `t_ctid` alanında token taşı | `INSERT ... ON CONFLICT` |

`SKIP_FSM`'in nerede seçildiği [../src/backend/commands/copyfrom.c:1000-1008](../src/backend/commands/copyfrom.c#L1000-L1008):

```c
	/*
	 * If the target file is new-in-transaction, we assume that checking FSM
	 * for free space is a waste of time.  This could possibly be wrong, but
	 * it's unlikely.
	 */
	if (RELKIND_HAS_STORAGE(cstate->rel->rd_rel->relkind) &&
		(cstate->rel->rd_createSubid != InvalidSubTransactionId ||
		 cstate->rel->rd_firstRelfilelocatorSubid != InvalidSubTransactionId))
		ti_options |= TABLE_INSERT_SKIP_FSM;
```

### 3.3 Sayfa seçimi ve yazma

[../src/backend/access/heap/heapam.c:2030-2039](../src/backend/access/heap/heapam.c#L2030-L2039)

```c
	/*
	 * Find buffer to insert this tuple into.  If the page is all visible,
	 * this will also pin the requisite visibility map page.
	 */
	buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
									   InvalidBuffer, options, bistate,
									   &vmbuffer, NULL,
									   0);
```

`InvalidBuffer` ikinci argüman `otherBuffer`'dır. INSERT'te tek sayfa kilitlenir; UPDATE'te eski sayfanın buffer'ı geçilir ve iki sayfa **düşük blok numarası önce** kuralıyla kilitlenir (bkz. [UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm 4.5).

Sonra kritik bölüm:

[../src/backend/access/heap/heapam.c:2065-2069](../src/backend/access/heap/heapam.c#L2065-L2069)

```c
	/* NO EREPORT(ERROR) from here till changes are logged */
	START_CRIT_SECTION();

	RelationPutHeapTuple(relation, buffer, heaptup,
						 (options & HEAP_INSERT_SPECULATIVE) != 0);
```

`RelationPutHeapTuple` ([../src/backend/access/heap/hio.c:35](../src/backend/access/heap/hio.c#L35)) line pointer'ı ekler ve `t_ctid`'yi kendine çevirir:

[../src/backend/access/heap/hio.c:58-78](../src/backend/access/heap/hio.c#L58-L78)

```c
	offnum = PageAddItem(pageHeader, tuple->t_data, tuple->t_len, InvalidOffsetNumber, false, true);
	if (offnum == InvalidOffsetNumber)
		elog(PANIC, "failed to add tuple to page");

	/* Update tuple->t_self to the actual position where it was stored */
	ItemPointerSet(&(tuple->t_self), BufferGetBlockNumber(buffer), offnum);

	/*
	 * Insert the correct position into CTID of the stored tuple, too (unless
	 * this is a speculative insertion, in which case the token is held in
	 * CTID field instead)
	 */
	if (!token)
	{
		ItemId		itemId = PageGetItemId(pageHeader, offnum);
		HeapTupleHeader item = (HeapTupleHeader) PageGetItem(pageHeader, itemId);

		item->t_ctid = tuple->t_self;
	}
```

**`t_ctid` alanının üç ayrı anlamı olduğuna dikkat:** normal tuple'da kendini gösterir, UPDATE'lenmiş tuple'da yeni sürümü gösterir, speculative tuple'da bir token taşır. Üçü aynı 6 bayta sığar.

### 3.4 Budama ipucu

[../src/backend/access/heap/heapam.c:2083-2096](../src/backend/access/heap/heapam.c#L2083-L2096)

```c
	/*
	 * Set pd_prune_xid to trigger heap_page_prune_and_freeze() once the page
	 * is full so that we can set the page all-visible in the VM on the next
	 * page access.
	 *
	 * Setting pd_prune_xid is also handy if the inserting transaction
	 * eventually aborts making this tuple DEAD and hence available for
	 * pruning. If no other tuple in this page is UPDATEd/DELETEd, the aborted
	 * tuple would never otherwise be pruned until next vacuum is triggered.
	 *
	 * Don't set it if we are in bootstrap mode or we are inserting a frozen
	 * tuple, as there is no further pruning/freezing needed in those cases.
	 */
	if (TransactionIdIsNormal(xid) && !(options & HEAP_INSERT_FROZEN))
		PageSetPrunable(page, xid);
```

İlginç: **INSERT bile sayfayı budanabilir işaretler.** Nedeni ROLLBACK'tir — iptal edilen INSERT ölü tuple bırakır.

### 3.5 WAL kaydı

[../src/backend/access/heap/heapam.c:2100-2133](../src/backend/access/heap/heapam.c#L2100-L2133)

```c
	/* XLOG stuff */
	if (RelationNeedsWAL(relation))
	{
		xl_heap_insert xlrec;
		xl_heap_header xlhdr;
		XLogRecPtr	recptr;
		uint8		info = XLOG_HEAP_INSERT;
		int			bufflags = 0;
		...
		/*
		 * If this is the single and first tuple on page, we can reinit the
		 * page instead of restoring the whole thing.  Set flag, and hide
		 * buffer references from XLogInsert.
		 */
		if (ItemPointerGetOffsetNumber(&(heaptup->t_self)) == FirstOffsetNumber &&
			PageGetMaxOffsetNumber(page) == FirstOffsetNumber)
		{
			info |= XLOG_HEAP_INIT_PAGE;
			bufflags |= REGBUF_WILL_INIT;
		}

		xlrec.offnum = ItemPointerGetOffsetNumber(&heaptup->t_self);
		xlrec.flags = 0;
		if (clear_all_visible)
			xlrec.flags |= XLH_INSERT_ALL_VISIBLE_CLEARED;
		if (options & HEAP_INSERT_SPECULATIVE)
			xlrec.flags |= XLH_INSERT_IS_SPECULATIVE;
```

Kayıt küçüktür: offset + bayraklar + tuple verisi. UPDATE'in `log_heap_update` fonksiyonundaki prefix/suffix sıkıştırması burada yoktur — çünkü karşılaştırılacak eski sürüm yoktur.

`XLOG_HEAP_INIT_PAGE` optimizasyonu, boş sayfaya ilk tuple yazılırken tam sayfa imajı (FPI) yazmayı önler. Toplu yüklemede WAL hacmini ciddi düşürür.

Son olarak sayaç:

[../src/backend/access/heap/heapam.c:2208](../src/backend/access/heap/heapam.c#L2208)

```c
	pgstat_count_heap_insert(relation, 1);
```

Yorum bir satır yukarıda uyarır: "Note: speculative insertions are counted too, even if aborted later" ([:2207](../src/backend/access/heap/heapam.c#L2207)). `pg_stat_user_tables.n_tup_ins` bu yüzden `ON CONFLICT` kullanan iş yükünde gerçek satır sayısından fazladır.

---

## 4. Sayfa seçimi ve Free Space Map

### 4.1 `RelationGetBufferForTuple` — hedef sayfa döngüsü

[../src/backend/access/heap/hio.c:500](../src/backend/access/heap/hio.c#L500)

Önce fillfactor'a göre ne kadar boşluk bırakılacağı hesaplanır:

[../src/backend/access/heap/hio.c:536-551](../src/backend/access/heap/hio.c#L536-L551)

```c
	/* Compute desired extra freespace due to fillfactor option */
	saveFreeSpace = RelationGetTargetPageFreeSpace(relation,
												   HEAP_DEFAULT_FILLFACTOR);

	/*
	 * Since pages without tuples can still have line pointers, we consider
	 * pages "empty" when the unavailable space is slight.  This threshold is
	 * somewhat arbitrary, but it should prevent most unnecessary relation
	 * extensions while inserting large tuples into low-fillfactor tables.
	 */
	nearlyEmptyFreeSpace = MaxHeapTupleSize -
		(MaxHeapTuplesPerPage / 8 * sizeof(ItemIdData));
	if (len + saveFreeSpace > nearlyEmptyFreeSpace)
		targetFreeSpace = Max(len, nearlyEmptyFreeSpace);
	else
		targetFreeSpace = len + saveFreeSpace;
```

Sonra hedef sayfa aranır. Sıralama önemlidir:

[../src/backend/access/heap/hio.c:558-596](../src/backend/access/heap/hio.c#L558-L596)

```c
	/*
	 * We first try to put the tuple on the same page we last inserted a tuple
	 * on, as cached in the BulkInsertState or relcache entry.  If that
	 * doesn't work, we ask the Free Space Map to locate a suitable page.
	 * Since the FSM's info might be out of date, we have to be prepared to
	 * loop around and retry multiple times. (To ensure this isn't an infinite
	 * loop, we must update the FSM with the correct amount of free space on
	 * each page that proves not to be suitable.)  If the FSM has no record of
	 * a page with enough free space, we give up and extend the relation.
	 */
	if (bistate && bistate->current_buf != InvalidBuffer)
		targetBlock = BufferGetBlockNumber(bistate->current_buf);
	else
		targetBlock = RelationGetTargetBlock(relation);

	if (targetBlock == InvalidBlockNumber && use_fsm)
	{
		/*
		 * We have no cached target page, so ask the FSM for an initial
		 * target.
		 */
		targetBlock = GetPageWithFreeSpace(relation, targetFreeSpace);
	}
```

Üç kademe: **(1)** bulk-insert durumundaki sayfa → **(2)** relcache'teki son hedef sayfa → **(3)** FSM. Hiçbiri işe yaramazsa son sayfa denenir, o da olmazsa tablo genişletilir.

Döngünün gövdesi:

[../src/backend/access/heap/hio.c:699-706](../src/backend/access/heap/hio.c#L699-L706)

```c
		pageFreeSpace = PageGetHeapFreeSpace(page);
		if (targetFreeSpace <= pageFreeSpace)
		{
			/* use this page as future insert target, too */
			RelationSetTargetBlock(relation, targetBlock);
			return buffer;
		}
```

Sığmadıysa — **FSM'in yanıldığı durum** — kilitler bırakılır ve FSM düzeltilerek yeniden sorulur:

[../src/backend/access/heap/hio.c:751-762](../src/backend/access/heap/hio.c#L751-L762)

```c
		else
		{
			/*
			 * Update FSM as to condition of this page, and ask for another
			 * page to try.
			 */
			targetBlock = RecordAndGetPageWithFreeSpace(relation,
														targetBlock,
														pageFreeSpace,
														targetFreeSpace);
		}
	}

	/* Have to extend the relation */
	buffer = RelationAddBlocks(relation, bistate, num_pages, use_fsm,
							   &unlockedTargetBuffer);
```

Bu **kendini düzelten döngü** FSM tasarımının kilit noktasıdır: FSM yanlış olabilir, ama her yanlış cevap düzeltilerek döngü sonlanır.

### 4.2 FSM neden yaklaşık?

[../src/backend/storage/freespace/README:12-19](../src/backend/storage/freespace/README#L12-L19)

```
It is important to keep the map small so that it can be searched rapidly.
Therefore, we don't attempt to record the exact free space on a page.
We allocate one map byte to each page, allowing us to record free space
at a granularity of 1/256th of a page.  Another way to say it is that
the stored value is the free space divided by BLCKSZ/256 (rounding down).
We assume that the free space must always be less than BLCKSZ, since
all pages have some overhead; so the maximum map value is 255.
```

**Sayfa başına 1 bayt.** 8 KB sayfa için çözünürlük 32 bayttır. 1 GB'lık bir tablonun FSM'i ~128 KB'dir — shared buffers'a rahat sığar. Kategori tablosu [../src/backend/storage/freespace/freespace.c:41-51](../src/backend/storage/freespace/freespace.c#L41-L51):

```
 * Range	 Category
 * 0	- 31   0
 * 32	- 63   1
 * ...    ...  ...
 * 8096 - 8127 253
 * 8128 - 8163 254
 * 8164 - 8192 255
```

### 4.3 FSM ağacı

[../src/backend/storage/freespace/README:21-37](../src/backend/storage/freespace/README#L21-L37)

```
To assist in fast searching, the map isn't simply an array of per-page
entries, but has a tree structure above those entries.  There is a tree
structure of pages, and a tree structure within each page, as described
below.
...
Within each FSM page, we use a binary tree structure where leaf nodes store
the amount of free space on heap pages (or lower level FSM pages, see
"Higher-level structure" below), with one leaf node per heap page. A non-leaf
node stores the max amount of free space on any of its children.

For example:

    4
 4     2
3 4   0 2    <- This level represents heap pages
```

Arama ve güncelleme kuralları:

[../src/backend/storage/freespace/README:39-50](../src/backend/storage/freespace/README#L39-L50)

```
To search for a page with X amount of free space, traverse down the tree
along a path where n >= X, until you hit the bottom. If both children of a
node satisfy the condition, you can pick either one arbitrarily.

To update the amount of free space on a page to X, first update the leaf node
corresponding to the heap page, then "bubble up" the change to upper nodes,
by walking up to each parent and recomputing its value as the max of its
two children.  Repeat until reaching the root or a parent whose value
doesn't change.
```

Kazanç: "boş sayfa yok" cevabı **tek düğüme** bakılarak verilir ([README:52-53](../src/backend/storage/freespace/README#L52-L53)).

### 4.4 Üç FSM giriş noktası

| Fonksiyon | Satır | Ne yapar |
|---|---|---|
| `GetPageWithFreeSpace` | [freespace.c:137](../src/backend/storage/freespace/freespace.c#L137) | Sadece ara |
| `RecordAndGetPageWithFreeSpace` | [freespace.c:154](../src/backend/storage/freespace/freespace.c#L154) | Önce yanlışı düzelt, sonra ara |
| `RecordPageWithFreeSpace` | [freespace.c:194](../src/backend/storage/freespace/freespace.c#L194) | Sadece güncelle |

Birleşik formun neden var olduğu [../src/backend/storage/freespace/freespace.c:145-151](../src/backend/storage/freespace/freespace.c#L145-L151):

```c
 * We provide this combo form to save some locking overhead, compared to
 * separate RecordPageWithFreeSpace + GetPageWithFreeSpace calls. There's
 * also some effort to return a page close to the old page; if there's a
 * page with enough free space on the same FSM page where the old one page
 * is located, it is preferred.
```

Son cümle **yerellik** (locality) optimizasyonudur: mümkünse eski sayfaya yakın bir sayfa döner.

Ve kritik uyarı — güncellemenin neden hemen görünmeyebileceği:

[../src/backend/storage/freespace/freespace.c:188-192](../src/backend/storage/freespace/freespace.c#L188-L192)

```c
 * Note that if the new spaceAvail value is higher than the old value stored
 * in the FSM, the space might not become visible to searchers until the next
 * FreeSpaceMapVacuum call, which updates the upper level pages.
 */
```

Yani **VACUUM'un boşalttığı yer, FSM üst seviyeleri güncellenene kadar kullanılamaz.** `VACUUM` sonunda `FreeSpaceMapVacuum` ([freespace.c:368](../src/backend/storage/freespace/freespace.c#L368)) tam da bunun içindir.

---

## 5. `COPY` ve `heap_multi_insert`

Tek tek `heap_insert` çağırmak pahalıdır: her tuple için buffer kilidi + ayrı WAL kaydı. `heap_multi_insert` bunu birleştirir.

[../src/backend/access/heap/heapam.c:2296-2306](../src/backend/access/heap/heapam.c#L2296-L2306)

```c
/*
 *	heap_multi_insert	- insert multiple tuples into a heap
 *
 * This is like heap_insert(), but inserts multiple tuples in one operation.
 * That's faster than calling heap_insert() in a loop, because when multiple
 * tuples can be inserted on a single page, we can write just a single WAL
 * record covering all of them, and only need to lock/unlock the page once.
 *
 * Note: this leaks memory into the current memory context. You can create a
 * temporary context before calling this, if that's a problem.
 */
```

Sayfaya sığdığı kadar tuple basılır:

[../src/backend/access/heap/heapam.c:2450-2460](../src/backend/access/heap/heapam.c#L2450-L2460)

```c
		/*
		 * RelationGetBufferForTuple has ensured that the first tuple fits.
		 * Put that on the page, and then as many other tuples as fit.
		 */
		RelationPutHeapTuple(relation, buffer, heaptuples[ndone], false);
		...
		for (nthispage = 1; ndone + nthispage < ntuples; nthispage++)
		{
			HeapTuple	heaptup = heaptuples[ndone + nthispage];

			if (PageGetHeapFreeSpace(page) < MAXALIGN(heaptup->t_len) + saveFreeSpace)
				break;

			RelationPutHeapTuple(relation, buffer, heaptup, false);
```

WAL kaydı tipi `XLOG_HEAP2_MULTI_INSERT` ([:2509](../src/backend/access/heap/heapam.c#L2509)) ve sayaç toplu artar ([:2688](../src/backend/access/heap/heapam.c#L2688)):

```c
	pgstat_count_heap_insert(relation, ntuples);
```

### 5.1 `COPY` ne zaman multi-insert kullanır?

Tampon sınırları [../src/backend/commands/copyfrom.c:65-71](../src/backend/commands/copyfrom.c#L65-L71):

```c
#define MAX_BUFFERED_TUPLES		1000
...
#define MAX_BUFFERED_BYTES		65535
```

Yani en çok 1000 satır veya 64 KB — hangisi önce dolarsa. Karar mantığı [../src/backend/commands/copyfrom.c:1018-1076](../src/backend/commands/copyfrom.c#L1018-L1076) beş kez `CIM_SINGLE`'a düşer:

| Koşul | Neden multi-insert olmaz |
|---|---|
| BEFORE / INSTEAD OF ROW trigger var | Trigger tabloyu sorgulayabilir; tamponda bekleyen satırları göremez |
| Foreign table + batching yok | FDW toplu ekleme desteklemiyor |
| Partitioned + statement-level insert trigger | `CopyMultiInsertInfoFlush` tek ilişki varsayar |
| `volatile_defexprs` | Volatile DEFAULT ifadesi tabloyu sorgulayabilir |
| `WHERE` içinde volatile fonksiyon | Aynı gerekçe |

Ortak tema: **tamponda bekleyen satırların henüz görünmemesi.** Kullanıcı kodu araya girebiliyorsa toplu yazma güvenli değildir.

Flush anında index ve AFTER trigger'lar tuple tuple işlenir:

[../src/backend/commands/copyfrom.c:556-582](../src/backend/commands/copyfrom.c#L556-L582)

```c
		table_multi_insert(resultRelInfo->ri_RelationDesc,
						   slots,
						   nused,
						   mycid,
						   ti_options,
						   buffer->bistate);
		...
			if (resultRelInfo->ri_NumIndices > 0)
			{
				List	   *recheckIndexes;

				cstate->cur_lineno = buffer->linenos[i];
				recheckIndexes =
					ExecInsertIndexTuples(resultRelInfo,
										  estate, 0, buffer->slots[i],
										  NIL, NULL);
```

> Sonuç: `COPY` heap yazımını toplulaştırır, **index yazımını toplulaştırmaz.** Çok index'li bir tabloya `COPY` yapmak hâlâ pahalıdır.

---

## 6. Index bakımı ve unique kontrolü

[../src/backend/executor/execIndexing.c:311](../src/backend/executor/execIndexing.c#L311)

Her index için bir `index_insert` çağrılır. Asıl karar hangi kontrol modunun geçileceğidir:

[../src/backend/executor/execIndexing.c:417-436](../src/backend/executor/execIndexing.c#L417-L436)

```c
		/*
		 * The index AM does the actual insertion, plus uniqueness checking.
		 *
		 * For an immediate-mode unique index, we just tell the index AM to
		 * throw error if not unique.
		 *
		 * For a deferrable unique index, we tell the index AM to just detect
		 * possible non-uniqueness, and we add the index OID to the result
		 * list if further checking is needed.
		 *
		 * For a speculative insertion (used by INSERT ... ON CONFLICT), do
		 * the same as for a deferrable unique index.
		 */
		if (!indexRelation->rd_index->indisunique)
			checkUnique = UNIQUE_CHECK_NO;
		else if (applyNoDupErr)
			checkUnique = UNIQUE_CHECK_PARTIAL;
		else if (indexRelation->rd_index->indimmediate)
			checkUnique = UNIQUE_CHECK_YES;
		else
			checkUnique = UNIQUE_CHECK_PARTIAL;
```

Dört mod:

| Mod | Anlamı |
|---|---|
| `UNIQUE_CHECK_NO` | Unique index değil, sadece ekle |
| `UNIQUE_CHECK_YES` | Çakışma varsa **hata ver** |
| `UNIQUE_CHECK_PARTIAL` | Çakışma olabilir → ekle, ama haber ver, bekleme |
| `UNIQUE_CHECK_EXISTING` | Sonradan yeniden kontrol (ertelenmiş kısıtlar) |

Sonra tek çağrı:

[../src/backend/executor/execIndexing.c:449-457](../src/backend/executor/execIndexing.c#L449-L457)

```c
		satisfiesConstraint =
			index_insert(indexRelation, /* index relation */
						 values,	/* array of index Datums */
						 isnull,	/* null flags */
						 tupleid,	/* tid of heap tuple */
						 heapRelation,	/* heap relation */
						 checkUnique,	/* type of uniqueness check to do */
						 indexUnchanged,	/* UPDATE without logical change? */
						 indexInfo);	/* index AM may need this */
```

Dosyanın başındaki tasarım notu unique'in neden burada değil de AM'de yapıldığını açıklar:

[../src/backend/executor/execIndexing.c:12-21](../src/backend/executor/execIndexing.c#L12-L21)

```c
 * Enforcing a unique constraint is straightforward.  When the index AM
 * inserts the tuple to the index, it also checks that there are no
 * conflicting tuples in the index already.  It does so atomically, so that
 * even if two backends try to insert the same key concurrently, only one
 * of them will succeed.  All the logic to ensure atomicity, and to wait
 * for in-progress transactions to finish, is handled by the index AM.
```

**Atomiklik** anahtar kelimedir: heap tarafında "önce bak sonra yaz" yarışı çözemez; btree sayfa kilidi altında ikisini birden yapar.

### 6.1 Exclusion kısıtları farklıdır

[../src/backend/executor/execIndexing.c:29-36](../src/backend/executor/execIndexing.c#L29-L36)

```c
 * Exclusion constraints are different from unique indexes in that when the
 * tuple is inserted to the index, the index AM does not check for
 * duplicate keys at the same time.  After the insertion, we perform a
 * separate scan on the index to check for conflicting tuples, and if one
 * is found, we throw an error and the transaction is aborted.  If the
 * conflicting tuple's inserter or deleter is in-progress, we wait for it
 * to finish first.
```

Yani exclusion'da sıra ters: **önce yaz, sonra tara.** Bunun bedeli deadlock riskidir ([:38-45](../src/backend/executor/execIndexing.c#L38-L45)) ve "zaten biri hata alacaktı" gerekçesiyle kabul edilir.

### 6.2 DELETE'te index'e ne olur?

Hiçbir şey. `ExecDelete` içindeki yorum tarihî ama nettir:

[../src/backend/executor/nodeModifyTable.c:2087-2095](../src/backend/executor/nodeModifyTable.c#L2087-L2095)

```c
		/*
		 * Note: Normally one would think that we have to delete index tuples
		 * associated with the heap tuple now...
		 *
		 * ... but in POSTGRES, we have no need to do this because VACUUM will
		 * take care of it later.  We can't delete index tuples immediately
		 * anyway, since the tuple is still visible to other transactions.
		 */
```

Son cümle asıl gerekçedir: **satır hâlâ başka transaction'lara görünür.** MVCC index girdisini hemen silmeyi imkânsız kılar.

---

## 7. `ON CONFLICT` — speculative insertion

Sorun şudur: "önce kontrol et, yoksa ekle" iki adımdır ve arada başkası aynı anahtarı ekleyebilir. Çözüm, tuple'ı **geri alınabilir** biçimde yazmaktır.

### 7.1 Ön kontrol

[../src/backend/executor/nodeModifyTable.c:1143-1165](../src/backend/executor/nodeModifyTable.c#L1143-L1165)

```c
			/*
			 * Do a non-conclusive check for conflicts first.
			 *
			 * We're not holding any locks yet, so this doesn't guarantee that
			 * the later insert won't conflict.  But it avoids leaving behind
			 * a lot of canceled speculative insertions, if you run a lot of
			 * INSERT ON CONFLICT statements that do conflict.
			 *
			 * We loop back here if we find a conflict below, either during
			 * the pre-check, or when we re-check after inserting the tuple
			 * speculatively.  Better allow interrupts in case some bug makes
			 * this an infinite loop.
			 */
	vlock:
			CHECK_FOR_INTERRUPTS();
			specConflict = false;
			if (!ExecCheckIndexConstraints(resultRelInfo, slot, estate,
										   &conflictTid, &invalidItemPtr,
										   arbiterIndexes))
```

`ExecCheckIndexConstraints` ([../src/backend/executor/execIndexing.c:543](../src/backend/executor/execIndexing.c#L543)) yalnızca **optimizasyondur**, garanti değildir:

[../src/backend/executor/execIndexing.c:529-538](../src/backend/executor/execIndexing.c#L529-L538)

```c
 *		Note that this doesn't lock the values in any way, so it's
 *		possible that a conflicting tuple is inserted immediately
 *		after this returns.  This can be used for either a pre-check
 *		before insertion or a re-check after finding a conflict.
```

Çakışma bulunursa üç yol: `DO UPDATE` → `ExecOnConflictUpdate`, `DO SELECT` → `ExecOnConflictSelect`, `DO NOTHING` → `ExecCheckTIDVisible` + `return NULL` ([:1166-1215](../src/backend/executor/nodeModifyTable.c#L1166-L1215)).

### 7.2 Token alma ve spekülatif yazma

[../src/backend/executor/nodeModifyTable.c:1226-1259](../src/backend/executor/nodeModifyTable.c#L1226-L1259)

```c
			/*
			 * Before we start insertion proper, acquire our "speculative
			 * insertion lock".  Others can use that to wait for us to decide
			 * if we're going to go ahead with the insertion, instead of
			 * waiting for the whole transaction to complete.
			 */
			INJECTION_POINT("exec-insert-before-insert-speculative", NULL);
			specToken = SpeculativeInsertionLockAcquire(GetCurrentTransactionId());

			/* insert the tuple, with the speculative token */
			table_tuple_insert_speculative(resultRelationDesc, slot,
										   estate->es_output_cid,
										   0,
										   NULL,
										   specToken);

			/* insert index entries for tuple */
			recheckIndexes = ExecInsertIndexTuples(resultRelInfo,
												   estate, EIIT_NO_DUPE_ERROR,
												   slot, arbiterIndexes,
												   &specConflict);

			/* adjust the tuple's state accordingly */
			table_tuple_complete_speculative(resultRelationDesc, slot,
											 specToken, !specConflict);

			/*
			 * Wake up anyone waiting for our decision.  They will re-check
			 * the tuple, see that it's no longer speculative, and wait on our
			 * XID as if this was a regularly inserted tuple all along.  Or if
			 * we killed the tuple, they will see it's dead, and proceed as if
			 * the tuple never existed.
			 */
			SpeculativeInsertionLockRelease(GetCurrentTransactionId());
```

Kilit noktası: **token bir transaction kilidi değildir.** Bekleyen backend'in tüm transaction'ın bitmesini beklemesi gerekmez; sadece "bu satır kalacak mı?" kararını bekler. Bu, `ON CONFLICT`'in yüksek eşzamanlılıkta çalışabilmesinin nedenidir.

Çakışma olursa baştan:

[../src/backend/executor/nodeModifyTable.c:1260-1269](../src/backend/executor/nodeModifyTable.c#L1260-L1269)

```c
			/*
			 * If there was a conflict, start from the beginning.  We'll do
			 * the pre-check again, which will now find the conflicting tuple
			 * (unless it aborts before we get there).
			 */
			if (specConflict)
			{
				list_free(recheckIndexes);
				goto vlock;
			}
```

### 7.3 Karşı taraf — `_bt_check_unique` ile yarış

Aynı anahtarı ekleyen ikinci backend btree'de bekler:

[../src/backend/access/nbtree/nbtinsert.c:213-235](../src/backend/access/nbtree/nbtinsert.c#L213-L235)

```c
		xwait = _bt_check_unique(rel, &insertstate, heapRel, checkUnique,
								 &is_unique, &speculativeToken);

		if (unlikely(TransactionIdIsValid(xwait)))
		{
			/* Have to wait for the other guy ... */
			_bt_relbuf(rel, insertstate.buf);
			insertstate.buf = InvalidBuffer;

			/*
			 * If it's a speculative insertion, wait for it to finish (ie. to
			 * go ahead with the insertion, or kill the tuple).  Otherwise
			 * wait for the transaction to finish as usual.
			 */
			if (speculativeToken)
				SpeculativeInsertionWait(xwait, speculativeToken);
			else
				XactLockTableWait(xwait, rel, &itup->t_tid, XLTW_InsertIndex);

			/* start over... */
			if (stack)
				_bt_freestack(stack);
			goto search;
		}
```

Token `SnapshotDirty` üzerinden taşınır ([../src/backend/access/nbtree/nbtinsert.c:589-600](../src/backend/access/nbtree/nbtinsert.c#L589-L600)):

```c
					xwait = (TransactionIdIsValid(SnapshotDirty.xmin)) ?
						SnapshotDirty.xmin : SnapshotDirty.xmax;

					if (TransactionIdIsValid(xwait))
					{
						if (nbuf != InvalidBuffer)
							_bt_relbuf(rel, nbuf);
						/* Tell _bt_doinsert to wait... */
						*speculativeToken = SnapshotDirty.speculativeToken;
```

### 7.4 Geri alma — `heap_abort_speculative`

[../src/backend/access/heap/heapam.c:6390](../src/backend/access/heap/heapam.c#L6390)

Normal DELETE değildir, "super deletion"dır: satır **hemen ve herkese** görünmez olur.

[../src/backend/access/heap/heapam.c:6463-6473](../src/backend/access/heap/heapam.c#L6463-L6473)

```c
	/*
	 * Set the tuple header xmin to InvalidTransactionId.  This makes the
	 * tuple immediately invisible everyone.  (In particular, to any
	 * transactions waiting on the speculative token, woken up later.)
	 */
	HeapTupleHeaderSetXmin(tp.t_data, InvalidTransactionId);

	/* Clear the speculative insertion token too */
	tp.t_data->t_ctid = tp.t_self;
```

`xmax` yazmak yetmez — `xmin`'in geçersizleştirilmesi gerekir, çünkü satırın **hiç var olmamış** sayılması istenir. Karşılaştırma için `heap_delete` sadece `xmax` yazar (bölüm 8.3).

Ayrıca: bu satır TOAST verisi de yazmış olabilir; onlar da `heap_abort_speculative` ile silinir ([../src/backend/access/heap/heapam.c:6512](../src/backend/access/heap/heapam.c#L6512)).

Onaylanan durumda ise sadece token temizlenir — `heap_finish_speculative` ([:6303](../src/backend/access/heap/heapam.c#L6303)):

[../src/backend/access/heap/heapam.c:6331-6336](../src/backend/access/heap/heapam.c#L6331-L6336)

```c
	/*
	 * Replace the speculative insertion token with a real t_ctid, pointing to
	 * itself like it does on regular tuples.
	 */
	htup->t_ctid = *tid;
```

---

## 8. `ExecDelete` ve `heap_delete`

### 8.1 Üç aşama

| Aşama | Satır | İş |
|---|---|---|
| `ExecDeletePrologue` | [nodeModifyTable.c:1736](../src/backend/executor/nodeModifyTable.c#L1736) | BEFORE ROW DELETE trigger |
| `ExecDeleteAct` | [nodeModifyTable.c:1768](../src/backend/executor/nodeModifyTable.c#L1768) | `table_tuple_delete` |
| `ExecDeleteEpilogue` | [nodeModifyTable.c:1795](../src/backend/executor/nodeModifyTable.c#L1795) | AFTER ROW DELETE trigger |

Prologue'un ne kadar az iş yaptığı yorumdan bellidir:

[../src/backend/executor/nodeModifyTable.c:1729-1734](../src/backend/executor/nodeModifyTable.c#L1729-L1734)

```c
 * Prepare executor state for DELETE.  Actually, the only thing we have to do
 * here is execute BEFORE ROW triggers.  We return false if one of them makes
 * the delete a no-op; otherwise, return true.
 */
```

Act tarafı da öyle:

[../src/backend/executor/nodeModifyTable.c:1774-1783](../src/backend/executor/nodeModifyTable.c#L1774-L1783)

```c
	if (changingPart)
		options |= TABLE_DELETE_CHANGING_PARTITION;

	return table_tuple_delete(resultRelInfo->ri_RelationDesc, tupleid,
							  estate->es_output_cid,
							  options,
							  estate->es_snapshot,
							  estate->es_crosscheck_snapshot,
							  true /* wait for commit */ ,
							  &context->tmfd);
```

`changingPart` bayrağı **cross-partition UPDATE**'ten gelir: UPDATE partition sınırını aşarsa satır önce silinir ([UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm 8).

### 8.2 Eşzamanlılık — `l1` döngüsü

[../src/backend/access/heap/heapam.c:2759](../src/backend/access/heap/heapam.c#L2759)

`heap_delete` de `heap_update` gibi bir etikete geri döner:

[../src/backend/access/heap/heapam.c:2820-2844](../src/backend/access/heap/heapam.c#L2820-L2844)

```c
l1:
	...
	result = HeapTupleSatisfiesUpdate(&tp, cid, buffer);

	if (result == TM_Invisible)
	{
		UnlockReleaseBuffer(buffer);
		ereport(ERROR,
				(errcode(ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE),
				 errmsg("attempted to delete invisible tuple")));
	}
	else if (result == TM_BeingModified && wait)
```

Bekleme mantığı `heap_update`'inkinin sadeleştirilmiş halidir — DELETE her zaman `LockTupleExclusive` ister, key/non-key ayrımı yoktur:

[../src/backend/access/heap/heapam.c:2913-2924](../src/backend/access/heap/heapam.c#L2913-L2924)

```c
		else if (!TransactionIdIsCurrentTransactionId(xwait))
		{
			/*
			 * Wait for regular transaction to end; but first, acquire tuple
			 * lock.
			 */
			LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
			heap_acquire_tuplock(relation, &(tp.t_self), LockTupleExclusive,
								 LockWaitBlock, &have_tuple_lock);
			XactLockTableWait(xwait, relation, &(tp.t_self), XLTW_Delete);
			LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
```

Sonuç kararı:

[../src/backend/access/heap/heapam.c:2947-2956](../src/backend/access/heap/heapam.c#L2947-L2956)

```c
		/*
		 * We may overwrite if previous xmax aborted, or if it committed but
		 * only locked the tuple without updating it.
		 */
		if ((tp.t_data->t_infomask & HEAP_XMAX_INVALID) ||
			HEAP_XMAX_IS_LOCKED_ONLY(tp.t_data->t_infomask) ||
			HeapTupleHeaderIsOnlyLocked(tp.t_data))
			result = TM_Ok;
		else if (!ItemPointerEquals(&tp.t_self, &tp.t_data->t_ctid))
			result = TM_Updated;
		else
			result = TM_Deleted;
```

**Yine `t_ctid` karar veriyor** — UPDATE'teki ile birebir aynı kural.

### 8.3 Kritik bölüm — tek alan

[../src/backend/access/heap/heapam.c:3036-3070](../src/backend/access/heap/heapam.c#L3036-L3070)

```c
	START_CRIT_SECTION();

	/*
	 * If this transaction commits, the tuple will become DEAD sooner or
	 * later.  Set flag that this page is a candidate for pruning once our xid
	 * falls below the OldestXmin horizon. ...
	 */
	PageSetPrunable(page, xid);
	...
	/* store transaction information of xact deleting the tuple */
	tp.t_data->t_infomask &= ~(HEAP_XMAX_BITS | HEAP_MOVED);
	tp.t_data->t_infomask2 &= ~HEAP_KEYS_UPDATED;
	tp.t_data->t_infomask |= new_infomask;
	tp.t_data->t_infomask2 |= new_infomask2;
	HeapTupleHeaderClearHotUpdated(tp.t_data);
	HeapTupleHeaderSetXmax(tp.t_data, new_xmax);
	HeapTupleHeaderSetCmax(tp.t_data, cid, iscombo);
	/* Make sure there is no forward chain link in t_ctid */
	tp.t_data->t_ctid = tp.t_self;
```

Bütün DELETE budur. Üç gözlem:

1. **`t_ctid = t_self`** — açıkça kendine çevrilir. Yorum "Make sure there is no forward chain link" der; satır daha önce HOT zincirinin parçasıysa bağ koparılır.
2. **`HeapTupleHeaderClearHotUpdated`** — zincir başı bayrağı temizlenir.
3. **Satır verisi hiç okunmaz.** Sadece başlık yazılır; bu yüzden DELETE satır genişliğinden bağımsız olarak ucuzdur.

### 8.4 `HEAP_XMAX_*` bitleri

[../src/include/access/htup_details.h:194-217](../src/include/access/htup_details.h#L194-L217)

```c
#define HEAP_XMAX_KEYSHR_LOCK	0x0010	/* xmax is a key-shared locker */
...
#define HEAP_XMAX_EXCL_LOCK		0x0040	/* xmax is exclusive locker */
#define HEAP_XMAX_LOCK_ONLY		0x0080	/* xmax, if valid, is only a locker */
...
#define HEAP_XMAX_COMMITTED		0x0400	/* t_xmax committed */
#define HEAP_XMAX_INVALID		0x0800	/* t_xmax invalid/aborted */
#define HEAP_XMAX_IS_MULTI		0x1000	/* t_xmax is a MultiXactId */
#define HEAP_UPDATED			0x2000	/* this is UPDATEd version of row */
```

`xmax` alanı üç ayrı şey olabilir:

| Durum | Bitler | Anlam |
|---|---|---|
| Silindi | `XMAX_INVALID` yok, `LOCK_ONLY` yok | `xmax` = silen transaction |
| Sadece kilitlendi | `HEAP_XMAX_LOCK_ONLY` | Satır canlı, `SELECT FOR ...` ile kilitli |
| Çok kilitleyici | `HEAP_XMAX_IS_MULTI` | `xmax` bir MultiXactId |
| Geçersiz | `HEAP_XMAX_INVALID` | `xmax` yok sayılır (ipucu biti) |

`HEAP_XMAX_COMMITTED`/`HEAP_XMAX_INVALID` **ipucu bitleridir** (hint bits): görünürlük testi bunları bulursa CLOG'a gitmez. İlk okuyan yazar, sonrakiler bedava faydalanır. Detay için bkz. [KILIT-MEKANIZMALARI.md](KILIT-MEKANIZMALARI.md).

### 8.5 WAL ve TOAST temizliği

WAL kaydı `XLOG_HEAP_DELETE`'tir ([../src/backend/access/heap/heapam.c:3153](../src/backend/access/heap/heapam.c#L3153)) ve `heap_abort_speculative` ile paylaşılır:

[../src/backend/access/heap/heapam.c:3074-3078](../src/backend/access/heap/heapam.c#L3074-L3078)

```c
	/*
	 * XLOG stuff
	 *
	 * NB: heap_abort_speculative() uses the same xlog record and replay
	 * routines.
	 */
```

Kaydın içinde tuple verisi **yoktur** — sadece offset, `xmax` ve infobit'ler. Ancak logical replication için replica identity yazılır ([:3103-3109](../src/backend/access/heap/heapam.c#L3103-L3109)); `REPLICA IDENTITY FULL` ise tüm eski satır WAL'a girer ve DELETE aniden pahalılaşır.

Buffer kilidi bırakıldıktan sonra TOAST zinciri temizlenir:

[../src/backend/access/heap/heapam.c:3170-3184](../src/backend/access/heap/heapam.c#L3170-L3184)

```c
	/*
	 * If the tuple has toasted out-of-line attributes, we need to delete
	 * those items too.  We have to do this before releasing the buffer
	 * because we need to look at the contents of the tuple, but it's OK to
	 * release the content lock on the buffer first.
	 */
	...
	else if (HeapTupleHasExternal(&tp))
		heap_toast_delete(relation, &tp, false);
```

Yani **DELETE TOAST satırlarını gerçekten siler** (onların `xmax`'ını yazar). Detay: [TOAST-NASIL-CALISIR.md](TOAST-NASIL-CALISIR.md).

### 8.6 Eşzamanlı DELETE — `TM_Updated` ve EvalPlanQual

[../src/backend/executor/nodeModifyTable.c:1980-2027](../src/backend/executor/nodeModifyTable.c#L1980-L2027)

```c
			case TM_Updated:
				{
					TupleTableSlot *inputslot;
					TupleTableSlot *epqslot;

					if (IsolationUsesXactSnapshot())
						ereport(ERROR,
								(errcode(ERRCODE_T_R_SERIALIZATION_FAILURE),
								 errmsg("could not serialize access due to concurrent update")));

					/*
					 * Already know that we're going to need to do EPQ, so
					 * fetch tuple directly into the right slot.
					 */
					EvalPlanQualBegin(context->epqstate);
					inputslot = EvalPlanQualSlot(context->epqstate, resultRelationDesc,
												 resultRelInfo->ri_RangeTableIndex);

					result = table_tuple_lock(resultRelationDesc, tupleid,
											  estate->es_snapshot,
											  inputslot, estate->es_output_cid,
											  LockTupleExclusive, LockWaitBlock,
											  TUPLE_LOCK_FLAG_FIND_LAST_VERSION,
											  &context->tmfd);
					...
							if (TupIsNull(epqslot))
								/* Tuple not passing quals anymore, exiting... */
								return NULL;
							...
							else
								goto ldelete;
```

Mekanizma UPDATE ile aynıdır ([UPDATE-NASIL-CALISIR.md](UPDATE-NASIL-CALISIR.md) bölüm 7); iki fark:

- DELETE'te yeni tuple kurulmaz — `goto ldelete` doğrudan silmeyi tekrarlar.
- `epqreturnslot` verilmişse silme yapılmaz, güncel satır geri döndürülür ([:2016-2027](../src/backend/executor/nodeModifyTable.c#L2016-L2027)). Bunu cross-partition UPDATE kullanır.

Diğer sonuçlar:

| `TM_Result` | Tepki | Satır |
|---|---|---|
| `TM_SelfModified` | `cmax == es_output_cid` ise sessizce atla, değilse hata | [:1942-1975](../src/backend/executor/nodeModifyTable.c#L1942-L1975) |
| `TM_Updated` | EPQ veya serialization hatası | [:1980](../src/backend/executor/nodeModifyTable.c#L1980) |
| `TM_Deleted` | Zaten silinmiş, atla (RR/SER'de hata) | [:2074-2081](../src/backend/executor/nodeModifyTable.c#L2074-L2081) |
| `TM_BeingModified` | `heap_delete` içinde beklenir | [heapam.c:2844](../src/backend/access/heap/heapam.c#L2844) |

`TM_SelfModified` hatasının metni öğreticidir:

[../src/backend/executor/nodeModifyTable.c:1968-1972](../src/backend/executor/nodeModifyTable.c#L1968-L1972)

```c
				if (context->tmfd.cmax != estate->es_output_cid)
					ereport(ERROR,
							(errcode(ERRCODE_TRIGGERED_DATA_CHANGE_VIOLATION),
							 errmsg("tuple to be deleted was already modified by an operation triggered by the current command"),
							 errhint("Consider using an AFTER trigger instead of a BEFORE trigger to propagate changes to other rows.")));
```

BEFORE trigger içinde aynı satırı güncellerseniz bu hatayı alırsınız.

---

## 9. Trigger makinesi ve ertelenmiş kısıtlar

### 9.1 Trigger çağrı noktaları

| Trigger | Fonksiyon | Satır |
|---|---|---|
| BEFORE STATEMENT INSERT | `ExecASInsertTriggers` | [trigger.c:2479](../src/backend/commands/trigger.c#L2479) |
| BEFORE ROW INSERT | `ExecBRInsertTriggers` | [trigger.c:2492](../src/backend/commands/trigger.c#L2492) |
| AFTER ROW INSERT | `ExecARInsertTriggers` | [trigger.c:2570](../src/backend/commands/trigger.c#L2570) |
| BEFORE STATEMENT DELETE | `ExecASDeleteTriggers` | [trigger.c:2708](../src/backend/commands/trigger.c#L2708) |
| BEFORE ROW DELETE | `ExecBRDeleteTriggers` | [trigger.c:2728](../src/backend/commands/trigger.c#L2728) |
| AFTER ROW DELETE | `ExecARDeleteTriggers` | [trigger.c:2828](../src/backend/commands/trigger.c#L2828) |

Hepsi sonunda tek yerden geçer — `ExecCallTriggerFunc` ([../src/backend/commands/trigger.c:2336](../src/backend/commands/trigger.c#L2336)):

[../src/backend/commands/trigger.c:2324-2334](../src/backend/commands/trigger.c#L2324-L2334)

```c
/*
 * Call a trigger function.
 *
 *		trigdata: trigger descriptor.
 *		tgindx: trigger's index in finfo and instr arrays.
 *		finfo: array of cached trigger function call information.
 *		instr: optional array of EXPLAIN ANALYZE instrumentation state.
 *		per_tuple_context: memory context to execute the function in.
 *
 * Returns the tuple (or NULL) as returned by the function.
 */
```

Kritik fark: **BEFORE ROW trigger dönüş değeri kullanılır** (NULL = iptal, farklı tuple = değiştir), **AFTER ROW trigger'ın dönüşü yok sayılır.** AFTER trigger hemen çalışmaz; kuyruğa girer.

### 9.2 Ertelenmiş kuyruk — `AfterTriggerEndQuery`

[../src/backend/commands/trigger.c:5186](../src/backend/commands/trigger.c#L5186)

[../src/backend/commands/trigger.c:5173-5183](../src/backend/commands/trigger.c#L5173-L5183)

```c
/* ----------
 * AfterTriggerEndQuery()
 *
 *	Called after one query has been completely processed. At this time
 *	we invoke all AFTER IMMEDIATE trigger events queued by the query, and
 *	transfer deferred trigger events to the global deferred-trigger list.
 *
 *	Note that this must be called BEFORE closing down the executor
 *	with ExecutorEnd, because we make use of the EState's info about
 *	target relations.  Normally it is called from ExecutorFinish.
 * ----------
 */
```

İki kuyruk vardır:

```
  Sorgu sırasında              Sorgu bitince                 Commit'te
  ─────────────────            ─────────────                 ─────────
  AfterTriggerSaveEvent   →    AfterTriggerEndQuery     →    AfterTriggerFireDeferred
  (trigger.c:6262)             (trigger.c:5186)              (trigger.c:5354)
        │                            │                             │
        ▼                            ▼                             ▼
  query_stack[depth]         IMMEDIATE olanlar ateşlenir     DEFERRED olanlar
     .events                 DEFERRED olanlar                  ateşlenir
                             afterTriggers.events'e taşınır
```

Ayrımı yapan fonksiyon `afterTriggerCheckState` ([../src/backend/commands/trigger.c:4055](../src/backend/commands/trigger.c#L4055)):

[../src/backend/commands/trigger.c:4061-4089](../src/backend/commands/trigger.c#L4061-L4089)

```c
	/*
	 * For not-deferrable triggers (i.e. normal AFTER ROW triggers and
	 * constraints declared NOT DEFERRABLE), the state is always false.
	 */
	if ((evtshared->ats_event & AFTER_TRIGGER_DEFERRABLE) == 0)
		return false;

	/*
	 * If constraint state exists, SET CONSTRAINTS might have been executed
	 * either for this trigger or for all triggers.
	 */
	if (state != NULL)
	{
		/* Check for SET CONSTRAINTS for this specific trigger. */
		for (i = 0; i < state->numstates; i++)
		{
			if (state->trigstates[i].sct_tgoid == tgoid)
				return state->trigstates[i].sct_tgisdeferred;
		}

		/* Check for SET CONSTRAINTS ALL. */
		if (state->all_isset)
			return state->all_isdeferred;
	}

	/*
	 * Otherwise return the default state for the trigger.
	 */
	return ((evtshared->ats_event & AFTER_TRIGGER_INITDEFERRED) != 0);
```

Yani öncelik sırası: **`NOT DEFERRABLE` > o oturumun `SET CONSTRAINTS`'i > `SET CONSTRAINTS ALL` > `INITIALLY DEFERRED` tanımı.**

`SET CONSTRAINTS` bunu `AfterTriggerSetState` ([:5851](../src/backend/commands/trigger.c#L5851)) ile ayarlar. Foreign key kontrolleri de birer AFTER trigger olduğundan, `DEFERRABLE INITIALLY DEFERRED` FK'lar commit'e kadar kontrol edilmez — dairesel referansları yüklemenin standart yolu budur.

Kuyruk döngüsünde ince bir nokta var:

[../src/backend/commands/trigger.c:5205-5219](../src/backend/commands/trigger.c#L5205-L5219)

```c
	 * Notice that we decide which ones will be fired, and put the deferred
	 * ones on the main list, before anything is actually fired.  This ensures
	 * reasonably sane behavior if a trigger function does SET CONSTRAINTS ...
	 * IMMEDIATE: all events we have decided to defer will be available for it
	 * to fire.
	 *
	 * We loop in case a trigger queues more events at the same query level.
```

Yani bir trigger yeni olay üretebilir; döngü boşalana kadar döner.

---

## 10. Tek sayfalık özet

```
INSERT INTO users VALUES (42,'ali');        DELETE FROM users WHERE id=42;

 PLANLAYICI                                  PLANLAYICI
 └─ junk kolon YOK                           └─ add_row_identity_columns → junk "ctid"

 EXECUTOR (nodeModifyTable.c)                EXECUTOR
 ├─ ExecGetInsertNewTuple  [:792]            ├─ junk ctid oku            [:4864]
 └─ ExecInsert             [:874]            └─ ExecDelete               [:1857]
     ├─ ExecPrepareTupleRouting                  ├─ ExecDeletePrologue   [:1736]
     ├─ ExecOpenIndices(spec?)                   │    └─ BEFORE ROW DELETE
     ├─ BEFORE ROW INSERT trigger               ├─ ExecDeleteAct        [:1768]
     ├─ ExecComputeStoredGenerated              │    └─ table_tuple_delete ──┐
     ├─ RLS WITH CHECK                          ├─ index'e DOKUNULMAZ        │
     ├─ ExecConstraints  (NOT NULL, CHECK)      ├─ ExecDeleteEpilogue   [:1795]
     ├─ ExecPartitionCheck                      │    └─ AFTER ROW DELETE     │
     │                                          └─ RETURNING (ESKİ satır)    │
     ├─ ON CONFLICT? ───► speculative:                                       │
     │    vlock: ExecCheckIndexConstraints  (ön kontrol, garanti değil)      │
     │           SpeculativeInsertionLockAcquire → token                     │
     │           table_tuple_insert_speculative  (t_ctid = token)            │
     │           ExecInsertIndexTuples(NO_DUPE_ERROR)                        │
     │           complete_speculative → onayla | heap_abort_speculative      │
     │           çakışma varsa → goto vlock                                  │
     ├─ değilse: table_tuple_insert ─────────┐                               │
     ├─ ExecInsertIndexTuples  [execIndexing.c:311]                          │
     │    └─ index_insert(checkUnique) → btree atomik kontrol                │
     ├─ AFTER ROW INSERT trigger             │                               │
     └─ RETURNING (YENİ satır)               │                               │
                                             ▼                               ▼
 HEAP: heap_insert  [heapam.c:2005]           HEAP: heap_delete [heapam.c:2759]
 ├─ heap_prepare_insert  [:2228]              ├─ l1: HeapTupleSatisfiesUpdate
 │    ├─ xmin=xid, xmax=0, XMAX_INVALID       │    └─ TM_BeingModified → bekle
 │    └─ t_len > TOAST_TUPLE_THRESHOLD?       │       t_ctid≠self → TM_Updated
 │        └─► heap_toast_insert_or_update     │       t_ctid=self → TM_Deleted
 ├─ RelationGetBufferForTuple  [hio.c:500]    ├─ compute_new_xmax_infomask
 │    ├─ bistate → relcache → FSM             ├─ KRİTİK BÖLÜM  [:3036]
 │    ├─ sığmadı? RecordAndGetPageWithFreeSpace│    ├─ PageSetPrunable
 │    └─ hiç yok? RelationAddBlocks           │    ├─ xmax = xid, cmax = cid
 ├─ KRİTİK BÖLÜM  [:2066]                     │    ├─ ClearHotUpdated
 │    ├─ RelationPutHeapTuple → t_ctid=self   │    ├─ t_ctid = t_self  ← ZİNCİR YOK
 │    ├─ PageSetPrunable  (ROLLBACK için)     │    └─ XLOG_HEAP_DELETE
 │    └─ XLOG_HEAP_INSERT [+INIT_PAGE]        ├─ heap_toast_delete  [:3184]
 └─ pgstat_count_heap_insert  [:2208]         └─ pgstat_count_heap_delete [:3202]

 FSM  [freespace.c]                           SONRASI
 ├─ sayfa başına 1 BAYT (1/256 çözünürlük)    ├─ index girdisi ölü tuple'a bakmaya devam
 ├─ ikili ağaç: iç düğüm = çocukların max'ı   ├─ heap_page_prune_opt ölüleri toplar
 ├─ "yer yok" cevabı tek düğümden okunur      └─ VACUUM index girdilerini temizler
 └─ YAKLAŞIK: yanlış sayfa dönebilir → döngü

 SORGU SONU
 └─ AfterTriggerEndQuery  [trigger.c:5186]
      ├─ IMMEDIATE olayları ateşle
      └─ DEFERRED olayları COMMIT kuyruğuna taşı → AfterTriggerFireDeferred [:5354]
```

---

## 11. İzleme ve hata ayıklama

**Temel sayaçlar:**

```sql
SELECT relname,
       n_tup_ins,
       n_tup_del,
       n_live_tup,
       n_dead_tup,
       pg_size_pretty(pg_relation_size(relid)) AS size
FROM pg_stat_user_tables
ORDER BY n_tup_ins DESC;
```

Kaynakları: `n_tup_ins` ← [heapam.c:2208](../src/backend/access/heap/heapam.c#L2208) ve [:2688](../src/backend/access/heap/heapam.c#L2688), `n_tup_del` ← [heapam.c:3202](../src/backend/access/heap/heapam.c#L3202).

> `n_tup_ins`, iptal edilen speculative insertion'ları da sayar. `ON CONFLICT` yoğun bir tabloda bu sayıyı gerçek satır sayısı sanmayın.

**FSM'in ne bildiğini görmek** (`pg_freespacemap` eklentisi):

```sql
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

SELECT blkno, avail
FROM pg_freespace('users')
WHERE avail > 0
ORDER BY blkno
LIMIT 20;

-- Toplam "kullanılabilir" boşluk
SELECT pg_size_pretty(sum(avail)::bigint) FROM pg_freespace('users');
```

`avail` değeri 32 baytın katlarıdır — bölüm 4.2'deki kategori tablosunun sonucudur.

DELETE sonrası bu sayı hemen artmaz; artması için VACUUM'un `FreeSpaceMapVacuum`'u çağırması gerekir ([freespace.c:368](../src/backend/storage/freespace/freespace.c#L368)).

**Tablo genişlemesi FSM'i mi atlıyor?** Yeni sayfa açılması `RelationAddBlocks` ([hio.c:236](../src/backend/access/heap/hio.c#L236)) demektir. Yoğun eşzamanlı INSERT'te bu darboğaz olur:

```sql
SELECT pid, wait_event_type, wait_event, state, left(query, 60)
FROM pg_stat_activity
WHERE wait_event IN ('RelationExtend', 'BufferMapping', 'ExtendRelation');
```

**Bir satırın DELETE sonrası halini görmek** (`pageinspect`):

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid,
       (t_infomask & 128)  <> 0 AS xmax_lock_only,   -- HEAP_XMAX_LOCK_ONLY
       (t_infomask & 1024) <> 0 AS xmax_committed,   -- HEAP_XMAX_COMMITTED
       (t_infomask & 2048) <> 0 AS xmax_invalid,     -- HEAP_XMAX_INVALID
       (t_infomask & 4096) <> 0 AS xmax_is_multi     -- HEAP_XMAX_IS_MULTI
FROM heap_page_items(get_raw_page('users', 0));
```

Silinmiş satırda `t_xmax` doludur ve **`t_ctid` hâlâ `(blok,offset)` olarak kendini gösterir** — UPDATE'ten ayırt etme yolu budur.

**Speculative insertion'ı yakalamak:** `t_ctid` alanı anlamsız bir değer gösteriyorsa ve `t_xmin` çalışan bir transaction ise, o an spekülatif bir tuple'a bakıyorsunuz demektir. Bekleyen tarafı görmek:

```sql
SELECT pid, wait_event_type, wait_event, left(query, 60)
FROM pg_stat_activity
WHERE wait_event = 'SpeculativeToken';
```

**Ertelenmiş kısıtları görmek:**

```sql
SELECT conname, contype, condeferrable, condeferred
FROM pg_constraint
WHERE conrelid = 'users'::regclass;

-- Oturum içinde ertele
BEGIN;
  SET CONSTRAINTS ALL DEFERRED;
  -- ... dairesel referanslı yüklemeler ...
COMMIT;   -- kontroller burada çalışır
```

`condeferred = true` olanlar `AFTER_TRIGGER_INITDEFERRED` bayrağıyla kuyruğa girer ([trigger.c:4088](../src/backend/commands/trigger.c#L4088)).

**Trigger maliyetini ölçmek:**

```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO users VALUES (43, 'veli');
```

`EXPLAIN ANALYZE` çıktısında "Trigger for constraint ...: time=... calls=..." satırları görülür; kaynağı `ExecCallTriggerFunc`'a geçilen `instr` dizisidir ([trigger.c:2336](../src/backend/commands/trigger.c#L2336)).

**`COPY` multi-insert kullanıyor mu?** Doğrudan bir sayaç yok; dolaylı test: tabloya BEFORE ROW trigger veya volatile DEFAULT ekleyip `COPY` hızını karşılaştırın (bölüm 5.1 tablosu).

---

## Pratik sonuçlar

1. **DELETE ucuzdur, temizlik pahalıdır.** Silme tek alan yazar; asıl maliyet VACUUM'a ve index temizliğine ertelenir. Toplu silmeden sonra `VACUUM` çalıştırmadan yer geri gelmez.
2. **INSERT'in maliyetini index sayısı belirler.** Heap'e yazma sabit; her index ayrı bir `index_insert` demektir ve unique index'ler ayrıca bekleyebilir.
3. **`fillfactor` INSERT için değil UPDATE içindir.** `RelationGetBufferForTuple` yorumu bunu açıkça söyler ([hio.c:491-494](../src/backend/access/heap/hio.c#L491-L494)): ayrılan boşluk, aynı sayfada kalacak yeni sürümler içindir.
4. **`COPY` `INSERT`'ten hızlıdır ama sihir değildir.** Kazanç heap yazımı ve WAL'dadır; index yazımı yine satır satırdır. En hızlı yükleme: index'siz `COPY` + sonra `CREATE INDEX`.
5. **`ON CONFLICT` gerçekten yazar.** Çakışan denemeler de heap'e tuple yazıp geri alır; `n_tup_ins` şişer ve sayfalarda ölü tuple birikir. Çakışma oranı yüksekse `DO NOTHING` yerine önce `SELECT` etmeyi ölçün.
6. **Ertelenmiş kısıtlar hatayı COMMIT'e taşır.** `SET CONSTRAINTS ALL DEFERRED` sonrası hata satırın hangisi olduğunu göstermez; dairesel yüklemeler dışında kullanmayın.
7. **`REPLICA IDENTITY FULL` DELETE'i pahalılaştırır.** WAL kaydı normalde sadece `xmax` taşır; `FULL` ile tüm eski satır yazılır ([heapam.c:3103-3109](../src/backend/access/heap/heapam.c#L3103-L3109)).
