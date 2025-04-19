+++
date = '2025-04-19T23:51:59+08:00'
draft = false 
title = 'Badger Scan 性能探究'
searchHidden = true
ShowReadingTime =  true
ShowBreadCrumbs =  true
ShowPostNavLinks =  true
ShowWordCount =  true
ShowRssButtonInSectionTermList =  true
UseHugoToc = true
showToc = true
TocOpen = false
hidemeta = false
comments = false
description = ''
disableHLJS = true 
disableShare = false
hideSummary = false
tags = ["Badger", "Performance", "KV"]
+++

[Badger](https://docs.hypermode.com/badger/overview) 是go实现的高性能KV库，与RocksDB类似，也是使用LSM实现的。但不同的是，对于大value, 为了减少了LSM的读写放大的问题，把value记录到单独的value log 中。在LSM记录的value 只是相对应的value log 的offset , 也就是log file ID 和所在文件的offset。本文描述是基于 1.6.2 版本。

KKV是一种数据结构，尤其在推荐领域广泛用到。比如说用户浏览过的文章，视频或者商品列表。对于同一个用户来说，浏览过的物料实际是一个list 。可以简单表示如下：

| user_id | Item_id | Timestamp  |
| ------- | ------- | ---------- |
| User1   | Item1   | 1745070155 |
| User1   | Item2   | 1745070155 |
| User1   | Item3   | 1745070155 |

那么在底层，如果使用badger进行存储的时候，会把（user_id, item_id） 当成 key 存储。那么如果针对某个 user_id ，我们需要进行前缀匹配操作，匹配到user_id开头的数据都获取到，进而获取到item_id 列表。badger 的scan操作，官方的例子如下：

```go
db.View(func(txn *badger.Txn) error {
  it := txn.NewIterator(badger.DefaultIteratorOptions)
  defer it.Close()
  prefix := []byte("1234")
  for it.Seek(prefix); it.Valid(); it.Next() {
    item := it.Item()
    k := item.Key()
    err := item.Value(func(v []byte) error {
      fmt.Printf("key=%s, value=%s\n", k, v)
      return nil
    })
    if err != nil {
      return err
    }
  }
  return nil
})
```

我们简单看下整体的逻辑。

1. txn.NewIterator 实际会构造一个遍历列表，包括 memtable 和 ssttable 。而且这些table 中，key 都是有序的。
2. it.Seek 根据prefix 定位包含prefix的开始的key ，从这里开始，后面的一个或者多个 key 都会包含 prefix 。而且会进行一定的预取。
3. it.Valid 会判断当前的key 是否符合条件（也会判断整个Iterator列表是否遍历完了)，与 prefix 也进行比较。如果不满足条件，退出循环。
4. 如果符合条件，通过 it.Next 遍历下一个 key 。

我们先看最简单的情况，根据userid 写入一批itemid，然后查询 userid 对应的列表。 

```go
func ScanDeleteValueV1() {
	path := "/Users/bruceding/Projects/go/src/badger-test/scandelv1"
	db, err := badger.Open(badger.DefaultOptions(path).WithMaxTableSize(16 << 20).WithCompactL0OnClose(false))
	if err != nil {
		panic(err)
	}
	defer db.Close()
	for i := 0; i < 500; i++ {
		for j := 0; j < 1000; j++ {
			val := RandomString(32)
			key := fmt.Sprintf("user_%d#item_%d", i, j)
			db.Update(func(txn *badger.Txn) error {
				err := txn.Set([]byte(key), []byte(val))
				return err
			})
		}
	}
	doScanV1(db, 0)
}
func doScanV1(db *badger.DB, i int) {
	start := time.Now()
	db.View(func(txn *badger.Txn) error {
		buf := bytes.NewBufferString(fmt.Sprintf("user_%d#", i))
		prefix := buf.Bytes()

		opts := badger.DefaultIteratorOptions
		opts.PrefetchValues = false
		opts.Prefix = prefix
		it := txn.NewIterator(opts)
		defer it.Close()

		var keys []string
		for it.Seek(prefix); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			keys = append(keys, string(key))
		}
		fmt.Println(len(keys))
		return nil
	})

	fmt.Println("time cost", time.Since(start))
}
```

