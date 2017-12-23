# Vol 哈希算法

ATS 在建立Vol hash表时用了一种特殊的一致性hash 算法

```
void
build_vol_hash_table(CacheHostRecord *cp)
{
  int num_vols          = cp->num_vols;
  unsigned int *mapping = (unsigned int *)ats_malloc(sizeof(unsigned int) * num_vols);
  Vol **p               = (Vol **)ats_malloc(sizeof(Vol *) * num_vols);

  memset(mapping, 0, num_vols * sizeof(unsigned int));
  memset(p, 0, num_vols * sizeof(Vol *));
  uint64_t total = 0;
  int bad_vols   = 0;
  int map        = 0;
  uint64_t used  = 0;
  // initialize number of elements per vol
  for (int i = 0; i < num_vols; i++) {
    if (DISK_BAD(cp->vols[i]->disk)) {
      bad_vols++;
      continue;
    }
    mapping[map] = i;
    p[map++]     = cp->vols[i];
    total += (cp->vols[i]->len >> STORE_BLOCK_SHIFT);
  }

  num_vols -= bad_vols;

  if (!num_vols || !total) {
    // all the disks are corrupt,
    if (cp->vol_hash_table) {
      new_Freer(cp->vol_hash_table, CACHE_MEM_FREE_TIMEOUT);
    }
    cp->vol_hash_table = nullptr;
    ats_free(mapping);
    ats_free(p);
    return;
  }

  unsigned int *forvol   = (unsigned int *)ats_malloc(sizeof(unsigned int) * num_vols);
  unsigned int *gotvol   = (unsigned int *)ats_malloc(sizeof(unsigned int) * num_vols);
  unsigned int *rnd      = (unsigned int *)ats_malloc(sizeof(unsigned int) * num_vols);
  unsigned short *ttable = (unsigned short *)ats_malloc(sizeof(unsigned short) * VOL_HASH_TABLE_SIZE);
  unsigned short *old_table;
  unsigned int *rtable_entries = (unsigned int *)ats_malloc(sizeof(unsigned int) * num_vols);
  unsigned int rtable_size     = 0;

  // estimate allocation
  for (int i = 0; i < num_vols; i++) {
    // 按比例划分 VOL_HASH_TABLE_SIZE 空间
    forvol[i] = (VOL_HASH_TABLE_SIZE * (p[i]->len >> STORE_BLOCK_SHIFT)) / total;
    used += forvol[i];
    // 按8M 划分整个vol 的空间，既划分采样本。vol 越大被采样的几率越大
    rtable_entries[i] = p[i]->len / VOL_HASH_ALLOC_SIZE;
    rtable_size += rtable_entries[i];
    gotvol[i] = 0;
  }

  // 除不净的从头开始重新分配 VOL_HASH_TABLE_SIZE 是一个小于2^15次方的最大质数
  // 所以很可能除不尽
  // spread around the excess
  int extra = VOL_HASH_TABLE_SIZE - used;
  for (int i = 0; i < extra; i++) {
    forvol[i % num_vols]++;
  }
  // seed random number generator
  for (int i = 0; i < num_vols; i++) {
    uint64_t x = p[i]->hash_id.fold();
    rnd[i]     = (unsigned int)x;
  }
  // initialize table to "empty"
  for (int i = 0; i < VOL_HASH_TABLE_SIZE; i++) {
    ttable[i] = VOL_HASH_EMPTY;
  }
  // generate random numbers proportaion to allocation
  rtable_pair *rtable = (rtable_pair *)ats_malloc(sizeof(rtable_pair) * rtable_size);
  int rindex          = 0;
  // 初始化rtable 空间随机值和对应vol 索引， 每次rval会通过next_rand重新计算
  for (int i = 0; i < num_vols; i++) {
    for (int j = 0; j < (int)rtable_entries[i]; j++) {
      rtable[rindex].rval = next_rand(&rnd[i]);
      rtable[rindex].idx  = i;
      rindex++;
    }
  }
  ink_assert(rindex == (int)rtable_size);
  // sort (rand #, vol $ pairs)
  // 排序rtable 随机值，使rtable内容充分打散
  qsort(rtable, rtable_size, sizeof(rtable_pair), cmprtable);
  // 规定每个bucket采样间隔 为width， 这里随机数类型为unsigned int
  unsigned int width = (1LL << 32) / VOL_HASH_TABLE_SIZE;
  unsigned int pos; // target position to allocate
  // select vol with closest random number for each bucket
  int i = 0; // index moving through the random numbers
  // 填充每个bucket
  for (int j = 0; j < VOL_HASH_TABLE_SIZE; j++) {
    // 这里 pos 起始值不能为0 应为随机值不为0 且不能为 uint32 max 因为 所有值 < uint32_max
    // 所以折中一下， 取样空间在 width / 2 < x < (VOL_HASH_TABLE_SIZE - 1) * width
    pos = width / 2 + j * width; // position to select closest to
    // 找到离采样点最近的随机数（样本）
    while (pos > rtable[i].rval && i < (int)rtable_size - 1) {
      i++;
    }
    // 设置采样点对应的vol 索引
    ttable[j] = mapping[rtable[i].idx];
    gotvol[rtable[i].idx]++;
  }
  for (int i = 0; i < num_vols; i++) {
    Debug("cache_init", "build_vol_hash_table index %d mapped to %d requested %d got %d", i, mapping[i], forvol[i], gotvol[i]);
  }
  // install new table
  if (nullptr != (old_table = ink_atomic_swap(&(cp->vol_hash_table), ttable))) {
    new_Freer(old_table, CACHE_MEM_FREE_TIMEOUT);
  }
  ats_free(mapping);
  ats_free(p);
  ats_free(forvol);
  ats_free(gotvol);
  ats_free(rnd);
  ats_free(rtable_entries);
  ats_free(rtable);
}
```

next_rand
----
next_rand其实是一个固定算法，如果给的参数一定，则next_rand的计算结果一定

ATS vol比例划分 hash 区间
-----
ats 根据每个vol的大小 划分采样样本数量，之后更具width 进行采样。vol 越大，vol对应样本越多，则被采样的概率越大。

ATS hash一致性
----
当某个vol 改变时，如果他的hash id变了，并且他的某个样本更靠近采样点，则很可能会侵占别的vol的bucket。这就会导致一定的偏差，但是这种侵占只保持在小范围内。
同理当某个vol 被踢掉时，其对应的空间会被其他vol划分。所以对应原始bucket hash并不会变。保障原始hash的一致性。

###### 参考
http://blog.chinaunix.net/uid-23242010-id-93352.html
