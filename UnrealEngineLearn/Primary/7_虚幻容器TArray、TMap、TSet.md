`TArray`、`TMap` 和 `TSet` 都是虚幻引擎中的模板容器类，它们用于存储和管理数据。`TArray` 是一个有序的元素列表，`TMap` 是键值对的集合，`TSet` 是唯一元素的集合
以下是它们的一些共同点：

1. **模板类**：它们都是模板类，可以用于存储任何类型的数据。例如，你可以有一个 `TArray<int32>` 来存储整数，或者一个 `TMap<FString, UObject*>` 来存储字符串到对象的映射
2. **动态大小**：它们的大小是动态的，可以在运行时添加或删除元素
3. **内存管理**：它们都提供了内存管理功能，当添加或删除元素时，它们会自动处理内存的分配和释放
4. **迭代器支持**：它们都支持迭代器，可以使用范围基于的 for 循环来遍历容器中的所有元素
5. **虚幻引擎特性**：它们都支持虚幻引擎的一些特性，如垃圾回收和序列化

下面是打印TArray、TMap、TSet的方法：
```cpp
//TArray
for(auto It = MyArray.CreateConstIterator();It;It++)
{
	UE_LOG(LogTemp,display,TEXT("%d"),*It)
}

//TMap
for(auto& PrintMap:MyMap)
{
	UE_LOGUE_LOG(LogTemp,display,TEXT("Key:%d" "Value:%d"),PrintMap.Key,PrintMap.Value);
}

//TSet
for(auto& PrintSets:MySet)
{
	UE_LOG(LogTemp,display,TEXT("%s"),*TestSet);
}
```
## TArray
在虚幻引擎中，`TArray` 是一个模板类，用于表示动态数组。它类似于 C++ 的 `std::vector`，可以存储任何类型的元素，并且可以动态地增加和减少元素。

以下是 `TArray` 的一些主要方法：
- `Add(Element)`: 向数组的末尾添加一个元素。
- `AddUnique(Element)`: 如果元素尚未存在于数组中，则将其添加到数组的末尾。
- `Remove(Element)`: 从数组中删除一个元素。
- `Empty()`: 清空数组。
- `Num()`: 返回数组中的元素数量。
- `Sort()`: 对数组进行排序。
- `Contains(Element)`: 检查数组是否包含特定元素

### 创建TArray
```cpp
TArrat<int32>MyArray;

//Add Element
MyArray.Add(10);
```

## TMap
`TMap` 是虚幻引擎中的一个模板类，用于表示键值对的集合，类似于 C++ 的 `std::map` 或 `std::unordered_map`。`TMap` 中的每个元素都由一个键`Key`和一个值`Value`组成，键是唯一的，可以用来快速查找对应的值。

以下是 `TMap` 的一些主要方法：
- `Add(Key, Value)`: 向映射中添加一个键值对，`Add` 需要先创建键值对然后再插入
- `Emplace(Key,value)`：向映射中添加一个键值对，`Emplace`不需要先创建键值对然后再插入，可以避免不必要的对象复制
- `Remove(Key)`: 从映射中删除一个键值对
- `Empty`：清空数据
- `Contains(Key)`: 检查映射是否包含特定的键，进行两次查找
- `Find(Key)`: 查找特定的键，进行一次查找，并返回对应的值的指针。如果找不到键，则返回 `nullptr`
- `FindKey(value)`：查找特定的值，并返回对应的键的指针。如果找不到键，则返回 `nullptr`

### 创建`TMap`:
```cpp
TMap<FString, int32> MyMap;

// Add key-value pair
MyMap.Add(TEXT("KeyOne"), var1);
```
### 获取键和值
```cpp
TArray<int32> TestKey;
TArrat<int32> TestValues;
MyMap.GenerateKeyArray(TestKey);
MyMap.GenerateValueArray(TestValue);
```
`TestKey`和`TestValue`中分别储存着键和值的数据

## TSet
`TSet` 是虚幻引擎中的一个模板类，用于表示一组唯一的元素。它类似于 C++ 的 `std::set` 或 `std::unordered_set`，但提供了一些额外的功能，如内存管理和序列化，使其更适合用于虚幻引擎的游戏开发。
`TSet`是一种快速容器类，（通常）用于在排序不重要的情况下储存唯一元素。`TSet`可以非常快速添加、查找、删除元素（恒定时间）， `-TSet`也是值类型，支持常规复制、复制和析构函数操作，以及其元素较强的所有权
`TSet` 类似于[TMap](7_虚幻容器TArray、TMap、TSet.md##TMap)和`TMultiMap`，但有一个重要的区别：`TSet`是通过元素求值的可覆盖函数，使用该数据本身作为键，而不是将数据值与独立的键相关联

`TSet`的常见方法：
- `Add(Element)`: 向集合中添加一个元素。如果元素已经存在于集合中，则不会添加
- `Append(TMap)`：向集合中添加传入的集合的所有元素，如果有任何重复的元素，它们将被忽略
- `Emplace(Element)`：向集合中添加一个元素。可以在集合中直接构造元素，而不需要先构造对象
- `Remove(Element)`: 从集合中删除一个元素
- `Contains(Element)`: 检查集合是否包含特定元素，找到返回True
- `Find(Element)`：检查元素是否包含特定的元素，找到返回指向元素数值的指针
- `Num()`: 返回集合中的元素数量
- `Empty()`: 清空集合并且删除内存
- `Reset()`：清空集合但是保留内存
- `Array()`：函数会返回一个TArray，其中填充了TSet每一个元素的副本
- `Sort([](ConditionInput){Condition})`：排序TSet
- `Reserve(num)`：预先分配内存，诺输入的num大于元素的个数，则会产生闲置内存(Slack)
- `Shrink`：删除TSet中末端的空元素
- `Compact`：将容器中所有空白元素集合到一起放在末尾一起删除。Compact可能会导致该改变元素之间的顺序
- `CompactStable`：将将容器中所有空白元素集合到一起放在末尾一起删除。CompactStable不会导致该改变元素之间的顺序
- 
### 创建TMap
```cpp
TSet<FString>MySet

//Add variable
MySet.Add(TEXT("One"));
MySet.Add(TEXT("TWO"));
```

### 索引
```cpp
FSetElementID Index = Set("Two");
Set[index] = TEXT("One");
```