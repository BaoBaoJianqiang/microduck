# fsutil.rs 文件解析

## 文件位置

`d:\microduck\updater\src\fsutil.rs`

## 定位

持久化文件系统原语。设计的崩溃保证依赖两个 rename **既持久又有序**：boot-counter 记录必须在使 symlink 交换可见的断电中存活。`rename(2)` 原子但不持久（目录项可能仍在页缓存），所以每个依赖的 rename 后都 fsync 包含它的目录。

## `fsync_parent(path)`

fsync `path` 的父目录，使其中的 rename 持久。处理 `parent()` 为 `Some("")`（裸文件名）的情况→用 `.`。

## `write_atomic(path, bytes)`

通过临时文件写入+rename，使读者永不看到部分文件，然后 fsync 文件与目录使结果在断电中存活。顺序：写 tmp → fsync 文件内容 → rename → fsync 父目录。

## 关键摘要

fsutil.rs 提供崩溃安全的文件操作：`fsync_parent` 保证目录项持久（rename 本身不持久），`write_atomic` 通过 tmp+rename+双 fsync 保证读者不见部分文件且断电存活。
