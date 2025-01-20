### 本文为 《Lua 源码赏析》的阅读笔记，并补充了一些源码，Lua版本为5.4

Lua5.4 中，关于 table 定义了一些数据结构
```
/*
** Union of all Lua values
*/
typedef union Value {
  struct GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  lua_CFunction f; /* 轻量C函数 */
  lua_Integer i;   /* 整数 */
  lua_Number n;    /* 浮点数 */
  /* 没被用到， 但可以避免未初始化值 warning */
  lu_byte ub;
} Value;
#define TValuefields	Value value_; lu_byte tt_
/*
** 哈希表的节点有以下域: 一对 TValue 值，即键值对 (key-value pairs)，以及一个 'next' 域来链接冲突域。
** 'key_tt' 和 'key_val' 没有组成一个 'TValue' 是为了允许 'Node' 可以使用 4 字节和 8 字节偏移
*/
typedef union Node {
  struct NodeKey {
    TValuefields;  /* fields for value */
    lu_byte key_tt;  /* key type */
    int next;  /* for chaining */
    Value key_val;  /* key value */
  } u;
  TValue i_val;  /* direct access to node's value as a proper 'TValue' */
} Node;

typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array */
  unsigned int alimit;  /* "limit" of 'array' array */
  TValue *array;  /* 数组部分 */
  Node *node;   // 哈希表
  Node *lastfree;  /* 哈希表此位置前的都可用 */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

Table 的数组部分被存储在 array 域中。哈希表存储在 node 中，哈希表的大小使用 lsizenode 表示。
每个 table 结构，最多会由三块连续的内存构成。一个 Table 结构，一块存放了连续整数索引的数组，和一块大小为2的整数次幂的哈希表。哈希表的最小尺寸为2的0次幂，即1。为了减少空表的维护成本，Lua做了以下优化，它定义了一个不可改写的空哈希表：dummynode。让空表被初始化时，node 域指向这个dummy 节点。它虽然是一个全局变量，但因为对其是只读的，所以不会引起线程安全问题。
```
/* 当 't' 使用 'dummynode' 作为它的哈希部分时为true */
#define isdummy(t)		((t)->lastfree == NULL)

#define dummynode		(&dummynode_)

static const Node dummynode_ = {
  {{NULL}, LUA_VEMPTY,  /* value's value and type */
   LUA_VNIL, 0, {NULL}}  /* key type, next, and key value */
};
```

阅读luaH_new 和 luaH_free 两个 api 的实现，可以了解这一层次的数据结构
```
#define gnode(t,i)	(&(t)->node[i])
#define gval(n)		(&(n)->i_val)
#define gnext(n)	((n)->u.next)
/*
** 使用给定的size来创建一个 Table 的哈希表部分, 或者当size为0时，使用 dummy node。
** 计算size 是否溢出有两步：第一步比较能确保第二步的左移不会溢出
*/
static void setnodevector (lua_State *L, Table *t, unsigned int size) {
  if (size == 0) {  /* 哈希表没有元素? */
    t->node = cast(Node *, dummynode);  /* 使用 'dummynode' */
    t->lsizenode = 0;
    t->lastfree = NULL;  /* 标志目前是 dummy node */
  }
  else {
    int i;
    int lsize = luaO_ceillog2(size);    // 哈希表大小
    if (lsize > MAXHBITS || (1u << lsize) > MAXHSIZE)   // 确保 size 不會溢出
      luaG_runerror(L, "table overflow");
    size = twoto(lsize);
    t->node = luaM_newvector(L, size, Node);    // 分配一块内存给哈希表
    for (i = 0; i < cast_int(size); i++) {  // 初始化 t->node
      Node *n = gnode(t, i);
      gnext(n) = 0;
      setnilkey(n);
      setempty(gval(n));
    }
    t->lsizenode = cast_byte(lsize);
    t->lastfree = gnode(t, size);  /* 哈希表所有位置都可用 */
  }
}

Table *luaH_new (lua_State *L) {
  GCObject *o = luaC_newobj(L, LUA_VTABLE, sizeof(Table));
  Table *t = gco2t(o);
  t->metatable = NULL;
  t->flags = cast_byte(maskflags);  /* table 还没元方法 */
  t->array = NULL;
  t->alimit = 0;
  setnodevector(L, t, 0);
  return t;
}

