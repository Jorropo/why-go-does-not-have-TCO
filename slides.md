# Why go does not have TCO ?

https://github.com/Jorropo/thunderdup

## `thunderdup` - Fast concurrent linux file deduplicator

```
~/k/go (master)> thunderdup
Scanning ...
2024-11-12 16:02:26:
	unique:    3548	632 MiB
	duplicate: 34	2.9 MiB
	queue length: 0
	currently working workers: 0/48
Deduplicating ...
total dedupped: 2.9 MiB
dedupping errors: 0
```

Theses slides are licensed under [CC BY](https://creativecommons.org/licenses/by/4.0/) license.

---

# Profiling `(*os.File).Read`.

```go
func read() error {
  f, err := os.Open("big.bin")
  if err != nil { return err }
  defer f.Close()

  _, err = io.Copy(io.Discard, f)
  return err
}
```

## pprof result:

```
          │          
                     
          │          
          │          
┌─────────▼─────────┐
│(*os.File).Read 95%│
└─────────┬─────────┘
          │          
          │          
                     
          │          
                     
          │          
                     
          │          
          │          
  ┌───────▼─────┐    
  │Syscall6 90% │    
  └─────────────┘    
```

---

# Profiling `(*os.File).Read`; bringing up the big guns:

```
doas perf record -g ./read-benchmark.test                
 ▲    ▲    ▲      ▲   ▲                                  
 │    │    │      │   │                                  
 │    │    │      │   File to profile                    
 │    │    │      Collect the callgraph                  
 │    │    Record profile                                
 │    Linux performance measurement tool                 
 opendoas, run as root (otherwise kernel is not profiled)
```

---

# Profiling `(*os.File).Read`; bringing up the big guns:

```
doas perf record -g ./read-benchmark.test                
 ▲    ▲    ▲      ▲   ▲                                  
 │    │    │      │   │                                  
 │    │    │      │   File to profile                    
 │    │    │      Collect the callgraph                  
 │    │    Record profile                                
 │    Linux performance measurement tool                 
 opendoas, run as root (otherwise kernel is not profiled)
```
```
...
- 89.34% vfs_read
    - 83.81% filemap_read
      - 73.67% filemap_get_pages
          - 70.13% page_cache_ra_unbounded
            + 30.32% filemap_add_folio
            + 23.82% folio_alloc_noprof
            + 11.70% read_pages
```

---

# Page cache is expensive

```
- 89.34% vfs_read
 + 83.81% filemap_read
```

```c
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter, ssize_t already_read)
```

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
        ssize_t ret;

        if (!(file->f_mode & FMODE_READ))
                return -EBADF;
        if (!(file->f_mode & FMODE_CAN_READ))
                return -EINVAL;
        if (unlikely(!access_ok(buf, count)))
                return -EFAULT;

        ret = rw_verify_area(READ, file, pos, count);
        if (ret)
                return ret;
        if (count > MAX_RW_COUNT)
                count =  MAX_RW_COUNT;

        if (file->f_op->read)
                ret = file->f_op->read(file, buf, count, pos);
        else if (file->f_op->read_iter)
                ret = new_sync_read(file, buf, count, pos); // ←
        else
                ret = -EINVAL;
        if (ret > 0) {
                fsnotify_access(file);
                add_rchar(current, ret);
        }
        inc_syscr(current);
        return ret;
}
```

---

# 🪄✨

```diff
 NOSTDINC_FLAGS :=
-CFLAGS_MODULE   =
+CFLAGS_MODULE   = -fno-optimize-sibling-calls -fno-omit-frame-pointer
 RUSTFLAGS_MODULE =
 AFLAGS_MODULE   =
 LDFLAGS_MODULE  =
-CFLAGS_KERNEL  =
+CFLAGS_KERNEL  = -fno-optimize-sibling-calls -fno-omit-frame-pointer
 RUSTFLAGS_KERNEL =
 AFLAGS_KERNEL  =
 LDFLAGS_vmlinux =
