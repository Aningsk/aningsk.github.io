# 简介
在使用C语言写上层应用代码时，常需要list的数据结构；但普通C库中并不含有这方面内容，需要自己实现；而Linux内核中，有一套list的实现，使用起来也相对简洁优雅，可以拿到上层来使用。

不过，我常常记不清一些使用细节，而且内核中与list相关的内容也不完全在一个头文件中——其实不完全在同一个文件中，是很符合其中的设计理念——只不过，在写上层代码只想用list，就没有必要划分出好多list相关的头文件。（哈希表没有在这里保留。）

所以呢，为了方便以后使用，我把Linux内核中的list相关部分提到一个文件中（会附加在文章下面），并写写例程；这样自己就不用在每次想用时，因遗忘又重新研究一遍了。

# 开始使用list
通常在学校学习的链表，它的每一个节点中都是有数据存在的。而Linux list的使用不是这样：它是先定义一个 _HEAD_ ，这个链表头不会包含数据，仅提供结构——也体现了Linux内核的设计哲学啊。

所有访问整个链表的操作，都要拿到这个 _HEAD_。

## 定义方式
定义链表头，如下：

```c
LIST_HEAD(name);
```

这是一个在list.h中定义的宏，它会以 **_name_** 为名称，定义一个 struct list\_head的变量，我们就是用这个变量作为 _HEAD_ 。（如附录例程main.c所示）

## 数据结构中的使用
list自身不带有任何数据部分，它仅实现了两个指针，通过两个指针描述出双向链表。在使用list时，必须将 _struct list\_head_ 的对象（在这里称为**node**），作为**实际数据结构之内的一个元素**（例如main.c中定义的 struct some就是实际使用的数据结构体）。并且，在初始化实际数据结构时，我们不用管里面的**node**，只需专心处理数据（例如main.c中的函数 _set\_some\_num()_ 的内容）。

### 访问数据结构体 - list\_entry
如何在操作链表**node**时访问真正的数据呢？

可以使用宏：

```c
/**
 * list_entry - 获得包含此list node的结构体的指针
 * @ptr:	结构体中struct list_head的指针
 * @type:	结构体的类型
 * @member: 结构体中struct list_head的名字
 */
#define list_entry(ptr, type, member)
```

其真正的实现就是Linux内核中非常有名的 _container\_of()_ 啦。

### 向链表中添加成员 - list\_add
这个函数的两个参数都是 _struct list\_head_  （专注链表不管别的）

```c
/**
 * list_add - 添加新节点
 * @new: 新节点的node指针
 * @head: 代表整个链表的HEAD的指针
 */
void list_add(struct list_head *new, struct list_head *head)
```

如list.h中的注释， _list\_add()_ 加入的节点，有**先进后出**的特点（栈）； _list\_add\_tail()_ 加入的节点，有**先进先出**的特点（队列）。这个特点可以从例程main.c的运行结果看出来。

另外，也注意例程main.c中，将一个节点重新加入原位置的方法（主要是第二个参数 - 一个普通的**node**也可以当作_HEAD_使用。

### 删除节点 - list\_del
```c
/**
 * list_del - 删除新节点
 * @new: 要删除节点的node指针
 */
void list_del(struct list_head *entry)
```

### 遍历链表 - list\_for\_each
这个宏其实就是一个for语句的头，有好几个变种，详细参见list.h中的定义。

```c
/**
 * list_for_each	-	遍历列表
 * @pos:	一个可指向struct list_head的指针变量，作为游标
 * @head:	代表整个链表的HEAD的指针
 */
#define list_for_each(pos, head) 

/**
 * list_for_each_safe - 安全版的遍历列表
 * @pos:	一个可指向struct list_head的指针变量，作为游标
 * @n:		一个可指向struct list_head的指针变量，做临时保存
 * @head:	代表整个链表的HEAD的指针
 */
#define list_for_each_safe(pos, n, head)

/**
 * list_for_each_entry	-	遍历数据结构列表
 * @pos:	一个可指向数据结构类型的指针变量，作为游标
 * @head:	代表整个链表的HEAD的指针
 * @member:	数据结构中struct list_head的名字
 */
#define list_for_each_entry(pos, head, member)
```

