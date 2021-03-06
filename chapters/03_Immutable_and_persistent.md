# Immutable and persistent data {#sec:immutable}

What prevents us from implementing traditional imperative-language data structures in R is the immutability of data. As a general rule, you can modify environments---so you can assign to variables---but you cannot modify actual data. Whenever R makes it look like you are modifying data, it is lying. When you assign to an element in a vector

```{r, eval=FALSE}
x[i] <- v
```

the vector will look modified to you, but behind the curtain R has simply replaced the vector that `x` refers to with a new copy, identical to the old `x` except for element number `i`. It tries to do this efficiently, so it will only copy the vector if there are other references to it, but conceptually, it still makes a copy. 

Now, you could reasonably argue that there is little difference between actually modifying data and simply having the illusion of modifying data, and you would be right, except that the illusion is only skin deep. Since R creates the illusion by making copies of data and assigning the copies to variables in the local environment, it doesn't affect other references to the original data. Data you pass to a function as a parameter will be referenced by a local function variable. If we "modify" such data, we are modifying the local function environment---the caller of the function has a different reference to the same data, and that reference is to the original data that will not be affected by what we do with the local function environment in any way. R is not entirely side-effect free, as a programming language, but side effects are contained to I/O, random number generation, and affecting variable-value bindings in environments. Modifying actual data is not something we can do via function side effects.[^sideeffects] If we want to update a data structure, we have to do what R does when we try to modify data: we need to build a *new* data structure, looking like the one we wanted to change the old one into. Functions that should update data structures need to construct new versions and return them to the caller.

[^sideeffects]: Strictly speaking, we *can* create side effects that affect data structures, we just have to modify environments. The reference class system, R6, emulate objects with mutable state by modifying environments, and we can do the same via closures. When we get to [Chapter @sec:bags-stacks-and-queues], where we will implement queues, we will need to introduce side effects of member queries, and there we will use this trick. Unless we represent all data structures by collections of environments, however, the trick only gets os so far. We still need to build data structures without modifying data; we just get to remember the result in an environment we constructed for this purpose.

## Persistent data structures

Since we cannot modify data, we might as well make a virtue out of necessity. What we get out of immutable data is persistent data structures, that is data structures that retain earlier versions of themselves. Not all types of data structures have persistent versions of themselves, and some persistent data structures can be less efficient than their non-persistent, or ephemeral, counterparts, but constructing persistent data structures is an active area of research so there are data structures enough to pick from when you need one.

To see what I mean by data structures being persistent in R, we look at the simple linked list again. I've defined it below, using slightly shorter names than earlier now that we don't need to remind ourselves that it is a linked list, and I'm using the sentinel trick to create the "empty" list.

```{r, eval=FALSE}
is_empty <- function(x) UseMethod("is_empty")

list_cons <- function(elem, lst)
  structure(list(item = elem, tail = lst), class = "linked_list")

list_nil <- list_cons(NA, NULL)
is_empty.linked_list <- function(x) identical(x, list_nil)
empty_list <- function() list_nil

list_head <- function(lst) lst$item
list_tail <- function(lst) lst$tail
```

With these definitions, we can create three lists like this:

```{r, eval=FALSE}
x <- list_cons(2, list_cond(1, empty_list()))
y <- list_cons(3, x)
z <- list_cons(4, empty_list())
```

The lists will be represented in memory as shown in [@fig:linked-lists-construction-1]. In the figure I have shown the content of the lists, the head of each, in the white boxes and the tail pointer as a grey box and an error. I have explicitly shown the empty list sentinel in this figure, but in future figures I will simply leave it out. The variables, `x`, `y`, and `z` are shown as pointers to the lists. For `x` and `z` , the lists were created by updating the empty list; for `y` , the list was created by updating `x`. But as we can clearly see, the updated lists are still there. We just need to keep a pointer to them to get them back. That is the essence of persistent data structures and how we need to work with data structures in R.

