### 4.5.3 预分配的源码
这个部分很长, 只选择了核心的逻辑部分. 具体实现需要自行阅读. 实际上具体实现中主要是针对memoryManager 进行加减锁操作而已.
```scala
 /**
   * Unroll the given block in memory safely.
   *
   * The safety of this operation refers to avoiding potential OOM exceptions caused by
   * unrolling the entirety of the block in memory at once. This is achieved by periodically
   * checking whether the memory restrictions for unrolling blocks are still satisfied,
   * stopping immediately if not. This check is a safeguard against the scenario in which
   * there is not enough free memory to accommodate the entirety of a single block.
   *
   * This method returns either an array with the contents of the entire block or an iterator
   * containing the values of the block (if the array would have exceeded available memory).
   */
  def unrollSafely(
      blockId: BlockId,
      values: Iterator[Any],
      droppedBlocks: ArrayBuffer[(BlockId, BlockStatus)])
    : Either[Array[Any], Iterator[Any]] = {
      // Number of elements unrolled so far
    var elementsUnrolled = 0
    // Whether there is still enough memory for us to continue unrolling this block
    var keepUnrolling = true
    // Initial per-task memory to request for unrolling blocks (bytes). Exposed for testing.
    val initialMemoryThreshold = unrollMemoryThreshold
    // How often to check whether we need to request more memory
    val memoryCheckPeriod = 16
    // Memory currently reserved by this task for this particular unrolling operation
    var memoryThreshold = initialMemoryThreshold
    // Memory to request as a multiple of current vector size
    val memoryGrowthFactor = 1.5
    // Keep track of pending unroll memory reserved by this method.
    var pendingMemoryReserved = 0L
    // Underlying vector for unrolling the block
    var vector = new SizeTrackingVector[Any]

    // Request enough memory to begin unrolling
    keepUnrolling = reserveUnrollMemoryForThisTask(blockId, initialMemoryThreshold, droppedBlocks)

  

    // Unroll this block safely, checking whether we have exceeded our threshold periodically
    try {
      while (values.hasNext && keepUnrolling) {
        vector += values.next()
        if (elementsUnrolled % memoryCheckPeriod == 0) {
          // If our vector's size has exceeded the threshold, request more memory
          val currentSize = vector.estimateSize()
          if (currentSize >= memoryThreshold) {
            // 申请1.5倍的内存
          }
        }
        elementsUnrolled += 1
      }

      if (keepUnrolling) {
        // We successfully unrolled the entirety of this block
        Left(vector.toArray)
      } else {
        // We ran out of space while unrolling the values for this block
        logUnrollFailureMessage(blockId, vector.estimateSize())
        Right(vector.iterator ++ values)
      }

    } finally {
      // If we return an array, the values returned here will be cached in `tryToPut` later.
      // In this case, we should release the memory only after we cache the block there.
      if (keepUnrolling) {
        val taskAttemptId = currentTaskAttemptId()
        memoryManager.synchronized {
          // Since we continue to hold onto the array until we actually cache it, we cannot
          // release the unroll memory yet. Instead, we transfer it to pending unroll memory
          // so `tryToPut` can further transfer it to normal storage memory later.
          // TODO: we can probably express this without pending unroll memory (SPARK-10907)
          unrollMemoryMap(taskAttemptId) -= pendingMemoryReserved
          pendingUnrollMemoryMap(taskAttemptId) =
            pendingUnrollMemoryMap.getOrElse(taskAttemptId, 0L) + pendingMemoryReserved
        }
      } else {
        // Otherwise, if we return an iterator, we can only release the unroll memory when
        // the task finishes since we don't know when the iterator will be consumed.
      }
    }
}
```

### 4.5.4 把数据刷进预分配的内存里去
在下刷的时候会二次检查内存状态, 以防止对value的计算错误导致内存不足

```scala
/**
   * Try to put in a set of values, if we can free up enough space. The value should either be
   * an Array if deserialized is true or a ByteBuffer otherwise. Its (possibly estimated) size
   * must also be passed by the caller.
   *
   * `value` will be lazily created. If it cannot be put into MemoryStore or disk, `value` won't be
   * created to avoid OOM since it may be a big ByteBuffer.
   *
   * Synchronize on `memoryManager` to ensure that all the put requests and its associated block
   * dropping is done by only on thread at a time. Otherwise while one thread is dropping
   * blocks to free memory for one block, another thread may use up the freed space for
   * another block.
   *
   * All blocks evicted in the process, if any, will be added to `droppedBlocks`.
   *
   * @return whether put was successful.
   */
  private def tryToPut(
      blockId: BlockId,
      value: () => Any,
      size: Long,
      deserialized: Boolean,
      droppedBlocks: mutable.Buffer[(BlockId, BlockStatus)]): Boolean = {

    /* TODO: Its possible to optimize the locking by locking entries only when selecting blocks
     * to be dropped. Once the to-be-dropped blocks have been selected, and lock on entries has
     * been released, it must be ensured that those to-be-dropped blocks are not double counted
     * for freeing up more space for another block that needs to be put. Only then the actually
     * dropping of blocks (and writing to disk if necessary) can proceed in parallel. */

    memoryManager.synchronized {
      // Note: if we have previously unrolled this block successfully, then pending unroll
      // memory should be non-zero. This is the amount that we already reserved during the
      // unrolling process. In this case, we can just reuse this space to cache our block.
      // The synchronization on `memoryManager` here guarantees that the release and acquire
      // happen atomically. This relies on the assumption that all memory acquisitions are
      // synchronized on the same lock.
      releasePendingUnrollMemoryForThisTask()
      val enoughMemory = memoryManager.acquireStorageMemory(blockId, size, droppedBlocks)
      if (enoughMemory) {
        // We acquired enough memory for the block, so go ahead and put it
        val entry = new MemoryEntry(value(), size, deserialized)
        entries.synchronized {
          entries.put(blockId, entry)
        }
        val valuesOrBytes = if (deserialized) "values" else "bytes"
        logInfo("Block %s stored as %s in memory (estimated size %s, free %s)".format(
          blockId, valuesOrBytes, Utils.bytesToString(size),
          Utils.bytesToString(maxMemory - blocksMemoryUsed)))
      } else {
        // Tell the block manager that we couldn't put it in memory so that it can drop it to
        // disk if the block allows disk storage.
        lazy val data = if (deserialized) {
          Left(value().asInstanceOf[Array[Any]])
        } else {
          Right(value().asInstanceOf[ByteBuffer].duplicate())
        }
        val droppedBlockStatus = blockManager.dropFromMemory(blockId, () => data)
        droppedBlockStatus.foreach { status => droppedBlocks += ((blockId, status)) }
      }
      enoughMemory
    }
  }
```
# 5. 压缩
压缩对于Spark这样的系统来说是必要的, 计算比内存廉价.
spark提供三种压缩方式

官方推荐使用snappy, 但根据**我查的各种资料显示, lz4是性价比最高的**. lz4付出多一点点的计算代价, 获得数倍的压缩比.
```scala
  private val shortCompressionCodecNames = Map(
    "lz4" -> classOf[LZ4CompressionCodec].getName,
    "lzf" -> classOf[LZFCompressionCodec].getName,
    "snappy" -> classOf[SnappyCompressionCodec].getName)

```