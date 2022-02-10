#### 浅析并发编程(五)synchronize源码分析

`synchronize`关键字大家并不陌生，是用来实现线程同步的锁。自`JDK1.6`版本起，`synchronize`引入偏向锁、轻量级锁再加上重量级锁，提升了性能，而本文中synchronize的源码是常用的`JDK1.8`版本。

1. 加锁流程

  当共享资源被同一线程访问，当前资源的`object header`会偏向当前线程，之后当该线程访问当前共享资源则不需要再加锁，这就是偏向锁存在的意义。在学习之前我们需要一点前置知识，

   ```c++
   //BasicObjectLock类中有两个变量_lock、_obj，而该类的实例对象就是我们认知中的Lock Record
   class BasicObjectLock VALUE_OBJ_CLASS_SPEC {
     friend class VMStructs;
    private:
     BasicLock _lock;
     oop       _obj; 
     ...
   }  
   ```

  `BasicObjectLock`类中`_lock`锁对应的类是`BasicLock`， 而`BasicLock`类中的变量是`_displaced_header`，对应的是类型是`markOop`该类在<B>浅析并发编程(三)对象布局</B>文章中是用来保存Mark Word，所以相应的，在该变量在后续的轻量级锁中，会用来保存共享对象object header的`Mark Word`。

   ```c++
   class BasicLock VALUE_OBJ_CLASS_SPEC {
     friend class VMStructs;
    private:
     volatile markOop _displaced_header;
     ...
     }
   ```

  `oop`类中的变量是_o，是指向对象头的指针。

   ```c++
   class oop {
     oopDesc* _o;
   }
   ```

  所以`BasicObjectLock`类的实例就是Lock Record，其中`_obj`会指向对象头，`_lock`可以用来存放Mark Word。之后我们找到以下代码行`src/share/vm/interpreter/bytecodeInterpreter.cpp:1816`，

   ```c++
   CASE(_monitorenter): {
       //获取锁对象，下文中Lock Record会指向当前锁对象，所以此处判空。
       oop lockee = STACK_OBJECT(-1);
       CHECK_NULL(lockee);    
       //BasicObjectLock的实例，在此处即为Lock Record
       //在此处monitor_base到stack_base存储的都是Lock Record，而此处是从stack_base开始到
       //monitor_base寻找一个新的Lock Record，用来指向当前锁对象。为了方便大家理解贴上图示：
       
       // Layout of C++ interpreter frame: (While executing in BytecodeInterpreter::run)
       //
       //                             <- SP (current esp/rsp)
       //    [local variables         ] BytecodeInterpreter::run local variables
       //    ...                        BytecodeInterpreter::run local variables
       //    [local variables         ] BytecodeInterpreter::run local variables
       //    [old frame pointer       ]   fp [ BytecodeInterpreter::run's ebp/rbp ]
       //    [return pc               ]  (return to frame manager)
       //    [interpreter_state*      ]  (arg to BytecodeInterpreter::run)   --------------
       //    [expression stack        ] <- last_Java_sp                           |
       //    [...                     ] * <- interpreter_state.stack              |
       //    [expression stack        ] * <- interpreter_state.stack_base         |
       //    [monitors                ]   \                                       |
       //     ...                          | monitor block size                   |
       //    [monitors                ]   / <- interpreter_state.monitor_base     |
       //    [struct interpretState   ] <-----------------------------------------|
       //    [return pc               ] (return to callee of frame manager [1]
       //    [locals and parameters   ]
       //                               <- sender sp
       BasicObjectLock* limit = istate->monitor_base();
       BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
       BasicObjectLock* entry = NULL;
       while (most_recent != limit ) {
           if (most_recent->obj() == NULL) entry = most_recent;
           else if (most_recent->obj() == lockee) break;
           most_recent++;
       }
       //当找到一个空的Lock Record，首先Lock Record指向当前锁对象。
       if (entry != NULL) {
           entry->set_obj(lockee);
           int success = false;
           //在继续看后面的代码之前，先一起来复习一下对象布局中的部分知识。
           //#define nth_bit(n)        (n >= BitsPerWord ? 0 : OneBit << (n))
   		//#define right_n_bits(n)   (nth_bit(n) - 1)
   		//#define left_n_bits(n)    (right_n_bits(n) << (n >= BitsPerWord ? 0 : 										(BitsPerWord - n)))
           //在64位版本中，BitsPerWord为64。在32位的版本中，BitsPerWord为32，所以nth_bit的返回值为		   //2^n (n<64), right_n_bits的方法的返回值为2^n-1 (n<64)。
           //再打开src/share/vm/oops/markOop.hpp:111的
           //
           // enum { age_bits                 = 4,
           // lock_bits                = 2,
           // biased_lock_bits         = 1,
           // max_hash_bits            = BitsPerWord - age_bits - lock_bits - 			                                       biased_lock_bits,
           // hash_bits                = max_hash_bits > 31 ? 31 : max_hash_bits,
           // cms_bits                 = LP64_ONLY(1) NOT_LP64(0),
           //epoch_bits               = 2
           //};
           
           //其中在Mark Word中age占4bits、lock占2bit、biased_lock_bits占1bit、max_hash_bits为
           //57bits、hash_bits占31bits、cms_bits在64位版本中占1bits、epoch_bits占2bits
           //在src/share/vm/oops/markOop.hpp:130
           //enum { lock_mask                = right_n_bits(lock_bits),                     
   	 	//	   lock_mask_in_place       = lock_mask << lock_shift,                     
   	 	//	   biased_lock_mask         = right_n_bits(lock_bits + biased_lock_bits), 
   	 	//	   biased_lock_mask_in_place= biased_lock_mask << lock_shift,             
   	 	//	   biased_lock_bit_in_place = 1 << biased_lock_shift,                     
   	 	//	   age_mask                 = right_n_bits(age_bits),                     
   	 	//	   age_mask_in_place        = age_mask << age_shift,                           	   //	  epoch_mask               = right_n_bits(epoch_bits),                   
   	    //       epoch_mask_in_place      = epoch_mask << epoch_shift,                 
   	 	//	   cms_mask                 = right_n_bits(cms_bits),                     
   	 	//	   cms_mask_in_place        = cms_mask << cms_shift                       
   		//	};
           
           //按照上述代码可以得知：(已按照Mark Word的位置进行排列)
           //lock_mask                                11
           //lock_mask_in_place                       11
           //biased_lock_mask                        111
           //biased_lock_mask_in_place               111
           //biased_lock_bit_in_place                100
           //age_mask                            1111
           //age_mask_in_place                   1111000
           //epoch_mask                       11
           //epoch_mask_in_place              1100000000
           //cms_mask                           1
           //cms_mask_in_place                  10000000
           //（如有不清楚可以和 浅析并发编程(三)对象布局两篇文章对照学习）
           
           //所以在此处epoch_mask_in_place为1100000000   
           uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;
           //获取锁对象的Mark Word
           markOop mark = lockee->mark();
           //参考src/share/vm/oops/markOop.hpp:157， 此处hash为0。
           intptr_t hash = (intptr_t) markOopDesc::no_hash;
           //锁对象开启了偏向锁模式，也就是判断锁对象Mark Word的后三位是否为101，也就是5。
           if (mark->has_bias_pattern()) {
               uintptr_t thread_ident;
               uintptr_t anticipated_bias_locking_value;
               thread_ident = (uintptr_t)istate->thread();
               //此处将锁对象的Klass中的Mark Word或运算当前线程ID之后，再异或锁对象Mark Word
               //然后与非1111000，也就是忽略age的影响。
               //而此处剩余的影响因素剩下：thread、epoch、baise_lock、lock
               anticipated_bias_locking_value =
                   (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) &
                   ~((uintptr_t) markOopDesc::age_mask_in_place);
               //当前值为0，表明锁对象和对应的Klass的Mark Word在忽略GC年龄的影响下二者相同。
               //也就是说当前锁对象已经偏向当前线程，此为轻量级锁重入现象，将success置为true即可。
               //没有后续操作，在此处达成偏向锁重入不需要再加锁的情况。
               if  (anticipated_bias_locking_value == 0) {
                   //默认false 用于打印偏向锁统计信息，参src/share/vm/runtime/globals.hpp:1290
                   if (PrintBiasedLockingStatistics) {
                       (* BiasedLocking::biased_lock_entry_count_addr())++;
                   }
                   success = true;
               }
               //此处biased_lock_mask_in_place的值为 111 且与结果不为0，而锁对象的后三位为101，
               //anticipated_bias_locking_value的后三位是Klass的Mark Word和锁对象异或的结果，
               //也就是说Klass的Mark Word后三位不是101，也就是说Klass不是偏向锁模式，而锁对象是偏向锁
               //模式，此处尝试撤销偏向锁。
               else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
                   //获取Klass的Mark Word
                   markOop header = lockee->klass()->prototype_header();
                   if (hash != markOopDesc::no_hash) {
                       header = header->copy_set_hash(hash);
                   }
                   //此处是CAS算法，检查当前锁对象的Mark Word是否还是 过去的Mark Word快照版本，
                   //相同则让当前锁对象等于Klass的Mark Word，又因为Klass的Mark Word不是偏向锁模式
                   //所以实际上当锁对象是偏向锁模式，但是Klass的Mark Word不是，采用CAS方式撤销偏向				 //锁。
                   if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
                       if (PrintBiasedLockingStatistics)
                           (*BiasedLocking::revoked_lock_entry_count_addr())++;
                   }
               }
               //此处类似，epoch_mask_in_place的值为1100000000，且判断语句下与结果不为0，
               //而anticipated_bias_locking_value的epoch值是锁对象和Klass的异或的结果，
               //也就是说锁对象的epoch值和Klass的epoch的值不同，而批量重偏向情况下，epoch的值才会发生             //改变，所以就是说 当偏向锁过期了，此时需要重偏向。
               else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
                   //得到偏向当前线程的Mark Word。
                   markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
                   if (hash != markOopDesc::no_hash) {
                       new_header = new_header->copy_set_hash(hash);
                   }
                   //此处雷同，使用CAS尝试将锁对象的Mark Word赋值为偏向当前线程的Mark Word。
                   if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
                       if (PrintBiasedLockingStatistics)
                           (* BiasedLocking::rebiased_lock_entry_count_addr())++;
                   }
                   //锁对象的Mark Word发生变化，当前出现线程竞争的情况，进入monitorenter方法
                   else {
                       CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
                   }
                   success = true;
            }
               else {
                   //代码能走到这个位置，所以只能是anticipated_bias_locking_value的threadID出现				  //了问题，并且锁对象没有偏向当前线程，此时尝试偏向当前线程。
                   //因为biased_lock_mask_in_place为111，age_mask_in_place为1111000，
                   //epoch_mask_in_place为1100000000，所以header的结果就是
                   //锁对象的mark Word & 1101111111，结果也就是忽略CMS的影响下的锁对象的Mark Word
                   //此时header的值 还有高位的54bit用于存放thread_id没有赋值。
                   markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |
                                                                   (uintptr_t)markOopDesc::age_mask_in_place |
                                                                   epoch_mask_in_place));
                   if (hash != markOopDesc::no_hash) {
                       header = header->copy_set_hash(hash);
   
                   }
                   //变量new_header已偏向当前线程
                   markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
                   //用于DEBUG调试使用
                   DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
                       //CAS方式将偏向当前线程的以及忽略了CMS影响的锁对象的Mark Word赋值给当前锁对象
                       if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
                           if (PrintBiasedLockingStatistics)
                               (* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
                       }
                   else {
                       //锁对象的Mark Word不等于之前的header的快照信息，说明出现线程竞争，调用
                       //monitorenter方法
                       CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
                   }
                   success = true;
               }
           }
   
           //当开启偏向锁时，锁对象已经偏向当前线程，或者epoch过期进行重偏向，又或者锁对象没有偏向当前		   //线程，都会使得success置为true。反之，只有Klass关闭了偏向模式，导致偏向锁撤销，或者锁对象没        //有开启偏向锁，代码就会走到这里。此处将进入轻量级锁模式。
           if (!success) {
               //将锁对象的Mark Word和unlocked_value也就是001，进行或运算之后得到变量displaced
               //的未加锁状态的Mark Word，也就是常规状态下的Mark Word。之后将该Mark Word保存在
               //Lock Record的_displaced_header变量中。
               markOop displaced = lockee->mark()->set_unlocked();
               entry->lock()->set_displaced_header(displaced);
               //UseHeavyMonitors默认为false，参考src/share/vm/runtime/globals.hpp:2708
               bool call_vm = UseHeavyMonitors;
               //将锁对象的Mark Word指向当前Lock Record的指针。
               if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
                   //此处为轻量级锁重入的现象，将Lock Record的_displaced_header变量置为NULL，
                   //所以对于轻量级锁来说，只有从interpreter_state.stack_base自上而下方向到
                   //interpreter_state.monitor_base中，只有第一个Lock Record的                               //_displaced_header保存着锁对象的Mark Word，其他的Lock Record关于这个变量全部                 //为NULL
                   if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
                       entry->lock()->set_displaced_header(NULL);
                   } else {
                       //锁对象没有偏向当前线程，调用monitorenter
                       CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
                   }
               }
           }
           UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
       } else {
           //Lock Record不够，重新进入该方法
           istate->set_msg(more_monitors);
           UPDATE_PC_AND_RETURN(0);
       }
   }
   ```

  当出现线程竞争的情况就会进入到`InterpreterRuntime::monitorenter`方法，

   ```c++
   IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
   #ifdef ASSERT
     thread->last_frame().interpreter_frame_verify_monitor(elem);
   #endif
     if (PrintBiasedLockingStatistics) {
       Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
     }
     //创建名为h_obj的Handle类的构造方法，入参为当前线程和Lock Record的Object Header指针。
     Handle h_obj(thread, elem->obj());
     assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
            "must be NULL or an object");
     //开启偏向锁进入fast_enter方法避免无意义的锁膨胀，用于性能优化。
     if (UseBiasedLocking) {
       ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
     } else {
       ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
     }
     assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
            "must be NULL or an object");
   #ifdef ASSERT
     thread->last_frame().interpreter_frame_verify_monitor(elem);
   #endif
   IRT_END
   ```

   接着我们继续来看`ObjectSynchronizer::fast_enter`方法，

   ```c++
   void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
    //开启偏向锁校验，由于进入fast_enter之前已经校验偏向锁，此处为double-check   
    if (UseBiasedLocking) {
       //我们先来看src/share/vm/runtime/safepoint.hpp:60
       //  enum SynchronizeState {
       // _not_synchronized = 0,     在safepoint中线程未同步
       // _synchronizing    = 1,     线程同步中
       // _synchronized     = 2      在safepoint中所有Java线程全部暂停，此时只有VM虚拟机线程在运									  转。
       //SafepointSynchronize::is_at_safepoint()该方法判断状态是否为_synchronized，
       //此时Java线程全部暂停，只有VM线程正在运行。此处取反，Java线程正在运行，未到safepoint。 
       //所以此处revoke_and_rebias为Java线程调用 与此同时attempt_rebias 为true
       //出参的内容为：
       //  enum Condition {
       //     NOT_BIASED = 1,    //未偏向
       //     BIAS_REVOKED = 2,  //撤销偏向
       //     BIAS_REVOKED_AND_REBIASED = 3  //撤销并重偏向
       //  }; 
        
       //开启偏向锁之后，Java线程执行revoke_and_rebias，返回枚举的值为撤销并重偏向当前线程，执行结束。
       //其他情况进入锁膨胀的逻辑。 
       if (!SafepointSynchronize::is_at_safepoint()) {
         BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);  
         if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
           return;
         }
       } else {
         //Java线程暂停，VM线程可以运行，此时调用 revoke_at_safepoint  
         assert(!attempt_rebias, "can not rebias toward VM thread");
         BiasedLocking::revoke_at_safepoint(obj);
       }
       assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
    }
    //进入锁膨胀的逻辑
    slow_enter (obj, lock, THREAD) ;
   }
   ```



  2. 偏向锁的撤销和重偏向

   我们一起来看`BiasedLocking::revoke_and_rebias`方法，

   ```c++
   BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
     assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
   
   
     markOop mark = obj->mark();
     //当锁对象的是匿名偏向且attempt_rebias为false，例如: 调用System.identityHashCode生成         //identityHashCode，此时需要撤销偏向锁。而如果在生成identityHashCode之前，已经偏向某一个线程
     //除了撤销偏向锁之外，还需要膨胀为轻量级锁，这块我们到具体的代码再讲。  
     if (mark->is_biased_anonymously() && !attempt_rebias) {
       markOop biased_value       = mark;  
       //参考代码： 
       //  static markOop prototype() {
       //		return markOop( no_hash_in_place | no_lock_in_place );
     	//		}
       //no_hash_in_place表示Mark Word的hash位置的信息为0 之后或上001，之后再获取锁对象的GC年龄
       //这样变量unbiased_prototype的Mark Word信息高位为0，GC年龄信息，后三位为001
       //之后用CAS的方式，将锁对象的Mark Word赋值为unbiased_prototype，完成偏向锁的撤销
       markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
       markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
       if (res_mark == biased_value) {
         return BIAS_REVOKED;
       }
     //此处再次判断锁对象是否开启偏向锁，属于代码性能的优化的方式，避免直接进入slow_enter锁膨胀的逻辑    
     } else if (mark->has_bias_pattern()) {  
       Klass* k = obj->klass();
       markOop prototype_header = k->prototype_header();
       //这边的逻辑与刚进偏向锁的逻辑相似，锁对象对应Klass没有开启偏向锁，此时撤销偏向锁
       //使用CAS的方式将锁对象的Mark Word置为Klass的Mark Word  
       if (!prototype_header->has_bias_pattern()) {
         markOop biased_value       = mark;
         markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
         assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
         return BIAS_REVOKED;
       //当Klass的epoch值和当前锁对象的epoch值不一致时，也就是说明触发了偏向锁的批量重偏向，
       //也就是说epoch过期，此时如果attempt_rebias为true，重偏向开关打开，可以重偏向 
       } else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
         if (attempt_rebias) {
           assert(THREAD->is_Java_thread(), "");
           markOop biased_value       = mark;
           //重新生成偏向当前线程的 && epoch等于Klass epoch值 && 保留锁对象的GC年龄 &&后三位为101
           //的Mark Word，参考： src/share/vm/oops/markOop.hpp:313
           //并通过CAS的方式，将新的Mark Word赋值给锁对象，成功则返回BIAS_REVOKED_AND_REBIASED
           //对应的枚举值。参考fast_enter方法得知，当偏向锁的epoch过期，重新偏向当前线程，之后就可以提		  //前退出，从而避免进入slow_enter的方法。  
           markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());  
           markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
           if (res_mark == biased_value) {
             return BIAS_REVOKED_AND_REBIASED;
           }
         //相反，当epoch过期attempt_rebias为false，此时需要撤销偏向锁。
         //处理逻辑和匿名偏向锁撤销类似，不在赘述。    
         } else {
           markOop biased_value       = mark;
           markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
           markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
           if (res_mark == biased_value) {
             return BIAS_REVOKED;
           }
         }
       }
     }
     //我们先来看下，什么样的情况会走到这边的代码，当锁对象已经偏向线程A且epoch没有过期，也就是说
     //锁对象的epoch值等于Klass的Mark Word的epoch，也就是说没有出现偏向锁的批量重偏向，而在此时
     //线程B尝试获取锁。现在，我们继续往下看。update_heuristics该方法是探索性的方法，按照探索
     //的结果进行不同的处理。下文将会展开。  
     HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
     //无需偏向  
     if (heuristics == HR_NOT_BIASED) {
       return NOT_BIASED;
     //单个撤销    
     } else if (heuristics == HR_SINGLE_REVOKE) {
       Klass *k = obj->klass();
       markOop prototype_header = k->prototype_header();
       //当锁对象偏向的是当前线程，并且epoch值没有过期。也就是说需要撤销的是偏向当前线程的锁，
       //当前情况情况是为调用了System.identityHashCode来计算identityHashCode
       //由于偏向锁对应的Mark Word的高位存储的是ThreadID，此时需要撤销偏向锁，然后膨胀为
       //轻量级锁，此逻辑将在revoke_bias方法中实现，将在后文展开。 
       if (mark->biased_locker() == THREAD &&
           prototype_header->bias_epoch() == mark->bias_epoch()) {
         ResourceMark rm;
         if (TraceBiasedLocking) {
           tty->print_cr("Revoking bias by walking my own stack:");
         }
         EventBiasedLockSelfRevocation event;
         //当前入参中allow_rebias和is_bulk都为false，由于此处撤销的是偏向当前线程的偏向锁，所以
         //直接调用revoke_bias撤掉偏向锁，不需要交给VM线程异步执行。  
         BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD, NULL);
         ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
         assert(cond == BIAS_REVOKED, "why not?");
         if (event.should_commit()) {
           event.set_lockClass(k);
           event.commit();
         }
         return cond;
       } else {
         //此处需要单个撤销偏向锁，且锁对象没有偏向当前线程，此时使用构造方法生成VM_RevokeBias的对象
         //并交由VM虚拟机线程来执行。  
         EventBiasedLockRevocation event;
         VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
         VMThread::execute(&revoke);
         if (event.should_commit() && (revoke.status_code() != NOT_BIASED)) {
           event.set_lockClass(k);
           //保存相关信息
           event.set_safepointId(SafepointSynchronize::safepoint_counter() - 1);
           event.set_previousOwner(revoke.biased_locker());
           event.commit();
         }
         return revoke.status_code();
       }
     }
     //到此处，此时探索性的方法给出的结论是批量撤销，使用构造方法生成VM_BulkRevokeBias的实例对象
     //之后交由VM线程在safepoint，也就是在Java线程暂停的时候，进行批量撤销。  
     assert((heuristics == HR_BULK_REVOKE) ||
            (heuristics == HR_BULK_REBIAS), "?");
     EventBiasedLockClassRevocation event;
     VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                   (heuristics == HR_BULK_REBIAS),
                                   attempt_rebias);
     VMThread::execute(&bulk_revoke);
     if (event.should_commit()) {
       event.set_revokedClass(obj->klass());
       event.set_disableBiasing((heuristics != HR_BULK_REBIAS));
       event.set_safepointId(SafepointSynchronize::safepoint_counter() - 1);
       event.commit();
     }
     return bulk_revoke.status_code();
   }
   ```

   现在我们来看下`update_heuristics`这个探索性的方法的实现方式，

   ```c++
   static HeuristicsResult update_heuristics(oop o, bool allow_rebias) {
     markOop mark = o->mark();
     //当锁对象的后三位不是101，给出的结论是不偏向  
     if (!mark->has_bias_pattern()) {
       return HR_NOT_BIASED;
     }
   
     //在接着看以下代码之前，先来认识一下几个变量：
     //BiasedLockingBulkRebiasThreshold  20
     //BiasedLockingBulkRevokeThreshold  40
     //BiasedLockingDecayTime 25000ms
     //当偏向锁撤销计数器大于20 且小于40，并且当前时间距离上次批量撤销时间超过25000ms，也就是25秒，
     //当前不是第一次批量撤销，因为每次批量撤销都会记录当前批量撤销时间，将会重置批量撤销计数。 
     Klass* k = o->klass();
     jlong cur_time = os::javaTimeMillis();
     jlong last_bulk_revocation_time = k->last_biased_lock_bulk_revocation_time();
     int revocation_count = k->biased_lock_revocation_count();
     if ((revocation_count >= BiasedLockingBulkRebiasThreshold) &&
         (revocation_count <  BiasedLockingBulkRevokeThreshold) &&
         (last_bulk_revocation_time != 0) &&
         (cur_time - last_bulk_revocation_time >= BiasedLockingDecayTime)) {
       k->set_biased_lock_revocation_count(0);
       revocation_count = 0;
     }
   
     //当偏向锁的撤销计数器小于40，每次进入该探索方法都会自增计数一次
     if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
       revocation_count = k->atomic_incr_biased_lock_revocation_count();
     }
     //相应的，当偏向锁的撤销计数器达到40次，表示当前适合批量撤销
     if (revocation_count == BiasedLockingBulkRevokeThreshold) {
       return HR_BULK_REVOKE;
     }
     //而当偏向锁计数器小于40次，但是达到了20次表示当前适合批量重偏向
     if (revocation_count == BiasedLockingBulkRebiasThreshold) {
       return HR_BULK_REBIAS;
     }
     //当偏向计数器小于20次，表示当前适合单个撤销
     return HR_SINGLE_REVOKE;
   }
   ```

   我们先来捋一下探索性方法的逻辑：

   * 当前锁对象不是偏向锁模式，给出的结论为不偏向
   * 当偏向锁的撤销计数器小于20，表示当前适合单个撤销
   * 当偏向锁的撤销计数器达到20但是小于40，表示当前适合批量重偏向
   * 当偏向锁的撤销计数器达到40，表示当前适合批量撤销
   * 单次进入探索性方法，计数器会自增一
   * <B>当偏向锁的撤销次数大于20小于40 且距离上次批量撤销时间超过25秒，重置计数器为0</B>