使用 _list\_for\_each\_entry()_ 版本，会直接得到数据结构的指针，实际使用数据时更为直接。

使用方法如上文的注释，但关于安全版本还是要说明一下：在遍历列表时，如果有可能**删除节点**，就必须使用安全版本；否则将导致链表断裂，出现野指针。

### 剪断链表 - list\_cut\_position
将一个链表在某个节点处，剪断为两个链表。

```c
/**
 * list_cut_position - 剪断链表
 * @list: 将剪下的一串挂在新的HEAD的指针
 * @head: 原链表的HEAD的指针
 * @entry: 从原链表的哪一个位置剪断
 */
void list_cut_position(struct list_head *list,
		struct list_head *head, struct list_head *entry)
```

举个例子：  
list\_cut\_position(HEAD-new, HEAD, 3);  
原链表HEAD,5,4,3,2,1  
在3处剪断，得到的新链表为：HEAD-new,5,4,3  
剩下的旧链表为：HEAD,2,1  

### 合并链表 - list\_splice
将两个链表合入到其中的一个，我自己更建议使用 _list\_splice\_init()_ 版本。

```c
/**
 * list_splice - join two lists, this is designed for stacks
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
void list_splice(const struct list_head *list,
				struct list_head *head)
```

举个例子：  
HEAD-a,2,1  
HEAD-b,5,4,3  
list\_splice(HEAD-a, HEAD-b);  
得到合并后的是HEAD-b，其内容为：  
HEAD-b,5,4,3,2,1  

更建议使用 _list\_splice\_init()_ 版本的原因是：带有init的版本，会在合并后，将无用的HEAD-a置空；而 _list\_splice()_ 不会这样做；不置空的话，HEAD-a中的指针还保节点指向，如果不小心遍历了HEAD-a会有意外的结果。

# 写在最后
上文也只是说了最简单的使用list的方法，还有甚多API没有说明；但知道了这些，再看其他的也就很简单啦～！附录中也包含了list.h和main.c的代码，可供参考。

网上也有很好的说明：  
[Linux kernel中的list怎么使用][1] 侧重讲使用方法  
[linux kernel list详解][2] 侧重讲代码原理  
感谢两位博主的工作！  

好！赶在520结束前写完啦～！

# 附录

## list.h
头文件。