在我们的测试中，会返回如下，看时间性能还是可以的。

```
1000
time cost 202.708µs
```

我们看一个特殊情况，就是某个userid 会对应一批itemid 列表，然后把userid 的数据全部删除，然后再根据 userid 查询会发生什么？示例代码如下：

```go
func ScanDeleteValueV1() {
	path := "/Users/bruceding/Projects/go/src/badger-test/scandelv1"
	db, err := badger.Open(badger.DefaultOptions(path).WithMaxTableSize(16 << 20).WithCompactL0OnClose(false))
	if err != nil {
		panic(err)
	}
	defer db.Close()
	var keys []string
	for i := 0; i < 500; i++ {
		for j := 0; j < 1000; j++ {
			val := RandomString(32)
			key := fmt.Sprintf("user_%d#item_%d", i, j)
			keys = append(keys, key)
			db.Update(func(txn *badger.Txn) error {
				err := txn.Set([]byte(key), []byte(val))
				return err
			})
		}
	}
  // 删除 keys
	for _, key := range keys {
		db.Update(func(txn *badger.Txn) error {
			err := txn.Delete([]byte(key))
			return err
		})
	}

	doScanV1(db, 0)
	doScanV2(db, 0)
}
func doScanV1(db *badger.DB, i int) {
	start := time.Now()
	db.View(func(txn *badger.Txn) error {
		buf := bytes.NewBufferString(fmt.Sprintf("user_%d#", i))
		prefix := buf.Bytes()

		opts := badger.DefaultIteratorOptions
		opts.PrefetchValues = false
		opts.Prefix = prefix
		it := txn.NewIterator(opts)
		defer it.Close()

		var keys []string
		for it.Seek(prefix); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			keys = append(keys, string(key))
		}
		fmt.Println(len(keys))
		return nil
	})

	fmt.Println("time cost", time.Since(start))
}
```

那么会如下输出，这个时候看到性能是相当差的。

```
0
time cost 81.809833ms
```

那么具体的原因要从 it.Seek 这个函数看起。

```go
func (it *Iterator) Seek(key []byte) {
	for i := it.data.pop(); i != nil; i = it.data.pop() {
		i.wg.Wait()
		it.waste.push(i)
	}

	it.lastKey = it.lastKey[:0]
	if len(key) == 0 {
		key = it.opt.Prefix
	}
	if len(key) == 0 {
		it.iitr.Rewind()
		it.prefetch()
		return
	}

	if !it.opt.Reverse {
		key = y.KeyWithTs(key, it.txn.readTs)
	} else {
		key = y.KeyWithTs(key, 0)
	}
	it.iitr.Seek(key)
	it.prefetch()
}
```

实际上主要最后两行代码:

1. it.iitr.Seek(key) 会定位，匹配 key(prefix) 开始的第一个key ,从第一个key 开始进行遍历数据
2. it.prefetch() 会预取数据, 默认会预取两条。因为所有的key 都是被删除的，没有一个合法的key 。 那么it.parseItem 那么总是返回false, 数据会一直进行遍历。直到把所有的数据都遍历完。那么这也是性能的差的原因。这里遍历不是只遍历了前缀匹配的 key ，而且整个数据的key 都遍历了一遍，如果 key 的数量很多的话，性能自然会差。

```go
func (it *Iterator) prefetch() {
	prefetchSize := 2
	if it.opt.PrefetchValues && it.opt.PrefetchSize > 1 {
		prefetchSize = it.opt.PrefetchSize
	}

	i := it.iitr
	var count int
	it.item = nil
	for i.Valid() {
		if !it.parseItem() {
			continue
		}
		count++
		if count == prefetchSize {
			break
		}
	}
}
```

在 parseItem 实现中，我们可以看到类似的代码，AllVersions 为true 的情况下，会返回 true，会及时退出遍历。