我们继续看下`revoke_bias`的执行逻辑，

```c++
//方法的出参为Condition枚举，先来看一下:
//enum Condition {
//  NOT_BIASED = 1,
//  BIAS_REVOKED = 2,
//  BIAS_REVOKED_AND_REBIASED = 3
//};

static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread, JavaThread** biased_locker) {
  markOop mark = obj->mark();
  //当锁对象的Mark Word不是偏向锁模式， 则不需要撤销偏向锁 
  if (!mark->has_bias_pattern()) {
    if (TraceBiasedLocking) {
      ResourceMark rm;
      tty->print_cr("  (Skipping revocation of object of type %s because it's no longer biased)",
                    obj->klass()->external_name());
    }
    return BiasedLocking::NOT_BIASED;
  }
  //保存GC年龄
  uint age = mark->age();
  //biased_prototype后三位为101 且拥有锁对象的GC年龄，是为匿名偏向锁Mark Word
  //unbiased_prototype后三位为001，且拥有锁对象GC年龄，是为无锁模式的Mark Word  
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);

  if (TraceBiasedLocking && (Verbose || !is_bulk)) {
    ResourceMark rm;
    tty->print_cr("Revoking bias of object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s , prototype header " INTPTR_FORMAT " , allow rebias %d , requesting thread " INTPTR_FORMAT,
                  p2i((void *)obj), (intptr_t) mark, obj->klass()->external_name(), (intptr_t) obj->klass()->prototype_header(), (allow_rebias ? 1 : 0), (intptr_t) requesting_thread);
  }

  JavaThread* biased_thread = mark->biased_locker();
  //首先代码执行到这个位置，锁对象的后三位为101，当biased_thread == NULL，
  //表示当前为匿名偏向锁模式。如果allow_rebias为false，则不允许重偏向，锁对象Mark Word
  //置为无锁模式的Mark Word  
  if (biased_thread == NULL) {
    if (!allow_rebias) {
      obj->set_mark(unbiased_prototype);
    }
    if (TraceBiasedLocking && (Verbose || !is_bulk)) {
      tty->print_cr("  Revoked bias of anonymously-biased object");
    }
    return BiasedLocking::BIAS_REVOKED;
  }

  //判断当前线程是否存活：
  //1. 如果锁对象偏向锁偏向当前线程
  //2. 遍历所有线程，能够找到当前线程表示存活。  
  bool thread_is_alive = false;
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  } else {
    for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
      if (cur_thread == biased_thread) {
        thread_is_alive = true;
        break;
      }
    }
  }
  //如果当前线程不存活，allow_rebias== true 表示允许重偏向，对于偏向锁的撤销的逻辑转变为
  //将锁对象的Mark Word置为匿名偏向模式，否则当allow_rebias== false，不允许重偏向
  //则将锁对象置为无锁模式的Mark Word  
  if (!thread_is_alive) {
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      obj->set_mark(unbiased_prototype);
    }
    if (TraceBiasedLocking && (Verbose || !is_bulk)) {
      tty->print_cr("  Revoked bias of object biased toward dead thread");
    }
    return BiasedLocking::BIAS_REVOKED;
  }
  //如果当前线程存活，则需要遍历所有的MonitorInfo
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    //_owner指向当前线程表示 当前线程还在执行同步代码块的代码，
    //此时需要膨胀为轻量级锁，先将_displaced_header置为NULL，防止重入现象。
    //轻量级锁加锁的时候，最高位的Lock Record的_displaced_header变量为锁对象的Mark Word
    //其余该变量均为NULL，此处雷同。  
    if (mon_info->owner() == obj) {
      if (TraceBiasedLocking && Verbose) {
        tty->print_cr("   mon_info->owner (" PTR_FORMAT ") == obj (" PTR_FORMAT ")",
                      p2i((void *) mon_info->owner()),
                      p2i((void *) obj));
      }
        
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header( );
    } else {
      if (TraceBiasedLocking && Verbose) {
        tty->print_cr("   mon_info->owner (" PTR_FORMAT ") != obj (" PTR_FORMAT ")",
                      p2i((void *) mon_info->owner()),
                      p2i((void *) obj));
      }
    }
  }
  if (highest_lock != NULL) {
	//此处将_displaced_header置为无锁模式的Mark Word，用来保存锁对象的Mark Word信息
    //之后将锁对象的Mark Word设置为轻量级锁的最高位的Lock Record  
    highest_lock->set_displaced_header(unbiased_prototype);
    obj->release_set_mark(markOopDesc::encode(highest_lock));
    assert(!obj->mark()->has_bias_pattern(), "illegal mark state: stack lock used bias bit");
    if (TraceBiasedLocking && (Verbose || !is_bulk)) {
      tty->print_cr("  Revoked bias of currently-locked object");
    }
  } else {
    //highest_lock为空，表示当前线程存活，但是不在同步代码块中
    //如果allow_rebias == true，将锁对象MarkWord 置为匿名偏向锁
    //否则，直接将锁对象置为无锁状态  
    if (TraceBiasedLocking && (Verbose || !is_bulk)) {
      tty->print_cr("  Revoked bias of currently-unlocked object");
    }
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      obj->set_mark(unbiased_prototype);
    }
  }

#if INCLUDE_JFR
  if (biased_locker != NULL) {
    *biased_locker = biased_thread;
  }
#endif  
  return BiasedLocking::BIAS_REVOKED;
}
```

