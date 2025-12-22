# 定义
二叉树是n（$i\geq1$）个结点的有限集合：
- 或者为空二叉树，即n=0。
- 或者由一个根结点和两个互不相交的被称为根的左子树和右子树组成。左子树和右子树又分别是一棵二叉树。
特点：
- 每个结点至多只有两棵子树
- 左右子树不能颠倒（二叉树是有序树）

# 特殊二叉树
## 满二叉树
一棵高度为h，且含有$2^h-1$个结点的二叉树
特点：
- 只有最后一层有叶子结点
- 不存在度为1的结点
- 按层序从1开始编号，结点i的左孩子为2i，右孩子为2i+1；结点i的父节点为Li/2]（如果有的话）
## 完全二叉树
当且仅当每个结点都与高度为h的满二叉树中编号为1~n的结点一一对应时，称为完全二叉树特点：
- 只有最后两层可能有叶子结点
- 最多只有一个度为1的结点
- 按层序从1开始编号，结点i的左孩子为2i，右孩子为2i+1；结点i的父节点为\[i/2\]（如果有的话）
- i≤\[n/2\]为分支结点，i>\[n/2\]为叶子结点

## 二叉排序树
一棵二叉树或者是空二叉树，或者是具有如下性质的二叉树：
- 左子树上所有结点的关键字均小于根结点的关键字；右子树上所有结点的关键字均大于根结点的关键字。
- 左子树和右子树又各是一棵二叉排序树。
二叉排序树可用于元素的排序、搜索

## 平衡二叉树
树上任一结点的左子树与右子树的深度之差不超过1（这样能有更高的搜索效率）。

# 性质
- 性质1：设非空二叉树中度为0、1和2的结点个数分别为 $n_0$、$n_1$和$n_2$，则$n_0=n_2+1$（叶子结点比二分支结点多一个）
- 性质2：二叉树第i层至多有 $2^{i-1}$个结点($i\geq1$)，m叉树第i层至多有 $m^{i-1}$个结点($i\geq1$)
- 性质3：高度为h的二叉树至多有$2^h-1$个结点（满二叉树），高度为h的m叉树至多有$\frac{m^h-1}{m-1}$
## 完全二叉树性质
- 性质1：具有n个（n>0）结点的完全二叉树的高度h为$\ulcorner log_2()n+1 \urcorner$或$\llcorner log_2n\lrcorner+1$
- 性质2：对于完全二叉树，可以由的结点数n推出度为0、1和2的结点个数为$n_0$、$n_1$ 和 $n_2$。完全二叉树最多只有一个度为1的结点，即$n_1$=0或1，$n_0=n_2+1 \rightarrow n_0+n_2$一定是奇数。

# 二叉树的存储
二叉树的存储顺序存储（数组实现）和链式存储（指针/引用实现）的核心区别在于物理存储的连续性和节点关系的表示方法
## 顺序存储
用连续的数组空间存储二叉树节点，通过数组下标隐式表示节点间的父子关系。
存储规则
- 根节点：放在数组索引 `0` 或 `1` 的位置（通常从 `1` 开始计算更直观）。
- 父子关系（假设根节点索引为 `1`）：
    - 父节点 `i` 的左子节点：`2i`
    - 父节点 `i` 的右子节点：`2i + 1`
    - 子节点 `i` 的父节点：`⌊i/2⌋`（向下取整）

| 优点              | 缺点             |
| --------------- | -------------- |
| 无需额外存储指针，节省空间   | 非完全二叉树会导致空间浪费  |
| 随机访问快（通过下标直接定位） | 插入/删除节点需移动大量元素 |
| 适合静态二叉树或堆结构     | 动态扩展数组可能触发拷贝开销 |
## 链式存储
每个节点用结构体/对象表示，通过指针或引用显式维护父子关系。

```c
struct TreeNode {
    int data;           // 节点数据
    TreeNode* left;     // 左子节点指针
    TreeNode* right;    // 右子节点指针
    // 可选：TreeNode* parent; // 父指针（非必需）
};
```

| 优点              | 缺点                |
| --------------- | ----------------- |
| 灵活支持任意树形结构      | 每个节点需额外存储指针，空间开销大 |
| 插入/删除节点高效（O(1)） | 无法随机访问，查找需遍历      |
| 动态扩展无需连续内存      | 指针操作可能引发内存泄漏      |

# 二叉树遍历方式
二叉树有三种主要的遍历方式：前序遍历、后序遍历和层序遍历