```

---

```c
static ssize_t btrfs_file_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
        ssize_t ret = 0;

        if (iocb->ki_flags & IOCB_DIRECT) { // ¿
                ret = btrfs_direct_read(iocb, to);
                if (ret < 0 || !iov_iter_count(to) ||
                    iocb->ki_pos >= i_size_read(file_inode(iocb->ki_filp)))
                        return ret;
        }

        return filemap_read(iocb, to, ret); // ←←←
}
```

---

```diff
 func read() error {
-  f, err := os.Open("big.bin")
+  f, err := os.OpenFile("big.bin", os.O_RDONLY|syscall.O_DIRECT, 0)
   if err != nil { return err }
   defer f.Close()
```

With the winds blowing south-south-east at 18 knots on my particular machine this yields 2GiB/s → 6GiB/s.

---

# Why this happens ?

```
┌──────►                      
│                             
│                             
│                             
│      ┌─────────────────────┐
│ ┌────►vfs_read:            │
│ │    │RP = ...             │
└─┼────┼RB                   │
  │    │...                  │
  │    ├─────────────────────┤
  │    │btrfs_file_read_iter:│
  │    │RP = vfs_read        │
  └────┼RB                   │
       │...                  │
       └─────────────────────┘
```
```
┌──────►...                   
│                             
│                             
│                             
│      ┌─────────────────────┐
│ ┌────►vfs_read:            │
│ │    │RP = ...             │
└─┼────┼BP                   │
  │    │...                  │
  │    ├─────────────────────┤
  │    │filemap_read:        │
  │    │RP = vfs_read        │
  └────┼BP                   │
       │...                  │
       └─────────────────────┘
```

---

# What is Tail-Call-Optimization (TCO) to begin with ?

Go source code:

```go
//go:noinline
func F(a, b uint) uint {
  return a + b
}

func G() uint {
  return F(33, 44)
}
```

---

# What is Tail-Call-Optimization (TCO) to begin with ?

Go source code:

```go
//go:noinline
func F(a, b uint) uint {
  return a + b
}

func G() uint {
  return F(33, 44)
}
```

Pseudo assembly (riscv64):

```go
func F(returnPointer, R1, R2) {
  R1 = R1 + R2
  goto returnPointer
}

func G(returnPointer) {
  R1 = 33
  R2 = 44
  savedReturnPointer := returnPointer
  returnPointer = &afterTheFunction
  goto F
afterTheFunction:
  goto savedReturnPointer
}
```

---

# What is Tail-Call-Optimization (TCO) to begin with ?

Go source code:

```go
//go:noinline
func F(a, b uint) uint {
  return a + b
}

func G() uint {
  return F(33, 44)
}
```

Pseudo assembly with TCO (riscv64):

```go
func F(returnPointer, R1, R2) {
  R1 = R1 + R2
  goto returnPointer
}
```

```diff
 func G(returnPointer) {
   R1 = 33
   R2 = 44
-  savedReturnPointer := returnPointer
-  returnPointer = &afterTheFunction
   goto F
-afterTheFunction:
-  goto savedReturnPointer
 }
```

```go 
func G(returnPointer) {
  R1 = 33
  R2 = 44
  goto F
 }
```

---

# Part of the std:

```go
package runtime

func Callers(skip int, pc []uintptr) int
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
```

---

# An optimization about codestyle not efficiency

- Helps a bit with memory by freeing stack frames early.
  - But how big are you stack frames really ?

```go
func loop(f func(), n uint) {
  if n == 0 { return }
  f()
  loop(f, n-1)
}

loop(func() { fmt.Println("Bonjour !") }, 1024*1024*1024)
```

---

# An optimization about codestyle not efficiency

- Helps a bit with memory by freeing stack frames early.
  - But how big are you stack frames really ?

```go
func loop(f func(), n uint) {
  if n == 0 { return }
  f()
  loop(f, n-1)
}

loop(func() { fmt.Println("Bonjour !") }, 1024*1024*1024)
```

## Issue churn for the go compilers ecosystem

Once one compiler implements TCO for the example above, rather than being "wrong" everywhere, users could file bugs arguing « but it works on $COMPILER ».

It's not fun to have go code that only works on some go compilers.

---

# Thanks

[Slides](https://github.com/Jorropo/why-go-does-not-have-TCO)