代码看到这个位置，我们再一起来看下`BiasedLocking::revoke_at_safepoint(Handle h_obj)`方法，在VM线程中偏向锁的撤销。

```c++
void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
  assert(SafepointSynchronize::is_at_safepoint(), "must only be called while at safepoint");
oop obj = h_obj();
   //探索性方法，用于分析当前情况适合哪种处理方式，查看之前代码，此处不再赘述。  
   HeuristicsResult heuristics = update_heuristics(obj, false);
   //单个撤销，调用revoke_bias来撤销偏向锁，相关代码已经分析，查看上文。  
   if (heuristics == HR_SINGLE_REVOKE) {
     revoke_bias(obj, false, false, NULL, NULL);
   //批量重偏向和批量撤销，调用bulk_revoke_or_rebias_at_safepoint代码    
   } else if ((heuristics == HR_BULK_REBIAS) ||
              (heuristics == HR_BULK_REVOKE)) {
     bulk_revoke_or_rebias_at_safepoint(obj, (heuristics == HR_BULK_REBIAS), false, NULL);
   }
   //遍历线程List清理所有Monitor缓存  
   clean_up_cached_monitor_info();
 }
```

 3. 批量撤销和批量重偏向

 接下来，我们重点关注`bulk_revoke_or_rebias_at_safepoint`批量撤销、批量重偏向的核心方法。

