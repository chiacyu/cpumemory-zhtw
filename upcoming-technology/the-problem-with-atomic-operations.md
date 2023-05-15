# 8.1 原子操作(Atomic Operation)的問題

共享資料的同步化通常透過兩種方式

- 透過系統提供的互斥鎖
- 透過免鎖(lock-free)資料結構

免鎖資料的問題在處理器必須提供原子操作的相關指令，但這種支援往往有限。大多數架構中，僅支援對一個字進行原子性讀寫。有兩種的實現方式（參見 6.4.2 節）：

- 使用比較與交換(CAS)操作
- 使用 LL/SC pair

使用 LL/SC 可以實現 CAS 操作，因此 CAS 操作是大多數原子操作和免鎖結構的基礎。

一些處理器，提供了一套更精細的原子操作。特別是 x86 和 x86-64 有針對特定目的 CAS 操作進行最佳化 。例如：原子地增加某個值到記憶體位置可以使用 CAS 和 LL/SC 操作來實做，但 x86/x86-64 處理器的指令支援更加快速。對於程式撰寫者來說，了解這些操作和可使用的內建函式是重要的。

這兩種架構的特別之處在於具有雙字（double-word）的CAS（DCAS）操作。這對某些應用場景來說非常重要但並非所有應用都如此（參見[5]）。以下以使用 DCAS 的方式來實做一個免鎖陣列的堆疊/後進先出()資料結構。圖 8.1 為使用 gcc 內建函式的實作。

```c
struct elem {
    data_t d;
    struct elem *c;
};

struct elem *top;

void push(struct elem *n) {
    n->c = top;
    top = n;
}

struct elem *pop(void) {
    struct elem *res = top;
    if (res != NULL)
        top = res->c;
        return res;
}
```
圖 8.1 非執行緒安全的後進先出資料結構(LIFO)

這段程式碼顯然不具備執行緒安全。在不同的執行緒中並行存取全域變數 `top` 。元素可能會遺失或已移除的元素可能會突然重新出現。可使用互斥鎖來解決這個問題，但這裡我們試著僅使用原子操作。

修正問題的第一個嘗試是在安裝或移除元素時使用 CAS 操作。修正後的程式碼如圖 8.2 所示。


```c
#define CAS __sync_bool_compare_and_swap
struct elem {
    data_t d;
    struct elem *c;
};
struct elem *top;

void push(struct elem *n) {
    do
    n->c = top;
    while (!CAS(&top, n->c, n));
}

struct elem *pop(void) {
    struct elem *res;
    while ((res = top) != NULL)
        if (CAS(&top, res, res->c))
        break;
    return res;
}
```

圖 8.2 CAS 實作 LIFO


乍看下似乎是個可行方案。除非 `top` 與操作開始時的頂部元素相同，否則永遠不會修改 `top`。但必須考慮並行下的所有可能發生的狀況。最糟糕的狀況，另一個正在處理資料的執行緒可能會被排程器調度，這正是典型的 [ABA](https://en.wikipedia.org/wiki/ABA_problem) 問題。如果在 CAS 操作之前另一個執行緒被調度並執行以下操作：

```c
l = pop()
push(newelem)
push(l)
```

操作結束後原本 LIFO 的頂部元素回到頂部，但第二個元素不同。對第一個執行緒，因為頂部元素沒有變化，CAS 操作會成功。但是 `res->c` 的值不正確，它指向 LIFO 原先的第二個元素而不是 `newelem` 。因此造成這個新元素遺失。在文獻 [10] 中可以找到一些建議來解決這個問題但牽涉到處理器上的特性。具體而言，與 x86 和 x86-64 處理器執行 DCAS 操作的能力有關。圖 8.3 中的程式碼為此例

```c
#define CAS __sync_bool_compare_and_swap
struct elem {
    data_t d;
    struct elem *c;
};

struct lifo {
    struct elem *top;
    size_t gen;
} l;

void push(struct elem *n) {
    struct lifo old, new;
        do {
            old = l;
            new.top = n->c = old.top;
            new.gen = old.gen + 1;
        } while (!CAS(&l, old, new));
}

struct elem *pop(void) {
    struct lifo old, new;
        do {
            old = l;
            if (old.top == NULL) return NULL;
                new.top = old.top->c;
                new.gen = old.gen + 1;
        } while (!CAS(&l, old, new));
            return old.top;
}
```
圖 8.3 雙字 CAS 實作 LIFO


不像先前兩個例子，上例仍是虛擬碼因為 gcc 無法理解 CAS 內在函式中使用資料的方法。在指向 LIFO 頂部資料的指標中增加了一個計數器。在每次操作（ `push` 或 `pop`）時都會改變計數器的數值，這樣就可解決先前提到的 ABA 問題。當第一個執行緒通過實際交換 `top` 指標回復執行時，計數器已增加三次，因此 CAS 操作會失敗，在下一輪執行中，LIFO 的第一和第二個元素將會是正確值且 LIFO 不會被破壞。大功告成！

但這真的是解決方法嗎？ [10] 的作者們確實讓它聽起來還不錯。實際上確實可以用上述方式構建出 LIFO 的資料結構。但從一般情況來看，這種方法與之前的方法一樣注定會失敗。但程式並行的問題依然存在只是發生在不同的地方。假設一個執行緒執行 `pop` 操作，稍後在測試 `old.top == NULL` 後被中斷。第二個執行緒使用 `pop` 操作並取得先前 `LIFO` 第一個元素的所有權。接著執行緒可以對元素進行任何操作，包括更改所有值或甚至在動態分配元素的情況下釋放記憶體。

緊接著第一個執行緒恢復執行。`old` 變數仍存有先前的 LIFO 頂部元素。具體而言 `top` 成員指向第二個執行緒彈出的元素。在 `new.top = old.top->c` 中，第一個執行緒對元素進行參考。但指標內所指向的元素可能已經被釋放，因此該部分的記憶體位址可能無法存取，進而導致行程崩潰。但解決此問題的方法非常昂貴：至少在釋放記憶體之前必須事先驗證沒有任何執行緒在引用該記憶體空間。考慮到免鎖資料結構應該具備快速且更高的並行性，但這些額外要求完全破壞了任何優勢。某些語言通過垃圾回收來解決該問題，但這也有其代價。

對於更複雜的資料結構而言情況通常更糟。上述引用的論文還描述一個 FIFO 的實作（在後續論文中進行了改進）。但這段程式碼也存在相同的問題。因為現有的硬體上（x86、x86-64），CAS 操作僅限於修改兩個在記憶體中連續的字，所以對於其他常見情況幫助不大。例如：無法在雙向鏈結串列中的任意位置進行原子性增加或刪除。

多數情況通常涉及多個記憶體位址，只有在這些位址的值都沒有被同時更改的情況下，整個操作才能成功。這是資料庫處理中常見的概念，也正是其中一個最有希望解決此困境的提案。