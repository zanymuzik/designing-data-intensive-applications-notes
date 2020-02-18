# Red-black trees

* Red-black trees are a type of self-balancing binary search tree. They have $O(\log{n})$ read times, making them highly efficient.
* In addition to the properties of a binary search tree red-black trees have the following invariants:
  * Each node is either red or black.
  * All leaves are `null` and black.
  * If a node is red, then both its children are black.
  * Every path from a given node to any of its descendant `null` nodes contains the same number of black nodes.
* Red–black trees offer worst-case guarantees for insertion time, deletion time, and search time.
* Read operations are the same as in binary search trees.
* Insert and delete are much more compliated, but still $O(\log{n})$.
* Here's the tutorial I'm following: http://staff.ustc.edu.cn/~csli/graduate/algorithms/book6/chap14.htm.


* A


```python
class Node:
    def __init__(self, color, left, right, parent, value=None):
        self.color = color
        self.left = left
        self.right = right
        self.parent = parent
        self.value = value
        

class Tree:
    def __init__(self, root):
        self.root = root
        
        
def set_left(node, parent):
    # If the node is the right child of the parent, remove it from there.
    if parent.right == node:
        parent.right = None
    
    # If the parent has an existing left child set that child's parent to None (overwrite).
    if parent.left is not None:
        parent.left.parent = None
    
    # Attach the node to the parent.
    parent.left = node
    
    # If the node is non-null assign it the parent pointer.
    if node is not None:
        node.parent = parent
    
        
def set_right(node, parent):
    if parent.left == node:
        parent.left = None
    
    if parent.right is not None:
        parent.right.parent = None    
    
    parent.right = node
    
    if node is not None:
        node.parent = parent

    
def left_rotate(T, x):
    y = x.right
    
    set_right(node=y.left, parent=x)
    
    if x.parent is None:
        T.root = y  # TODO: ?
    else:
        if x == x.parent.left:
            set_left(node=y, parent=x.parent)
        else:
            set_right(node=y, parent=x.parent)
            
    set_left(node=x, parent=y)
    
    
def right_rotate(T, x):
    y = x.left
    
    set_left(node=y.right, parent=x)
    
    if x.parent is None:
        T.root = y
    else:
        if x == x.parent.right:
            set_right(node=y, parent=x.parent)
        else:
            set_left(node=y, parent=x.parent)
            
    set_right(node=x, parent=y)
    
    
def binary_insert(T, val):
    """Insert as though inserting into a binary tree, with the color red."""
    n = T.root
    while True:
        if val < n.value:
            if n.left is None:
                node = Node('red', None, None, n, value=val)
                set_left(node=node, parent=n)
                return node
            else:
                n = n.left
        else:
            if n.right is None:
                node = Node('red', None, None, n, value=val)
                set_right(node=node, parent=n)
                return node
            else:
                n = n.right
    
    
def insert(T, v):
    # Perform a binary insert.
    x = binary_insert(T, v)
    
    
    # Now correct violations of the red-black property.
    while x != T.root and x.parent.color == 'red':
        if x.parent == x.parent.parent.left:
            y = x.parent.parent.right
            
            # Case 1, both the node and its parent are red. 
            # Simply recolor.
            # This can move the issue further up the tree, which is why we have a while loop.
            # So we can fix those issues as well.
            if y.color == 'red':
                x.parent.color = 'black'
                y.color = 'black'
                x.parent.parent.color = 'red'
                x = x.parent.parent
                
            else:
                # Case 2:
                # * x is not the tree root.
                # * x's parent is red.
                # * x's parent is to the left of its grandparent.
                # * x's uncle (its parent's right sibling) is black.
                # * x is on its parent's right.
                # 
                # x's parent is red but its uncle is black, which violates invariant four.
                # We can fix this by performing a left rotation.                
                if x == x.parent.right:
                    x = x.parent
                    left_rotate(T, x)
                
                # Case 3. After fixing case 2, we will always end up with a doubled red node further up the tree.
                # Perform a right rotation to fix this.
                x.parent.color = 'black'
                x.parent.parent.color = 'red'
                right_rotate(T, x.parent.parent)
                
        else:  # x.parent == x.parent.parent.right
            # Same logic, just with right and left exchanged.
            y = x.parent.parent.left
            
            if y.color == 'red':
                x.parent.color = 'black'
                y.color = 'black'
                x.parent.parent.color = 'red'
                x = x.parent.parent
                
            else:
                if x == x.parent.left:
                    x = x.parent
                    right_rotate(T, x)
                    
                x.parent.color = 'black'
                x.parent.parent.color = 'red'
                left_rotate(T, x.parent.parent)
    
    T.root.color = 'black'
```


