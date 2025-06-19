# axalloc

> documentation = "https://arceos-org.github.io/arceos/axalloc/index.html"

1. ## 架构概览

   - `axalloc` 采用双层分配器架构，结合了字节级分配器和页级分配器。这种设计的核心思想是：

     - **字节分配器 (balloc)**: 处理小块内存分配，支持任意大小的内存请求

     - **页分配器 (palloc)**: 管理页级内存，为字节分配器提供内存池

   - 在 ArceOS 运行时中，`axalloc` 的初始化过程如下

     ```rust
     fn init_allocator() {
         use axhal::mem::{MemRegionFlags, memory_regions, phys_to_virt};
     
         info!("Initialize global memory allocator...");
         info!("  use {} allocator.", axalloc::global_allocator().name());
     
         let mut max_region_size = 0;
         let mut max_region_paddr = 0.into();
         for r in memory_regions() {
             if r.flags.contains(MemRegionFlags::FREE) && r.size > max_region_size {
                 max_region_size = r.size;
                 max_region_paddr = r.paddr;
             }
         }
         for r in memory_regions() {
             if r.flags.contains(MemRegionFlags::FREE) && r.paddr == max_region_paddr {
                 axalloc::global_init(phys_to_virt(r.paddr).as_usize(), r.size);
                 break;
             }
         }
         for r in memory_regions() {
             if r.flags.contains(MemRegionFlags::FREE) && r.paddr != max_region_paddr {
                 axalloc::global_add_memory(phys_to_virt(r.paddr).as_usize(), r.size)
                     .expect("add heap memory region failed");
             }
         }
     }
     ```

     初始化流程：

     - 找到最大的可用内存区域作为主堆
     - 将其他内存区域添加为额外的内存池
     - 支持多个不连续内存区域的统一管理

2. ## 分配策略支持

   1. ### 算法选择机制

      1. `axalloc` 支持三种不同的字节分配策略，通过 Cargo features 进行选择

         ```rust
         cfg_if::cfg_if! {
             if #[cfg(feature = "slab")] {
                 /// The default byte allocator.
                 pub type DefaultByteAllocator = allocator::SlabByteAllocator;
             } else if #[cfg(feature = "buddy")] {
                 /// The default byte allocator.
                 pub type DefaultByteAllocator = allocator::BuddyByteAllocator;
             } else if #[cfg(feature = "tlsf")] {
                 /// The default byte allocator.
                 pub type DefaultByteAllocator = allocator::TlsfByteAllocator;
             }
         }
         ```

      2. 这些算法在 `axalloc` 中作为字节分配器使用，与页分配器配合工作

         ```rust
         pub struct GlobalAllocator {
             balloc: SpinNoIrq<DefaultByteAllocator>,
             palloc: SpinNoIrq<BitmapPageAllocator<PAGE_SIZE>>,
         }
         ```

   2. ### 算法介绍

      1. #### TLSF (Two-Level Segregated Fit): 实时性能优秀，适合对延迟敏感的场景

         - **核心思想**：TLSF 是一种实时内存分配器，设计目标是在 O(1) 时间内完成分配和释放操作，同时保持较低的内存碎片。

         - **工作原理**：

           - **两级索引：**
             - 第一级：按 2 的幂次方划分大小类别
             - 第二级：在每个大小类别内进一步细分
           - **位图索引**：使用位图快速定位可用的内存块
           - **立即合并**：释放时立即与相邻空闲块合并

         - **数据结构**：

           ```
           First Level Index (FLI): log2(block_size)  
           Second Level Index (SLI): 细分每个 FLI 类别  
           ```

         - **优势**：

           - **实时性能**：分配和释放都是 O(1) 时间复杂度
           - **低碎片**：通过精细的大小分类减少碎片
           - **可预测性**：性能不依赖于分配历史

         - **适用场景**：

           - 实时系统
           - 对延迟敏感的应用
           - 需要可预测性能的场景

      2. #### Slab: 适合固定大小对象的频繁分配/释放

         - **核心思想：**专门针对固定大小对象的频繁分配和释放进行优化。它预先分配一系列固定大小的内存块（称为 slab），每个 slab 包含多个相同大小的对象。通过预分配固定大小的内存池减少内部碎片，提升高频操作性能。

         - **工作原理**：

           - **预分配**：为常用的对象大小预先分配内存池
           - **快速分配**：从对应大小的 slab 中直接取出一个对象
           - **快速释放**：将对象直接放回对应的 slab
           - **缓存友好**：重复使用相同的内存位置，提高缓存命中率

         - **适用场景**：

           - 频繁分配/释放相同大小的对象
           - 对象生命周期较短的场景
           - 需要减少内存碎片的系统

           从依赖关系可以看出，slab_allocator 还依赖于 buddy_system_allocator 作为底层支持

      3. #### Buddy: 经典的伙伴系统，减少外部碎片

         - **核心思想**：Buddy 系统将内存按 2 的幂次方大小进行分割和合并，有效减少外部碎片。
         - **工作原理**：
           - **分割**：当需要分配内存时，找到最小的满足需求的 2^n 大小块
           - **合并**：释放时检查相邻的"伙伴"块是否空闲，如果是则合并成更大的块
           - **树状管理**：使用二叉树结构管理不同大小的空闲块
         - **优势**：
           - 有效减少外部碎片
           - 分配和释放的时间复杂度为 O(log n)
           - 实现相对简单
         - **缺点**：
           - 内部碎片较大（最坏情况下浪费近 50% 内存）
           - 只能分配 2 的幂次方大小的内存块

   3. ### 性能对比

      | 算法      | 分配时间 | 释放时间 | 内存碎片       | 适用场景             |
      | --------- | -------- | -------- | -------------- | -------------------- |
      | **SLAB**  | O(1)     | O(1)     | 低（固定大小） | 固定大小对象频繁分配 |
      | **Buddy** | O(log n) | O(log n) | 中等           | 通用场景，简单实现   |
      | **TLSF**  | O(1)     | O(1)     | 低             | 实时系统，低延迟要求 |

