# 定义
顺续表也可用“Array-based List”表示。顺序表是用一段**连续的存储单元**（通常为数组）依次存储线性表元素的物理结构，其特点是逻辑相邻的元素在物理位置上也相邻，支持通过下标直接访问（随机访问）。

# 静态分配和动态分配
| 特性   | 静态顺序表           | 动态顺序表        |
| ---- | --------------- | ------------ |
| 存储方式 | 固定大小数组          | 动态分配数组       |
| 容量   | 编译时确定，不可变       | 运行时可变（可扩容）   |
| 内存管理 | 自动分配/释放（栈或全局内存） | 需手动管理（堆内存）   |
| 灵活性  | 低（可能浪费或不足）      | 高（按需调整）      |
| 适用场景 | 数据量固定的简单应用      | 数据量变化较大的复杂应用 |

##  静态顺序表（Static Sequential List）
使用**固定大小的数组**存储数据，容量在编译时确定，无法动态扩展。适用于数据量已知且固定，不需要频繁插入/删除。
```c
#define MAX_SIZE 100  // 最大容量固定

typedef struct {
    int data[MAX_SIZE];  // 静态数组存储
    int length;          // 当前元素个数
} StaticSeqList;
```

## 动态顺序表（Dynamic Sequential List）
使用**动态分配的数组**，可随数据增长扩容。适用于数据量不确定，需要动态调整容量。
```c
typedef struct {
    int *data;      // 动态数组指针
    int length;     // 当前元素个数
    int capacity;   // 当前总容量
} DynamicSeqList;

// 初始化（初始容量为initCapacity）
void InitList(DynamicSeqList *L, int initCapacity) {
    L->data = (int *)malloc(initCapacity * sizeof(int));
    L->length = 0;
    L->capacity = initCapacity;
}

// 扩容（新容量为newCapacity）
void Expand(DynamicSeqList *L, int newCapacity) {
    int *newData = (int *)realloc(L->data, newCapacity * sizeof(int));
    if (newData) {
        L->data = newData;
        L->capacity = newCapacity;
    } else {
        printf("Expand failed!\n");
        exit(1);
    }
}
```