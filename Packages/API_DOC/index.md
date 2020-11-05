## API详解

### normalizeVNode
child == null 或者child是布尔值，创建并返回Comment类型VNode。

child为数组类型，创建并返回Fragment类型VNnode，child作为子VNode。

child为其他类型，创建并返回Text类型VNode，child作为子VNode。

#### 语法
> normalizeVNode(child as VNodeChild)

#### 参数（选填）
child，VNodeChild类型

#### 返回值
VNode

### getFallthroughAttrs
遍历attrs，返回，过滤掉键不为'class'或者'style'或者正则匹配为on开头的键值对，并返回。

#### 语法
> getFallthroughAttrs(attrs: Data): Data | undefined

#### 返回值
undefined | Data类型