3. ## GlobalAlloc 实现

   1. `axalloc` 实现了 Rust 标准的 `GlobalAlloc` trait，使其能够作为全局分配器

      ```rust
      unsafe impl GlobalAlloc for GlobalAllocator {
          unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
              if let Ok(ptr) = GlobalAllocator::alloc(self, layout) {
                  ptr.as_ptr()
              } else {
                  alloc::alloc::handle_alloc_error(layout)
              }
          }
      
          unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
              GlobalAllocator::dealloc(self, NonNull::new(ptr).expect("dealloc null ptr"), layout)
          }
      }
      ```

   2. 通过 `#[global_allocator]` 属性注册为系统默认分配器

      ```rust
      #[cfg_attr(all(target_os = "none", not(test)), global_allocator)]
      static GLOBAL_ALLOCATOR: GlobalAllocator = GlobalAllocator::new();
      ```

   3. ### 初始化

      1. `GlobalAllocator` 初始化方法把内存区域添加到分配器

         ```rust
         pub fn init(&self, start_vaddr: usize, size: usize) {
             assert!(size > MIN_HEAP_SIZE);
             let init_heap_size = MIN_HEAP_SIZE;
             self.palloc.lock().init(start_vaddr, size);
             let heap_ptr = self
                 .alloc_pages(init_heap_size / PAGE_SIZE, PAGE_SIZE)
                 .unwrap();
             self.balloc.lock().init(heap_ptr, init_heap_size);
         }
         ```

         1. 将整个内存区域添加到页分配器
         2. 从页分配器分配 32KB 作为初始堆
         3. 用这 32KB 初始化字节分配器

   4. ### 核心分配逻辑

      1. `GlobalAllocator` 的核心分配方法实现了智能的两级分配策略

         ```rust
         pub fn alloc(&self, layout: Layout) -> AllocResult<NonNull<u8>> {
             // simple two-level allocator: if no heap memory, allocate from the page allocator.
             let mut balloc = self.balloc.lock();
             loop {
                 if let Ok(ptr) = balloc.alloc(layout) {
                     return Ok(ptr);
                 } else {
                     let old_size = balloc.total_bytes();
                     let expand_size = old_size
                         .max(layout.size())
                         .next_power_of_two()
                         .max(PAGE_SIZE);
                     let heap_ptr = self.alloc_pages(expand_size / PAGE_SIZE, PAGE_SIZE)?;
                     debug!(
                         "expand heap memory: [{:#x}, {:#x})",
                         heap_ptr,
                         heap_ptr + expand_size
                     );
                     balloc.add_memory(heap_ptr, expand_size)?;
                 }
             }
         }
         ```

      2. 分配算法详解：

         1. **首次尝试**: 从字节分配器直接分配
         2. **动态扩展**: 如果失败，计算扩展大小（取当前堆大小、请求大小的最大值，然后向上取 2 的幂次方）
         3. **页面分配**: 从页分配器获取新页面
         4. **堆扩展**: 将新页面添加到字节分配器
         5. **重试分配**: 循环直到分配成功

      3. 释放操作

         1. 释放操作相对简单，直接委托给字节分配器

            ```rust
            pub fn dealloc(&self, pos: NonNull<u8>, layout: Layout) {
                self.balloc.lock().dealloc(pos, layout)
            }
            ```

      4. 页面级分配接口

         1. `GlobalAllocator` 还提供直接的页面分配接口

            ```rust
            pub fn alloc_pages(&self, num_pages: usize, align_pow2: usize) -> AllocResult<usize> {
                self.palloc.lock().alloc_pages(num_pages, align_pow2)
            }
            ```

   5. ### GlobalAlloc Trait 实现

      1. 为了作为 Rust 的全局分配器，`GlobalAllocator` 实现了 `GlobalAlloc` trait。

         ```rust
         unsafe impl GlobalAlloc for GlobalAllocator {
             unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
                 if let Ok(ptr) = GlobalAllocator::alloc(self, layout) {
                     ptr.as_ptr()
                 } else {
                     alloc::alloc::handle_alloc_error(layout)
                 }
             }
         
             unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
                 GlobalAllocator::dealloc(self, NonNull::new(ptr).expect("dealloc null ptr"), layout)
             }
         }
         ```

         这个实现：

         - 将内部的 `AllocResult` 转换为标准的指针返回
         - 分配失败时调用 `handle_alloc_error` 处理错误
         - 释放时进行空指针检查

