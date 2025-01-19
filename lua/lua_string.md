下面是lua字符串的定义
```
/*
** basic types
*/
...
#define LUA_TSTRING		4
...
/* LUA_TSTRING又被分为两大类：短字符串和长字符串 */
/* add variant bits to a type */
#define makevariant(t,v)	((t) | ((v) << 4))
/* Variant tags for strings */
#define LUA_VSHRSTR	makevariant(LUA_TSTRING, 0)  /* short strings */
#define LUA_VLNGSTR	makevariant(LUA_TSTRING, 1)  /* long strings */

/* LUAI_MAXSHORTLEN 限制了短字符串的最大长度为40，这些都是内部化字符串。 */
#if !defined(LUAI_MAXSHORTLEN)
#define LUAI_MAXSHORTLEN	40
#endif
```
相同的短字符串在同一个Lua State 中只存在唯一一份，这被称为字符串的内部化(internalized)。

字符串类型是这样定义的：
```
typedef struct TString {
  CommonHeader;     /* 用于GC的object： struct GCObject *next; lu_byte tt; lu_byte marked
  lu_byte extra;  /* 对于短字符串可用来记录这个字符串是否为保留字，这个标记用于词法分析器对保留字的快速判断; 对于长字符串可用于惰性求哈希值 */
  lu_byte shrlen;  /* 短字符串长度, 在长字符串中是 0xFF */
  unsigned int hash;
  union {
    size_t lnglen;  /* 长字符串长度 */
    struct TString *hnext;  /* 哈希链表 */
  } u;
  char contents[1];
} TString;
```
使用 getstr,getlngstr,getshrstr宏可以获得字符串内容
```
#define getstr(ts)	((ts)->contents)
#define getlngstr(ts)	check_exp((ts)->shrlen == 0xFF, (ts)->contents)
#define getshrstr(ts)	check_exp((ts)->shrlen != 0xFF, (ts)->contents)
```
所有的短字符串均被存放在全局表（global_State）的strt(string table)域中。
```
typedef struct stringtable {
  TString **hash;
  int nuse;  /* number of elements */
  int size;
} stringtable;
```
短字符串会被内部化放在全局的一张哈希字符串表中。
长字符串则独立存放，从外部压入一个长字符串时，简单复制一遍字符串，并不立刻计算其hash值，
而是标记一下extra域，直到需要对字符串做键匹配时，才惰性计算其hash值，加快以后的键比较过程。
lua使用一个随机种子进行hash值的计算，这个随机种子是在 Lua State 创建时放在全局表中的。它利用了构造状态机的内存地址随机性及用户可配置的一个随机量同时来决定。
```
unsigned int luaS_hash (const char *str, size_t l, unsigned int seed) {
  unsigned int h = seed ^ cast_uint(l);
  for (; l > 0; l--)
    h ^= ((h<<5) + (h>>2) + cast_byte(str[l - 1]));
  return h;
}

/*
** Compute an initial seed with some level of randomness.
** Rely on Address Space Layout Randomization (if present) and
** current time.
*/
#define addbuff(b,p,e) \
  { size_t t = cast_sizet(e); \
    memcpy(b + p, &t, sizeof(t)); p += sizeof(t); }

static unsigned int luai_makeseed (lua_State *L) {
  char buff[3 * sizeof(size_t)];
  unsigned int h = cast_uint(time(NULL));
  int p = 0;
  addbuff(buff, p, L);  /* heap variable */
  addbuff(buff, p, &h);  /* local variable */
  addbuff(buff, p, &lua_newstate);  /* public function */
  lua_assert(p == sizeof(buff));
  return luaS_hash(buff, p, h);
}
```
短字符串的比较
```
/*
** 先确认两者是否都是短字符串，是的话直接比较它们是否相同一样就行，因为它们都被内部唯一化了
*/
#define eqshrstr(a,b)	check_exp((a)->tt == LUA_VSHRSTR, (a) == (b))
```

长字符串的比较
```
int luaS_eqlngstr (TString *a, TString *b) {
  size_t len = a->u.lnglen;
  lua_assert(a->tt == LUA_VLNGSTR && b->tt == LUA_VLNGSTR);    //先确认是否都是长字符串类型
  return (a == b) ||  /* 是否是同一个对象 */
    ((len == b->u.lnglen) &&  /* 比较两者长度 */
     (memcmp(getlngstr(a), getlngstr(b), len) == 0));  /* 比较内容 */
}
```

