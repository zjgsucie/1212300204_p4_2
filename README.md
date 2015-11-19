# 1212300204_p4_2
焦雁飞 计科1201
When a reference is released, the call to kobject_put() will decrement the
reference count and, possibly, free the object. Note that kobject_init()
sets the reference count to one, so the code which sets up the kobject will
need to do a kobject_put() eventually to release that reference.
当一个引用被释放时，调用 kobject_put() 将递减引用计数，并且可能的话，释放对象。
请注意，kobject_init() 将引用计数设置为 1，所以在建立 kobject 的代码里需要执行
kobject_put() 用于最终释放引用。

Because kobjects are dynamic, they must not be declared statically or on
the stack, but instead, always allocated dynamically.  Future versions of
the kernel will contain a run-time check for kobjects that are created
statically and will warn the developer of this improper usage.
因为 kobject 是动态的，所以它们不能被声明为静态的或者在栈区分配空间，它们
应该始终被动态分配。为来的 kernel 版本将会包含一个对 kobject 是否静态创建的
运行时检查，并且会警告开发人员这种不正确的使用。

If all that you want to use a kobject for is to provide a reference counter
for your structure, please use the struct kref instead; a kobject would be
overkill.  For more information on how to use struct kref, please see the
file Documentation/kref.txt in the Linux kernel source tree.
如果你使用 kobject 的理由仅仅是使用引用计数的话，那么请使用 struct kref 替代
kobject。更多关于 struct kref 的信息请参考 Linux 内核文档 Documentation/kref.txt


Creating "simple" kobjects
创建“简单”的 kobject

Sometimes all that a developer wants is a way to create a simple directory
in the sysfs hierarchy, and not have to mess with the whole complication of
ksets, show and store functions, and other details.  This is the one
exception where a single kobject should be created.  To create such an
entry, use the function:
有时开发人员希望有一种创建一个在 sysfs 层次中简单目录的方式，而并不想搞乱本来
就错综复杂的 ksets，show 和 store 函数，或一些其他的细节。这是一个创建单独的
kobjects 的一个例外。要创建这样的条目，使用这个函数：

struct kobject *kobject_create_and_add(char *name, struct kobject *parent);

This function will create a kobject and place it in sysfs in the location
underneath the specified parent kobject.  To create simple attributes
associated with this kobject, use:
此函数将创建一个 kobject，并将其放置在指定的父 kobject 的 sysfs 中下层
的位置。要创建简单的与这个 kobject 关联的属性，使用函数：

int sysfs_create_file(struct kobject *kobj, struct attribute *attr);
or
int sysfs_create_group(struct kobject *kobj, struct attribute_group *grp);

Both types of attributes used here, with a kobject that has been created
with the kobject_create_and_add(), can be of type kobj_attribute, so no
special custom attribute is needed to be created.
这两种使用 kobject_create_and_add() 创建的 kobject 的属性类型都可以是
kobj_attribute，所以没有创建自定义属性的需要。

See the example module, samples/kobject/kobject-example.c for an
implementation of a simple kobject and attributes.
查看示例模块 samples/kobject/kobject-example.c，一个简单的 kobject
和属性的实现。



ktypes and release methods
ktypes 和 release 方法

One important thing still missing from the discussion is what happens to a
kobject when its reference count reaches zero. The code which created the
kobject generally does not know when that will happen; if it did, there
would be little point in using a kobject in the first place. Even
predictable object lifecycles become more complicated when sysfs is brought
in as other portions of the kernel can get a reference on any kobject that
is registered in the system.
到目前为止我们遗漏了一个重要的事情，那就是当一个 kobject 的引用计数达到 0 时将
会发生什么。通常，创建 kobject 的代码并不知道这种情况何时发生。如果它们知道何
时发生，那么把 kobject 放在结构体的首位可能会有那么一点点帮助。即使是一个可预
测的对象生命周期也将变得复杂，特别是在当 sysfs 作为 kernel 的一部分被引入时，它
可以获取到任意在系统中注册的 kobject 的引用。

