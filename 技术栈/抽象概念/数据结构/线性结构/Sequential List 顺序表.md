# 定义
顺续表也可用“Array-based List”表示。顺序表是用一段**连续的存储单元**（通常为数组）依次存储线性表元素的物理结构，其特点是逻辑相邻的元素在物理位置上也相邻，支持通过下标直接访问（随机访问）。

# 静态分配和动态分配
##  静态顺序表（Static Sequential List）
```c
#define MAX_SIZE 100  // 最大容量固定

typedef struct {
    int data[MAX_SIZE];  // 静态数组存储
    int length;          // 当前元素个数
} StaticSeqList;
```