4. ## GlobalPage实现

   1. `axalloc` 还提供了 `GlobalPage` 结构体，用于 RAII 风格的页面管理，当对象被销毁时会自动释放页面

      ```rust
      #[derive(Debug)]
      pub struct GlobalPage {
          start_vaddr: VirtAddr,
          num_pages: usize,
      }
      ```

      结构包含两个字段：

      - `start_vaddr: VirtAddr` - 起始虚拟地址
      - `num_pages: usize` - 页面数量

   2. `GlobalPage` 提供了多种便利方法：

      - `alloc()`: 分配单个页面

        ```rust
        pub fn alloc() -> AxResult<Self> {
            global_allocator()
                .alloc_pages(1, PAGE_SIZE)
                .map(|vaddr| Self {
                    start_vaddr: vaddr.into(),
                    num_pages: 1,
                })
                .map_err(alloc_err_to_ax_err)
        }
        ```

      - `alloc_zero()`: 分配并清零的页面

        ```rust
        /// Allocate one 4K-sized page and fill with zero.
        pub fn alloc_zero() -> AxResult<Self> {
            let mut p = Self::alloc()?;
            p.zero();
            Ok(p)
        }
        ```

      - `alloc_contiguous()`: 分配连续页面

        ```rust
        /// Allocate contiguous 4K-sized pages.
        pub fn alloc_contiguous(num_pages: usize, align_pow2: usize) -> AxResult<Self> {
            global_allocator()
                .alloc_pages(num_pages, align_pow2)
                .map(|vaddr| Self {
                    start_vaddr: vaddr.into(),
                    num_pages,
                })
                .map_err(alloc_err_to_ax_err)
        }
        ```

   3. 提供了多种访问页面内容的方法：

      1. `as_ptr()` / `as_mut_ptr()` - 获取原始指针

         ```rust
         pub fn as_ptr(&self) -> *const u8 {
             self.start_vaddr.as_ptr()
         }
         
         /// Convert to a mutable raw pointer.
         pub fn as_mut_ptr(&mut self) -> *mut u8 {
             self.start_vaddr.as_mut_ptr()
         }
         ```

      2. `as_slice()` / `as_slice_mut()` - 转换为字节切片 

         ```rust
         pub fn as_slice(&self) -> &[u8] {
             unsafe { core::slice::from_raw_parts(self.as_ptr(), self.size()) }
         }
         
         /// Forms a mutable slice that can write data.
         pub fn as_slice_mut(&mut self) -> &mut [u8] {
             unsafe { core::slice::from_raw_parts_mut(self.as_mut_ptr(), self.size()) }
         }
         ```

      3. `fill()` / `zero()` - 填充页面内容 

         ```rust
         pub fn fill(&mut self, byte: u8) {
             unsafe { core::ptr::write_bytes(self.as_mut_ptr(), byte, self.size()) }
         }
         
         /// Fill `self` with zero.
         pub fn zero(&mut self) {
             self.fill(0)
         } 
         ```

      4. 自动释放机制

         实现了 `Drop` trait，确保页面在对象销毁时自动释放 。释放通过调用 `global_allocator().dealloc_pages()` 完成。

         ```rust
         impl Drop for GlobalPage {
             fn drop(&mut self) {
                 global_allocator().dealloc_pages(self.start_vaddr.into(), self.num_pages);
             }
         }
         ```

         

5. ## 统计和监控

   1. `axalloc` 提供了丰富的内存使用统计接口：

      ```rust
      /// Returns the number of allocated bytes in the byte allocator.
      pub fn used_bytes(&self) -> usize {
          self.balloc.lock().used_bytes()
      }
      
      /// Returns the number of available bytes in the byte allocator.
      pub fn available_bytes(&self) -> usize {
          self.balloc.lock().available_bytes()
      }
      
      /// Returns the number of allocated pages in the page allocator.
      pub fn used_pages(&self) -> usize {
          self.palloc.lock().used_pages()
      }
      
      /// Returns the number of available pages in the page allocator.
      pub fn available_pages(&self) -> usize {
          self.palloc.lock().available_pages()
      }
      ```

      这些接口支持：

      1. 字节级和页级的使用情况监控
      2. 内存碎片分析
      3. 性能调优和调试