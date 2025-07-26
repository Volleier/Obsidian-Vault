# 定义
**静态链表（Static Linked List）** 是用 **数组模拟链表** 的一种数据结构，通过数组下标代替指针实现节点的逻辑连接。它结合了顺序表和链表的特性：
- 物理结构：数据存储在连续数组中（静态分配）。
- 逻辑结构：通过数组元素的“游标”（`cursor`，类似指针）链接成链式结构。

| 优点                     | 缺点              |
| ---------------------- | --------------- |
| 无动态内存分配，避免内存泄漏风险。      | 容量固定，无法动态扩展。    |
| 插入/删除快（_O(1)_，无需移动元素）。 | 需要手动管理备用链，逻辑复杂。 |
| 适合嵌入式等资源受限环境。          | 随机访问效率低（需遍历）。   |
# 操作
##  初始化
```c
void InitList(StaticLinkedList *list) {
    // 初始化所有节点为备用链
    for (int i = 0; i < MAX_SIZE - 1; i++) {
        list->nodes[i].next = i + 1;  // 每个节点指向下一个
    }
    list->nodes[MAX_SIZE - 1].next = -1;  // 末尾节点指向空
    list->head = -1;                      // 主链初始为空
    list->free = 0;                       // 备用链从头开始
}
```

## 分配空闲节点（类似malloc）
```c
int MallocNode(StaticLinkedList *list) {
    if (list->free == -1) return -1;  // 无空闲节点
    int allocated = list->free;       // 分配备用链的第一个节点
    list->free = list->nodes[allocated].next;  // 更新备用链头
    return allocated;                 // 返回分配的节点下标
}
```

## 插入节点（在位置i后插入e）
```c
void Insert(StaticLinkedList *list, int i, int e) {
    int newNode = MallocNode(list);   // 申请新节点
    if (newNode == -1) return;       // 空间不足
    list->nodes[newNode].data = e;   // 写入数据

    // 找到插入位置的前驱节点
    int prev = i;
    list->nodes[newNode].next = list->nodes[prev].next;
    list->nodes[prev].next = newNode;
}
```

##  删除节点（删除位置i的节点）
```c
void Delete(StaticLinkedList *list, int i) {
    int prev = FindPrev(list, i);    // 找到前驱节点（需自定义函数）
    int toDelete = list->nodes[prev].next;
    list->nodes[prev].next = list->nodes[toDelete].next;

    // 释放节点到备用链
    list->nodes[toDelete].next = list->free;
    list->free = toDelete;
}
```