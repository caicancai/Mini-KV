# Mini-KV
一个基于bitcask模型的简单kv存储

GET 方法需要先从内存中取出索引信息，判断是否存在，不存在直接返回，存在的话从磁盘当中取出数据。

```go
func (db *MiniDB) Get(key []byte) (val []byte, err error) {
   // 从内存当中取出索引信息
   offset, ok := db.indexes[string(key)]
   // key 不存在
   if !ok {
      return
   }

   // 从磁盘中读取数据
   var e *Entry
   e, err = db.dbFile.Read(offset)
   if err != nil && err != io.EOF {
      return
   }
   if e != nil {
      val = e.Value
   }
   return
}
```

PUT先更新磁盘，写入一条记录，再更新内存

```go
func (db *MiniDB) Put(key []byte, value []byte) (err error) {
  
   offset := db.dbFile.Offset
   // 封装成 Entry
   entry := NewEntry(key, value, PUT)
   // 追加到数据文件当中
   err = db.dbFile.Write(entry)

   // 写到内存
   db.indexes[string(key)] = offset
   return
}
```

删除操作，这里并不会定位到原记录进行删除，而还是将删除的操作封装成 Entry，追加到磁盘文件当中，只是这里需要标识一下 Entry 的类型是删除,然后在内存当中的哈希表删除对应的 key 的索引信息

```go
func (db *MiniDB) Del(key []byte) (err error) {
   // 从内存当中取出索引信息
   _, ok := db.indexes[string(key)]
   // key 不存在，忽略
   if !ok {
      return
   }

   // 封装成 Entry 并写入
   e := NewEntry(key, nil, DEL)
   err = db.dbFile.Write(e)
   if err != nil {
      return
   }

   // 删除内存中的 key
   delete(db.indexes, string(key))
   return
}
```

merge 的思路也很简单，需要取出原数据文件的所有 Entry，将有效的 Entry 重新写入到一个新建的临时文件中，最后将原数据文件删除，临时文件就是新的数据文件了

```go
func (db *MiniDB) Merge() error {
   // 读取原数据文件中的 Entry
   for {
      e, err := db.dbFile.Read(offset)
      if err != nil {
         if err == io.EOF {
            break
         }
         return err
      }
      // 内存中的索引状态是最新的，直接对比过滤出有效的 Entry
      if off, ok := db.indexes[string(e.Key)]; ok && off == offset {
         validEntries = append(validEntries, e)
      }
      offset += e.GetSize()
   }

   if len(validEntries) > 0 {
      // 新建临时文件
      mergeDBFile, err := NewMergeDBFile(db.dirPath)
      if err != nil {
         return err
      }
      defer os.Remove(mergeDBFile.File.Name())

      // 重新写入有效的 entry
      for _, entry := range validEntries {
         writeOff := mergeDBFile.Offset
         err := mergeDBFile.Write(entry)
         if err != nil {
            return err
         }

         // 更新索引
         db.indexes[string(entry.Key)] = writeOff
      }

      // 删除旧的数据文件
      os.Remove(db.dbFile.File.Name())
      // 临时文件变更为新的数据文件
      os.Rename(mergeDBFile.File.Name(), db.dirPath+string(os.PathSeparator)+FileName)

      db.dbFile = mergeDBFile
   }
   return nil
}
```
参考：https://zhuanlan.zhihu.com/p/385820641
