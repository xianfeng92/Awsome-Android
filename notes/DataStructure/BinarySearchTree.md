# Binary Search Tree

二叉查找树（Binary Search Tree），它是一种数据结构，支持多种动态集合操作，如 Search、Insert、Delete、Minimum 和 Maximum 等。

二叉查找树要么是一棵空树，要么是一棵具有如下性质的非空二叉树:

1. 若左子树非空，则左子树上的所有结点的关键字值均小于根结点的关键字值。

2. 若右子树非空，则右子树上的所有结点的关键字值均大于根结点的关键字值。

3. 左、右子树本身也分别是一棵二叉查找树（二叉排序树）。

可以看出，二叉查找树是一个递归的数据结构，且对二叉查找树进行中序遍历，可以得到一个递增的有序序列。

首先，我们来定义一下 BST 的结点结构体:

```
     class Node {
        public E e;
        public Node left, right;

        public Node(E e) {
            this.e = e;
            left = null;
            right = null;
        }
    }
```

## BST 的插入

二叉查找树作为一种动态结构，其特点是树的结构通常不是一次生成的，而是在查找过程中，当树中不存在结点的关键字等于给定值时再进行插入。
由于二叉查找树是递归定义的，插入结点的过程是：若原二叉查找树为空，则直接插入；否则，若关键字 k 小于根结点关键字，则插入到左子树中，
若关键字 k 大于根结点关键字，则插入到右子树中。

```
    //向以 node 为根的二分搜索树中插入元素 e，递归算法
    private void add(Node node, E e){
        // 已存在插入值的情况，则直接返回
        if(e.equals(node.e))
            return;
        // node 结点的左子树为空时，构造新结点，并直接挂载到 node.left
        else if(e.compareTo(node.e) < 0 && node.left == null){
            node.left = new Node(e);
            size ++;
            return;
        }// node 结点的右子树为空时，构造新结点，并直接挂载到 node.right
        else if(e.compareTo(node.e) > 0 && node.right == null){
            node.right = new Node(e);
            size ++;
            return;
        }
        // 递归
        if(e.compareTo(node.e) < 0)
            add(node.left, e);
        else //e.compareTo(node.e) > 0
            add(node.right, e);
    }
```
向以node为根的二分搜索树中插入元素e过程如下：

1. 如果 e 的值刚好为根结点 node.e 的值，则直接返回

2. 如果 e 的值小于根结点 node.e 的值 & node 的左子树为空，则直接用 e 构造一个新结点挂载到 node 的左孩子上。

3. 如果 e 的值大于根结点 node.e 的值 & node 的右子树为空，则直接用 e 构造一个新结点挂载到 node 的右孩子上。

4. 以上条件的都不满足时：当 e 的值小于根结点 node.e 的值，以 node.left 为根结点进行递归调用。否则以及 node.right 为根结点进行递归调用。



## BST的查找

对于二叉查找树，最常见的操作就是查找树中的某个关键字。除了 Search 操作外，二叉查找树还能支持如 Minimum（最小值）、Maximum（最大值）、Predecessor（前驱）、
Successor（后继）等查询。对于高度为 n 的树，这些操作都可以在 Θ(n) 时间内完成。

### 查找

```
    // 以 node 为根的二分搜索树中是否包含元素e, 递归算法
    private boolean BST_Search(Node node, E e){
        if(node == null)
            return false;
        if(e.compareTo(node.e) == 0)
            return true;
        else if(e.compareTo(node.e) < 0)
            return contains(node.left, e);
        else // e.compareTo(node.e) > 0
            return contains(node.right, e);
    }
```
查找以 node 为根的二分搜索树中是否含有元素 e 过程如下：

1. 如果 node 为 null，则返回 false

2. 将 e 的值与 node.e 比对，如果刚好相等，则返回 true

3. 上述条件都不满足时，则进行递归处理


##　前序遍历

```
    // 二分搜索树的非递归前序遍历
    public void preOrderNR(){
        Stack<Node> stack = new Stack<>();
        stack.push(root);
        while(!stack.isEmpty()){
            Node cur = stack.pop();
            System.out.println(cur.e);
            if(cur.right != null)
                stack.push(cur.right);
            if(cur.left != null)
                stack.push(cur.left);
        }
    }
```

```
    private void preOrder(Node node){
        if(node == null){
            return;
        }
        System.out.println(node.e);
        preOrder(node.left);
        preOrder(node.right);
    }
```


## 中序遍历

```
    private void inOrder(Node node){
        if (node == null){
            return;
        }
        inOrder(node.left);
        System.out.println(node.e);
        inOrder(node.right);
    }
```

## 后序遍历

```
    private void postOrder(Node node){
        if (node == null){
            return;
        }
        postOrder(node.left);
        postOrder(node.right);
        System.out.println(node.e);
    }
```


## BST 最大值

```
   private Node maximum(Node node){
        if (node.right == null){
            return node;
        }
            return maximum(node.right);
    }
```

## BST 最小值

```
    private Node minimum(Node node){
        if (node.left == null){
            return node;
        }
        return minimum(node.left);
    }
```