```c
/**
 * list.h
 * Come from linux-3.16.83 
 * 2020-05-15
 *
 * Sometimes, I wanna use list in user space program.
 * And Linux kernel list.h is a good choice. So I copy them,
 * and add other code from kernel to support list, such as
 * container_of().
 *
 * Of course, all these code is under GPLv2.
 */

#ifndef _LINUX_LIST_H
#define _LINUX_LIST_H

/* copy from include/linux/stddef.h */
#undef offsetof
#ifdef __compiler_offsetof
#define offsetof(TYPE, MEMBER)	__compiler_offsetof(TYPE, MEMBER)
#else
#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE *)0)->MEMBER)
#endif

/* copy from include/linux/kernel.h */
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({			\
	const typeof( ((type *)0)->member ) *__mptr = (ptr);	\
	(type *)( (char *)__mptr - offsetof(type,member) );})

/* copy from include/linux/poison.h */
/*
 * These are non-NULL pointers that will result in page faults
 * under normal circumstances, used to verify that nobody uses
 * non-initialized list entries.
 */
#define POISON_POINTER_DELTA 	0
#define LIST_POISON1  ((void *) 0x100 + POISON_POINTER_DELTA)
#define LIST_POISON2  ((void *) 0x200 + POISON_POINTER_DELTA)

/* copy from include/linux/types.h */
struct list_head {
	struct list_head *next, *prev;
};

/*
 * Simple doubly linked list implementation.
 *
 * Some of the internal functions ("__xxx") are useful when
 * manipulating whole lists rather than single entries, as
 * sometimes we already know the next/prev entries and we can
 * generate better code by using them directly rather than
 * using the generic single-entry routines.
 */

#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}

/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}

/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}


/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}

/*
 * Delete a list entry by making the prev/next entries
 * point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	prev->next = next;
}

/**
 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty() on entry does not return true after this, the entry is
 * in an undefined state.
 */
static inline void __list_del_entry(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
}

static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = LIST_POISON1;
	entry->prev = LIST_POISON2;
}

/**
 * list_replace - replace old entry by new one
 * @old : the element to be replaced
 * @new : the new element to insert
 *
 * If @old was empty, it will be overwritten.
 */
static inline void list_replace(struct list_head *old,
				struct list_head *new)
{
	new->next = old->next;
	new->next->prev = new;
	new->prev = old->prev;
	new->prev->next = new;
}

static inline void list_replace_init(struct list_head *old,
					struct list_head *new)
{
	list_replace(old, new);
	INIT_LIST_HEAD(old);
}

/**
 * list_del_init - deletes entry from list and reinitialize it.
 * @entry: the element to delete from the list.
 */
static inline void list_del_init(struct list_head *entry)
{
	__list_del_entry(entry);
	INIT_LIST_HEAD(entry);
}

/**
 * list_move - delete from one list and add as another's head
 * @list: the entry to move
 * @head: the head that will precede our entry
 */
static inline void list_move(struct list_head *list, struct list_head *head)
{
	__list_del_entry(list);
	list_add(list, head);
}

/**
 * list_move_tail - delete from one list and add as another's tail
 * @list: the entry to move
 * @head: the head that will follow our entry
 */
static inline void list_move_tail(struct list_head *list,
				  struct list_head *head)
{
	__list_del_entry(list);
	list_add_tail(list, head);
}

/**
 * list_is_last - tests whether @list is the last entry in list @head
 * @list: the entry to test
 * @head: the head of the list
 */
static inline int list_is_last(const struct list_head *list,
				const struct list_head *head)
{
	return list->next == head;
}

/**
 * list_empty - tests whether a list is empty
 * @head: the list to test.
 */
static inline int list_empty(const struct list_head *head)
{
	return head->next == head;
}

/**
 * list_empty_careful - tests whether a list is empty and not being modified
 * @head: the list to test
 *
 * Description:
 * tests whether a list is empty _and_ checks that no other CPU might be
 * in the process of modifying either member (next or prev)
 *
 * NOTE: using list_empty_careful() without synchronization
 * can only be safe if the only activity that can happen
 * to the list entry is list_del_init(). Eg. it cannot be used
 * if another CPU could re-list_add() it.
 */
static inline int list_empty_careful(const struct list_head *head)
{
	struct list_head *next = head->next;
	return (next == head) && (next == head->prev);
}

/**
 * list_rotate_left - rotate the list to the left
 * @head: the head of the list
 */
static inline void list_rotate_left(struct list_head *head)
{
	struct list_head *first;

	if (!list_empty(head)) {
		first = head->next;
		list_move_tail(first, head);
	}
}

/**
 * list_is_singular - tests whether a list has just one entry.
 * @head: the list to test.
 */
static inline int list_is_singular(const struct list_head *head)
{
	return !list_empty(head) && (head->next == head->prev);
}

static inline void __list_cut_position(struct list_head *list,
		struct list_head *head, struct list_head *entry)
{
	struct list_head *new_first = entry->next;
	list->next = head->next;
	list->next->prev = list;
	list->prev = entry;
	entry->next = list;
	head->next = new_first;
	new_first->prev = head;
}

/**
 * list_cut_position - cut a list into two
 * @list: a new list to add all removed entries
 * @head: a list with entries
 * @entry: an entry within head, could be the head itself
 *	and if so we won't cut the list
 *
 * This helper moves the initial part of @head, up to and
 * including @entry, from @head to @list. You should
 * pass on @entry an element you know is on @head. @list
 * should be an empty list or a list you do not care about
 * losing its data.
 *
 */
static inline void list_cut_position(struct list_head *list,
		struct list_head *head, struct list_head *entry)
{
	if (list_empty(head))
		return;
	if (list_is_singular(head) &&
		(head->next != entry && head != entry))
		return;
	if (entry == head)
		INIT_LIST_HEAD(list);
	else
		__list_cut_position(list, head, entry);
}

static inline void __list_splice(const struct list_head *list,
				 struct list_head *prev,
				 struct list_head *next)
{
	struct list_head *first = list->next;
	struct list_head *last = list->prev;

	first->prev = prev;
	prev->next = first;

	last->next = next;
	next->prev = last;
}

/**
 * list_splice - join two lists, this is designed for stacks
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
static inline void list_splice(const struct list_head *list,
				struct list_head *head)
{
	if (!list_empty(list))
		__list_splice(list, head, head->next);
}

/**
 * list_splice_tail - join two lists, each list being a queue
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
static inline void list_splice_tail(struct list_head *list,
				struct list_head *head)
{
	if (!list_empty(list))
		__list_splice(list, head->prev, head);
}

/**
 * list_splice_init - join two lists and reinitialise the emptied list.
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 *
 * The list at @list is reinitialised
 */
static inline void list_splice_init(struct list_head *list,
				    struct list_head *head)
{
	if (!list_empty(list)) {
		__list_splice(list, head, head->next);
		INIT_LIST_HEAD(list);
	}
}

/**
 * list_splice_tail_init - join two lists and reinitialise the emptied list
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 *
 * Each of the lists is a queue.
 * The list at @list is reinitialised
 */
static inline void list_splice_tail_init(struct list_head *list,
					 struct list_head *head)
{
	if (!list_empty(list)) {
		__list_splice(list, head->prev, head);
		INIT_LIST_HEAD(list);
	}
}

/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

/**
 * list_first_entry - get the first element from a list
 * @ptr:	the list head to take the element from.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_first_entry(ptr, type, member) \
	list_entry((ptr)->next, type, member)

/**
 * list_last_entry - get the last element from a list
 * @ptr:	the list head to take the element from.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_last_entry(ptr, type, member) \
	list_entry((ptr)->prev, type, member)

/**
 * list_first_entry_or_null - get the first element from a list
 * @ptr:	the list head to take the element from.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 *
 * Note that if the list is empty, it returns NULL.
 */
#define list_first_entry_or_null(ptr, type, member) \
	(!list_empty(ptr) ? list_first_entry(ptr, type, member) : NULL)

/**
 * list_next_entry - get the next element in list
 * @pos:	the type * to cursor
 * @member:	the name of the list_struct within the struct.
 */
#define list_next_entry(pos, member) \
	list_entry((pos)->member.next, typeof(*(pos)), member)

/**
 * list_prev_entry - get the prev element in list
 * @pos:	the type * to cursor
 * @member:	the name of the list_struct within the struct.
 */
#define list_prev_entry(pos, member) \
	list_entry((pos)->member.prev, typeof(*(pos)), member)

/**
 * list_for_each	-	iterate over a list
 * @pos:	the &struct list_head to use as a loop cursor.
 * @head:	the head for your list.
 */
#define list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

/**
 * list_for_each_prev	-	iterate over a list backwards
 * @pos:	the &struct list_head to use as a loop cursor.
 * @head:	the head for your list.
 */
#define list_for_each_prev(pos, head) \
	for (pos = (head)->prev; pos != (head); pos = pos->prev)

/**
 * list_for_each_safe - iterate over a list safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_safe(pos, n, head) \
	for (pos = (head)->next, n = pos->next; pos != (head); \
		pos = n, n = pos->next)

/**
 * list_for_each_prev_safe - iterate over a list backwards safe against removal of list entry
 * @pos:	the &struct list_head to use as a loop cursor.
 * @n:		another &struct list_head to use as temporary storage
 * @head:	the head for your list.
 */
#define list_for_each_prev_safe(pos, n, head) \
	for (pos = (head)->prev, n = pos->prev; \
	     pos != (head); \
	     pos = n, n = pos->prev)

/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_first_entry(head, typeof(*pos), member);	\
	     &pos->member != (head);					\
	     pos = list_next_entry(pos, member))

/**
 * list_for_each_entry_reverse - iterate backwards over list of given type.
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry_reverse(pos, head, member)			\
	for (pos = list_last_entry(head, typeof(*pos), member);		\
	     &pos->member != (head); 					\
	     pos = list_prev_entry(pos, member))

/**
 * list_prepare_entry - prepare a pos entry for use in list_for_each_entry_continue()
 * @pos:	the type * to use as a start point
 * @head:	the head of the list
 * @member:	the name of the list_struct within the struct.
 *
 * Prepares a pos entry for use as a start point in list_for_each_entry_continue().
 */
#define list_prepare_entry(pos, head, member) \
	((pos) ? : list_entry(head, typeof(*pos), member))

/**
 * list_for_each_entry_continue - continue iteration over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Continue to iterate over list of given type, continuing after
 * the current position.
 */
#define list_for_each_entry_continue(pos, head, member) 		\
	for (pos = list_next_entry(pos, member);			\
	     &pos->member != (head);					\
	     pos = list_next_entry(pos, member))

/**
 * list_for_each_entry_continue_reverse - iterate backwards from the given point
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Start to iterate over list of given type backwards, continuing after
 * the current position.
 */
#define list_for_each_entry_continue_reverse(pos, head, member)		\
	for (pos = list_prev_entry(pos, member);			\
	     &pos->member != (head);					\
	     pos = list_prev_entry(pos, member))

/**
 * list_for_each_entry_from - iterate over list of given type from the current point
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Iterate over list of given type, continuing from current position.
 */
#define list_for_each_entry_from(pos, head, member) 			\
	for (; &pos->member != (head);					\
	     pos = list_next_entry(pos, member))

/**
 * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_first_entry(head, typeof(*pos), member),	\
		n = list_next_entry(pos, member);			\
	     &pos->member != (head); 					\
	     pos = n, n = list_next_entry(n, member))

/**
 * list_for_each_entry_safe_continue - continue list iteration safe against removal
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Iterate over list of given type, continuing after current point,
 * safe against removal of list entry.
 */
#define list_for_each_entry_safe_continue(pos, n, head, member) 		\
	for (pos = list_next_entry(pos, member), 				\
		n = list_next_entry(pos, member);				\
	     &pos->member != (head);						\
	     pos = n, n = list_next_entry(n, member))

/**
 * list_for_each_entry_safe_from - iterate over list from current point safe against removal
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Iterate over list of given type from current point, safe against
 * removal of list entry.
 */
#define list_for_each_entry_safe_from(pos, n, head, member) 			\
	for (n = list_next_entry(pos, member);					\
	     &pos->member != (head);						\
	     pos = n, n = list_next_entry(n, member))

/**
 * list_for_each_entry_safe_reverse - iterate backwards over list safe against removal
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 *
 * Iterate backwards over list of given type, safe against removal
 * of list entry.
 */
#define list_for_each_entry_safe_reverse(pos, n, head, member)		\
	for (pos = list_last_entry(head, typeof(*pos), member),		\
		n = list_prev_entry(pos, member);			\
	     &pos->member != (head); 					\
	     pos = n, n = list_prev_entry(n, member))

/**
 * list_safe_reset_next - reset a stale list_for_each_entry_safe loop
 * @pos:	the loop cursor used in the list_for_each_entry_safe loop
 * @n:		temporary storage used in list_for_each_entry_safe
 * @member:	the name of the list_struct within the struct.
 *
 * list_safe_reset_next is not safe to use in general if the list may be
 * modified concurrently (eg. the lock is dropped in the loop body). An
 * exception to this is if the cursor element (pos) is pinned in the list,
 * and list_safe_reset_next is called after re-taking the lock and before
 * completing the current iteration of the loop body.
 */
#define list_safe_reset_next(pos, n, member)				\
	n = list_next_entry(pos, member)

#endif

```

