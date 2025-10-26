```
// 单线程双缓冲路径+Σ增量更新伪代码
// 约定：root.parent == nil，root.depth == 0

type Node struct {
    id       int32
    parent   *Node
    depth    int32
    sigma    T          // 随节点携带的附加数据
}

type PathBuffer struct {
    nodes  []int32      // 根→末端的有序节点id
    sigmas []T          // 与nodes一一对应的sigma值
}

var cache struct {
    main PathBuffer      // 最近一次执行的完整路径
    alt  PathBuffer      // 次近一次（备用链）
    tail *Node           // main对应的末端节点
    altT *Node           // alt对应的末端节点
}

// 入口：每次执行前调用，返回当前节点cur的path与sigma序列
func getPath(cur *Node) (path []int32, sigma []T) {
    // 1. 先尝试主缓存
    lca := LCA(cur, cache.tail)
    if lca != nil {                      // 主缓存命中
        reuse := lca.depth + 1           // 可复用长度
        if reuse <= cache.main.len {
            // 直接截断
            cache.main.nodes = cache.main.nodes[:reuse]
            cache.main.sigmas = cache.main.sigmas[:reuse]
            appendTail(&cache.main, cur, lca) // 补cur→lca.parent段
            return cache.main.nodes, cache.main.sigmas
        }
    }

    // 2. 主缓存未命中，试备用缓存
    lca = LCA(cur, cache.altT)
    if lca != nil {                      // 备用缓存命中
        reuse := lca.depth + 1
        if reuse <= cache.alt.len {
            // 把备用链swap到主位
            swapBuffers()
            cache.main.nodes = cache.main.nodes[:reuse]
            cache.main.sigmas = cache.main.sigmas[:reuse]
            appendTail(&cache.main, cur, lca)
            return cache.main.nodes, cache.main.sigmas
        }
    }

    // 3. 双缓存均未命中，全量重建
    rebuildFull(cur)
    return cache.main.nodes, cache.main.sigmas
}

// ---------- 子过程 ----------

// 计算两个节点的LCA（单线程，无锁）
func LCA(a, b *Node) *Node {
    if a == b { return a }
    // 先拉到同一深度
    for a.depth > b.depth { a = a.parent }
    for b.depth > a.depth { b = b.parent }
    // 再一起向上
    for a != b {
        a = a.parent
        b = b.parent
    }
    return a
}

// 把src缓存与alt缓存互换（仅交换切片头，不复制数据）
func swapBuffers() {
    cache.main, cache.alt = cache.alt, cache.main
    cache.tail, cache.altT = cache.altT, cache.tail
}

// 从已有尾部补全到cur（逆序填充）
func appendTail(pb *PathBuffer, cur, lca *Node) {
    // 先算需要新增多少级
    add := cur.depth - lca.depth
    // 保证容量
    need := len(pb.nodes) + add
    if cap(pb.nodes) < need {
        // 按2倍扩容
        newNodes  := make([]int32, len(pb.nodes), need*2)
        newSigmas := make([]T,    len(pb.sigmas), need*2)
        copy(newNodes,  pb.nodes)
        copy(newSigmas, pb.sigmas)
        pb.nodes, pb.sigmas = newNodes, newSigmas
    }
    // 逆序写新增段
    tmp := cur
    for i := len(pb.nodes) + add - 1; i >= len(pb.nodes); i-- {
        pb.nodes[i]  = tmp.id
        pb.sigmas[i] = tmp.sigma
        tmp = tmp.parent
    }
    pb.nodes  = pb.nodes[:need]
    pb.sigmas = pb.sigmas[:need]
    // 更新主尾指针
    cache.tail = cur
}

// 全量重建（罕见路径）
func rebuildFull(cur *Node) {
    d := cur.depth + 1
    // 复用主缓存的底层数组
    if cap(cache.main.nodes) < d {
        cache.main.nodes  = make([]int32, d, d+32)
        cache.main.sigmas = make([]T,    d, d+32)
    }
    cache.main.nodes  = cache.main.nodes[:d]
    cache.main.sigmas = cache.main.sigmas[:d]
    // 一次性倒填
    tmp := cur
    for i := d - 1; i >= 0; i-- {
        cache.main.nodes[i]  = tmp.id
        cache.main.sigmas[i] = tmp.sigma
        tmp = tmp.parent
    }
    // 把旧主链丢到备用位
    cache.alt, cache.main = cache.main, cache.alt
    cache.altT = cache.tail
    cache.tail = cur
}
```