# bluestore 状态机源码注释

```c++
int BlueStore::queue_transactions(
  CollectionHandle& ch,
  vector<Transaction>& tls,
  TrackedOpRef op,
  ThreadPool::TPHandle *handle)
{
  FUNCTRACE(cct);
  list<Context *> on_applied, on_commit, on_applied_sync;
  ObjectStore::Transaction::collect_contexts(
    tls, &on_applied, &on_commit, &on_applied_sync);

  if (cct->_conf->objectstore_blackhole) {
    dout(0) << __func__ << " objectstore_blackhole = TRUE, dropping transaction"
	    << dendl;
    for (auto& l : { on_applied, on_commit, on_applied_sync }) {
      for (auto c : l) {
	delete c;
      }
    }
    return 0;
  }
  utime_t start = ceph_clock_now();

  Collection *c = static_cast<Collection*>(ch.get());
  OpSequencer *osr = c->osr.get();
  dout(10) << __func__ << " ch " << c << " " << c->cid << dendl;

  // prepare
  TransContext *txc = _txc_create(static_cast<Collection*>(ch.get()), osr,
				  &on_commit);

  for (vector<Transaction>::iterator p = tls.begin(); p != tls.end(); ++p) {
    txc->bytes += (*p).get_num_bytes();
    _txc_add_transaction(txc, &(*p));
  }
  _txc_calc_cost(txc);

  _txc_write_nodes(txc, txc->t);

  // journal deferred items
  if (txc->deferred_txn) {
    txc->deferred_txn->seq = ++deferred_seq;
    bufferlist bl;
    encode(*txc->deferred_txn, bl);
    string key;
    get_deferred_key(txc->deferred_txn->seq, &key);
    txc->t->set(PREFIX_DEFERRED, key, bl);
  }

  _txc_finalize_kv(txc, txc->t);
  if (handle)
    handle->suspend_tp_timeout();

  utime_t tstart = ceph_clock_now();
  throttle_bytes.get(txc->cost);
  if (txc->deferred_txn) {
    // ensure we do not block here because of deferred writes
    if (!throttle_deferred_bytes.get_or_fail(txc->cost)) {
      dout(10) << __func__ << " failed get throttle_deferred_bytes, aggressive"
	       << dendl;
      ++deferred_aggressive;
      deferred_try_submit();
      {
	// wake up any previously finished deferred events
	std::lock_guard<std::mutex> l(kv_lock);
	kv_cond.notify_one();
      }
      throttle_deferred_bytes.get(txc->cost);
      --deferred_aggressive;
   }
  }
  utime_t tend = ceph_clock_now();

  if (handle)
    handle->reset_tp_timeout();

  logger->inc(l_bluestore_txc);

  // execute (start)
  _txc_state_proc(txc);

  // we're immediately readable (unlike FileStore)
  for (auto c : on_applied_sync) {
    c->complete(0);
  }
  for (auto c : on_applied) {
    finishers[osr->shard]->queue(c);
  }

  logger->tinc(l_bluestore_submit_lat, ceph_clock_now() - start);
  logger->tinc(l_bluestore_throttle_lat, tend - tstart);
  return 0;
}
```