The end result is that a structure protected by a kobject cannot be freed
before its reference count goes to zero. The reference count is not under
the direct control of the code which created the kobject. So that code must
be notified asynchronously whenever the last reference to one of its
kobjects goes away.
结论就是一个被 kobject 保护的结构体不能在这个 kobject 引用计数到 0 之前被释放。
而这个引用计数又不被创建这个 kobject 的代码所直接控制，所以必须在这个 kobject
的最后一个引用消失时异步通知这些代码。

Once you registered your kobject via kobject_add(), you must never use
kfree() to free it directly. The only safe way is to use kobject_put(). It
is good practice to always use kobject_put() after kobject_init() to avoid
errors creeping in.
一旦你通过 kobject_add() 注册你的 kobject 之后，永远也不要使用 kfree() 去释
放直接它。唯一安全的途径是使用 kobject_put()。总是在 kobject_init() 之后使
用 kobject_put() 是避免错误蔓延的很好的做法。

This notification is done through a kobject's release() method. Usually
such a method has a form like:
这个通知是通过 kobject 的 release() 函数完成的。该函数通常具有这样的形式：

void my_object_release(struct kobject *kobj)
{
   struct my_object *mine = container_of(kobj, struct my_object, kobj);

   kfree(mine);
}

One important point cannot be overstated: every kobject must have a
release() method, and the kobject must persist (in a consistent state)
until that method is called. If these constraints are not met, the code is
flawed.  Note that the kernel will warn you if you forget to provide a
release() method.  Do not try to get rid of this warning by providing an
"empty" release function; you will be mocked mercilessly by the kobject
maintainer if you attempt this.
很重要的一点怎么强调也不过分：每个 kobject 必须有一个 release() 方法，而且
在这个方法被调用之前 kobject 必须继续存在（保持一致的状态）。如果不符合这
些限制，那么代码是有缺陷的。需要注意的是，如果你忘记提供一个 release() 方法，
kernel 会警告你。不要试图提供一个“空”的 release 函数来摆脱这个警告，如果你
这样做你会受到 kobject 维护者们无情的嘲笑。


Note, the name of the kobject is available in the release function, but it
must NOT be changed within this callback.  Otherwise there will be a memory
leak in the kobject core, which makes people unhappy.
注意，尽管在 release 函数中 kobject 的 name 是可用的，但是千万不要在这个回调函
数中修改它。否则将会在 kobject core 中发生令人不愉快的内存泄露问题。

Interestingly, the release() method is not stored in the kobject itself;
instead, it is associated with the ktype. So let us introduce struct
kobj_type:
有趣的是 release() 方法并没有保存在 kobject 之中，而是关联在它的 ktype
成员中。让我们来介绍 struct kobj_type:

struct kobj_type {
    void (*release)(struct kobject *);
    const struct sysfs_ops *sysfs_ops;
    struct attribute **default_attrs;
};

This structure is used to describe a particular type of kobject (or, more
correctly, of containing object). Every kobject needs to have an associated
kobj_type structure; a pointer to that structure must be specified when you
call kobject_init() or kobject_init_and_add().
这个结构体是用来描述一个特定类型的 kobject（或者更确切的说，包含它的对象）。
每个 kobject 都需要一个关联的 kobj_type 结构体，当你调用 kobject_init() 或
kobject_init_and_add() 时必须指定一个指向 kobj_type 结构体的指针。

The release field in struct kobj_type is, of course, a pointer to the
release() method for this type of kobject. The other two fields (sysfs_ops
and default_attrs) control how objects of this type are represented in
sysfs; they are beyond the scope of this document.
当然 struct kobj_type 中的 release 字段就是一个指向同类型对象 release() 方法的
函数指针。另外两个字段（sysfs_ops 和 default_attrs）是用来控制这些类型的对象在
sysfs 里是如何表现的，这已经超出了本文的讨论范围。

The default_attrs pointer is a list of default attributes that will be
automatically created for any kobject that is registered with this ktype.
default_attrs 成员指针是一个在任何属于这个 ktype 的 kobject 注册时自动创
建的默认属性列表。


ksets

A kset is merely a collection of kobjects that want to be associated with
each other.  There is no restriction that they be of the same ktype, but be
very careful if they are not.
kset 仅仅是一个需要相互关联的 kobject 集合。在这里没有任何规定它们必须是同
样的 ktype，但如果它们不是一样的 ktype 则一定要小心处理。

