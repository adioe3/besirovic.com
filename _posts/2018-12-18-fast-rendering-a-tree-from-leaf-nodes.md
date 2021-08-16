---
layout: post
title: "Fast rendering a tree from leaf nodes"
date: 2018-12-18
---

At work we ran into an interesting problem: given a tree and a list of leaf node IDs, how do you get a list of the nodes, sorted in a depth-first order and render it into HTML in one pass quickly?

For example, given this tree (blue lines indicates ordering of same-level nodes):

![Basic graph](/assets/fast-tree/basic.svg)

and the list of IDs [7, 8] we want to first get this subtree:

![First subtree](/assets/fast-tree/first-subtree.svg)

and generate this HTML out of it in a single pass:

```html
<section id="1" data-level="1">
    <section id="2" data-level="2">
        <section id="4" data-level="3">
            <section id="7" data-level="4">...</section>
            <section id="8" data-level="4">...</section>
        </section>
    </section>
</section>
```

while preserving the ordering of same-level nodes (i.e. 3 should always come before 4, 5 before 6 and so on).

## Ways to generate the HTML

Since this is a tree stucture, one method to generate the HTML is by using recursion. Oddly, this is one example where recursion is easier to understand than the alternative. Here is a very and clear working example:

```php
<?php
class Node {
    public $id;
    public $children = [];
}

function generate_html(Node $node, $level = 1) {
    echo "<section id='{$node->id}' data-level='$level'>";

    // Node content would be handled here

    foreach ($node->children as $child) {
        generate_html($child, $level + 1);
    }

    echo "</section>";
}
```

The team working on this before my time took another approach (I assume they tried recursion and it was too slow). Their solution was a single-pass which, knowing the maximum depth beforehand and that the nodes are sorted in a depth-first search ordering, is possible to do with something like this:

```php
<?php
define("MAX_DEPTH", 4);
$leaf = false;
$last = 0;

foreach ($nodes as $node)
{
    if ($node->level == MAX_DEPTH)
    {
        $leaf = true;
        echo "<section id='{$node->id}' data-level='{$node->level}'>";
        // Handle leaf node content here
        echo "</section>";
    }
    else
    {
        if ($leaf) {
            echo str_repeat("</section>", MAX_DEPTH - $node->level);
            $leaf = false;
        }

        echo "<section id='{$node->id}' data-level='{$node->level}'>";
        // Handle container node content here
    }

    $last = $node->level;
}

echo str_repeat("</section>", $last - 1);
```

Their solution utilized materialized paths stored in a relational database.

## Materialized paths

For the graph above, the database table would look like this:

```
id | parent_id | position | path
---|-----------|----------|--------
 1 |      NULL |        1 | 1
 2 |         1 |        1 | 1/2
 3 |         2 |        1 | 1/2/3
 4 |         2 |        2 | 1/2/4
 5 |         3 |        1 | 1/2/3/5
 6 |         3 |        2 | 1/2/3/6
 7 |         4 |        1 | 1/2/4/7
 8 |         4 |        2 | 1/2/4/8
```

The path field makes lookups relatively easy, going top-down we can quickly get all nodes below say node 3 by comparing the path:

```sql
SELECT * FROM table WHERE path LIKE '1/2/3/%';
```

The reverse is somewhat less intuitive but also possible:

```sql
SELECT * FROM table WHERE LOCATE(path, '1/2/3') = 1;
```

The problem however is the ordering - it is impossible to sort this in a depth-first search ordering. To accomplish that recursive queries on the parent_id and position fields are required but that degrades performance quickly as the number of queries issued to the database grows exponentially.

One possible optimization is to build a second materialized path for the position field:

```
id | parent_id | position | path    | position_path
---|-----------|----------|---------|---------------
 1 |      NULL |        1 | 1       | 1
 2 |         1 |        1 | 1/2     | 1/1
 3 |         2 |        1 | 1/2/3   | 1/1/1
 4 |         2 |        2 | 1/2/4   | 1/1/2
 5 |         3 |        1 | 1/2/3/5 | 1/1/1/1
 6 |         3 |        2 | 1/2/3/6 | 1/1/1/2
 7 |         4 |        1 | 1/2/4/7 | 1/1/2/1
 8 |         4 |        2 | 1/2/4/8 | 1/1/2/2
```

This enables ordering the results easily:

```sql
SELECT * FROM table WHERE LOCATE(path, '1/2/3') = 1 ORDER BY position_path ASC;
```

While the benefits of the materialized paths are obvious for fetching data, moving things around becomes very expensive quickly because each time we move a node to a new parent we have to update the paths of all the nodes below.

Below is an idea how to implement a performant solution to this using the C programming language. In theory, any programming language with a concept similar to pointers would do the trick.

## Graph structure

Looking at the original graph above there are five relationships and two pieces of info we need to know about each node. The info is basically the node ID and content, the relationships are:

- what is the parent node?
- what is to the left and right of this node on the same level?
- what is the first and/or last child of this node?

Given that, our node structure would visually look like this:

## Node structure

and can be described with this structure:

```c
typedef struct _Node {
    int id;
    void* data;

    struct _Node* parent;
    struct _Node* prev;
    struct _Node* next;
    struct _Node* first_child;
    struct _Node* last_child;
} Node;
```

Traversing a tree of these structures is very easy, top-to-bottom:

```c
void traverse(Node *node) {
    /* process this node's content here */

    /* we're doing a depth-first so process child nodes first */
    if (node->first_child != NULL) {
        traverse(node->first_child);
    }

    /* then move to the next neighbor if any */
    if (node->next != NULL) {
        traverse(node->next);
    }

}
```

Climbing up the tree would be:

```c
while (node != NULL) {
    /* process current node, then */
    node = node->parent;
}
```

Coming back to the original problem, searching this tree boils down to checking if the node’s ID matches or if any of it’s children have a match:

```c
int traverse(Node *node) {
    int match = 0, child_match = 0;

    if (id_in_array(node->id)) {
        match = 1;
    }

    if (node->first_child != NULL) {
        match |= traverse(node->first_child);
    }

    if (match) {
        add_node_to_result_list();
    }

    if (node->next != NULL) {
        match |= traverse(node->next);
    }

    return match;
}
```
