# 定义
线索二叉树是一种对普通二叉树进行优化的数据结构，通过利用二叉树中的空指针域来存储遍历顺序的前驱或后继信息，从而加快遍历速度并减少空间浪费。
## 基本概念
在n个节点的二叉树中，有**n+1个空指针域**（每个节点有2个指针，共2n个指针，实际使用n-1个指针连接节点）。线索二叉树利用这些空指针来存储遍历顺序的前驱或后继信息。

线索化类型：
- 中序线索二叉树：按照中序遍历顺序添加线索
- 前序线索二叉树：按照前序遍历顺序添加线索
- 后序线索二叉树：按照后序遍历顺序添加线索

## 优点
- 提高遍历效率：
    - 无需使用递归或栈即可实现遍历    
    - 时间复杂度O(n)，空间复杂度O(1)
- 快速查找前驱后继：
    - 可以直接通过线索找到前驱或后继节点
- 节约空间：
    - 利用了原本为NULL的指针域
# 二叉树的线索化

## 中序线索化
### 递归线索化
```c
ThreadNode *pre = NULL;  // 全局变量，指向当前访问节点的前驱

void InThread(ThreadTree p) {
    if (p != NULL) {
        InThread(p->lchild);  // 线索化左子树
        
        if (p->lchild == NULL) {  // 左孩子为空，建立前驱线索
            p->lchild = pre;
            p->ltag = 1;
        }
        if (pre != NULL && pre->rchild == NULL) {  // 前驱的右孩子为空，建立后继线索
            pre->rchild = p;
            pre->rtag = 1;
        }
        pre = p;
        
        InThread(p->rchild);  // 线索化右子树
    }
}
```

### 非递归遍历线索二叉树
```c
void InOrder_Thread(ThreadTree T) {
    ThreadNode *p = T;
    while (p != NULL) {
        // 找到最左下节点
        while (p->ltag == 0) {
            p = p->lchild;
        }
        visit(p);
        
        // 如果有后继线索，直接访问
        while (p->rtag == 1 && p->rchild != NULL) {
            p = p->rchild;
            visit(p);
        }
        
        // 否则转向右子树
        p = p->rchild;
    }
}
```
## 前序线索化

### 递归实现
```c
void PreThreading(ThreadTree p) {
    if (p != NULL) {
        // 处理当前节点
        if (p->lchild == NULL) { // 左孩子为空，建立前驱线索
            p->ltag = 1;
            p->lchild = pre;
        } else {
            p->ltag = 0;
        }
        
        if (pre != NULL && pre->rchild == NULL) { // 前驱的右孩子为空，建立后继线索
            pre->rtag = 1;
            pre->rchild = p;
        } else if (pre != NULL) {
            pre->rtag = 0;
        }
        
        pre = p; // 更新前驱节点
        
        if (p->ltag == 0) { // 只有左孩子是实际子节点时才递归
            PreThreading(p->lchild);
        }
        if (p->rtag == 0) { // 只有右孩子是实际子节点时才递归
            PreThreading(p->rchild);
        }
    }
}
```

### 非递归遍历前序线索二叉树
```c
void PreOrderTraverse_Thr(ThreadTree T) {
    ThreadNode *p = T;
    while (p != NULL) {
        printf("%c ", p->data);
        
        if (p->ltag == 0) { // 有左孩子
            p = p->lchild;
        } else { // 无左孩子，找后继
            p = p->rchild;
        }
    }
}
```

## 后序线索化

### 递归实现

```c
void PostThreading(ThreadTree p) {
    if (p != NULL) {
        PostThreading(p->lchild); // 线索化左子树
        PostThreading(p->rchild); // 线索化右子树
        
        // 处理当前节点
        if (p->lchild == NULL) { // 左孩子为空，建立前驱线索
            p->ltag = 1;
            p->lchild = pre;
        } else {
            p->ltag = 0;
        }
        
        if (pre != NULL && pre->rchild == NULL) { // 前驱的右孩子为空，建立后继线索
            pre->rtag = 1;
            pre->rchild = p;
        } else if (pre != NULL) {
            pre->rtag = 0;
        }
        
        pre = p; // 更新前驱节点
    }
}
```

### 后序线索二叉树的遍历
后序线索二叉树的遍历较为复杂，通常需要借助栈或记录父节点指针：
```c
void PostOrderTraverse_Thr(ThreadTree T) {
    ThreadNode *p = T;
    ThreadNode *lastVisited = NULL;
    
    // 找到第一个要访问的节点（最左下的节点）
    while (p->ltag == 0) {
        p = p->lchild;
    }
    
    while (p != NULL) {
        // 如果右孩子是线索或已访问过
        if (p->rtag == 1 || p->rchild == lastVisited) {
            printf("%c ", p->data);
            lastVisited = p;
            p = p->rchild; // 沿着线索回溯
        } else {
            // 转向右子树，并找到该子树的最左下节点
            p = p->rchild;
            while (p != NULL && p->ltag == 0) {
                p = p->lchild;
            }
        }
    }
}
```