A kset serves these functions:
一个 kset 提供以下功能：

 - It serves as a bag containing a group of objects. A kset can be used by
   the kernel to track "all block devices" or "all PCI device drivers."
－ 它就像一个装有一堆对象袋子。kernel 可以用一个 kset 来跟踪像“所有的
   块设备”或者“所有的 PCI 设备驱动”这样的东西。

 - A kset is also a subdirectory in sysfs, where the associated kobjects
   with the kset can show up.  Every kset contains a kobject which can be
   set up to be the parent of other kobjects; the top-level directories of
   the sysfs hierarchy are constructed in this way.
－ 一个 kset 也是一个 sysfs 里的子目录，该目录里能够看见这些相关的 kobject。
   每个 kset 都另外包含一个 kobject，这个 kobject 可以用来设置成其他 kobjects
   的 parent。sysfs 层次结构中的顶层目录就是通过这样的方法构建的。

 - Ksets can support the "hotplugging" of kobjects and influence how
   uevent events are reported to user space.
－ kset 还可以支持 kobject 的“热插拔”，并会影响 uevent 事件如何报告给用户空间。

In object-oriented terms, "kset" is the top-level container class; ksets
contain their own kobject, but that kobject is managed by the kset code and
should not be manipulated by any other user.
以面向对象的观点来看，“kset” 是一个顶层容器类。kset 包含有它们自己的 kobject，
这个 kobject 是在 kset 代码管理之下的，而且不允许其他任何用户对其操作。

A kset keeps its children in a standard kernel linked list.  Kobjects point
back to their containing kset via their kset field. In almost all cases,
the kobjects belonging to a kset have that kset (or, strictly, its embedded
kobject) in their parent.
一个 kset 使用标准的 kernel 链表来保存它的 children。kobjects 通过它们的 kset
字段回指向包含它们的 kset。在几乎所有的情况下，属于某 kset 的 kobject 的
parent 都指向这个 kset（严格的说是嵌套进这个 kset 的 kobject）。

As a kset contains a kobject within it, it should always be dynamically
created and never declared statically or on the stack.  To create a new
kset use:
正是因为一个 kset 包含了一个 kobject，就应该始终动态创建这个 kset，千万
不要将其声明为静态的或在栈区分配空间。创建一个 kset 使用：

struct kset *kset_create_and_add(const char *name,
                                 struct kset_uevent_ops *u,
                                 struct kobject *parent);

When you are finished with the kset, call:
当你使用完一个 kset 时，调用这个函数：

void kset_unregister(struct kset *kset);
to destroy it.
来销毁它。

An example of using a kset can be seen in the
samples/kobject/kset-example.c file in the kernel tree.
内核树中的 samples/kobject/kset-example.c 文件是一个
有关于如何使用 kset 的示例。

If a kset wishes to control the uevent operations of the kobjects
associated with it, it can use the struct kset_uevent_ops to handle it:
如果一个 kset 希望控制那些与它相关联的 kobject 的 uevent 操作，可以使用
struct kset_uevent_ops 处理。

struct kset_uevent_ops {
    int (*filter)(struct kset *kset, struct kobject *kobj);
    const char *(*name)(struct kset *kset, struct kobject *kobj);
    int (*uevent)(struct kset *kset, struct kobject *kobj,
                  struct kobj_uevent_env *env);
};

The filter function allows a kset to prevent a uevent from being emitted to
userspace for a specific kobject.  If the function returns 0, the uevent
will not be emitted.
filter 函数允许 kset 阻止一个特定的 kobject 的 uevent 是否发送到用户空间。如果
这个函数返回 0，则 uevent 将不会被发出。

The name function will be called to override the default name of the kset
that the uevent sends to userspace.  By default, the name will be the same
as the kset itself, but this function, if present, can override that name.
name 函数用来重写那些将被 uevent 发送到用户空间的 kset 的默认名称。默认情况
下，这个名称应该和 kset 本身的名称相同，但这个函数的出现将可以改写这个名称。

