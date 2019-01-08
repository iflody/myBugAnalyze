经典 UaF。锁定代码：https://chromium.googlesource.com/chromium/src.git/+/9d04d4a29a4f488d74e04bdc3133db1b72262462/content/browser/indexed_db/indexed_db_database.cc#1500

```c++
void IndexedDBDatabase::OpenCursor(IndexedDBTransaction* transaction, int64_t object_store_id, int64_t index_id, std::unique_ptr<IndexedDBKeyRange> key_range, blink::WebIDBCursorDirection direction, bool key_only, blink::WebIDBTaskType task_type, scoped_refptr<IndexedDBCallbacks> callbacks) {
	DCHECK(transaction);
	IDB_TRACE1("IndexedDBDatabase::OpenCursor", "txn.id", transaction->id());

	if (!ValidateObjectStoreIdAndOptionalIndexId(object_store_id, index_id))
		return;

    std::unique_ptr<OpenCursorOperationParams> params(
    base::MakeUnique<OpenCursorOperationParams>());
    params->object_store_id = object_store_id;
    params->index_id = index_id;
    params->key_range = std::move(key_range);
    params->direction = direction;
    params->cursor_type =
    key_only ? indexed_db::CURSOR_KEY_ONLY : indexed_db::CURSOR_KEY_AND_VALUE;
    params->task_type = task_type;
    params->callbacks = callbacks;
    transaction->ScheduleTask(base::Bind(
    &IndexedDBDatabase::OpenCursorOperation, this, base::Passed(&params)));
}
```

这里做了进一步封装，执行函数 OpenCursorOperation。

https://chromium.googlesource.com/chromium/src.git/+/9d04d4a29a4f488d74e04bdc3133db1b72262462/content/browser/indexed_db/indexed_db_database.cc#1529

```c++
leveldb::Status IndexedDBDatabase::OpenCursorOperation(std::unique_ptr<OpenCursorOperationParams> params, IndexedDBTransaction* transaction) {
    IDB_TRACE1("IndexedDBDatabase::OpenCursorOperation", "txn.id", transaction->id());
    // The frontend has begun indexing, so this pauses the transaction
    // until the indexing is complete. This can't happen any earlier
    // because we don't want to switch to early mode in case multiple
    // indexes are being created in a row, with Put()'s in between.
    if (params->task_type == blink::kWebIDBTaskTypePreemptive)
    	transaction->AddPreemptiveEvent();
    leveldb::Status s = leveldb::Status::OK();
    std::unique_ptr<IndexedDBBackingStore::Cursor> backing_store_cursor;
    if (params->index_id == IndexedDBIndexMetadata::kInvalidId) {
        if (params->cursor_type == indexed_db::CURSOR_KEY_ONLY) {
            DCHECK_EQ(params->task_type, blink::kWebIDBTaskTypeNormal);
            backing_store_cursor = backing_store_->OpenObjectStoreKeyCursor(transaction->BackingStoreTransaction(), id(), params->object_store_id, *params->key_range, params->direction, &s);
        } else {
            backing_store_cursor = backing_store_->OpenObjectStoreCursor(transaction->BackingStoreTransaction(), id(), params->object_store_id, *params->key_range, params->direction, &s);
        }
    } else {
        DCHECK_EQ(params->task_type, blink::kWebIDBTaskTypeNormal);
        if (params->cursor_type == indexed_db::CURSOR_KEY_ONLY) {
            backing_store_cursor = backing_store_->OpenIndexKeyCursor(transaction->BackingStoreTransaction(), id(), params->object_store_id, params->index_id, *params->key_range, params->direction, &s);
        } else {
            backing_store_cursor = backing_store_->OpenIndexCursor(transaction->BackingStoreTransaction(), id(), params->object_store_id, params->index_id, *params->key_range, params->direction, &s);
        }
    }

    if (!s.ok()) {
        DLOG(ERROR) << "Unable to open cursor operation: " << s.ToString();
        return s;
    }

    if (!backing_store_cursor) {
        // Occurs when we've reached the end of cursor's data.
        params->callbacks->OnSuccess(nullptr);
        return s;
    }

    std::unique_ptr<IndexedDBCursor> cursor = base::MakeUnique<IndexedDBCursor>(std::move(backing_store_cursor), params->cursor_type, params->task_type, transaction);
    IndexedDBCursor* cursor_ptr = cursor.get();
    transaction->RegisterOpenCursor(cursor_ptr);
    params->callbacks->OnSuccess(std::move(cursor), cursor_ptr->key(),
    cursor_ptr->primary_key(), cursor_ptr->Value());
    return s;
}
```

其中 **cursor_ptr** 是 raw pointer，被 transaction 持有，实际存在的 cursor 对象却被转移了所有权到回调函数内， **cursor_ptr** 指向 cursor，进回调函数 **OnSuccess** 会发现，实际上还是进一步封装，将其传入 **SendSuccessCursor** 函数中。继续跟进


```c++
void IndexedDBCallbacks::IOThreadHelper::SendSuccessCursor(
    SafeIOThreadCursorWrapper cursor,
    const IndexedDBKey& key,
    const IndexedDBKey& primary_key,
    blink::mojom::IDBValuePtr value,
    const std::vector<IndexedDBBlobInfo>& blob_info) {
  DCHECK_CURRENTLY_ON(BrowserThread::IO);
  if (!callbacks_)
    return;
  if (!dispatcher_host_) {
    OnConnectionError();
    return;
  }
  auto cursor_impl = std::make_unique<CursorImpl>(
      std::move(cursor.cursor_), origin_, dispatcher_host_.get(), idb_runner_);
  if (value && !CreateAllBlobs(blob_info, &value->blob_or_file_info))
    return;
  blink::mojom::IDBCursorAssociatedPtrInfo ptr_info;
  auto request = mojo::MakeRequest(&ptr_info);
  dispatcher_host_->AddCursorBinding(std::move(cursor_impl),
                                     std::move(request));
  callbacks_->SuccessCursor(std::move(ptr_info), key, primary_key,
                            std::move(value));
}
```


漏洞关键出现了，cursor 对象为 **SafeIOThreadCursorWrapper** 类型的对象，其中持有 **unique_ptr** 指针，该指针持有 **IndexedDBCursor** 类型的对象，如果该函数没有执行到

```c++
auto cursor_impl = std::make_unique<CursorImpl>(
      std::move(cursor.cursor_), origin_, dispatcher_host_.get(), idb_runner_);
```

这句就提前 return 的话，cursor 超过这个作用域会析构，其中持有的 unique_ptr 没有转移所有权，其中的 **IndexedDBCursor** 对象也会被析构，还记得吗，现在 transaction 还持有着一个指向该 **IndexedDBCursor** 对象的 raw pointer，后期如 transaction 调用了该对象的话，就会造成 Use After Free。

那么如何让它提前 return 呢？其实就是让该 IO Thread 提前结束即可，刷新页面是一种做法，作者 nw 提到的是刷新页面或者关闭浏览器。

而后续 transaction 关闭时会这样操作：

```c++
void IndexedDBTransaction::CloseOpenCursors() {
  IDB_TRACE1("IndexedDBTransaction::CloseOpenCursors", "txn.id", id());
  for (auto* cursor : open_cursors_)
    cursor->Close();
  open_cursors_.clear();
}
```

这里就调用了前面持有的已经 free 的 **IndexedDBCursor** 类型的指针。