```c++
static BiasedLocking::Condition bulk_revoke_or_rebias_at_safepoint(oop o,
                                                                    bool bulk_rebias,
                                                                 bool attempt_rebias_of_object,
                                                                    JavaThread* requesting_thread) {
   assert(SafepointSynchronize::is_at_safepoint(), "must be done at safepoint");
 
   //默认false，参考：src/share/vm/runtime/globals.hpp:1417  
   if (TraceBiasedLocking) {
     tty->print_cr("* Beginning bulk revocation (kind == %s) because of object "
                   INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                   (bulk_rebias ? "rebias" : "revoke"),
                   p2i((void *) o), (intptr_t) o->mark(), o->klass()->external_name());
   }
   //保存当前时间为批量撤销时间，该值影响探索性方法的决策结果
   jlong cur_time = os::javaTimeMillis();
   o->klass()->set_last_biased_lock_bulk_revocation_time(cur_time);
 
 
   Klass* k_o = o->klass();
   Klass* klass = k_o;
   //批量重偏向的情况下，该值为true，相应的批量撤销的情况下，该值为false。
   //参考src/share/vm/runtime/biasedLocking.cpp:697  
   //当Klass的Mark Word具有偏向锁模式，bulk_rebias为true也就是批量重偏向的逻辑。
   //相应的，我们回顾之前的代码发现，当单次撤销次数达到20，触发重偏向。
   //在此处，自增Klass的Mark Word的epoch值。  
   if (bulk_rebias) {
     if (klass->prototype_header()->has_bias_pattern()) {
       int prev_epoch = klass->prototype_header()->bias_epoch();
       klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
       int cur_epoch = klass->prototype_header()->bias_epoch();
 
       //关于MonitorInfo 参考：src/share/vm/runtime/vframe.hpp:242
       //用于存放当前锁的所有者等信息。
       //此处遍历所有线程，对于每个线程的所有MonitorInfo数据，(owner->klass() == k_o) && mark-	  //>has_bias_pattern() 表示当前线程正在使用该偏向锁。遇到批量重偏向的操作，此时需要一同更新
       //相应的epoch值  
       for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
         GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
         for (int i = 0; i < cached_monitor_info->length(); i++) {
           MonitorInfo* mon_info = cached_monitor_info->at(i);
           oop owner = mon_info->owner();
           markOop mark = owner->mark();
           if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
             assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
             owner->set_mark(mark->set_bias_epoch(cur_epoch));
           }
         }
       }
     }
     //之后调用撤销偏向的核心方法
     revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread, NULL);
   } else {
      
     if (TraceBiasedLocking) {
       ResourceMark rm;
       tty->print_cr("* Disabling biased locking for type %s", klass->external_name());
     }
 
 	//此处为批量撤销偏向锁的逻辑，将Klass的Mark Word撤销偏向模式，此后对应该类的新的实例变为无锁模式，
     //和当前类现有实例一样，需要加锁的时候，会变成轻量级锁。 
     klass->set_prototype_header(markOopDesc::prototype());
 
     //开始遍历所有线程，对于每个线程的所有MonitorInfo数据，发现当前线程正在使用该偏向锁，
     //此时需要将其膨胀为轻量级锁，具体逻辑在revoke_bias核心代码中实现。  
     for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
       GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
       for (int i = 0; i < cached_monitor_info->length(); i++) {
         MonitorInfo* mon_info = cached_monitor_info->at(i);
         oop owner = mon_info->owner();
         markOop mark = owner->mark();
         if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
           revoke_bias(owner, false, true, requesting_thread, NULL);
         }
       }
     }
 
     //调用revoke_bias核心方法
     revoke_bias(o, false, true, requesting_thread, NULL);
   }
 
   if (TraceBiasedLocking) {
     tty->print_cr("* Ending bulk revocation");
   }
 
   BiasedLocking::Condition status_code = BiasedLocking::BIAS_REVOKED;
   //当attempt_rebias_of_object==true，且锁对象和Klass都具有偏向模式，
   //此时构造新的Mark Word，该Mark Word偏向当前的线程，表示撤销重偏向成功。  
   if (attempt_rebias_of_object &&
       o->mark()->has_bias_pattern() &&
       klass->prototype_header()->has_bias_pattern()) {
     markOop new_mark = markOopDesc::encode(requesting_thread, o->mark()->age(),
                                            klass->prototype_header()->bias_epoch());
     o->set_mark(new_mark);
     status_code = BiasedLocking::BIAS_REVOKED_AND_REBIASED;
     if (TraceBiasedLocking) {
       tty->print_cr("  Rebiased object toward thread " INTPTR_FORMAT, (intptr_t) requesting_thread);
     }
   }
 
   assert(!o->mark()->has_bias_pattern() ||
          (attempt_rebias_of_object && (o->mark()->biased_locker() == requesting_thread)),
          "bug in bulk bias revocation");
 
   return status_code;
 }
```

 4. 锁膨胀

 在一起来看slow_enter之前，我们先弄清楚什么样的情况，代码走到这里。在偏向锁或者轻量级锁加锁过程遇到线程竞争的时候，进入`InterpreterRuntime::monitorenter`方法，由于`UseBiasedLocking`默认为true，参考：`src/share/vm/runtime/globals.hpp:1284`，之后进入fast_enter，只有结果为`BIAS_REVOKED_AND_REBIASED`，偏向撤销重偏向的时候，表示虽然出现竞争，但是最后锁对象重偏向到当前线程。否则其他情况都会进入到`slow_enter`的逻辑。

