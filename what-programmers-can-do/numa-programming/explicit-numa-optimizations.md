# 6.5.7. 明確的 NUMA 最佳化

假如所有節點上的所有執行緒都需要存取相同的記憶體區域時，所有的本地記憶體與親和性規則都無法幫上忙。當然，簡單地將執行緒的數量限制為直接連接到記憶體節點的處理器所能支援的數量是可能的。不過，這並沒有善用 SMP NUMA 機器，因此並不是個實際的選項。

若是所需的資料是唯讀的，有個簡單的解法：複製（replication）。每個節點能夠得到它自己的資料副本，這樣就不必進行節點之間的存取了。這樣做的程式碼看起來可能像這樣：

```c
void *local_data(void) {
  static void *data[NNODES];
  int node =
    NUMA_memnode_self_current_idx();
  if (node == -1)
    /* Cannot get node, pick one. */
    node = 0;
  if (data[node] == NULL)
    data[node] = allocate_data();
  return data[node];
}
void worker(void) {
  void *data = local_data();
  for (...)
    compute using data
}
```

在這段程式中，函數 `worker` 藉由一次 `local_data` 的呼叫來取得一個資料的本地副本的指標來進行準備。接著它繼續執行使用了這個指標的迴圈。`local_data` 函數保存一個已經被分配的資料副本的列表。每個系統擁有有限的記憶體節點，所以帶有各節點記憶體副本的指標的陣列大小是受限的。來自 libNUMA 的 `NUMA_memnode_system_count` 函數回傳了這個數字。若是給定節點的記憶體還沒被分配給當前節點（由 `data` 在 `NUMA_memnode_self_current_idx` 呼叫所回傳的索引位置的空指標來識別），就分配一個新的副本。

重要的是要意識到，如果在 `getcpu` 系統呼叫之後，執行緒被排程在另一個連接到不同記憶體節點的 CPU 上時，是不會發生什麼可怕的事情的。[^43]它只代表在 `worker` 中使用 `data` 變數的存取，存取了另一個記憶體節點上的記憶體。這直到 `data` 被重新計算為止會拖慢程式，但就這樣了。系統核心總是會避免不必要的、每顆 CPU 執行佇列的重新平衡。若是這種轉移發生了，通常是為了一個很好的理由，並且在不久的未來不會再次發生。

當處理中的記憶體區域是可寫入的，事情會更為複雜。在這種情況下，單純的複製是行不通的。根據具體情況，或許有一些可能的解法。

舉例來說，若是可寫入的記憶體區域是用來累積（accumulate）結果的，首先為結果被累積的每個記憶體節點建立一個分離的區域。接著，當這項工作完成時，所有的每節點的記憶體區域會被結合以得到全體的結果。即使運作從不真的停止，這項技術也能行得通，但中介結果是必要的。這個方法的必要條件是，結果的累積是無狀態的（stateless）。即，它不會依賴先前收集起來的結果。

不過，擁有對可寫入記憶體區域的直接存取總是比較好的。若是對記憶體區域的存取數量很可觀，那麼強迫系統核心將處理中的記憶體分頁遷移到本地節點可能是個好點子。若是存取的數量非常高，並且在不同節點上的寫入並未同時發生，這可能有幫助。但要留意，系統核心無法產生奇蹟：分頁遷移是一個複製操作，並且以此而論並不便宜。這個成本必須被分期償還。



[^43]: 使用者層級的 `sched_getcpu` 介面是使用 `getcpu` 系統呼叫來實作的。後者不該被直接使用，並且有一個不同的介面。