![Memory layout of linked lists.](figures/linked-lists-construction-1){#fig:linked-lists-construction-1}

## List functions

When we implemented a set through linked lists we saw how to add and search in a list. To get more of a feeling for working with immutable data structures we will try to implement a few more functions manipulating lists. We can start simply by writing a function for reversing a list. This is slightly more complicated than just searching in a list because we will have to construct the reversed list from the wrong end, so to speak. We need to first construct the list that contains the last element of the input list and the empty list. Then we need to put the second last elements at the head of this list, and so on.

When writing a function that operates on persistent data, I always find it easiest to think in terms of recursion. It might not be immediately obvious how to reverse a list as a recursive function, though. If we recurse all the way down to the end of the list, we get hold of the first element we should have in the reversed list, but how do we then fit that into the list we construct going up in the recursion again? There is no simple way to do this. We can, however, use the trick of bring an accumulator with is in the recursive calls and construct the reversed list in this. If you are not familiar with accumulators in recursive functions, I cover it in some detail in my book on functional programming in R [@mailund2017functional], but you can probably follow the idea in the code below. The idea is that the variable `acc` contains the reversed list we have constructed so far. When we get to the end of the recursion, we have the entire reversed list in `acc` so we can just return it. Otherwise, we can recurse on the remaining list but put the head element at the top of the accumulator. With a recursive helper function, the list reversal can look like this:

```{r, eval=FALSE}
list_reverse_helper <- function(lst, acc) {
  if (is_empty(lst)) acc
  else list_reverse_helper(list_tail(lst),
                           list_cons(list_head(lst), acc))
}
list_reverse_rec <- function(lst) 
  list_reverse_helper(lst, empty_list())
```

The running time of this function is linear---we need to run through the entire original list, but each operation we do when we construct the new list takes constant time.

I have shown the iterations for reversing a list of length three in [@fig:list-reversal]. In this figure I have not shown the sentinel empty string---I just show the empty string as a pointer to nothing---but you will see how the variable `lst` refers to different positions in the original list as we recurse, while the original list doesn't change at all, while we build a new list pointed to by `acc`.

![Iterations in the recursive list reversal.](figures/list-reversal){#fig:list-reversal}

In a pure functional programming language, this would probably be the best approach to reversing a list. The function uses tail recursion (again, you can read about that in my other book [@mailund2017functional]), so it is essentially a loop we have written. Unfortunately, R does *not* implement tail recursion, so we have a potential problem. If we have a very long list, we can run out of stack space before we finish reversing it. We can, however, almost automatically translate tail recursive functions into loops, and a loop version for reversing a list would then look like this:

```{r, eval=FALSE}
list_reverse_loop <- function(lst) {
  acc <- empty_list()
  while (!is_empty(lst)) {
    acc <- list_cons(list_head(lst), acc)
    lst <- list_tail(lst)
  }
  acc
}
```

If the lists are short, there is no immediate benefit in using one solution over the other. There is some overhead in function calls, but there is also some overhead in loops, and the two solutions work equally well for short lists, see [@fig:list-reverse-comparison]. Whenever I can get away with it, I prefer recursive solutions---I find them easier to implement and simpler to understand---but the loop version will be able to deal with much longer lists than the recursive one.

![Looping versus recursive reversal of lists.](figures/list-reverse-comparison){#fig:list-reverse-comparison}


Another thing we might want to do with lists is concatenate two of them. With mutable pointers, this is something that we can do in constant time if we have pointers to both the beginning and end of our linked lists, but with immutable data it becomes a linear time function---we need to construct the new concatenated list without modifying the two original lists, so we need to move to the end of the first list and then create a new one that contains the same elements followed by the second list. So strictly speaking, it isn't a linear time algorithm in the length of both lists, but it is linear in the length of the first list.

Again, it is easiest to construct the function recursively. Here, the base case is when the first list is empty. Then, the concatenation of the two lists is just the second list. Otherwise, we have to put the head of the first list in front of  the concatenation of the tail of the first list and the entire second list. As an R function, we can implement that idea like this:

```{r, eval=FALSE}
list_concatenate <- function(l1, l2) {
  if (is_empty(l1)) l2
  else list_cons(list_head(l1), 
                 list_concatenate(list_tail(l1), l2))
}
```

The new list we construct contain `l2` as the last part of it. We do not need to copy this---it is, after all, an immutable data structure so there is no chance of it changing in the future---but we are putting a new copy of `l1` in front of it. The structure of the lists after we concatenate them is shown in  [@fig:list-concatenation]. The two original lists are alive and well, and we have a new concatenated version.

![Persistent lists in list concatenation.](figures/list-concatenation){#fig:list-concatenation}

It is a little harder to implement concatenation without recursion. We construct the new list as we return from the recursive calls, so if we want to implement this iteratively, we need to emulate the call stack. We can do this, however, following the way we implemented list reversal: we can construct a reversal of the first list moving down the list and once we read the end of the first list we can construct the result of the concatenation by putting head elements in front of the new list. We can implement a looping version this way:

```{r, eval=FALSE}
list_concatenate_loop <- function(l1, l2) {
  rev_l1 <- empty_list()
  while (!is_empty(l1)) {
    rev_l1 <- list_cons(list_head(l1), rev_l1)
    l1 <- list_tail(l1)
  }
  result <- l2
  while (!is_empty(rev_l1)) {
    result <- list_cons(list_head(rev_l1), result)
    rev_l1 <- list_tail(rev_l1)
  }
  result
}
```

The loop version has to first construct the reversed of the first list and then construct the concatenated list, so it does more work than the recursive version, and therefore it is slower, see [@fig:list-concatenate-comparison]. I think you would also agree that the loop-version is quite a bit more complex than the recursive version. It can, however, deal with longer lists because it doesn't require a deep call stack. Still, unless we run out of stack in the recursive function, it is probably the better choice.

![Time comparison of the two concatenation functions.](figures/list-concatenate-comparison){#fig:list-concatenate-comparison}


What about removing elements from a list, then? Again, we cannot modify lists, so removing means constructing a new list, and once again, the easiest approach to solving the problem is to write a recursive function. The base case is removing an element from an empty list. That is easily done, because we can just return the empty list. Otherwise, we have two cases---assuming we only want to remove the first occurrence of an element, as I will assume there. Either the head of the list is equal to the element we want to remove, in which case we just return the tail of the list, otherwise we need to concatenate the current head to a recursive call to remove the element from the tail of the list. The solution could look like this:

```{r, eval=FALSE}
list_remove <- function(lst, elm) {
  if (is_empty(lst)) lst
  else if (list_head(lst) == elm) list_tail(lst)
  else list_cons(list_head(lst), list_remove(list_tail(lst), elm))
}
```

An example of the new list constructed by a removal is shown in [@fig:list-removal].

![Removing an element from a list.](figures/list-removal){#fig:list-removal}

We could make a loop version of the removal function as well, but just as for the concatenation function, it gets more complicated than the recursive solution, and I will not bother with it this time around.

The general problem with both the concatenation and removal functions, that prevents us from writing simple loop versions, is that they are not tail recursive. It is generally easy to translate tail recursive functions into loop, but in both the concatenation and the removal functions we need to build a new list as part of the recursion, and that complicates matters. There are general approaches to get around this and translate your functions into tail recursive ones, using what is called continuations, but this adds an overhead in several ways, not least because you would still have to emulate actual tail recursion to avoid reaching the stack limit, so *if* recursive functions cause you problems, you are better off trying to write a costume looping version instead. That being said, because list functions typically have linear running time, if you have to worry about stack limits, you should probably reconsider the data structure in any case and use a more efficient one. For functional data structures, that almost always means using a tree instead of a list.

## Trees

Linked lists are the workhorses of much functional programming, but when it comes to efficiency, trees usually are better alternatives. In fact, most of the remaining chapters of this book will use variations of trees to implement various abstract data structures. For now, however, we will just consider how to construct and traverse trees. For this, we implement a simple, unbalanced, search tree. Search trees are trees with the following properties:

1. Each node in the tree contains a number (or any other element from an ordered set)
2. Each node has two children, we call them the "left" and the "right" children. These children are sub-trees.
3. All elements in the left subtree of a node contains values smaller than the element in the node.
4. All elements in the right subtree of a node contains values larger than the element in the node.

An example of a search tree, containing the elements 1, 3, 4, 6, and 9, is shown in [@fig:search-tree].

The asymptotic efficiency of search trees come from how they are balanced. Searching in a search tree involves recursing down the tree structure and will take time proportional to the depth we search. The worst case for a tree would be one that is essentially a list---we could get such a tree if all nodes only had a single child---where the search time would be linear, just as searching in lists. We can typically balance them, though, in which case the depth is only going to be logarithmic in the number of elements we save in them. To see this, consider a binary tree where at each level all nodes have two children. That would double the number of elements we have in the tree at each level, and we can only double a number 
#ifdef EPUB
log(n)
#else
$\log(n)$
#endif
many times before we have
#ifdef EPUB
n
#else
$n$
#endif
elements.

The tree we construct in this chapter will not necessarily be balanced. We implement none of the tricks needed to keep it balanced, so the depth of the tree will depend on the order in which we insert elements.

![An example of a search tree.](figures/search-tree){#fig:search-tree}

Anyway, we construct our tree out of nodes that contain a value and a reference to the left and to the right subtrees. As usual, we create a sentinel object to represent an empty tree. So our tree implementation could look like this:

```{r, eval=FALSE}
search_tree_node <- function(
  value
  , left = empty_search_tree()
  , right = empty_search_tree()
) {
  structure(list(left = left, value = value, right = right),
            class = c("unbalanced_search_tree"))
}

empty_search_tree_node = search_tree_node(NA, NULL, NULL)
empty_search_tree <- function() empty_search_tree_node
is_empty.unbalanced_search_tree <- function(x) 
  identical(x, empty_search_tree_node)
```

We want three generic functions for working with sets in general, so we define functions for inserting and removing elements in a set and a function for testing membership in a set. All three will be implemented for our search tree with complexity proportional to the depth of the tree.

```{r, eval=FALSE}
insert <- function(x, elm) UseMethod("insert")
remove <- function(x, elm) UseMethod("remove")
member <- function(x, elm) UseMethod("member")
```

Of these three functions, the `member` function is the simplest to implement for a search tree. The invariant for search trees tells us that if we have a node and the element we are looking for is smaller than the value there, then the element must be found in the left search tree if it exists; otherwise, it must be found in the right search tree. This leads naturally to a recursive function. The base case is when we search in an empty tree: there we will always answer that the element isn't found there. Otherwise, we essentially have to check three possibilities: We might have found the node where the element is, or we have to search to the left, or we have to search to the right. We can implement that function thus:

```{r, eval=FALSE}
member.unbalanced_search_tree <- function(x, elm) {
  if (is_empty(x)) return(FALSE)
  if (x$value == elm) return(TRUE)
  if (elm < x$value) member(x$left, elm)
  else member(x$right, elm)
}
```

In this solution, we do to comparisons in each recursive call if we are not at an empty tree: we compare for equality with the element we are looking for and we check if it is less than the value in the node. We can actually delay one of the comparisons and halve the number of comparisons. We just have to remember the last element that *could* be the element we are looking for. We then can call recursively to the left or right after just checking if the element in the node is larger than the element we are searching for. If we get all the way down to an empty tree, we check if the element is the one we could have checked equality on earlier. That solution looks like this:

```{r, eval=FALSE}
st_member <- function(x, elm, candidate = NA) {
  if (is_empty(x)) return(!is.na(candidate) && elm == candidate)
  if (elm < x$value) st_member(x$left, elm, candidate)
  else st_member(x$right, elm, x$value)
}
member.unbalanced_search_tree <- function(x, elm) {
  st_member(x, elm)
}
```

This solution reduces the number of comparisons per inner node in the tree, but it also risks going deeper in the tree that necessary because it will move past the element we are searching for. In practise, on random search trees, the two solutions have roughly the same performance, see [@fig:search-tree-member-comparison]. In that figure I have divided the running time with 
#ifdef EPUB
log(n)
#else
$\log(n)$
#endif
since that is the expected depth of a random tree, and I ran the experiments on random trees. It is not the worst case tree depth, though. We see this when we consider how we add elements to a tree.

![Search tree member functions comparison.](figures/search-tree-member-comparison){#fig:search-tree-member-comparison}

To insert an element in a search tree we have to, as always, construct a new structure that represent the updated tree. We can do this by searching recursively down the tree to the position where we should insert the element---going left if we are inserting an element smaller than the value in a node and going right if we are inserting a larger value---and then constructing the new tree going up the recursion. A recursive solution will work like this: if we reach an empty tree, we have found the place where we should insert the element, so we construct a new leaf and return that from the recursion. Otherwise, we check the value in the inner node. If the value is larger than the element we are inserting, we need to construct the new tree consisting of the current value, the existing right tree, and a version of the left tree where we have inserted the element. If the element in the node is smaller than the element we are inserting, we do the symmetric constructing where we insert the element in the right tree. Of course, if the element is equal to the value in the node, it is already in the tree so we can just return the existing tree. In R code, this solutions could look like this:

```{r, eval=FALSE}
insert.unbalanced_search_tree <- function(x, elm) {
  if (is_empty(x)) return(search_tree_node(elm))
  if (elm < x$value)
    search_tree_node(x$value, insert(x$left, elm), x$right)
  else if (elm > x$value)
    search_tree_node(x$value, x$left, insert(x$right, elm))
  else
    x # the value is already in the tree
}
```

The insert function searches down the recursion for the place to put the new value and then constructs a new tree going up the recursion again. All the operations we do going down and up the recursion are constant time and we only copy parts of the original tree that are directly on the search part. The other parts of the tree we just keep the existing references to. An example of an updated tree, after inserting 5 in the tree from [@fig:search-tree] is shown in [@fig:search-tree-insert].

![Result of inserting 5 in the search tree from [@fig:search-tree].](figures/search-tree-insert){#fig:search-tree-insert}

If we have a set of elements we want to construct a search tree on, and we insert them in a random order, each element is equally likely to be put to the left or the right of the root, on average, so we end up with a fairly balanced tree---which is why we got flat plots when we divided by the logarithm of the tree size when we compared membership test functions. However, if we were inserting the element in increasing order, we would always be inserting the next element in the rightmost child of the tree, and we would end up with a tree with linear depth.

If we construct search trees by inserting one element at a time, the running time would be the number of elements time the average depth. If the tree is unbalanced, this becomes an 
#ifdef EPUB
O(n^2^)
#else
$O(n^2)$
#endif
constructing time because the average tree depth will be 
#ifdef EPUB
n/2.
#else
$n/2$.
#endif
The contrast between inserting the elements in a random order or in increasing order is shown in [@fig:search-tree-construction-comparison]. It is actually slightly worse than the running time suggests: since the unbalanced trees can be linear in depth, the recursive functions we use to manipulate them can run into problems with stack depths long before their balanced counterparts. We return to techniques for keeping trees balanced in [@sec:sets-and-search-trees]. For now, we just continue with the simple operations, and we have made it to the last function we need to implement: removal of elements.

![Comparison of balanced and unbalanced search tree constructions.](figures/search-tree-construction-comparison){#fig:search-tree-construction-comparison}

Removing elements is the most complex function we need to implement for our search trees. The basic pattern is the same as for insertion: we search for the element to remove going down a recursion and then construct an updated tree going up the recursion again. The complication comes when we actually have to remove an element. Removing leaves is easy: we can just return an empty tree from the recursion when we construct the updated tree. It is also easy to remove nodes with a single child: here we can just return the child and it will be inserted in the correct position in the tree we construct. When the element we need to remove is in an inner node with two children, however, we cannot easily construct a new inner node with references to the two children but not the value in the node.

We need to construct a new node with references to the two children but containing a value that is not the one we want to delete, that should already be in the tree---we don't want to add values when we are removing one---and it should satisfy the invariant that it is larger than all elements in the left subtree and smaller than all elements in the right subtree.

The trick is to get hold of the leftmost node in the right subtree. This node contains the smallest value in the right subtree, so if we put this value in the new inner node, and remove it from the right subtree, then the invariant will be satisfied. The leftmost node in the subtree is larger than all values in the left subtree---because is is currently in the right subtree---and it will be smaller than all the other values in the right subtree because it is the leftmost value. Also, it is easy to remove the leftmost node in a tree because it can have at most one (right) child, and those cases we can handle easily.

We need a helper function for finding the leftmost node in a subtree, but after that, we just have to handle different cases when we delete an element. If it isn't in the tree, we eventually hit an empty tree in the search and we just return that. Otherwise, if we find the element, we handle the cases where it has at most one child directly and do the leftmost trick otherwise. The remaining cases are just handling when we are still searching down in the recursion: we need to create a node that contains the original left or right subtree---depending on the value in the node---and use the result of removing the element in the other subtree. The full implementation can look like this:

```{r, eval=FALSE}
st_leftmost <- function(x) {
  while (!is_empty(x)) {
    value <- x$value
    tree <- x$left
  }
  value
}

remove.unbalanced_search_tree <- function(x, elm) {
  # if we reach an empty tree, there is nothing to do
  if (is_empty(x)) return(x)

  if (x$value == elm) {
    a <- x$left
    b <- x$right
    if (is_empty(a)) return(b)
    if (is_empty(b)) return(a)

    s <- st_leftmost(x)
    return(search_tree_node(s, a, remove(b, s)))
  }

  # we need to search further down to remove the element
  if (elm < x$value)
    search_tree_node(x$value, remove(x$left, elm), x$right)
  else # (elm > x$value)
    search_tree_node(x$value, x$left, remove(x$right, elm))
}
```


[@Fig:search-tree-remove] shows the result of removing 6 from the tree in [@fig:search-tree]. When the search finds the node that contains 6 it will go down and get the leftmost node in the right subtree, which is the leaf containing 9. It then deletes 9  from the right subtree---the result will be an empty tree---and constructs a new node that contains the value 9, has the subtree containing 4 as its left subtree and the empty subtree as the right subtree. It then creates a copy of the nodes above the removal point going up the recursion to construct the final updated tree.

![Removing an element from a subtree.](figures/search-tree-remove){#fig:search-tree-remove}

Lists and trees are the basic building blocks for all the data structures we will examine in this book, and the way we have written functions for updating them will be the way we approach all the data structures we see. The methods we have seen in this chapter are not efficient in most cases---although the search tree is pretty good if the data we store in it is random---but the data structures that *are* efficient are build on the foundation we are familiar with now.


## Random access lists

As a small example, before we move on to more traditional data structures, we will consider how to build random access lists, described in @Okasaki95purelyfunctional, using lists and trees. Random access lists behave like lists---we can extend them at the front and we can access or remove the front element in constant time---but have two additional operations: we can access any element by its index and we can change any element referenced by its index (but not delete elements at arbitrary positions). The data structure from @Okasaki95purelyfunctional implements these two additional operations in $O(\\log n)$ time.

The random access lists are constructed from fully balanced binary trees. With this data structure we won't be implementing different versions, so I will not bother with classes here and simply implement a bare-bones version of the data structures, so the actual structure can easily be seen and not be overshadowed by interface design. So, we implement trees as lists and we will use `NULL` to indicate an empty tree. We can construct a tree like this:

```{r, eval=FALSE}
ral_binary_tree <- function(value, left, right) {
  list(value = value, left = left, right = right)
}
```

Since we use trees to represent lists, we need a way of mapping positions in lists to nodes in the trees. In this data structure we will map indices to the in-order position in the trees; this means that the first element in a tree is the value in the root, the follows all elements in the left subtree and then all elements in the right subtree. We choose this in-order mapping because we then have constant time access to the front of the list---the root of the tree.

To find the node in a tree that corresponds to a given index, we search recursively. Index 1 is always the root, so if we want to access this index we have access to it right away. Otherwise, we have to search in either the left or the right subtree. If the tree has size $n$, then the root has 1 element and each subtree has $(n-1)/2$ elements, since all the trees we will manipulate are fully balanced. So if we are looking for an index $i$ larger than one, we know that we need to search in the left subtree if $i \leq (n-1)/2 + 1$ and that we need to search in the right subtree otherwise. In a recursive call we then need to modify the index to skip past the root---when searching to the left---or the root and the left subtree---when searching to the right. Based on this algorithm, we can implement the lookup operation like this:

```{r, eval=FALSE}
ral_is_leaf <- function(tree)
  !is.null(tree) && is.null(tree$left) && is.null(tree$right)

ral_tree_lookup <- function(tree, tree_size, idx) {
  if (idx == 1) return(tree$value)

  if (ral_is_leaf(tree)) # a leaf but idx is not one!
    if (idx > 1) stop("Index error in lookup")

  child_size <- (tree_size - 1) / 2
  if (idx <= child_size + 1)
    ral_tree_lookup(tree$left, child_size, idx - 1)
  else
    ral_tree_lookup(tree$right, child_size, idx - 1 - child_size)
}
```

Updating a tree is only slightly more involved. Locating the position where we need to change a value follows exactly the same search algorithm, of course, we just need to construct a new tree in recursive calls. Whenever we call recursively, we construct a new tree from the existing values and trees, but modify either the left or the right subtree via a recursive call to the update operation:

```{r, eval=FALSE}
ral_tree_update <- function(tree, tree_size, idx, value) {
  if (ral_is_leaf(tree)) {
    if (idx == 1)
      return(ral_binary_tree(value, NULL, NULL))
    # a leaf but idx is not one!
    stop("Index error")
  }

  if (idx == 1) {
    ral_binary_tree(value, tree$left, tree$right)
  } else {
    child_size <- (tree_size - 1) / 2
    if (idx <= child_size + 1) {
      ral_binary_tree(tree$value,
                      ral_tree_update(tree$left, child_size, 
                      idx - 1, value),
                      tree$right)
    } else {
      ral_binary_tree(tree$value,
                      tree$left,
                      ral_tree_update(tree$right, child_size,
                                      idx - 1 - child_size, value))
    }
  }
}
```

Since the trees are fully balanced, the maximal depth of a tree of size $n$ is $O(\\log n)$, and both the lookup and the update operations thus run in this time.

Obviously, we cannot represent all lists with $n$ elements in fully balanced trees. For any balanced tree with $n$ nodes, $n$ must be $2^k - 1$ for some $k$. Because of this, there is a close corresponded between fully balanced trees and [skew binary numbers](https://en.wikipedia.org/wiki/Skew_binary_number_system), that we will exploit in this data structure. Skew binary numbers are positional numbers---meaning that the position of a digit determine its value, as you are used to with decimal and binary numbers---where a digit $d$ at position $k$ represent the number $d \\times 2^{k+1} - 1$. The digit $d$ can be 0, 1, or 2, but is only allowed to be 2 if it is the least significant non-zero digit. The first six numbers are shown in decimal, binary and skew binary representations below:

Decimal |	Binary	| Skew binary
-------:+------:+--------------:
      0	|     0	|            0
      1	|     1	|            1
      2	|    10	|            2
      3	|    11	|           10
      4	|   100	|           11
      5	|   101	|           12

There are two important properties of skew binary numbers that we will exploit for our random access lists: any number $n$ can be represented in $O(\\log n)$ digits (of which at most one is 2, so even if we count that one twice we have logarithmically many digits), and we can increment and decrement numbers in this representation in constant time. The first property should be obvious: we use fewer digits in skew binary numbers than in binary numbers, and for binary numbers we only have $O(\\log n)$ digits to represent the number $n$. Constant time increment and decrement can be achieved by representing only the non-zero digits as a linked list. We represent a number by a linked list where the elements in the list are the positions of the non-zero digits, where the digit 2 is represented by two elements with the same position, and we keep this list sorted in increasing position order, see [@fig:skew-binary-numbers].

![Representation of skew binary numbers.](figures/skew-binary-numbers){#fig:skew-binary-numbers}

To increment a number, we first check if the first two elements in the list contain the same position. If they do, we cannot prepend a position zero element since only the least significant non-zero digit can be two. Instead, we can merge the two front elements into a single list element containing the next position in the number representation. Since $2 \\times (2^{k+1}-1) + 1 = 2^{k+2} - 1$, if the two front element contain the position $k$, then merging them this way increments the number $n$. Since only the two front elements can be duplicated, the new number we create this way doesn't invalidate the invariant---it might contain the same position as the next element in the list, but before we merge the front elements, that would be the only element containing that index. If the two front elements contain different positions, we can just add a zero-position element to the front of the list---this might change a 1 at position 0 to a 2, but that is fine since there was only one position 0 element to begin with.

To decrement, we do the reverse of the increment operation. If there is a position zero element at the front of the list, we just remove that element. Otherwise, we take the first element and replace it with two elements containing the position one smaller.

Getting back to the lists, we are going to represent these as linked lists similar to the skew binary number representation. We are just going to have fully balanced binary trees of size $2^{k+1}-1$ instead of the position $k$ as list elements. We represent a list of $n$ elements by having trees of sizes corresponding to the digits in the skew binary number representation of $n$. The representation is a linked list---we have a `siblings` pointer to the next element in the list---and each element contains a tree and the tree size. 

```{r, eval=FALSE}
ral_node <- function(tree, tree_size, siblings) {
  list(tree = tree, tree_size = tree_size, siblings = siblings)
}
```

The empty list is represented by `NULL`, as for empty trees.

```{r, eval=FALSE}
ral_is_empty <- function(ral) is.null(ral)
```

We will keep the trees sorted such that the first elements in the list are in the first tree, the following in the second tree, and so on. Then, because the order of elements in individual trees are in-order, we can get the head of the list from the root of the first tree:

```{r, eval=FALSE}
ral_head <- function(ral) {
  ral$tree$value
}
```

The other lists operations---appending an element to the front of the list, or removing the front element---follows the operations on skew binary numbers. For the `cons` operation, if the two front trees have different sizes, we just prepend a size 1 tree. If the two front trees have the same size, we construct a new tree with the new front element at the root, the two original front trees as the left and right trees, respectively---the first tree must go to the left and the second to the right---and we replace the two front trees with the new tree. Taking the tail of a list is just the reverse of these operations: if the front tree has size 1, simply remove it, otherwise split the front tree into two and get rid of the root, see [@fig:random-access-lists-cons-and-tail]. The implementation just checks the cases and does the appropriate operations based on them:

![List operations on random access lists.](figures/random-access-lists-cons-and-tail){#fig:random-access-lists-cons-and-tail}

```{r, eval=FALSE}
ral_singleton_node <- function(value, siblings = NULL) {
  singleton_tree <- ral_binary_tree(value, NULL, NULL)
  ral_node(singleton_tree, 1, siblings)
}
ral_cons <- function(elem, ral) {
  if (ral_is_empty(ral) || ral_is_empty(ral$siblings))
      return(ral_singleton_node(elem, ral))

  first <- ral$tree
  first_size <- ral$tree_size
  second <- ral$siblings$tree
  second_size <- ral$siblings$tree_size
  rest <- ral$siblings$siblings

  if (first_size < second_size)
    ral_singleton_node(elem, ral)
  else
    ral_node(ral_binary_tree(elem, first, second), 
             first_size + second_size + 1, rest)
}
```


```{r, eval=FALSE}
ral_is_singleton <- function(ral) {
  ral$tree_size == 1
}

ral_tail <- function(ral) {
  if (ral_is_singleton(ral))
    ral$siblings
  else {
    left <- ral$tree$left
    right <- ral$tree$right
    size <- (ral$tree_size - 1) / 2
    ral_node(left, size, ral_node(right, size, ral$siblings))
  }
}
```

What remains to be implemented are the lookup and update operations. We already have functions for looking up or updating a value in a tree, so we just need code for picking the right tree from the list of trees. This is a simple matter of keeping track of the index we are looking for. For the lookup, we iterate through the trees and check if the index is less than the size of the current tree. If it is, we look into the tree. Otherwise, we decrease the index to reflect that we have moved past part of the list and then move to the next tree:

```{r, eval=FALSE}
ral_lookup <- function(ral, idx) {
  while (!is.null(ral)) {
    if (idx <= ral$tree_size)
      return(ral_tree_lookup(ral$tree, ral$tree_size, idx))
    idx <- idx - ral$tree_size
    ral <- ral$siblings
  }
  stop("Index out of bounds")
}
```

For the update operation, we are essentially doing the same thing as for the lookup operation, but since we need to construct a new list down to the tree we end up modifying, we cannot use a loop. We must do the search in a recursion that constructs a new list. So each recursive call must be wrapped in a construction of a new node.

```{r, eval=FALSE}
ral_update <- function(ral, idx, value) {
  if (idx < 1) stop("Index out of bounds")
  if (idx <= ral$tree_size)
    ral_node(ral_tree_update(ral$tree, ral$tree_size, idx, value),
             ral$tree_size, ral$siblings)
  else
    ral_node(ral$tree, ral$tree_size,
             ral_update(ral$siblings, idx - ral$tree_size, value))

}
```

The way the data structure is updated is illustrated in [@fig:random-access-lists-update]. Here, the index we must update is found in the third tree, so the new list will consist of three new elements and will then be followed by elements from the old list. The first two list elements just point to the first two original trees while the third element points to a modified tree. In the figure, the original and updated trees are shown as disjoint, but in fact they will share most of their structure; only elements on the path down to the node that is updated will differ between the two trees.

![Updating random access lists.](figures/random-access-lists-update){#fig:random-access-lists-update}