短字符串的内部化  
```
/*
** 如果短字符串存在，直接复用它，否则创建一个新的短字符串。
*/
static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  TString *ts;
  global_State *g = G(L);
  stringtable *tb = &g->strt;   // 短字符串表
  unsigned int h = luaS_hash(str, l, g->seed);  // hash值
  TString **list = &tb->hash[lmod(h, tb->size)];    // 找到对应的 hash 链表
  lua_assert(str != NULL);  /* otherwise 'memcmp'/'memcpy' are undefined */
  for (ts = *list; ts != NULL; ts = ts->u.hnext) {
    if (l == ts->shrlen && (memcmp(str, getshrstr(ts), l * sizeof(char)) == 0)) {   // 比较长度，再比较字符串
      /* 找到了 */
      if (isdead(g, ts))  /* 寄了但还没被gc */
        changewhite(ts);  /* 复用它 */
      return ts;
    }
  }
  /* 否则必须创建一个新string */
  if (tb->nuse >= tb->size) {  /* 需不需要扩展 stringtable 来避免冲突发生 */
    growstrtab(L, tb);
    list = &tb->hash[lmod(h, tb->size)];  /* rehash with new size */
  }
  ts = createstrobj(L, l, LUA_VSHRSTR, h);
  ts->shrlen = cast_byte(l);
  memcpy(getshrstr(ts), str, l * sizeof(char));
  ts->u.hnext = *list;
  *list = ts;
  tb->nuse++;
  return ts;
}
```
当一个字符串被放入短字符串表中时，先检查是否已有相同的字符串，有的话，则复用该字符串。没有就创建一个新的字符串，并简单链到同一哈希位的链表上。
注意如果有相同的字符串，还需要检查它是否已经“死”了，因为Lua的gc是分布完成的。有可能在标记完字符串后发现有些字符串没有任何引用，但在下个步骤中又产生了相同的字符串导致其复活。
当哈希表中的字符串数量（nuse）超过预定容量（size）时，需要调用growstrtab来扩展字符串的哈希链表数组，并通过 luaS_resize 重排所有字符串的位置。
```
static void tablerehash (TString **vect, int osize, int nsize) {
  int i;
  for (i = osize; i < nsize; i++)  /* 清理新的哈希位内容 */
    vect[i] = NULL;
  for (i = 0; i < osize; i++) {  /* 重新排列旧的哈希位字符串位置 */
    TString *p = vect[i];
    vect[i] = NULL;
    while (p) {  /* 对于所有在此哈希位的字符串列表 */
      TString *hnext = p->u.hnext;  /* 保存下一个字符串 */
      unsigned int h = lmod(p->hash, nsize);  /* 计算得到此字符串的新哈希位 */
      p->u.hnext = vect[h];  /* 链到对应的哈希位链表上 */
      vect[h] = p;
      p = hnext;
    }
  }
}

void luaS_resize (lua_State *L, int nsize) {
  stringtable *tb = &G(L)->strt;
  int osize = tb->size;
  TString **newvect;
  if (nsize < osize)  /* 缩减 tb 的大小 */
    tablerehash(tb->hash, osize, nsize);  /* 减少哈希表大小 */
  newvect = luaM_reallocvector(L, tb->hash, osize, nsize, TString*);
  if (l_unlikely(newvect == NULL)) {  /* 分配空间失败 */
    if (nsize < osize)  /* 是否是缩减哈希表大小？ */
      tablerehash(tb->hash, nsize, osize);  /* 恢复为原来的大小 */
  }
  else {  /* 成功分配新的空间 */
    tb->hash = newvect;
    tb->size = nsize;
    if (nsize > osize)
      tablerehash(newvect, osize, nsize);  /* 根据新空间重新排列字符串位置 */
  }
}

static void growstrtab (lua_State *L, stringtable *tb) {
  if (l_unlikely(tb->nuse == MAX_INT)) {  /* 当前字符串数量超过了最大值 */
    luaC_fullgc(L, 1);  /* 尝试清理一些 */
    if (tb->nuse == MAX_INT)  /* 还是超过了最大值 */
      luaM_error(L);  /* 无法创建 */
  }
  if (tb->size <= MAXSTRTB / 2)  /* 是否可以扩展 stringtable */
    luaS_resize(L, tb->size * 2);
}
```

在Lua 状态机内创建一个字符串，都会按 C 风格字符串存放，以兼容 C 接口。即在末尾添加 '\0'。
```
static TString *createstrobj (lua_State *L, size_t l, int tag, unsigned int h) {
  TString *ts;
  GCObject *o;
  size_t totalsize;  /* total size of TString object */
  totalsize = sizelstring(l);
  o = luaC_newobj(L, tag, totalsize);
  ts = gco2ts(o);
  ts->hash = h;
  ts->extra = 0;
  getstr(ts)[l] = '\0';  /* ending 0 */
  return ts;
}
```

Userdata 在 lua 中没有太特别的地方，在存储形式上和字符串相同。可以看成是拥有独立元表，不被内部化处理，也不需要追加'\0'的字符串。在实现上，只是对象结构从TString 变为 UData。所以实现代码也放在 lstring.c中，其api 也以 LuaS 开头。
```
/* Ensures that addresses after this type are always fully aligned. */
typedef union UValue {
  TValue uv;
  LUAI_MAXALIGN;  /* ensures maximum alignment for udata bytes */
} UValue;

typedef struct Udata {
  CommonHeader;
  unsigned short nuvalue;  /* number of user values */
  size_t len;  /* number of bytes */
  struct Table *metatable;
  GCObject *gclist;
  UValue uv[1];  /* user values */
} Udata;
```
Userdata 的数据部分和字符串一样，紧接在这个结构后面，并没有单独分配内存。Userdata 的构造函数如下：
```
Udata *luaS_newudata (lua_State *L, size_t s, int nuvalue) {
  Udata *u;
  int i;
  GCObject *o;
  if (l_unlikely(s > MAX_SIZE - udatamemoffset(nuvalue)))
    luaM_toobig(L);
  o = luaC_newobj(L, LUA_VUSERDATA, sizeudata(nuvalue, s));
  u = gco2u(o);
  u->len = s;
  u->nuvalue = nuvalue;
  u->metatable = NULL;
  for (i = 0; i < nuvalue; i++)
    setnilvalue(&u->uv[i].uv);
  return u;
}
```