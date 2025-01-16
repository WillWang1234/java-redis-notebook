# java-redis-notebook
Java
# java-redis-notebook
Java
redis的设计与实现 用简洁的方式讲解 redis的内部运行机制  通过阅读本书 你可以俩姐到redis从数据结构 到服务器构造在内的几乎所有知识

        第一部分 ： 内部数据结构
          redis和其他许多键值数据库的不同之处在于，redis不仅支持简单的字符串键值对还提供了 一系列数据结构类型值，比如列表，哈希，集合和有序集，并在这些数据结构类型上定义了一套强大的api

          在redis的内部，数据结构类型值由高效的数据结构和算法进行支持，并且在redis在自身的构件中，也大量使用了这些数据结构。

          这一部分对redis的内存所使用的数据结构和算法进行介绍
















          第一节：简单动态字符串

             简单动态字符串 sds 是redis底层所使用的字符串表示 几乎所有的redis模块都使用了sds

              { sds的用途 主要有两个 ：
               1.实现字符串对象 stringObject
               2.在redis程序内部用作char*类型的替代品

               首先讲实现字符串对象：
               redis是个key-value DataBase ，数据库的值可以是字符串，集合列表等多种类型的对象，而数据库的key则总是字符串对象
               对于那些包含字符串值的对象来说，每个字符串对象都包含一个sds值 
               “包含字符串值的字符串对象”，这种说法初听上去可能会有点奇怪， 但是在 Redis 中， 一个字符串对象除了可以保存字符串值之外， 还可以保存 long 类型的值， 所以为了严谨起见， 这里需要强调一下： 当字符串对象保存的是字符串时， 它包含的才是 sds 值， 否则的话， 它就是一个 long 类型的值

               举个例子：以下的命令创建了一个新的数据库键值对，这个键值对的key和value都是字符串对象，它们都有一个sds值：
               redis> SET book "Mastering C++ in 21 days"
                OK

               redis> GET book
               "Mastering C++ in 21 days"
               
               以下命令创建了另一个键值对， 它的键是字符串对象， 而值则是一个集合对象：
               redis>SADD nosql "redis""MongoDB""NEo4j"
               (integer)3
               redis> SMEMBERS nosql
                1) "Neo4j"
                2) "Redis"
                3) "MongoDB"
               总结下来就是key一定是字符串对象 都有sds值 值不一定 也可能是long型的对象 这个时候就是long型的值

               第二个用途：就是 用sds来取代c默认的char*类型
                   因为 char* 类型的功能单一， 抽象层次低， 并且不能高效地支持一些 Redis 常用的操作（比如追加操作和长度计算操作）， 所以在 Redis 程序内部， 绝大部分情况下都会使用 sds 而不是 char* 来表示字符串。

               性能问题在稍后介绍 sds 定义的时候就会说到， 因为我们还没有了解过 Redis 的其他功能模块， 所以也没办法详细地举例说那里用到了 sds ， 不过在后面的章节中， 我们会经常看到其他模块（几乎每一个）都用到了 sds 类型值。

               目前来说， 只要记住这个事实即可： 在 Redis 中， 客户端传入服务器的协议内容、 aof 缓存、 返回给客户端的回复， 等等， 这些重要的内容都是由 sds 类型来保存的。
               }
               
               {redis中的字符串 
               在c语言中，字符串可以用一个用\0结尾的char数组来表示
               比如说， hello world 在 C 语言中就可以表示为 "hello world\0" 。
               这种简单的字符串表示，在大多数情况下都能满足要求，但是，它并不能高效地支持长度计算和追加（append）这两种操作：

               每次计算字符串长度（strlen(s)）的复杂度为 θ(N)。
               对字符串进行 N 次追加，必定需要对字符串进行 N 次内存重分配（realloc）。
               在 Redis 内部， 字符串的追加和长度计算很常见， 而 **APPEND 和 **STRLEN 更是这两种操作，在 Redis 命令中的直接映射， 这两个简单的操作不应该成为性能的瓶颈。
               另外， Redis 除了处理 C 字符串之外， 还需要处理单纯的字节数组， 以及服务器协议等内容， 所以为了方便起见， Redis 的字符串表示还应该是二进制安全的： 程序不应对字符串里面保存的数据做任何假设， 数据可以是以 \0 结尾的 C 字符串， 也可以是单纯的字节数组， 或者其他格式的数据。
               二进制安全：
               二进制安全函数将其输入视为原始数据流，并忽略其可能具有的所有文本方面
               二进制安全文件读写
               虽然所有的文本数据都可以以二进制形式表示，但是必须通过字符编码来实现。除此之外，换行符的表示方式可能因所用平台而异。Windows、Linux 和 macOS 都以不同的二进制形式表示换行符。这意味着将文件读取为二进制数据，将其解析为文本，然后将其写回磁盘（从而将其重新转换回二进制形式）可能会导致 
              与最初使用的二进制表示不同的二进制表示。

              大多数编程语言都允许程序员决定是否将文件内容解析为文本，还是将其读取为二进制数据。为了传达此意图，在读取或写入文件到磁盘时存在特殊标志或不同函数。例如，在 PHP、C 和 C++ 编程语言中，开发人员必须使用fopen($filename, "rb")而不是fopen($filename, "r")将文件读取为二进制流，而不是将 
              文本数据解释为二进制流。这也可以称为以“二进制安全”模式读取。

              考虑到这两个原因（复杂度和二进制安全 ）所以redis使用了sds类型替换了c语言的默认字符串表示：sds既可以高效地实现追加和长度计算，同时是二进制安全地。


              sds的实现：
              实现由以下的两部分组成：

              typedef char * sds；
              struct sdshdr{
              // buf已占有长度
              int len；
              //buf剩余可用长度
              int free；
              //实际保存字符串数据的地方
              char buf[];
              };
              其中sds是char * 的别名（alias） 而结构sdshdr则保存了len，free和buf三个属性
              作为例子 以下是新创建的，同样保存了hello world 字符串中的sdshdr结构 
              stuct sdshdr{
              len = 11；
              free = 0；
              buf = “hello world\0”;//buf的实际长度为len + 1
               }
              通过len属性，sdshdr可以实现复杂度为0（1）的长度计算操作，
              另一方面， 通过对 buf 分配一些额外的空间， 并使用 free 记录未使用空间的大小， sdshdr 可以让执行追加操作所需的内存重分配次数大大减少， 下一节我们就会来详细讨论这一点。

              当然， sds 也对操作的正确实现提出了要求 —— 所有处理 sdshdr 的函数，都必须正确地更新 len 和 free 属性，否则就会造成 bug 。
              
              }

              {
              追加优化操作 
              利用sdshdr结构，除了可以用o（1）的复杂度获取字符串的长度之外，还可以减少追加（append）操作所需要的内存重分配次数
              接下来详细解释这个优化的原理


              用一个redis执行实例作为例子 解释一下执行以下代码的时候，redis内部发生了什么
              redis> SET msg "hello world"
              OK

              redis> APPEND msg " again!"
              (integer) 18

              redis> GET msg
              "hello world again!"
              首先set命令创建并保存了hello world和sdshdr中，这个sdshdr的值如下：
              struct sdshdr{
              len = 11;
              free = 0;
              buf = "hello world\0";}


              当执行append命令的时候，相应的sdshdr被更新，字符串again会被追加到原来的hello world之后：
              struct sdshdr {
              len = 18;
              free = 18;
              buf = "hello world again!\0                  ";     // 空白的地方为预分配空间，共 18 + 18 + 1 个字节
              }
              注意， 当调用 SET 命令创建 sdshdr 时， sdshdr 的 free 属性为 0 ， Redis 也没有为 buf 创建额外的空间 —— 而在执行 APPEND 之后， Redis 为 buf 创建了多于所需空间一倍的大小。
              
              在这个例子中， 保存 "hello world again!" 共需要 18 + 1 个字节， 但程序却为我们分配了 18 + 18 + 1 = 37 个字节 —— 这样一来， 如果将来再次对同一个 sdshdr 进行追加操作， 只要追加内容的长度不超过 free 属性的值， 那么就不需要对 buf 进行内存重分配。
              
              比如说， 执行以下命令并不会引起 buf 的内存重分配， 因为新追加的字符串长度小于 18 ：
              再次执行 APPEND 命令之后， msg 的值所对应的 sdshdr 结构可以表示如下：

              struct sdshdr {
                  len = 25;
                  free = 11;
                  buf = "hello world again! again!\0           ";     // 空白的地方为预分配空间，共 18 + 18 + 1 个字节
              }

              sds.c/sdsMakeRoomFor 函数描述了 sdshdr 的这种内存预分配优化策略， 以下是这个函数的伪代码版本：
                          def sdsMakeRoomFor(sdshdr, required_len):
                      
                          # 预分配空间足够，无须再进行空间分配
                          if (sdshdr.free >= required_len):
                              return sdshdr
                      
                          # 计算新字符串的总长度
                          newlen = sdshdr.len + required_len
                      
                          # 如果新字符串的总长度小于 SDS_MAX_PREALLOC
                          # 那么为字符串分配 2 倍于所需长度的空间
                          # 否则就分配所需长度加上 SDS_MAX_PREALLOC 数量的空间
                          if newlen < SDS_MAX_PREALLOC:
                              newlen *= 2
                          else:
                              newlen += SDS_MAX_PREALLOC
                      
                          # 分配内存
                          newsh = zrelloc(sdshdr, sizeof(struct sdshdr)+newlen+1)
                      
                          # 更新 free 属性
                          newsh.free = newlen - sdshdr.len
                      
                          # 返回
                          return newsh

                          在目前版本的 Redis 中， SDS_MAX_PREALLOC 的值为 1024 * 1024 ， 也就是说， 当大小小于 1MB 的字符串执行追加操作时， sdsMakeRoomFor 就为它们分配多于所需大小一倍的空间； 当字符串的大小大于 1MB ， 那么 sdsMakeRoomFor 就为它们额外多分配 1MB 的空间

                                  问题：
                                  这种分配策略会浪费内存吗？
                                  执行过 APPEND 命令的字符串会带有额外的预分配空间， 这些预分配空间不会被释放， 除非该字符串所对应的键被删除， 或者等到关闭 Redis 之后， 再次启动时重新载入的字符串对象将不会有预分配空间。
                                  
                                  因为执行 APPEND 命令的字符串键数量通常并不多， 占用内存的体积通常也不大， 所以这一般并不算什么问题。
                                  
                                  另一方面， 如果执行 APPEND 操作的键很多， 而字符串的体积又很大的话， 那可能就需要修改 Redis 服务器， 让它定时释放一些字符串键的预分配空间， 从而更有效地使用内存。
                                    
              

              
              }

              {
              sds模块的api：

              函数	    作用	                                             算法复杂度
              sdsnewlen	创建一个指定长度的 sds ，接受一个 C 字符串作为初始化值	 O(N)
              sdsempty	创建一个只包含空白字符串 "" 的 sds                   	O(1)
              sdsnew	根据给定 C 字符串，创建一个相应的 sds	                   O(N)
              sdsdup	复制给定 sds	                                           O(N)
              sdsfree	释放给定 sds                                          	O(N)
              sdsupdatelen	更新给定 sds 所对应 sdshdr 结构的 free 和 len	      O(N)
              sdsclear	清除给定 sds 的内容，将它初始化为 ""	                     O(1)
              sdsMakeRoomFor	对 sds 所对应 sdshdr 结构的 buf 进行扩展	         O(N)
              sdsRemoveFreeSpace	在不改动 buf 的情况下，将 buf 内多余的空间释放出去	O(N)
              sdsAllocSize	计算给定 sds 的 buf 所占用的内存总数	                  O(1)
              sdsIncrLen	对 sds 的 buf 的右端进行扩展（expand）或修剪（trim）     	O(1)
              sdsgrowzero	将给定 sds 的 buf 扩展至指定长度，无内容的部分用 \0 来填充	O(N)
              sdscatlen	按给定长度对 sds 进行扩展，并将一个 C 字符串追加到 sds 的末尾	O(N)
              sdscat	将一个 C 字符串追加到 sds 末尾	                               O(N)
              sdscatsds	将一个 sds 追加到另一个 sds 末尾	O(N)
              sdscpylen	将一个 C 字符串的部分内容复制到另一个 sds 中，需要时对 sds 进行扩展	O(N)
              sdscpy	将一个 C 字符串复制到 sds	O(N)
              sds 还有另一部分功能性函数， 比如 sdstolower 、 sdstrim 、 sdscmp ， 等等， 基本都是标准 C 字符串库函数的 sds 版本， 这里不一一列举了。

              
              }
              {这一部分的小结：
              Redis 的字符串表示为 sds ，而不是 C 字符串（以 \0 结尾的 char*）。
              对比 C 字符串， sds 有以下特性：
              可以高效地执行长度计算（strlen）；
              可以高效地执行追加操作（append）；
              二进制安全；
              sds 会为追加操作进行优化：加快追加操作的速度，并降低内存分配的次数，代价是多占用了一些内存，而且这些内存不会被主动释放。
              

              
              }


              第二节：双端链表

              链表作为数组之外的常用序列抽象， c语言本身不支持链表类型，大部分c语言程序都会自己实现一种链表类型，redis也不例外，双端链表作为一种常见的数据结构， 在大部分的数据结构或者算法书里都有讲解， 因此， 这一章关注的是 Redis 双端链表的具体实现， 以及该实现的 API ， 而对于双端链表本身， 以及双端链表所对应的算法， 则不做任何解释。             
              双端链表的应用：{
              双端链表作为一种通用的数据结构，在redis内部使用得非常多，既是redis列表结构得底层实现之一，同时也为大量redis快所用，用于构建redis得其他功能。
              实现redis得列表类型
              双端链表还是 Redis 列表类型的底层实现之一， 当对列表类型的键进行操作 —— 比如执行 **RPUSH 、 **LPOP 或 **LLEN 等命令时， 程序在底层操作的可能就是双端链表。

              redis> RPUSH brands Apple Microsoft Google
                (integer) 3
                
                redis> LPOP brands
                "Apple"
                
                redis> LLEN brands
                (integer) 2
                
                redis> LRANGE brands 0 -1
                1) "Microsoft"
                2) "Google"
                tips：redis列表使用两种数据结构作为底层实现：
                1.双端链表
                2.压缩列表 
                因为双端链表占用得内存比压缩列表要多，所以当创建新的列表键时，列表会优先考虑使用压缩列表作为底层实现，并且在有需要的时候，才从压缩列表实现转换到双端链表实现。




                redis自身功能的构建；
                除了实现列表类型以外 双端链表还被很多redis内部的模块所应用：
                事务模块使用双端链表依次序保存输入的命令 ；
                服务器模块使用双端链表来保存多个客户端；
                订阅或者发送模块使用双端链表来保存订阅模式的多个客户端；
                事件模块使用双端链表来保存时间事件
                类似的应用还有很多 
                }
                双端链表的实现 ：
                
                {
                双端链表的实现由listnode 和list两个数据结构构成 
                list      |--------------------------------------------|
                head ->   |     listNode <--- -|                       |
                tail -----|     prev value next |  ->  listNode        |->   listNode
                dup                            |-- -prev value next     prev value next
                free
                match
                len

                其中listnode是双端链表的节点：
                typedef struct listNode {

                  // 前驱节点
                  struct listNode *prev;
              
                  // 后继节点
                  struct listNode *next;
              
                  // 值
                  void *value;
              
                  } listNode;

                  list是双端链表本身

                  typedef struct list {

                    // 表头指针
                    listNode *head;
                
                    // 表尾指针
                    listNode *tail;
                
                    // 节点数量
                    unsigned long len;
                
                    // 复制函数
                    void *(*dup)(void *ptr);
                    // 释放函数
                    void (*free)(void *ptr);
                    // 比对函数
                    int (*match)(void *ptr, void *key);
                } list;

                注意listnode的value属性是void* 说明这个双端链表对节点所保存的值的类型不做限制
                对于不同类型的值，有时候需要不同的函数来处理这些值，因此， list 类型保留了三个函数指针 —— dup 、 free 和 match ，分别用于处理值的复制、释放和对比匹配。在对节点的值进行处理时，如果有给定这些函数，就会调用这些函数。

                举个例子：当删除一个 listNode 时，如果包含这个节点的 list 的 list->free 函数不为空，就会先调用删除函数 list->free(listNode->value) 来清空节点的值，再执行余下的删除操作（比如说，释放节点）。

                另外，从这两个数据结构的定义上，也可以了解到一些行为和性能特征：

                listNode 带有 prev 和 next 两个指针，因此，遍历可以双向进行：从表头到表尾，表尾到表头。
                list 保存了 head 和 tail 两个指针，因此，对链表的表头和表尾进行插入的复杂度都为 θ(1)
                 —— 这是高效实现 LPUSH 、 RPOP 、 RPOPLPUSH 等命令的关键。
                list 带有保存节点数量的 len 属性，所以计算链表长度的复杂度仅为 θ(1)
                 ，这也保证了 LLEN 命令不会成为性能瓶颈。

                 以下是用于操作双端链表的api 作用  和算法复杂度 ：
                  函数	      作用	                           算法复杂度
                  listCreate	创建新链表	                           O(1)
                  listRelease	释放链表，以及该链表所包含的节点     	 O(N)
                  listDup	创建给定链表的副本	                       O(N)
                  listRotate	取出链表的表尾节点，并插入到表头       	O(1)
                  listAddNodeHead	将包含给定值的节点添加到链表的表头 	O(1)
                  listAddNodeTail	将包含给定值的节点添加到链表的表尾	  O(1)
                  listInsertNode	将包含给定值的节点添加到某个节点的之前或之后	O(1)
                  listDelNode	删除给定节点	                          O(1)
                  listSearchKey	在链表中查找和给定 key 匹配的节点	    O(N)
                  listIndex	给据给定索引，返回列表中相应的节点	        O(N)
                  listLength	返回给定链表的节点数量	                O(1)
                  listFirst	返回链表的表头节点	                      O(1)
                  listLast	返回链表的表尾节点	                      0(1)
                  listPrevNode	返回给定节点的前一个节点	            O(1)
                  listNextNode	返回给定节点的后一个节点	            O(1)
                  listNodeValue	返回给定节点的值	                    O(1)

                 
                 
                                  
                
                }

                迭代器{
                Redis 为双端链表实现了一个迭代器 ， 这个迭代器可以从两个方向对双端链表进行迭代：

                沿着节点的 next 指针前进，从表头向表尾迭代；
                沿着节点的 prev 指针前进，从表尾向表头迭代；
                以下是迭代器的数据结构定义：
                Redis 为双端链表实现了一个迭代器 ， 这个迭代器可以从两个方向对双端链表进行迭代：
                typedef struct listIter {

                    // 下一节点
                    listNode *next;
                
                    // 迭代方向
                    int direction;
                
                } listIter;


                direction 记录迭代应该从那里开始：

                如果值为 adlist.h/AL_START_HEAD ，那么迭代器执行从表头到表尾的迭代；
                如果值为 adlist.h/AL_START_TAIL ，那么迭代器执行从表尾到表头的迭代；
                以下是迭代器的操作 API ，API 的作用以及算法复杂度：
                函数	作用	算法复杂度
                listGetIterator	创建一个列表迭代器	O(1)
                listReleaseIterator	释放迭代器	O(1)
                listRewind	将迭代器的指针指向表头	O(1)
                listRewindTail	将迭代器的指针指向表尾	O(1)
                listNext	取出迭代器当前指向的节点	O(1)
               
                小结：
                redis实现了自己的双端链表 
                双端链表主要有两个作用
                   作为redis列表类型的底层实现之一
                   作为通用的数据结构 被其他功能模块所使用 
                双端链表及其节点的性能特性如下：
                   节点带有前驱和后继指针，访问前驱节点和后继节点的复杂度为 O(1)
                       ，并且对链表的迭代可以在从表头到表尾和从表尾到表头两个方向进行；
                      链表带有指向表头和表尾的指针，因此对表头和表尾进行处理的复杂度为 O(1)
                       ；
                      链表带有记录节点数量的属性，所以可以在 O(1)
                       复杂度内返回链表的节点数量（长度）；
                                      



                
                }


                第三节 字典：{
                字典又名映射 （map）或者关联数组（associative array） 是一种抽线数据结构，由一集键值对组成，各个键值对各不相同，程序可以添加新的键值对到字典中，或者基于键进行查找，更新或删除等操作。
                
                   字典的应用：{
                   字典在redis中的应用广泛，使用频率可以说和sds以及双端链表不相上下，基本各个功能模块都有用到字典的地方 
                   主要用途是：
                   1.实现数据库键空间（key space）
                   2.用作Hash类型键的底层是实现之一

                           实现数据库键空间：{
                           redis是一个键值对数据库，数据库中的键值对由字典保存 每个数据库都有一个对应的字典，这个字典被称为键空间 key space
                           当用户添加一个键值对到数据库时（不论键值对是什么类型），程序就将该键值对添加到键空间；当用户从数据库中删除键值对时候，程序就会将这个键值对从键空间中删除，等等
                           举个例子：执行 FLUSHDB 可以清空键空间里所有的键值对数据：
                           redis> FLUSHDB
                           OK

                           执行 DBSIZE 则返回键空间里现有的键值对：

                            redis> DBSIZE
                            (integer) 0
                           
                           还可以用 SET 设置一个字符串键到键空间， 并用 GET 从键空间中取出该字符串键的值：

                            redis> SET number 10086
                            OK
                            
                            redis> GET number
                            "10086"
                            
                            redis> DBSIZE
                            (integer) 1
                            后面的《数据库》一章会对键空间以及数据库的实现作详细的介绍， 届时将看到， 大部分针对数据库的命令， 比如 DBSIZE 、 FLUSHDB 、 RANDOMKEY ， 等等， 都是构建于对字典的操作之上的； 而那些创建、更新、删除和查找键值对的命令， 也无一例外地需要在键空间上进行操作。
                            }

                            用作Hash类型键的底层实现之一：{
                            Redis的Hash类型键使用以下两种数据结构作为底层实现：
                            1.字典
                            2.压缩列表 
                            因为压缩列表比字典更节省内存，所以程序在创建新的Hash键时，默认使用压缩列表作为底层实现，当有需要的时候，程序才会从压缩列表转换到字典
                            当用户操作一个 Hash 键时， 键值在底层就可能是一个哈希表
                            redis> HSET book name "The design and implementation of Redis"
                            (integer) 1
                            
                            redis> HSET book type "source code analysis"
                            (integer) 1
                            
                            redis> HSET book release-date "2013.3.8"
                            (integer) 1
                            
                            redis> HGETALL book
                            1) "name"
                            2) "The design and implementation of Redis"
                            3) "type"
                            4) "source code analysis"
                            5) "release-date"
                            6) "2013.3.8"
                            《哈希表》章节给出了关于哈希类型键的更多信息， 并介绍了压缩列表和字典之间的转换条件。

                            介绍完了字典的用途， 现在让我们来看看字典数据结构的定义。
                            
                             字典的实现：{
                                 实现字典的方法有很多种
                                        最简单的就是使用链表或数组，但是这种方式只适用于元素个数不多的情况下 
                                        要兼顾高效和简单性，可以使用哈希表
                                        如果追求更为稳定的性能特征，并且高效地实现排序操作的话，则可以使用更为复杂地平衡树
                                        在众多可能地实现中，redis选择了高效实现简单的哈希表，作为字典的底层实现
                                  dict.h/dict 给出了这个字典的定义：
                                   /*
                                   * 字典
                                   *
                                   * 每个字典使用两个哈希表，用于实现渐进式 rehash
                                   */
                                  typedef struct dict {
                                  
                                      // 特定于类型的处理函数
                                      dictType *type;
                                  
                                      // 类型处理函数的私有数据
                                      void *privdata;
                                  
                                      // 哈希表（2 个）
                                      dictht ht[2];
                                  
                                      // 记录 rehash 进度的标志，值为 -1 表示 rehash 未进行
                                      int rehashidx;
                                  
                                      // 当前正在运作的安全迭代器数量
                                      int iterators;
                                  
                                  } dict   
                                  以下是用于处理 dict 类型的 API ， 它们的作用及相应的算法复杂度：

                                  操作	函数	算法复杂度
                                  创建一个新字典	dictCreate	O(1)
                                  添加新键值对到字典	dictAdd	O(1)
                                  添加或更新给定键的值	dictReplace	O(1)
                                  在字典中查找给定键所在的节点	dictFind	O(1)
                                  在字典中查找给定键的值	dictFetchValue	O(1)
                                  从字典中随机返回一个节点	dictGetRandomKey	O(1)
                                  根据给定键，删除字典中的键值对	dictDelete	O(1)
                                  清空并释放字典	dictRelease	O(N)
                                  清空并重置（但不释放）字典	dictEmpty	O(N)
                                  缩小字典	dictResize	O(N)
                                  扩大字典	dictExpand	O(N)
                                  对字典进行给定步数的 rehash	dictRehash	O(N)
                                  在给定毫秒内，对字典进行rehash	dictRehashMilliseconds	O(N)
                                  注意 dict 类型使用了两个指针，分别指向两个哈希表。
                                  
                                  其中， 0 号哈希表（ht[0]）是字典主要使用的哈希表， 而 1 号哈希表（ht[1]）则只有在程序对 0 号哈希表进行 rehash 时才使用。
                                  
                                  接下来两个小节将对哈希表的实现，以及哈希表所使用的哈希算法进行介绍。
                                  
              
                                  
                                 }


                                 hash表的实现{
                                 字典所使用的哈希表实现由dict.h/dictht类定义 ：
                                 /*
                                   * 哈希表
                                   */
                                  typedef struct dictht {
                                  
                                      // 哈希表节点指针数组（俗称桶，bucket）
                                      dictEntry **table;
                                  
                                      // 指针数组的大小
                                      unsigned long size;
                                  
                                      // 指针数组的长度掩码，用于计算索引值
                                      unsigned long sizemask;
                                  
                                      // 哈希表现有的节点数量
                                      unsigned long used;

                                  } dictht;
                                  table属性是个数组 数组的每个元素都是个指向dictEntry结构的指针
                                  每个dictEntry都保持着一个键值对，以及一个指向另一个dictEntry结构的指针；
                                  /*
                                   * 哈希表节点
                                   */
                                  typedef struct dictEntry {
                                  
                                      // 键
                                      void *key;
                                  
                                      // 值
                                      union {
                                          void *val;
                                          uint64_t u64;
                                          int64_t s64;
                                      } v;
                                  
                                      // 链往后继节点
                                      struct dictEntry *next;
                                  
                                  } dictEntry;
                                  
                                  next 属性指向另一个 dictEntry 结构， 多个 dictEntry 可以通过 next 指针串连成链表， 从这里可以看出， dictht 使用链地址法来处理键碰撞： 当多个不同的键拥有相同的哈希值时，哈希表用一个链表将这些键连接起来。

                                  下图展示了一个由 dictht 和数个 dictEntry 组成的哈希表例子：
                                 dictht
                                 table
                                 size：4                                                                   ------->       dictEntry
                                 sizemask：3   指针数组的长度掩码，用于计算索引值                                                            |        key1    value1 next      ------>null
                                 used： 3                                                                 |
                                                                    table            dictEntry**（bucket）|
                                                                                    0-------------------- |
                                                                                    1 ------------>null
                                                                                    2---------------------->dictEntry
                                                                                                          key2 value2 next ---->null
                                                                                      
                                                                                    3-----------------------dictEntry
                                                                                                           key3 value3 next ------>null
                                                                                                           
                                                                                                           
                                                                                                           
                                                                                                           
                                如果再加上之前列出的dict类型  那么整个字典结构可以表示如下：
                                dict
                                type                             ht[0]
                                privdata                     |------>   dictht
                                ht[2] -----------------------|          table ------------------->和上面的一样
                                rehashidx:-1           |                size:4
                                iterator:0             |               sizemask：3
                                                       |                used：3
                                                       |   ht[1]          
                                                       |----->dictht
                                                             table ------->null
                                                             size: 0
                                                             sizemask: 0
                                                             used:0

                               
                                                             
                                 }
                                      哈希算法：{
                                 redis主要使用两种不同的hash算法：
                                 1.murmurhash2 32 bit算法  这种算法的分布率和速度都非常
                                 2.基于 djb 算法实现的一个大小写无关散列算法：
                                 使用哪种算法取决于具体应用所处理的数据：
                                
                                命令表以及 Lua 脚本缓存都用到了算法 2 。
                                算法 1 的应用则更加广泛：数据库、集群、哈希键、阻塞操作等功能都用到了这个算法。
                                
                                                                 


                                 
                                 }
                                 创建新字典：{
                                 dictCreate 函数创建并返回一个新字典：

                                  dict *d = dictCreate(&hash_type, NULL);
                                 
                                    d 的值可以用图片表示如下：

                                dict
                                type                             ht[0]
                                privdata                     |------>   dictht
                                ht[2] -----------------------|          table ------------------->null
                                rehashidx:-1           |                size:4
                                iterator:0             |               sizemask：3
                                                       |                used：3
                                                       |   ht[1]          
                                                       |----->dictht
                                                             table ------->null
                                                             size: 0
                                                             sizemask: 0
                                                             used:0
                                新创建的两个哈希表都没有为 table 属性分配任何空间：

                                ht[0]->table 的空间分配将在第一次往字典添加键值对时进行；
                                ht[1]->table 的空间分配将在 rehash 开始时进行；
                                 }

                                添加键值对到字典：{

                                根据字典所处的状态， 将给定的键值对添加到字典可能会引起一系列复杂的操作：

                                如果字典为未初始化（即字典的 0 号哈希表的 table 属性为空），则程序需要对 0 号哈希表进行初始化；
                                如果在插入时发生了键碰撞，则程序需要处理碰撞；
                                如果插入新元素，使得字典满足了 rehash 条件，则需要启动相应的 rehash 程序；
                                当程序处理完以上三种情况之后，新的键值对才会被真正地添加到字典上。
                                
                                整个添加流程可以用下图表示：
                                                                        dictadd
                                                                           |
                                                                      键已经存在？
                                                                           |
                                                  ---------------------------------------------|否
                                                  |是                                          |
                                                  |                                     ht[0]未分配任何空间？
                                                  返回null                                      |
                                                  表示添加失败                                   |
                                                                          |------------------------------------------|否
                                                                          |是                                        |
                                                                          |                                          |
                                                                    初始化ht[0]                                       |
                                                                          |                                           |
                                                                           |                                          |
                                                                            |---------------------->          需要rehash？
                                                                                                          需要就开始渐进式rehash     不需要或者rehash正在进行
                                                                                                          
                                                                                                                     判断rehash正在进行中吗
                                                                                                                        
                                                                                                                是的话选择ht[1]作为新的键值对的添加目标     否选择ht[0]来作为新的键值对的添加目标
                                                                                                                
                                                                                                                                        根据给丁键 计算出hash值以及索引值
                                                                                                                                        创建新的dictEntry，并且保存给定键值对
                                                                                                                                        根据索引值将新节点添加到目标哈希表
                                                                                                                                        
                                在接下来的三节中， 我们将分别看到，添加操作如何在以下三种情况中执行：
                                
                                字典为空；
                                添加新键值对时发生碰撞处理；
                                添加新键值对时触发了 rehash 操作；
                                添加新元素到空白字典
                                                                                                                                        
                                } 
                                添加新元素到空白字典{
                                当第一次往空字典里添加键值对时， 程序会根据 dict.h/DICT_HT_INITIAL_SIZE 里指定的大小为 d->ht[0]->table 分配空间 （在目前的版本中， DICT_HT_INITIAL_SIZE 的值为 4 ）。
                                
                                以下是字典空白时的样子：
                                 dict
                                type                             ht[0]
                                privdata                     |------>   dictht
                                ht[2] -----------------------|          table ------------------->null
                                rehashidx:-1           |                size:4
                                iterator:0             |               sizemask：3
                                                       |                used：3
                                                       |   ht[1]          
                                                       |----->dictht
                                                             table ------->null
                                                             size: 0
                                                             sizemask: 0
                                                             used:0
                                
                                
                                以下是往空白字典添加了第一个键值对之后的样子
                                 dict
                                type                             ht[0]
                                privdata                     |------>   dictht
                                ht[2] -----------------------|          table ------------------->dictEntry**bucket
                                rehashidx:-1           |                size:4                          0        ------->null          
                                iterator:0             |               sizemask：3                      1 ----------------------->dictEntry
                                                       |                used：3                          2------>null            key1 value1 next ----->null
                                                       |   ht[1]                                         3------>null
                                                       |----->dictht
                                                             table ------->null
                                                             size: 0
                                                             sizemask: 0
                                                             used:0
                                
                                }
                                添加新的键值对时候发生碰撞处理的时候{
                                在哈希表实现中， 当两个不同的键拥有相同的哈希值时， 称这两个键发生碰撞（collision）， 而哈希表实现必须想办法对碰撞进行处理。

                               字典哈希表所使用的碰撞解决方法被称之为链地址法： 这种方法使用链表将多个哈希值相同的节点串连在一起， 从而解决冲突问题。
                               
                                对于一个新的键值对 key4 和 value4 ， 如果 key4 的哈希值和 key3 的哈希值相同， 那么它们将在哈希表的 0 号索引上发生碰撞。

                                  通过将 key4-value4 和 key3-value1 两个键值对用链表连接起来， 就可以解决碰撞的问题：
                              dictEntry**bucket
                               0        ------->null          
                               1 ----------------------->dictEntry
                               2------>null            key1 value1 next ----->null
                               3------null
                               4---------->       dictEntry
                               5 -->null         key2 value2 next --->null
                               6-------->dictEntey
                               7          key4 value4 next-------------->dictEntry
                                                                        key3 value3 next ------>null
                                                
                                
                               
                               
                               }
                               添加新键值对触发了rehash操作
                               对于使用链地址法来解决碰撞问题的哈希表 dictht 来说， 哈希表的性能取决于大小（size属性）与保存节点数量（used属性）之间的比率：

                                哈希表的大小与节点数量，比率在 1:1 时，哈希表的性能最好；
                                如果节点数量比哈希表的大小要大很多的话，那么哈希表就会退化成多个链表，哈希表本身的性能优势便不复存在；
                                举个例子， 下面这个哈希表， 平均每次失败查找只需要访问 1 个节点（非空节点访问 2 次，空节点访问 1 次）
                               
                                                                  

                               
                            
                            
                            
                   }




                   
                  
                  
                   
































                
                }

                

                


               
               