```c++
   void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
     assert(!mark->has_bias_pattern(), "should not see bias pattern here");
  //表示当前锁对象为无锁模式，尝试轻量级锁的加锁，步骤如下：
     //1. 将Lock Record中的_displaced_header变量设置为锁对象的Mark Word，保存Mark Word信息。
     //2. 锁对象指向Lock Record  
     if (mark->is_neutral()) {
       lock->set_displaced_header(mark);
       if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
         TEVENT (slow_enter: release stacklock) ;
         return ;
       }
     } 
       //对于锁重入的情况，需要将_displaced_header置为NULL，此时只有第一个轻量级的_displaced_header
       //保存着锁对象的Mark Word信息。
       else    
     if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
       assert(lock != mark->locker(), "must not re-lock the same lock");
       assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
       lock->set_displaced_header(NULL);
       return;
     }
   
   #if 0
     //锁对象的后三位为010，也就是重量锁，并且当前线程已经用了Monitor锁，所以此处为加锁成功
     //可以提前早退。  
     if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
       lock->set_displaced_header (NULL) ;
       return ;
     }
   #endif
   
     //进入锁膨胀逻辑，inflate方法返回ObjectMonitor的实例对象，之后调用enter()方法。
     //enter()方法的逻辑，具体查看  浅析并发编程(四)Monitor  
     lock->set_displaced_header(markOopDesc::unused_mark());
     ObjectSynchronizer::inflate(THREAD,
                                 obj(),
                                 inflate_cause_monitor_enter)->enter(THREAD);
   }
```

   现在我们来看下锁膨胀的具体逻辑，其实该方法只是用来获取`ObjectMonitor`的实例对象，方便之后执行

   `enter()`核心方法。

   ```c++
   ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self,
                                                        oop object,
                                                        const InflateCause cause) {
        assert (Universe::verify_in_progress() ||
                !SafepointSynchronize::is_at_safepoint(), "invariant") ;
      
        EventJavaMonitorInflate event;
        //这是一个循环代码，出口分为两类：
        //1. 当前已经是Monitor锁
        //2. 成功由无锁或轻量级锁膨胀为Monitor锁  
        for (;;) {
            const markOop mark = object->mark() ;
            assert (!mark->has_bias_pattern(), "invariant") ;
            //参考：src/share/vm/oops/markOop.hpp:273
            //  bool has_monitor() const {
         	 //	 return ((value() & monitor_value) != 0);
        	 //  }
            //monitor_value是为10, 轻量级锁后三位为000，无锁状态为001，偏向锁状态为101,
            //而只有重量级锁为010，所以此处表明锁对象当前指向Monitor锁，也就是重量级锁。
            //所以此处无需再次膨胀，直接返回。
            if (mark->has_monitor()) {
                ObjectMonitor * inf = mark->monitor() ;
                assert (inf->header()->is_neutral(), "invariant");
                assert (inf->object() == object, "invariant") ;
                assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
                return inf ;
            }
      
   		 //参考：static markOop INFLATING() { return (markOop) 0; }
            //表示当前有其他线程尝试将锁对象膨胀为 Monitor锁，等待。
            if (mark == markOopDesc::INFLATING()) {
               TEVENT (Inflate: spin while INFLATING) ;
               ReadStableMark(object) ;
               continue ;
            }
      
            //参考以下代码：
            //  bool has_locker() const {
      		 //	 	return ((value() & lock_mask_in_place) == locked_value);
     		 //	 }
            //locked_value为0，当锁对象Mark Word的后两位为0，也就是当前是轻量级锁的情况。
            //当前为轻量级锁膨胀为Monitor锁的情况。
            if (mark->has_locker()) {
                //分配一个ObjectMonitor实例对象也就是Monitor对象，
                //具体逻辑参考：src/share/vm/runtime/synchronizer.cpp:971
                ObjectMonitor * m = omAlloc (Self) ;
                //_Responsible这个是假定继承人的变量，OwnerIsThread用于轻量级锁膨胀为重量级锁的
                //标志，_recursions是为Monitor锁重入次数的计数，
                //这些东西在浅析并发编程(四)Monitor 这篇有提过，此处不再赘述。
                m->Recycle();
                m->_Responsible  = NULL ;
                m->OwnerIsThread = 0 ;
                m->_recursions   = 0 ;
                m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;
                //首先将锁对象的Mark Word置为膨胀中，也就是0。此后其他线程遇到膨胀中，处于等待状态，
                //直到当前线程将锁对象完全膨胀为重量级锁。同样，当此处CAS失败，说明有其他线程正在锁膨胀，			 //当前作跳出处理。
                markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
                if (cmp != mark) {
                   omRelease (Self, m, true) ;
                   continue ;
                }
      
                //  markOop displaced_mark_helper() const {
       		 //		assert(has_displaced_mark_helper(), "check");
       		 //		intptr_t ptr = (value() & ~monitor_value);
       		 //		return *(markOop*)ptr;
     			 //	}
                //monitor_value是为10，此处dmw的后两位为01，是为无锁状态Mark Word
                //此时Monitor的header存放Mark Word，_owner指向Mark Word，
                //此为轻量级锁第一次膨胀为重量级锁的情况，之后当其他线程获取Monitor锁，
                //_owner会指向线程。
                markOop dmw = mark->displaced_mark_helper() ;
                assert (dmw->is_neutral(), "invariant") ;
                m->set_header(dmw) ;
                m->set_owner(mark->locker());
                m->set_object(object);
                //当ObjectMonitor实例对象被初始化结束之后，
                //将后锁对象的Mark Word的后两位更新为10,至此Monitor实例对象已经初始化完成
                //参考：src/share/vm/oops/markOop.hpp:309
                //之后直接返回。
                guarantee (object->mark() == markOopDesc::INFLATING(), "invariant") ;
                object->release_set_mark(markOopDesc::encode(m));
   
                if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
                TEVENT(Inflate: overwrite stacklock) ;
                if (TraceMonitorInflation) {
                  if (object->is_instance()) {
                    ResourceMark rm;
                    tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                      (void *) object, (intptr_t) object->mark(),
                      object->klass()->external_name());
                  }
                }
                if (event.should_commit()) {
                  post_monitor_inflate_event(&event, object, cause);
                }
                return m ;
            }
      
   
            //代码能执行到这个位置，自然排出Monitor锁、正在膨胀中、轻量级锁
            //由于偏向锁不会执行到这个位置，所以当前为无锁状态。
            //初始化ObjectMonitor的实例对象，之后使用CAS尝试使锁对象指向Monitor锁对象生成的
            //Mark Word信息，出现失败则说明出现线程竞争，重新执行逻辑。
            //直到成功后，返回Monitor对象。
            assert (mark->is_neutral(), "invariant");
            ObjectMonitor * m = omAlloc (Self) ;
            m->Recycle();
            m->set_header(mark);
            m->set_owner(NULL);
            m->set_object(object);
            m->OwnerIsThread = 1 ;
            m->_recursions   = 0 ;
            m->_Responsible  = NULL ;
            m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;
            if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr(), mark) != mark) {
                m->set_object (NULL) ;
                m->set_owner  (NULL) ;
                m->OwnerIsThread = 0 ;
                m->Recycle() ;
                omRelease (Self, m, true) ;
                m = NULL ;
                continue ;
            }
            
            if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
            TEVENT(Inflate: overwrite neutral) ;
            if (TraceMonitorInflation) {
              if (object->is_instance()) {
                ResourceMark rm;
                tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                  (void *) object, (intptr_t) object->mark(),
                  object->klass()->external_name());
              }
            }
            if (event.should_commit()) {
              post_monitor_inflate_event(&event, object, cause);
            }
            return m ;
        }
      }
   ```

   5. 解锁流程
   
      解锁的流程相比加锁来说，比较简单。对于偏向锁而言，只需要将Lock Record的`_obj`指向锁对象`object_header`，置为NULL即可。而对于轻量级锁而言，需要将`_obj`置为NULL，同时将`_displaced_header`中保存的Mark Word重新赋值给锁对象的Mark Word。对于重量级锁，将`ObjectMonitor`的`_owner`字段置为NULL，同时唤醒下一个假定继承人，具体可以在<B>浅析并发编程(四)Monitor</B>查看

   ```c++
   CASE(_monitorexit): {
     oop lockee = STACK_OBJECT(-1);
     CHECK_NULL(lockee);
     //对比加锁的流程来看，加锁的时候从stack_base到monitor_base寻找到一个空的Lock Record
     //也就是BasicObjectLock实例对象的_obj变量为NULL的情况，每次重入一次。就寻找一个新的Lock Record,      //在此处执行解锁的流程，直接遍历所有的Lock Record，找到非空的Lock Record，也就是BasicObjectLock
     //的_obj指向当前锁对象的情况，挨个释放。  
     BasicObjectLock* limit = istate->monitor_base();
     BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
     while (most_recent != limit ) {
       //找到非空的LockRecord，对于偏向锁的解锁很简单，直接将Lock Record的_obj置为NULL，也就是不再
       //指向锁对象的对象头信息。 
       if ((most_recent)->obj() == lockee) {
         BasicLock* lock = most_recent->lock();
         markOop header = lock->displaced_header();
         most_recent->set_obj(NULL);             
         //当锁对象不是偏向锁模式，也就是说当前是轻量级锁或者是重量级锁，如果变量header不为空，
         //也就是当前不是重入状态，只需要使用CAS将_displaced_header存储的Mark Word赋值给
         //锁对象的Mark Word。但是如果失败了，则说明锁对象没有指向_displaced_header
         //没有指向Lock Record。说明当该线程执行期间，由于其他线程出现竞争，
         //导致膨胀为重量级锁，导致锁对象指向ObjectMonitor实例对象，
         //又或者存在其他线程正在膨胀该锁，使得锁对象指向markOopDesc::INFLATING()
         //处于中间状态，都会导致CAS失败，此时进入InterpreterRuntime::monitorexit方法  
         if (!lockee->mark()->has_bias_pattern()) {
           bool call_vm = UseHeavyMonitors;
           if (header != NULL || call_vm) {
             if (call_vm || Atomic::cmpxchg_ptr(header, lockee->mark_addr(), lock) != lock) {
               most_recent->set_obj(lockee);
               CALL_VM(InterpreterRuntime::monitorexit(THREAD, most_recent), handle_exception);
             }
           }
         }
         UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
       }
       most_recent++;
     }
     CALL_VM(InterpreterRuntime::throw_illegal_monitor_state_exception(THREAD), handle_exception);
     ShouldNotReachHere();
   }
   ```

