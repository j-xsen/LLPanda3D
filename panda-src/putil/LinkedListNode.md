# LinkedListNode

**Source:** `panda/src/putil/linkedListNode.h` / `.I` / `.cxx` (`.cxx` is empty — everything is inline)
**Inherits:** none
**Inherited by:** none in putil itself — meant to be a `protected` base mixed into other classes elsewhere in the engine that need a hand-rolled intrusive doubly-linked list instead of an STL container

Stores just the `_prev`/`_next` pointers needed to make an object a node of a
doubly-linked, circular, intrusive list. Every node in the list — including
the list's own root/sentinel — is expected to inherit from this class. All
members are `protected`, so this is purely a base-class building block, not
a usable standalone type.

## Behavior notes

- **Circular sentinel design.** The root of a list is constructed with
  `LinkedListNode(bool)` (the `bool` argument is unused, just a
  disambiguating tag), which sets `_prev = _next = this` — an empty list is
  a root node pointing to itself. Non-root nodes use the default
  constructor, which leaves `_prev`/`_next` at `nullptr` in debug builds
  (uninitialized in release) until inserted.
- **`is_on_list()` is `_next != nullptr`.** Because a root node's `_next` is
  never `nullptr` (it always points to itself when empty), `is_on_list()`
  "generally appears to always be a member of itself" for root nodes — the
  check is really only meaningful for a plain member node, not the root.
- **No implicit removal in the destructor.** `~LinkedListNode()` only
  asserts the node is either fully detached (`_next == _prev == nullptr`)
  or is an empty self-referencing root (`_next == _prev == this`) — it does
  *not* call `remove_from_list()` for you. A derived class that destructs
  while still linked into a list will trip an assert (debug) or corrupt the
  list (release). Callers must call `remove_from_list()` before the node's
  lifetime ends.
- **`insert_before()`/`insert_after()` require the node to currently be
  fully detached** (`_prev == _next == nullptr`) — you cannot move a node
  directly from one list position to another with a single call; remove it
  first.
- **Move constructor/move-assignment splice the moved-from node's list
  position onto the new object**, correctly fixing up the neighbors'
  pointers and leaving `from` detached (`nullptr`/`nullptr`) — this lets a
  node-owning object be moved without manually re-linking.
- **`take_list_from(other_root)` bulk-transfers an entire other list onto
  this one** (except the other root itself), splicing `other_root`'s
  members in just before `this` in the current list, then resets
  `other_root` back to an empty circular list. O(1) regardless of the other
  list's length.
- **Not thread-safe** — the header says so explicitly. Any derived class
  used from multiple threads must add its own mutex around every call into
  this base.

## API

All members are `protected` — only usable from a derived class's own code.

| Signature | Notes |
|---|---|
| `LinkedListNode()` | Plain node, not yet on a list |
| `LinkedListNode(bool)` | Constructs as an empty list root (self-circular) |
| `LinkedListNode(LinkedListNode&& from) noexcept` / `operator=(LinkedListNode&&)` | Splices this object into `from`'s list position |
| `bool is_on_list() const` | See caveat above re: root nodes |
| `void remove_from_list()` | Unlinks; must be called before a linked node is destroyed |
| `void insert_before(LinkedListNode* node)` / `insert_after(LinkedListNode* node)` | Node being inserted must currently be detached |
| `void take_list_from(LinkedListNode* other_root)` | Bulk-splice another list's members into this list, emptying the other |
| `LinkedListNode *_prev, *_next` | Raw pointers, directly accessible to derived classes |

## Usage

```cpp
class MyItem : public LinkedListNode {
public:
  MyItem() : LinkedListNode() {}
  ~MyItem() { if (is_on_list()) remove_from_list(); }
};

class MyList : public LinkedListNode {
public:
  MyList() : LinkedListNode(true) {}   // empty root
  void add(MyItem *item) { item->insert_after(this); }
};
```

## See also

[README.md](README.md)