static void freehash (lua_State *L, Table *t) {
  if (!isdummy(t))
    luaM_freearray(L, t->node, cast_sizet(sizenode(t)));
}

void luaH_free (lua_State *L, Table *t) {
  freehash(L, t);
  luaM_freearray(L, t->array, luaH_realasize(t));
  luaM_free(L, t);
}
```

Table 按照 lua 语言的定义，需要具有四种基本操作：读，写，迭代和获取长度。lua中没有删除操作，而是将对应 key 的值置为 nil。
写操作被实现为查询已有 key，若不存在则创建新 key。得到key 后，写入操作就是一次赋值。所以，在table 模块中，实际实现的基本操作为：创建，查询，迭代和获取长度。
创建操作的 api 为 luaH_newkey。它只负责在哈希表中创建出一个不存在的键，而不关数组部分的工作。因为table 的数组部分操作和C 语言数组没什么不同，不需要特殊处理。
```
/*
** 返回一个元素在哈希表中的主位置  (即它的哈希值索引).
*/
static Node *mainpositionTV (const Table *t, const TValue *key) {
  switch (ttypetag(key)) {
    case LUA_VNUMINT: {
      lua_Integer i = ivalue(key);
      return hashint(t, i);
    }
    case LUA_VNUMFLT: {
      lua_Number n = fltvalue(key);
      return hashmod(t, l_hashfloat(n));
    }
    case LUA_VSHRSTR: {
      TString *ts = tsvalue(key);
      return hashstr(t, ts);
    }
    case LUA_VLNGSTR: {
      TString *ts = tsvalue(key);
      return hashpow2(t, luaS_hashlongstr(ts));
    }
    case LUA_VFALSE:
      return hashboolean(t, 0);
    case LUA_VTRUE:
      return hashboolean(t, 1);
    case LUA_VLIGHTUSERDATA: {
      void *p = pvalue(key);
      return hashpointer(t, p);
    }
    case LUA_VLCF: {
      lua_CFunction f = fvalue(key);
      return hashpointer(t, f);
    }
    default: {
      GCObject *o = gcvalue(key);
      return hashpointer(t, o);
    }
  }
}

