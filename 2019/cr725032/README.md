# Cr725032

## Vulnerability Url

https://bugs.chromium.org/p/chromium/issues/detail?id=725032

## Vulnerability Detail

又一经典 UaF。就是 ned 说的有点模糊，分析了一下，首先是创建事务的过程中，会将指针保存在两个地方，看一下流程：

<https://chromium.googlesource.com/chromium/src.git/+/366c8bf9d9543605d97f63865b38b898876f2d22/content/browser/indexed_db/indexed_db_database.cc>

```c++

IndexedDBTransaction* IndexedDBDatabase::CreateTransaction(int64_t transaction_id, IndexedDBConnection* connection, const std::vector<int64_t>& object_store_ids, blink::mojom::IDBTransactionMode mode) {
  		IDB_TRACE1("IndexedDBDatabase::CreateTransaction", "txn.id", transaction_id);
  		DCHECK(connections_.count(connection));
  		UMA_HISTOGRAM_COUNTS_1000("WebCore.IndexedDB.Database.OutstandingTransactionCount", transaction_count_);
  		IndexedDBTransaction* transaction = connection->CreateTransaction(
        transaction_id,
        std::set<int64_t>(object_store_ids.begin(), object_store_ids.end()), mode,
        new IndexedDBBackingStore::Transaction(backing_store_.get()));
        TransactionCreated(transaction);
        return transaction;
}
```

transaction 是一个 raw_pointer，我们跟进 **connection->CreateTransaction** 看看：

```c++
IndexedDBTransaction* IndexedDBConnection::CreateTransaction(int64_t id, const std::set<int64_t>& scope, blink::mojom::IDBTransactionMode mode, IndexedDBBackingStore::Transaction* backing_store_transaction) {
  DCHECK_EQ(GetTransaction(id), nullptr) << "Duplicate transaction id." << id;
  std::unique_ptr<IndexedDBTransaction> transaction = IndexedDBClassFactory::Get()->CreateIndexedDBTransaction(
          id, this, scope, mode, backing_store_transaction);
  IndexedDBTransaction* transaction_ptr = transaction.get();
  transactions_[id] = std::move(transaction);
  return transaction_ptr;
}
```


看到 **transaction** 被保存在 **unique_ptr** 中，之后再放到 **IndexedDBConnection** 持有的 **transactions_** 这个 **map** 中， **key** 为事务的 **id**， **value** 是事务对象。这里的对象是智能指针 **unique_ptr** 持有的。

回到上一层，我们跟到 **TransactionCreated** 中，可以看到：

```c++
void IndexedDBDatabase::TransactionCreated(IndexedDBTransaction* transaction) {
  transaction_count_++;
  transaction_coordinator_.DidCreateTransaction(transaction);
}
```

跟入 **DidCreateTransaction** 中。

```c++
void IndexedDBTransactionCoordinator::DidCreateTransaction(IndexedDBTransaction* transaction) {
  DCHECK(!queued_transactions_.count(transaction));
  DCHECK(!started_transactions_.count(transaction));
  DCHECK_EQ(IndexedDBTransaction::CREATED, transaction->state());
  queued_transactions_.insert(transaction);
  ProcessQueuedTransactions();
}
```

看到最终 transaction raw pointer 被插入到 **queued_transactions_** 这个 list 中。

我们可以思考一下，如果连续创建两个 id 为 1 的事务，会发生什么？

**transactions_** 这个 map 中最终只保留了一个 id 为 1 的事务对象，其中一个被覆盖掉，随后析构，但是 **queued_transactions_** 这个 list 中却会有两个指针，其中一个将指向已经析构的对象，随后若使用的话，将造成 UaF。比如这个函数：

```c++
void IndexedDBTransactionCoordinator::ProcessQueuedTransactions() {
  if (queued_transactions_.empty())
    return;
  DCHECK(!IsRunningVersionChangeTransaction());
  // The locked_scope set accumulates the ids of object stores in the scope of
  // running read/write transactions. Other read-write transactions with
  // stores in this set may not be started. Read-only transactions may start,
  // taking a snapshot of the database, which does not include uncommitted
  // data. ("Version change" transactions are exclusive, but handled by the
  // connection sequencing in IndexedDBDatabase.)
  std::set<int64_t> locked_scope;
  for (auto* transaction : started_transactions_) {
    if (transaction->mode() == blink::mojom::IDBTransactionMode::ReadWrite) {
      // Started read/write transactions have exclusive access to the object
      // stores within their scopes.
      locked_scope.insert(transaction->scope().begin(),
                          transaction->scope().end());
    }
  }
  auto it = queued_transactions_.begin();
  while (it != queued_transactions_.end()) {
    IndexedDBTransaction* transaction = *it;
    ++it;
    if (CanStartTransaction(transaction, locked_scope)) {
      DCHECK_EQ(IndexedDBTransaction::CREATED, transaction->state());
      queued_transactions_.erase(transaction);
      started_transactions_.insert(transaction);
      transaction->Start();
      DCHECK_EQ(IndexedDBTransaction::STARTED, transaction->state());
    }
    if (transaction->mode() == blink::mojom::IDBTransactionMode::ReadWrite) {
      // Either the transaction started, so it has exclusive access to the
      // stores in its scope, or per the spec the transaction which was
      // created first must get access first, so the stores are also locked.
      locked_scope.insert(transaction->scope().begin(),
                          transaction->scope().end());
    }
  }
  RecordMetrics();
}
```

这里循环就会处理 queued_transactions_ 中的事务，造成 UaF。