我们一起来看`InterpreterRuntime::monitorexit`方法

```c++
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (elem == NULL || h_obj()->is_unlocked()) {
    THROW(vmSymbols::java_lang_IllegalMonitorStateException());
  }
  //调用slow_exit
  ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
  //最后将Lock Record的_obj置空，释放Lock Record。
  elem->set_obj(NULL);
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```

继续来看`ObjectSynchronizer::slow_exit`方法实现，

```c++
void ObjectSynchronizer::slow_exit(oop object, BasicLock* lock, TRAPS) {
  fast_exit (object, lock, THREAD) ;
}
```

继续来看`ObjectSynchronizer::fast_exit`的实现，

```c++
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
  markOop dhw = lock->displaced_header();
  markOop mark ;
  //当_displaced_header为空表示重入现象，直接早退。 
  if (dhw == NULL) {
     mark = object->mark() ;
     assert (!mark->is_neutral(), "invariant") ;
     if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
        assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
     }
     if (mark->has_monitor()) {
        ObjectMonitor * m = mark->monitor() ;
        assert(((oop)(m->object()))->mark() == mark, "invariant") ;
        assert(m->is_entered(THREAD), "invariant") ;
     }
     return ;
  }

  mark = object->mark() ;

  //如果锁对象的指向_displaced_header表示当前为轻量级锁，使用CAS将锁对象的Mark Word置为
  //_displaced_header中保存的Mark Word。  
  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }
  //获取膨胀完成的ObjectMonitor对象，调用exit(),将ObjectMonitor对象_owner变量置为NULL
  //释放Monitor锁。  
  ObjectSynchronizer::inflate(THREAD,
                              object,
                              inflate_cause_vm_internal)->exit(true, THREAD);
}
```

synchronize的源码相对复杂，需要<B>浅析并发编程(三)对象布局</B>、<B>浅析并发编程(四)Monitor重量级锁</B>和本篇文章一同学习。