## main.c
一个简单的测试程序。

```c
#include <stdio.h>
#include <stdlib.h>
#include "list.h"

struct test {
	int num1;
	int num2;
};

struct some {
	struct list_head node;
	struct test t;
};

int set_some_num(struct some *s, int num1, int num2)
{
	if (NULL == s) {
		printf("INVAL as NULL\n");
		return -1;
	}

	s->t.num1 = num1;
	s->t.num2 = num2;
	return 0;
}

int main(int argc, char *argv[])
{
	/* example for container_of() */
	struct test test;
	struct test *p = NULL;

	test.num1 = 1;
	test.num2 = 2;

	p = container_of(&test.num2, struct test, num2);
	printf("p = %p\n", p);
	printf("&test = %p\n", &test);

	/* simple example for list */
	printf("\n");

	LIST_HEAD(list);
	LIST_HEAD(list2);
	struct some *some1 = NULL;
	struct some *some2 = NULL;
	struct some *some3 = NULL;
	struct some *some4 = NULL;
	struct some *some5 = NULL;

	some1 = malloc(sizeof(some1));
	some2 = malloc(sizeof(some2));
	some3 = malloc(sizeof(some3));
	some4 = malloc(sizeof(some4));
	some5 = malloc(sizeof(some5));
	if (NULL == some1 || NULL == some2 ||
			NULL == some3 || NULL == some4 ||
			NULL == some5) {
		printf("malloc fail\n");
		return -1;
	}

	set_some_num(some1, 1, 2);
	set_some_num(some2, 2, 4);
	set_some_num(some3, 3, 6);
	set_some_num(some4, 4, 8);
	set_some_num(some5, 5, 10);

	if(list_empty(&list))
		printf("TEST: list is empty\n");
	else
		printf("TEST: list is not empty\n");

	/*
	 * Use list_add()
	 * 		will FIRST IN LAST OUT with list_for_each().
	 * 		HEAD 5 4 3 2 1
	 * Use list_add_tail()
	 * 		will FIRST IN FIRST OUT with list_for_each().
	 * 		HEAD 1 2 3 4 5
	 */
	printf("TEST: add 1 ~ 5 nodes into list\n");
	list_add(&some1->node, &list);
	list_add(&some2->node, &list);
	list_add(&some3->node, &list);
	list_add(&some4->node, &list);
	list_add(&some5->node, &list);

	struct list_head *pos = NULL;
	struct some *some = NULL; 

	printf("\tlist_for_each():");
	list_for_each(pos, &list) {
		some = list_entry(pos, struct some, node);
		printf(" %d,", some->t.num1);
	}
	printf("\n");

	/*
	 * Use list_del() solely, is safe for the list.
	 * But if use list_del() in list_for_each(),
	 * that will cause list break.
	 * So must use list_for_each_safe() with one
	 * temporary storage pointer @n. 
	 */
	printf("TEST: del 3 in list_for_each_safe():");
	struct list_head *n = NULL;
	list_for_each_safe(pos, n, &list) {
		some = list_entry(pos, struct some, node);
		if (3 == some->t.num1)
			list_del(pos);
	}
	printf("\n");

	/*
	 * list_for_each_entry() is similar to list_for_each().
	 * The difference is it can get entry for you directly.
	 */
	printf("\tlist_for_each_entry():");
	list_for_each_entry(some, &list, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	/*
	 * We want to add 3 back into list,
	 * just use list_add().
	 * And take some4 as HEAD.
	 */
	printf("TEST: re-add 3 into list\n");
	list_add(&some3->node, &some4->node);

	printf("\tlist_for_each_entry():");
	list_for_each_entry(some, &list, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	/*
	 * Cut list at 3,
	 * then the new list will have 3,
	 * while the old list will not have 3.
	 */
	printf("TEST: cut the list at 3\n");
	list_cut_position(&list2, &list, &some3->node);

	printf("\tNew list2:");
	list_for_each_entry(some, &list2, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	printf("\tOld list after cut:");
	list_for_each_entry(some, &list, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	/*
	 * Splice two lists as one.
	 * list_splice(A, B) will insert A at B's HEAD.
	 * list_splice_tail(A, B) will insert A at B's tail,
	 *
	 * Notice: in list_splice(), the A's HEAD will not re-init,
	 * that will cause circle list at A
	 * (and B's HEAD will be one node).
	 * That's bad thing.
	 * So, I think, list_splice_init() should be used,
	 * as it will re-init A, then A will empty.
	 */
	printf("TEST: splice two list\n");
	list_splice_init(&list, &list2);

	printf("\tlist2:");
	list_for_each_entry(some, &list2, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	printf("\tlist:");
	list_for_each_entry(some, &list, node)
		printf(" %d,", some->t.num1);
	printf("\n");

	free(some5);
	free(some4);
	free(some3);
	free(some2);
	free(some1);
	return 0;
}

```

[1]:	https://www.cnblogs.com/hwy89289709/p/6754300.html
[2]:	https://blog.csdn.net/Msy3TU4dFuUZ4/article/details/78930020