The uevent function will be called when the uevent is about to be sent to
userspace to allow more environment variables to be added to the uevent.
uevent 函数会向即将被发送到用户空间的 uevent 中添加更多的环境变量。

One might ask how, exactly, a kobject is added to a kset, given that no
functions which perform that function have been presented.  The answer is
that this task is handled by kobject_add().  When a kobject is passed to
kobject_add(), its kset member should point to the kset to which the
kobject will belong.  kobject_add() will handle the rest.
有人可能会问，当一个 kobject 加入到一个 kset 时并没有提供完成这些功能的函数。
答案就是这些任务由 kobject_add() 来完成，kobject 的成员 kset 必须指向它将被
加入到的 kset，然后kobject_add() 会帮你干完剩下的活。

If the kobject belonging to a kset has no parent kobject set, it will be
added to the kset's directory.  Not all members of a kset do necessarily
live in the kset directory.  If an explicit parent kobject is assigned
before the kobject is added, the kobject is registered with the kset, but
added below the parent kobject.
如果一个属于某个 kset 的 kobject 没有设置它的 parent kobject，那么它将被
添加到 kset 的目录中去。但并不是 kset 的所有成员都一定存在于 kset 目录下。
如果在 kobject 被添加前就指明了它的 parent kobject， 那么该 kobject 将被
注册到这个 kset 下，然后添加到它的 parent kobject 下。


Kobject removal
kobject 的移除

After a kobject has been registered with the kobject core successfully, it
must be cleaned up when the code is finished with it.  To do that, call
kobject_put().  By doing this, the kobject core will automatically clean up
all of the memory allocated by this kobject.  If a KOBJ_ADD uevent has been
sent for the object, a corresponding KOBJ_REMOVE uevent will be sent, and
any other sysfs housekeeping will be handled for the caller properly.
当 kobject core 成功的注册了一个 kobject之后，这个 kobject 必须在代码完成对它的
使用时销毁。要做到这一点，请调用 kobject_put()。通过这个调用， kobject core 会
自动清理所有通过这个 kobject 分配的内存空间。如果曾经为了这个 kobject 发送过一个
KOBJ_ADD uevent，那么一个相应的 KOBJ_REMOVE uevent 将会被发送，并且任何
其他的 sysfs 维护者都将为这个调用作出相应的处理。

If you need to do a two-stage delete of the kobject (say you are not
allowed to sleep when you need to destroy the object), then call
kobject_del() which will unregister the kobject from sysfs.  This makes the
kobject "invisible", but it is not cleaned up, and the reference count of
the object is still the same.  At a later time call kobject_put() to finish
the cleanup of the memory associated with the kobject.
如果你需要分两步来删除一个 kobject 的话（也就是说在你需要销毁一个对象时不允
许 sleep），那么请使用 kobject_del() 来从 sysfs 注销这个 kobject。这将使得
kobject “不可见”，但是它并没有被清理，并且它的引用计数也没变。在稍后的时候
调用 kobject_put() 去完成对这个 kobject 相关的内存空间的清理。

kobject_del() can be used to drop the reference to the parent object, if
circular references are constructed.  It is valid in some cases, that a
parent objects references a child.  Circular references _must_ be broken
with an explicit call to kobject_del(), so that a release functions will be
called, and the objects in the former circle release each other.
如果建立了一个循环引用的话，kobject_del() 可以用来删除指向 parent 对象的
引用。当某个 parnet 对象又引用了它的 child 时，这是非常有效的。循环引用必
须显示的通过调用 kobject_del() 来打破，这样做之后将会调用一个 release 函数，
并且这些先前在循环引用中的对象才会彼此释放。


Example code to copy from
示例代码

For a more complete example of using ksets and kobjects properly, see the
example programs samples/kobject/{kobject-example.c,kset-example.c},
which will be built as loadable modules if you select CONFIG_SAMPLE_KOBJECT.
想要获得更完整的关于正确使用 kset 和 konject 的例子，请参考示例程序
samples/kobject/{kobject-example.c,kset-example.c}。可以通过选择编译条件
CONFIG_SAMPLE_KOBJECT 来把这些示例编译成可装载模块。