```python
# Test suite for the swap operation.

def test_set_left_clean():
    r = Node('red', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    set_left(n, r)
    assert r.left == n
    assert n.parent == r
    assert r.right == None
    assert n.left == None
    assert n.right == None
    
    
def test_set_right_clean():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    set_right(n, r)
    assert r.right == n
    assert n.parent == r
    assert r.left == None
    assert n.left == None
    assert n.right == None
    
    
def test_set_left_overwrite():
    r = Node('black', None, None, None)
    t = Tree(r)
    n1 = Node('red', None, None, None)
    n2 = Node('red', None, None, None)
    
    r.left = n1
    n1.parent = r
    
    set_left(n2, r)
    assert r.left == n2
    assert n2.parent == r
    assert n1.parent == None

    
def test_set_right_overwrite():
    r = Node('black', None, None, None)
    t = Tree(r)
    n1 = Node('red', None, None, None)
    n2 = Node('red', None, None, None)
    
    r.right = n1
    n1.parent = r
    
    set_right(n2, r)
    assert r.right == n2
    assert n2.parent == r
    assert n1.parent == None


def test_set_left_swap():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    
    r.right = n
    n.parent = r
    
    set_left(n, r)
    assert r.left == n
    assert n.parent == r
    assert r.right == None
    
    
def test_set_right_swap():
    r = Node('black', None, None, None)
    t = Tree(r)
    n = Node('red', None, None, None)
    
    r.left = n
    n.parent = r
    
    set_right(n, r)
    assert r.right == n
    assert n.parent == r
    assert r.left == None

    
    
test_set_left_clean()
test_set_right_clean()
test_set_left_overwrite()
test_set_right_overwrite()
test_set_left_swap()
test_set_right_swap()
```


```python
def test_left_rotate_nonroot():
    r = Node('black', None, None, None)
    T = Tree(r)
    x = Node('red', None, None, None)
    y = Node('black', None, None, None)
    
    r.right = x
    x.parent = r
    x.right = y
    y.parent = x

    left_rotate(T, x)
    
    assert T.root == r
    assert T.root.right == y
    assert y.left == x
    
    
def test_left_rotate_root():
    r = x = Node('black', None, None, None)
    T = Tree(r)
    y = Node('black', None, None, None)
    
    r.right = y
    y.parent = r
    
    left_rotate(T, r)
    
    assert T.root == y
    assert T.root.left == r
    
    
def test_right_rotate_nonroot():
    r = Node('black', None, None, None)
    T = Tree(r)
    x = Node('red', None, None, None)
    y = Node('black', None, None, None)
    
    r.left = x
    x.parent = r
    x.left = y
    y.parent = x

    right_rotate(T, x)
    
    assert T.root == r
    assert T.root.left == y
    assert y.right == x
    
    
def test_right_rotate_root():
    r = x = Node('black', None, None, None)
    T = Tree(r)
    y = Node('black', None, None, None)
    
    r.left = y
    y.parent = r
    
    right_rotate(T, r)
    
    assert T.root == y
    assert T.root.right == r
    
    
test_left_rotate_nonroot()
test_left_rotate_root()
test_right_rotate_nonroot()
test_right_rotate_root()
```


```python
def test_binary_insert_left():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 0)
    
    assert T.root.left.value == 0
    
    
def test_binary_insert_right():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 2)
    
    assert T.root.right.value == 2

    
def test_binary_insert_dig():
    r = Node('black', None, None, None, value=1)
    T = Tree(r)
    binary_insert(T, 0)
    binary_insert(T, 2)
    
    binary_insert(T, 3)
    binary_insert(T, -1)
    
    assert T.root.right.right.value == 3
    assert T.root.left.left.value == -1

    
test_binary_insert_left()
test_binary_insert_right()
test_binary_insert_dig()
```


```python
def test_insert():
    rightright = Node('red', None, None, None, value=15)
    right = Node('black', None, rightright, None, value=14)
    rightright.parent = right
    
    leftleft = Node('black', None, None, None, value=1)
    leftrightright = Node('red', None, None, None, value=8)
    
    # leftrightleftleft = Node('red', None, None, value=4)
    leftrightleft = Node('red', None, None, None, value=5)
    leftright = Node('black', leftrightleft, leftrightright, None, value=7)
    leftrightleft.parent = leftright
    leftrightright.parent = leftright
    
    left = Node('red', leftleft, leftright, None, value=2)
    leftleft.parent = left
    leftright.parent = left
    
    root = Node('black', left, right, None, value=11)
    left.parent = root
    right.parent = root
    
    t = Tree(root)
    
    insert(t, 4)
    
test_insert()
```

* This case above emulates the case given in the reference.
* Not included: deletion, which is also $O(\log{n})$ and "only slightly more complicated than insertion".