/*
** 往哈希表中插入一个新 key; 首先，确认 key 的主位置是否可用。
** 如果不可用，确认有冲突的节点是否在它的主位置上；
** 如果不在，移动有冲突的节点到一个空位置，并将新key放在它的主位置上；
** 否则（即冲突节点在它的主位置上），新 key 放在一个空闲位置上。
*/
static void luaH_newkey (lua_State *L, Table *t, const TValue *key,
                                                 TValue *value) {
  Node *mp;
  TValue aux;
  if (l_unlikely(ttisnil(key)))
    luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {    // key 是浮點數
    lua_Number f = fltvalue(key);
    lua_Integer k;
    if (luaV_flttointeger(f, &k, F2Ieq)) {  /* 此浮点数能否转为整型 */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (l_unlikely(luai_numisnan(f)))
      luaG_runerror(L, "table index is NaN");
  }
  if (ttisnil(value))   // 值为 nil
    return;  /* 不需要插入 nil 值 */
  mp = mainpositionTV(t, key);  // 获得 key 的主位置
  if (!isempty(gval(mp)) || isdummy(t)) {  /* 主位置已被使用？ */
    Node *othern;
    Node *f = getfreepos(t);  /* 获得一个空闲位置 */
    if (f == NULL) {  /* 找不到可用位置？ */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      luaH_set(L, t, key, value);  /* insert key into grown table */
      return;
    }
    lua_assert(!isdummy(t));
    othern = mainpositionfromnode(t, mp);   // 找到 mp 的主位置
    if (othern != mp) {  /* 有冲突的节点不在它的主位置上 */
      /* 是的话将冲突节点移动到空闲位置上 */
      while (othern + gnext(othern) != mp)  /* 找到 'mp' 的前置节点 */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* 将空闲位置 'f' 链到 'othern' 上 */
      *f = *mp;  /* 再将 'mp' 复制到空闲节点 'f' 上 */
      if (gnext(mp) != 0) {
        gnext(f) += cast_int(mp - f);  /* 纠正 'f' 的 'next' 域 */
        gnext(mp) = 0;  /* 现在 'mp' 已经变为空闲节点 */
      }
      setempty(gval(mp));
    }
    else {  /* 冲突节点在它的主位置上 */
      /* 新节点会放在空闲节点上 */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* 链接新节点 'f' */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, mp, key);
  luaC_barrierback(L, obj2gco(t), key);
  lua_assert(isempty(gval(mp)));
  setobj2t(L, gval(mp), value);
}
```
Lua 的哈希表以闭散列方式实现，每个可能的 key 值，在哈希表中都有一个主位置，即 main position。创建一个新 key 时，检查 main position，若无人使用，则可以直接设置为这个新 key。若之前有其他 key 占据了这个位置，则检查占据此位置的 key 的主位置是不是这里。若两者位置冲突，则利用 Node 结构中的 next域，以一个单向链表的形式将它们链起来；否则，新 key 将占据这个位置，而老 key 更换到新位置并根据它的 key 找到属于它的单向链表中的上一个结点，重新链入。
无论是哪种冲突情况，都需要在哈希表中找到一个空闲可用的节点。这是在 getfreepos 中，递减 lastfree 域来实现的。lua 也不会在设置 key 的值为nil 时回收空间，而是在预先准备好的哈希空间使用完后惰性回收。即在 lastfree 递减到哈希空间头时，做一次 rehash 操作。
```
static Node *getfreepos (Table *t) {
  if (!isdummy(t)) {
    while (t->lastfree > t->node) {
      t->lastfree--;
      if (keyisnil(t->lastfree))
        return t->lastfree;
    }
  }
  return NULL;  /* 找不到空闲位置 */
}

/*
** nums[i] 表示 (2^(i - 1) < k <= 2^i] 这个区间的所有 key 的数量
** 计算 table 't'的数组部分中有多少 key：按2的整数次幂分片，分别计算每个区间 key 的数量并填入 nums[i]中
** 返回所有非 nil 值的key的总个数
*/
static unsigned int numusearray (const Table *t, unsigned int *nums) {
  int lg;
  unsigned int ttlg;  /* 2^lg */
  unsigned int ause = 0;  /* 'nums' 的总和 */
  unsigned int i = 1;  /* 遍历数组 key 的计数器 */
  unsigned int asize = limitasasize(t);  /* 数组大小 */
  /* traverse each slice */
  for (lg = 0, ttlg = 1; lg <= MAXABITS; lg++, ttlg *= 2) {
    unsigned int lc = 0;  /* 计数器，计算每片（（2^lg, ttlg]）中的所有元素数量 */
    unsigned int lim = ttlg;
    if (lim > asize) {
      lim = asize;  /* 调整上限 */
      if (i > lim)
        break;  /* 已经超上限了，无需进行后续计算 */
    }
    /* 计算这个区间的整数 key 的数量 (2^(lg - 1), 2^lg] */
    for (; i <= lim; i++) {
      if (!isempty(&t->array[i-1]))
        lc++;
    }
    nums[lg] += lc;
    ause += lc;
  }
  return ause;
}

static int numusehash (const Table *t, unsigned int *nums, unsigned int *pna) {
  int totaluse = 0;  /* 所有元素的总数量 */
  int ause = 0;  /* elements added to 'nums' (can go to array part) */
  int i = sizenode(t);
  while (i--) {     // 遍历所有节点
    Node *n = &t->node[i];
    if (!isempty(gval(n))) {    // 如果节点非空
      if (keyisinteger(n))      // 节点 key 是整数
        ause += countint(keyival(n), nums);     // 将整数 key 填到 nums 数组中
      totaluse++;
    }
  }
  *pna += ause;     // 同步散列部分的整数 key 到 na
  return totaluse;  // 返回散列部分所有元素的总数量
}

/*
** 为table 't' 的整数部分计算得到最优大小。'nums'是一个计数数组，nums[i]是指区间 (2^(i-1), 2^i]之间的所有整数。
** 'pna' 是指表中所有整数 key （整数部分+散列部分）的数量。返回最优大小。
*/
static unsigned int computesizes (unsigned int nums[], unsigned int *pna) {
  int i;
  unsigned int twotoi;  /* 2^i (最优大小的候补) */
  unsigned int a = 0;  /* 比 2^i 小的数 */
  unsigned int na = 0;  /* 放到整数部分的key数量 */
  unsigned int optimal = 0;  /* 整数部分的最优大小 */
  /* loop while keys can fill more than half of total size */
  /* 当 key 数量比总大小的一半多时，循环 */
  for (i = 0, twotoi = 1;
       twotoi > 0 && *pna > twotoi / 2;
       i++, twotoi *= 2) {
    a += nums[i];
    if (a > twotoi/2) {  /* 放入数组部分的整数key超过容量的一半，则此时整数部分的利用率时可以接受的，更新分配给数组部分的大小，并记录实际放到数组部分的数量 */
      optimal = twotoi;
      na = a; 
    }
  }
  lua_assert((optimal == 0 || optimal / 2 < na) && na <= optimal);
  *pna = na;    // 记录了放到数组部分的key的数量
  return optimal;   // 数组部分的大小
}

/*
** 使用新的给定大小来重排table 't'. 散列部分和数组部分的分配都可能会失败。
** 如果第一次分配（散列部分）失败，会报错。否则，在整体大小变小时，它会复制变小部分的数组到新的散列表中，并重新分配数组部分。
** 如果分配失败，表会变回初始状态；函数会析构新的散列部分并报错。
** 否则，它会将新的散列部分放到表中，并使用 nil 来初始化新的数组部分，并将旧散列部分的元素插入到新散列部分中
*/
void luaH_resize (lua_State *L, Table *t, unsigned int newasize,
                                          unsigned int nhsize) {
  unsigned int i;
  Table newt;  /* 用来维护新的散列部分 */
  unsigned int oldasize = setlimittosize(t);
  TValue *newarray;
  /* 使用 nhsize 创建一个新的散列部分到 'newt' */
  setnodevector(L, &newt, nhsize);
  if (newasize < oldasize) {  /* 数组部分会变小 */
    t->alimit = newasize;  /* 假设数组有新大小 */
    exchangehashpart(t, &newt);  /* 交换新旧两个 table 的散列部分 */
    /* 因为数组部分变小了，将被删除的区间内的元素插入到新的散列列表中 */
    for (i = newasize; i < oldasize; i++) {
      if (!isempty(&t->array[i]))
        luaH_setint(L, t, i + 1, &t->array[i]);
    }
    t->alimit = oldasize;  /* 恢复到原来的大小 */
    exchangehashpart(t, &newt);  /* 并交换回原来的散列表 */
  }
  /* 分配新的数组 */
  newarray = luaM_reallocvector(L, t->array, oldasize, newasize, TValue);
  if (l_unlikely(newarray == NULL && newasize > 0)) {  /* 分配失败？ */
    freehash(L, &newt);  /* release 新的散列表 */
    luaM_error(L);  /* 报错，且数组并未改变 */
  }
  /* 分配成功; 初始化新的数组部分 */
  exchangehashpart(t, &newt);  /* 交换散列表，此时t是新的散列表，newt是旧的散列表，新的散列表可能是空（数组大小 不缩减的情况），也可能有从数组中移出来的一部分（数组大小缩减的情况） */
  t->array = newarray;  /* 设置新的数组 */
  t->alimit = newasize;
  for (i = oldasize; i < newasize; i++)  /* 清理新增的数组空间 */
     setempty(&t->array[i]);
  /* 从旧的散列表插入元素到新的散列表 */
  reinsert(L, &newt, t);  /* 'newt' 是旧的散列表 */
  freehash(L, &newt);  /* 清理旧的散列部分 */
}

/*
** nums[i] 是指区间 (2^(i - 1), 2^i] 的所有整数 key，上述函数也会解释其含义  
*/
static void rehash (lua_State *L, Table *t, const TValue *ek) {
  unsigned int asize;  /* optimal size for array part */
  unsigned int na;  /* number of keys in the array part */
  unsigned int nums[MAXABITS + 1];
  int i;
  int totaluse;
  for (i = 0; i <= MAXABITS; i++) nums[i] = 0;  /* reset counts */
  setlimittosize(t);
  na = numusearray(t, nums);  /* 计算数组部分的key数量 */
  totaluse = na;  /* all those keys are integer keys */
  totaluse += numusehash(t, nums, &na);  /* 加上散列部分的key数量得到表中所有key的数量 */
  /* count extra key */
  if (ttisinteger(ek))
    na += countint(ivalue(ek), nums);
  totaluse++;
  /* 计算整数部分的新大小 */
  asize = computesizes(nums, &na);
  /* 用最新数组部分大小来调整table */
  luaH_resize(L, t, asize, totaluse - na);
}

```