| 遍历方式 | 访问顺序      | BST遍历结果 | 表达式树输出 |
| ---- | --------- | ------- | ------ |
| 前序   | 根→左→右     | 无特殊顺序   | 前缀表达式  |
| 中序   | 左→根→右     | 升序序列    | 中缀表达式  |
| 后序   | 左→右→根     | 无特殊顺序   | 后缀表达式  |
| 层序   | 从上到下，从左到右 | 无特殊顺序   | 无直接对应  |
## 前序遍历
**根节点 → 左子树 → 右子树**
- 第一个访问的总是根节点
- 适合用于复制二叉树结构
- 表达式树的前序遍历得到前缀表达式（波兰表示法）

代码实现：
```c
void PreOrder(BiTree T) {
    if (T != NULL) {
        visit(T);             // 访问根节点
        PreOrder(T->lchild);  // 遍历左子树
        PreOrder(T->rchild);  // 遍历右子树
    }
}
```

## 中序遍历
**左子树 → 根节点 → 右子树**
- 对二叉搜索树(BST)进行中序遍历，得到的是升序排列的元素序列。对于表达式树，中序遍历可以得到中缀表达式
- 遍历结果中，根节点位于左右子树结果之间
- 可以清晰看出左右子树的划分

代码实现：
```c
void InOrder(BiTree T) {
    if (T != NULL) {
        InOrder(T->lchild);  // 遍历左子树
        visit(T);            // 访问根节点
        InOrder(T->rchild);  // 遍历右子树
    }
}
```
## 后序遍历
**左子树 → 右子树 → 根节点**
- 最后一个访问的总是根节点
- 适合用于删除或释放二叉树
- 表达式树的后序遍历得到后缀表达式（逆波兰表示法）

代码实现：
```c
void PostOrder(BiTree T) {
    if (T != NULL) {
        PostOrder(T->lchild); // 遍历左子树
        PostOrder(T->rchild);  // 遍历右子树
        visit(T);             // 访问根节点
    }
}
```
## 层序遍历
**从上到下，从左到右，逐层访问**
- 按树的层级顺序访问
- 需要额外的队列数据结构辅助
- 适合用于计算树的深度、查找特定层级的节点等

代码实现：
```c
void LevelOrder(BiTree T) {
    LinkQueue Q;
    InitQueue(&Q);
    BiTree p;
    
    if (T != NULL) {
        EnQueue(&Q, T);
    }
    
    while (!IsEmpty(Q)) {
        DeQueue(&Q, &p);
        visit(p);
        
        if (p->lchild != NULL) {
            EnQueue(&Q, p->lchild);
        }
        if (p->rchild != NULL) {
            EnQueue(&Q, p->rchild);
        }
    }
}
```

### 完整层序遍历代码实现

```c
// 二叉树的结点（链式存储）
typedef struct BiTNode {
    char data;
    struct BiTNode *lchild, *rchild;
} BiTNode, *BiTree;

// 链式队列结点
typedef struct LinkNode {
    BiTNode *data;
    struct LinkNode *next;
} LinkNode;

// 链式队列
typedef struct {
    LinkNode *front, *rear;
} LinkQueue;

// 初始化队列
void InitQueue(LinkQueue *Q) {
    Q->front = Q->rear = NULL;
}

// 判断队列是否为空
int IsEmpty(LinkQueue Q) {
    return Q.front == NULL;
}

// 入队操作
void EnQueue(LinkQueue *Q, BiTree p) {
    LinkNode *newNode = (LinkNode *)malloc(sizeof(LinkNode));
    newNode->data = p;
    newNode->next = NULL;
    
    if (IsEmpty(*Q)) {
        Q->front = Q->rear = newNode;
    } else {
        Q->rear->next = newNode;
        Q->rear = newNode;
    }
}

// 出队操作
void DeQueue(LinkQueue *Q, BiTree *p) {
    if (IsEmpty(*Q)) {
        *p = NULL;
        return;
    }
    
    LinkNode *temp = Q->front;
    *p = temp->data;
    Q->front = temp->next;
    
    if (Q->front == NULL) {
        Q->rear = NULL;
    }
    
    free(temp);
}

// 访问结点
void visit(BiTree p) {
    printf("%c ", p->data);
}

// 层序遍历
void LevelOrder(BiTree T) {
    LinkQueue Q;
    InitQueue(&Q);
    BiTree p;
    
    if (T != NULL) {
        EnQueue(&Q, T);
    }
    
    while (!IsEmpty(Q)) {
        DeQueue(&Q, &p);
        visit(p);
        
        if (p->lchild != NULL) {
            EnQueue(&Q, p->lchild); // 左孩子入队
        }
        if (p->rchild != NULL) {
            EnQueue(&Q, p->rchild); // 右孩子入队
        }
    }
}

// 创建二叉树结点
BiTree CreateNode(char data) {
    BiTree newNode = (BiTree)malloc(sizeof(BiTNode));
    newNode->data = data;
    newNode->lchild = NULL;
    newNode->rchild = NULL;
    return newNode;
}
```