```go
	if it.opt.AllVersions {
		// Return deleted or expired values also, otherwise user can't figure out
		// whether the key was deleted.
		item := it.newItem()
		it.fill(item)
		setItem(item)
		mi.Next()
		return true
	}
```

那么我们通过设置 AllVersions 来看下性能如何？还是全部删除 key 的情况下：

```go
func ScanDeleteValueV1() {
	path := "/Users/bruceding/Projects/go/src/badger-test/scandelv1"
	db, err := badger.Open(badger.DefaultOptions(path).WithMaxTableSize(16 << 20).WithCompactL0OnClose(false))
	if err != nil {
		panic(err)
	}
	defer db.Close()
	var keys []string
	for i := 0; i < 500; i++ {
		for j := 0; j < 1000; j++ {
			val := RandomString(32)
			key := fmt.Sprintf("user_%d#item_%d", i, j)
			keys = append(keys, key)
			db.Update(func(txn *badger.Txn) error {
				err := txn.Set([]byte(key), []byte(val))
				return err
			})
		}
	}
	for _, key := range keys {
		db.Update(func(txn *badger.Txn) error {
			err := txn.Delete([]byte(key))
			return err
		})
	}

	doScanV1(db, 0)
	doScanV2(db, 0)
}
func doScanV1(db *badger.DB, i int) {
	start := time.Now()
	db.View(func(txn *badger.Txn) error {
		buf := bytes.NewBufferString(fmt.Sprintf("user_%d#", i))
		prefix := buf.Bytes()

		opts := badger.DefaultIteratorOptions
		opts.PrefetchValues = false
		opts.Prefix = prefix
		it := txn.NewIterator(opts)
		defer it.Close()

		var keys []string
		for it.Seek(prefix); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			keys = append(keys, string(key))
		}
		fmt.Println(len(keys))
		return nil
	})

	fmt.Println("time cost", time.Since(start))
}

func doScanV2(db *badger.DB, i int) {
	start := time.Now()
	db.View(func(txn *badger.Txn) error {
		buf := bytes.NewBufferString(fmt.Sprintf("user_%d#", i))
		prefix := buf.Bytes()

		opts := badger.DefaultIteratorOptions
		opts.PrefetchValues = false
		opts.Prefix = prefix
		opts.AllVersions = true
		it := txn.NewIterator(opts)
		defer it.Close()

		var keys []string
		deleteKeyMap := make(map[string]bool, 1024)
		for it.Seek(prefix); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			if item.IsDeletedOrExpired() {
				deleteKeyMap[string(key)] = true
				continue
			}
			if _, exist := deleteKeyMap[string(key)]; exist {
				continue
			}
			keys = append(keys, string(key))
		}
		fmt.Println(len(keys))
		return nil
	})
	fmt.Println("time cost", time.Since(start))

}
```

会看到如下输出，doScanV2 性能提升不少。

```
0
time cost 81.809833ms
0
time cost 344.042µs
```

doScanV2 主要原因就是遍历的key 数量会减少。it.Seek 返回之后，通过 it.Next 进行遍历下一个key ， it.Valid 会判断前缀是否匹配，如果没有匹配会及时退出。而 doScanV1 会在 it.Seek 里一直遍历数据运行，因为没有匹配前缀的判断逻辑，会一直遍历数据运行，直到全部遍历完成才行。

在LSM 中,数据的插入，更新，删除都是通过 append 的方式进行的。比如说插入一个 key1 = value1， 如果要删除 key1 时，会创建一个新条目标识key1 的元数据是被删除的，如果追加到 LSM 表中。这也是下面的代码的原因。如果 item 本身已经删除或者过期直接 continue ,同时记录删除的key 。当遍历到 插入key 的条目时，如果已经存在删除记录，也是直接 continue。

```go
		if item.IsDeletedOrExpired() {
				deleteKeyMap[string(key)] = true
				continue
			}
			if _, exist := deleteKeyMap[string(key)]; exist {
				continue
			}
```

最终的结论是在任何场景下，都推荐 doScanV2 的写法。
