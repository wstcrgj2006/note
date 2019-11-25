Java对象的分配，根据其过程，可以分为快速分配和慢速分配两种形式，其中快速分配使用无锁的指针碰撞技术在新生代的Eden区上进行分配，而慢速分配根据堆的实现方式、GC的实现方式、代的实现方式不同而具有不同的分配调用层次。下面以bytecodeInterpreter解释器对new指令的解释出发，分配实例对象的内存分配过程。

## 快速分配

[jvm/hotspot/src/share/vm/interpreter/**bytecodeInterpreter.cpp**]

```java
CASE(_new): {
u2 index = Bytes::get_Java_u2(pc+1);
ConstantPool* constants = istate->method()->constants();
// 验证类是否被解析过，在创建类的实例之前，必须确保该类型已经被正确解析和加载
if (!constants->tag_at(index).is_unresolved_klass()) {
  // Make sure klass is initialized and doesn't have a finalizer
  Klass* entry = constants->slot_at(index).get_klass();
  // 获取该类型在虚拟机中的表示InstanceKlass
  InstanceKlass* ik = InstanceKlass::cast(entry);
  // 如果该类已经被初始化并且可以被快速分配
  if (ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
    size_t obj_size = ik->size_helper();
    oop result = NULL;
    // If the TLAB isn't pre-zeroed then we'll have to do it
    bool need_zero = !ZeroTLAB;
    // 根据UseTLAB决定是否使用TLAB技术（Thread Local Allocation Buffers）
    if (UseTLAB) {
      // 若使用TLAB，则将分配工作交由线程自行完成
      // TLAB是每个线程在Java堆中预先分配了一小块内存，当有对象创建请求内存分配时，就会在该内存上进行分配，而不需要在Eden区通过同步控制进行内存分配
      result = (oop) THREAD->tlab().allocate(obj_size);
    }
    // Disable non-TLAB-based fast-path, because profiling requires that all
    // allocations go through InterpreterRuntime::_new() if THREAD->tlab().allocate
    // returns NULL.
#ifndef CC_INTERP_PROFILE
    // 如果不使用TLAB或在TLAB上分配失败，则会尝试在堆的Eden区上进行分配
    if (result == NULL) {
      need_zero = true;
      // Try allocate in shared eden
    retry:
      // Universe::heap()返回虚拟机内存体系所使用的CollectedHeap
      // top_addr()返回的是Eden区空闲快的起始地址变量_top的地址
      HeapWord* compare_to = *Universe::heap()->top_addr();
      // new_top􏰓􏰝􏰞􏱒􏲝􏲰􏲱􏲝􏰭􏰮􏰂􏰊􏲺􏰨􏰇􏲰􏲱􏲝􏲲􏲆􏲳􏲴为使用该块空闲快进行分配后新的空闲快起始地址
      HeapWord* new_top = compare_to + obj_size;
      // end_addr()返回的是Eden区空闲快的结束地址变量_end的地址
      if (new_top <= *Universe::heap()->end_addr()) {
        // 此处使用CAS操作进行空闲块的同步操作，即观察_top的预期值，若与compare_to相同，即没有其他线程操作该变量，
        // 则将new_top赋给_top真正成为新的空闲快起始地址值，这种分配技术叫做bump-the-pointer(指针碰撞技术)
        if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
          goto retry;
        }
        result = (oop) compare_to;
      }
    }
#endif
    if (result != NULL) {
      // Initialize object (if nonzero size and need) and then the header
      // 根据是否需要填0选项，堆分配空间的对象数据区进行填0
      if (need_zero ) {
        HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
        obj_size -= sizeof(oopDesc) / oopSize;
        if (obj_size > 0 ) {
          memset(to_zero, 0, obj_size * HeapWordSize);
        }
      }
      // 根据是否使用偏向锁，设置对象头信息，然后设置InstanceKlass引用（这样对象本身就获取了获取类型数据的途径）
      if (UseBiasedLocking) {
        result->set_mark(ik->prototype_header());
      } else {
        result->set_mark(markOopDesc::prototype());
      }
      result->set_klass_gap(0);
      result->set_klass(ik);
      // Must prevent reordering of stores for object initialization
      // with stores that publish the new object.
      OrderAccess::storestore();
      // 把对象地址引入栈，并继续执行下一个字节码
      SET_STACK_OBJECT(result, 0);
      UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
    }
  }
}
// 若该类型没有被解析，就会调用InterpreterRuntime的_new函数完成慢速分配
// Slow case allocation
CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
        handle_exception);
// Must prevent reordering of stores for object initialization
// with stores that publish the new object.
OrderAccess::storestore();
SET_STACK_OBJECT(THREAD->vm_result(), 0);
THREAD->set_vm_result(NULL);
UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
}
```

## 慢速分配

InterpreterRuntime的`_new`函数定义在`interpreterRuntime.cpp`中

[jvm/hotspot/src/share/vm/interpreter/**interpreterRuntime.cpp**]

```C++
IRT_ENTRY(void, InterpreterRuntime::_new(JavaThread* thread, ConstantPool* pool, int index))
  Klass* k_oop = pool->klass_at(index, CHECK);
  instanceKlassHandle klass (THREAD, k_oop);

  // Make sure we are not instantiating an abstract klass
  klass->check_valid_for_instantiation(true, CHECK);

  // Make sure klass is initialized
  klass->initialize(CHECK);

  // At this point the class may not be fully initialized
  // because of recursive initialization. If it is fully
  // initialized & has_finalized is not set, we rewrite
  // it into its fast version (Note: no locking is needed
  // here since this is an atomic byte write and can be
  // done more than once).
  //
  // Note: In case of classes with has_finalized we don't
  //       rewrite since that saves us an extra check in
  //       the fast version which then would call the
  //       slow version anyway (and do a call back into
  //       Java).
  //       If we have a breakpoint, then we don't rewrite
  //       because the _breakpoint bytecode would be lost.
  oop obj = klass->allocate_instance(CHECK);
  thread->set_vm_result(obj);
IRT_END
```

该函数进行了对象类的检查（确保不是抽象类）和对该类型进行初始化后，调用instanceKlassHandle的`allocate_instance`进行内存分配。

instanceKlassHandle类重载了成员访问运算符`->`，这里的一系列的成员方法的访问实际上是对InstanceKlass对象的访问。

[jvm/hotspot/src/share/vm/runtime/**handles.hpp**]

```C++
class instanceKlassHandle : public KlassHandle {
  ...
  InstanceKlass*       operator -> () const { return (InstanceKlass*)obj(); }
  ...
};
```

[jvm/hotspot/src/share/vm/oops/**instanceKlass.cpp**]

```C++
instanceOop InstanceKlass::allocate_instance(TRAPS) {
  // 检查是否设置了finalizer函数
  bool has_finalizer_flag = has_finalizer(); // Query before possible GC
  // 获取对象所需空间大小
  int size = size_helper();  // Query before forming handle.

  KlassHandle h_k(THREAD, this);

  instanceOop i;

  // 调用CollectedHeap的obj_allocate创建一个instanceOop（堆上的对象实例）
  i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
  // 根据情况注册finalizer函数
  if (has_finalizer_flag && !RegisterFinalizersAtInit) {
    i = register_finalizer(i, CHECK_NULL);
  }
  return i;
}
```

`CollectedHeap::obj_allocate`定义在`collectedHeap.hpp`中，它将转而调用内联函数`obj_allocate`：

```C++
inline static oop obj_allocate(KlassHandle klass, int size, TRAPS);
```

`obj_allocate`定义在`collectedHeap.inline.hpp`中：

[jvm/hotspot/src/share/vm/gc/shared/**collectedHeap.inline.hpp**]

```java
oop CollectedHeap::obj_allocate(KlassHandle klass, int size, TRAPS) {
  debug_only(check_for_valid_allocation_state());
  assert(!Universe::heap()->is_gc_active(), "Allocation during gc not allowed");
  assert(size >= 0, "int won't convert to size_t");
  HeapWord* obj = common_mem_allocate_init(klass, size, CHECK_NULL);
  post_allocation_setup_obj(klass, obj, size);
  NOT_PRODUCT(Universe::heap()->check_for_bad_heap_word_value(obj, size));
  return (oop)obj;
}
```

若当前处于GC状态，不允许进行内存分配申请，否则调用`common_mem_allocate_init`进行内存分配并返回获得内存的起始地址，随后调用`post_allocation_setup_obj`进行一些初始化工作。

`common_mem_allocate_init`分为两部分，分别调用`common_mem_allocate_noinit`进行内存空间的分配和调用`init_obj`进行对象空间的初始化。

```java
HeapWord* CollectedHeap::common_mem_allocate_noinit(KlassHandle klass, size_t size, TRAPS) {

  // Clear unhandled oops for memory allocation.  Memory allocation might
  // not take out a lock if from tlab, so clear here.
  CHECK_UNHANDLED_OOPS_ONLY(THREAD->clear_unhandled_oops();)

  if (HAS_PENDING_EXCEPTION) {
    NOT_PRODUCT(guarantee(false, "Should not allocate with exception pending"));
    return NULL;  // caller does a CHECK_0 too
  }

  HeapWord* result = NULL;
  // 若使用TLAB，则调用allocate_from_tlab尝试从TLAB分配内存，否则调用堆的mem_allocate尝试分配
  if (UseTLAB) {
    result = allocate_from_tlab(klass, THREAD, size);
    if (result != NULL) {
      assert(!HAS_PENDING_EXCEPTION,
             "Unexpected exception, will result in uninitialized storage");
      return result;
    }
  }
  bool gc_overhead_limit_was_exceeded = false;
  result = Universe::heap()->mem_allocate(size,
                                          &gc_overhead_limit_was_exceeded);
  if (result != NULL) {
    NOT_PRODUCT(Universe::heap()->
      check_for_non_bad_heap_word_value(result, size));
    assert(!HAS_PENDING_EXCEPTION,
           "Unexpected exception, will result in uninitialized storage");
    // 统计分配的字节数
    THREAD->incr_allocated_bytes(size * HeapWordSize);

    AllocTracer::send_allocation_outside_tlab_event(klass, size * HeapWordSize);

    return result;
  }

  // 若执行到此处，说明申请失败
  if (!gc_overhead_limit_was_exceeded) {
    // 若在申请过程中GC没有超时，则抛出OOM异常
    // -XX:+HeapDumpOnOutOfMemoryError and -XX:OnOutOfMemoryError support
    report_java_out_of_memory("Java heap space");

    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_JAVA_HEAP,
        "Java heap space");
    }

    THROW_OOP_0(Universe::out_of_memory_error_java_heap());
  } else {
    // -XX:+HeapDumpOnOutOfMemoryError and -XX:OnOutOfMemoryError support
    report_java_out_of_memory("GC overhead limit exceeded");

    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_JAVA_HEAP,
        "GC overhead limit exceeded");
    }

    THROW_OOP_0(Universe::out_of_memory_error_gc_overhead_limit());
  }
}
```

`init_obj`完成对对象内存空间的对齐和补充。

```java
void CollectedHeap::init_obj(HeapWord* obj, size_t size) {
  assert(obj != NULL, "cannot initialize NULL object");
  const size_t hs = oopDesc::header_size();
  assert(size >= hs, "unexpected object size");
  ((oop)obj)->set_klass_gap(0);
  Copy::fill_to_aligned_words(obj + hs, size - hs);
}
```

`hs`是对象头的大小，`fill_to_aligned_words`将对象空间除去对象头的部分做填0处理。该函数定义在`copy.hpp`中

[jvm/hotspot/src/share/vm/utilities/**copy.hpp**]

```java
static void fill_to_aligned_words(HeapWord* to, size_t count, juint value = 0) {
  assert_params_aligned(to);
  pd_fill_to_aligned_words(to, count, value);
}
```

接着调用`pd_fill_to_aligned_words`函数，此函数根据平台不同实现，以x86平台为例，该函数定义在`copy_x86.hpp`中，

[jvm/hotspot/src/cpu/x86/vm/**copy_x86.hpp**]

```java
static void pd_fill_to_aligned_words(HeapWord* tohw, size_t count, juint value) {
  pd_fill_to_words(tohw, count, value);
}
static void pd_fill_to_words(HeapWord* tohw, size_t count, juint value) {
#ifdef AMD64
  julong* to = (julong*) tohw;
  julong  v  = ((julong) value << 32) | value;
  while (count-- > 0) {
    *to++ = v;
  }
#else
  juint* to = (juint*)tohw;
  count *= HeapWordSize / BytesPerInt;
  while (count-- > 0) {
    *to++ = value;
  }
#endif // AMD64
}
```

该函数的作用就是先将地址类型转换，然后把堆的字节数转换为字节数，再对该段内存进行填0处理。

`post_allocation_setup_obj`调用了`post_allocation_setup_common`进行初始化工作，然后调用`post_allocation_notify`通知JVMTI和dtrace。

```java
void CollectedHeap::post_allocation_setup_obj(KlassHandle klass,
                                              HeapWord* obj_ptr,
                                              int size) {
  post_allocation_setup_common(klass, obj_ptr);
  oop obj = (oop)obj_ptr;
  assert(Universe::is_bootstrapping() ||
         !obj->is_array(), "must not be an array");
  // notify jvmti and dtrace
  post_allocation_notify(klass, obj, size);
}

void CollectedHeap::post_allocation_setup_common(KlassHandle klass,
                                                 HeapWord* obj_ptr) {
  post_allocation_setup_no_klass_install(klass, obj_ptr);
  oop obj = (oop)obj_ptr;
#if ! INCLUDE_ALL_GCS
  obj->set_klass(klass());
#else
  // Need a release store to ensure array/class length, mark word, and
  // object zeroing are visible before setting the klass non-NULL, for
  // concurrent collectors.
  obj->release_set_klass(klass());
#endif
}
```

`post_allocation_setup_no_klass_install`根据是否使用偏向锁，设置对象头信息等，即初始化`oop`的`_mark`字段。
