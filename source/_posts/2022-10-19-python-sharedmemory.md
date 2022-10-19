---
title: python共享内存SharedMemory踩坑记录
date: 2022-10-19 06:15:31
tags:
  - python
---
## 起因
在处理实时视频流分析中, 如果AI推理推理时间太长导致, 会导致视频流中断和延迟.
尝试情况:
- opencv + rtsp + ffmpeg + 多线程. 大概几十到几百帧就中断了
- opencv + rtsp + ffmpeg + 多线程 + 重连. 在重连阶段会出现没有图像
- opencv + rtsp + ffmpeg + 多进程 + 图像帧队列. 会出现中断, 可能是多进程间传输图像太大. 而且由于传输数据, 导致从抓取到推理时间变长很多
- opencv + rtsp + gstreamer + 多线程. 解决中断问题. 推理时间短的话, 图像延迟较少, 如果推理时间长, 会造成延迟很久


由于python GIL(Global Interpreter Lock) 全局锁的限制, 多线程无法利用多核, 在一个cpu上交替运行的. 
导致推理和读取视频交替, 如果推理时间长, 那么读取视频的次数就变少了, 而视频的帧数是相对不变的, 这样就造成了视频图像延迟.
所以多线程无法从根本上解决延迟.

多进程可以利用多核, 这样推理和读取视频在不同进程, 就互不影响, 读取视频可以做到实时读取. 而进程之间的通信方式有管道, 队列, 共享内存, 信号量, 信号, socket, 其中大数据传输最快的就是共享内存. 

python3.8开始支持共享内存`multiprocessing.shared_memory`. 经过测试, 多进程加共享内存, 可以解决中断和延迟问题.

## shared_memory 学习
官方文档比较简单, 但也覆盖了基本使用. 通过搜该关键字实际使用的也比较少, 比较相关的开源项目是 [blakeblackshear/frigate](https://github.com/blakeblackshear/frigate), 它是一个实时的本地IP摄像头对象检测的NVR(网络视频记录器).

这里说些文档没有的.
`size` 参数是字节长度. 比如保存的是图像, 图像的长是1920, 宽是1080, 彩色的, 是用`opencv`读取的, 那么 `size = 1920 * 1080 * 3`. 可以通过`img.nbytes`查看.

由于内存是一维的, 所以赋值的时候需要把三维的图像转换为一维, 例如`img.flatten()`.

从内存里面取出来的也是一维, 需要转换为三维图片. 而且需要指定读取的字节长度, 因为创建的长度可能因为平台不同, 有的平台会限制最小单位, 所以实际创建的会等于或者大于指定的`size`. 
```python
# count是字节长度, shape是返回的形状
np.frombuffer(bytes(shm.buf[:]), dtype=np.uint8, count=count).reshape(shape)
```

## 坑1: windows 上 close, unlink 和文档不一致
文档没有说明windows和linux的区别. 
在windwos上会出现的不一致情况:
- 在windows上调用close(), 如果之前还没有其他进程获取该内存块名的话, 那么其他线程无法在获取. 有人说是没有继承父进程的追踪器, 导致每个进程有自己的追踪器, 所以子进程的追踪器发现close, 就释放的该内存名, 其他子进程就获取不到了.  由于追踪器没有提供文档, 也不想在深入了解了, 而且也想出来一个简单的解决办法, p1进程先不close, 把共享内存名通过队列a传输到p2进程, 等p2进程处理完close, 在通过队列b传输内存名到p1进程, p1进程进行close. 
- unlink, 查看源码, 在windows上是不处理的.

## 坑2: windows 上内存泄漏
在windows上就算close了, 如果进程没有结束, 内存就没有释放掉, 造成内存一直增长.
这个bug 2020年在python3.8就发现, 并提出了解决办法,快2年多了都没处理, 可以看下面的参考链接. 目前只能通过 `monkey-patch` 来解决了, 通过引入下面的`monkey_shared_memory()` 方法.
```python
import os
import sys


def monkey_shared_memory():
    if os.name == "nt" and sys.version_info[:2] >= (3, 8):
        from . import shared_memory

```

```python
# shared_memory.py
"""
解决windows上shared-memory内存泄漏问题
https://bugs.python.org/issue40882
https://stackoverflow.com/questions/65968882/unlink-does-not-work-in-pythons-shared-memory-on-windows
"""
import ctypes
import ctypes.wintypes
import errno
import mmap
import multiprocessing
import multiprocessing.shared_memory
import os
import secrets
from multiprocessing.shared_memory import SharedMemory

if os.name == "nt":
    import _winapi

    _USE_POSIX = False
else:
    import _posixshmem

    _USE_POSIX = True


_O_CREX = os.O_CREAT | os.O_EXCL

# FreeBSD (and perhaps other BSDs) limit names to 14 characters.
_SHM_SAFE_NAME_LENGTH = 14

# Shared memory block name prefix
if _USE_POSIX:
    _SHM_NAME_PREFIX = "/psm_"
else:
    _SHM_NAME_PREFIX = "wnsm_"


def _make_filename():
    "Create a random filename for the shared memory object."
    # number of random bytes to use for name
    nbytes = (_SHM_SAFE_NAME_LENGTH - len(_SHM_NAME_PREFIX)) // 2
    assert nbytes >= 2, "_SHM_NAME_PREFIX too long"
    name = _SHM_NAME_PREFIX + secrets.token_hex(nbytes)
    assert len(name) <= _SHM_SAFE_NAME_LENGTH
    return name


UnmapViewOfFile = ctypes.windll.kernel32.UnmapViewOfFile
UnmapViewOfFile.argtypes = (ctypes.wintypes.LPCVOID,)
UnmapViewOfFile.restype = ctypes.wintypes.BOOL


class _SharedMemory(SharedMemory):
    def __init__(self, name=None, create=False, size=0):
        if not size >= 0:
            raise ValueError("'size' must be a positive integer")
        if create:
            self._flags = _O_CREX | os.O_RDWR
            if size == 0:
                raise ValueError("'size' must be a positive number different from zero")
        if name is None and not self._flags & os.O_EXCL:
            raise ValueError("'name' can only be None if create=True")

        if _USE_POSIX:

            # POSIX Shared Memory

            if name is None:
                while True:
                    name = _make_filename()
                    try:
                        self._fd = _posixshmem.shm_open(
                            name, self._flags, mode=self._mode
                        )
                    except FileExistsError:
                        continue
                    self._name = name
                    break
            else:
                name = "/" + name if self._prepend_leading_slash else name
                self._fd = _posixshmem.shm_open(name, self._flags, mode=self._mode)
                self._name = name
            try:
                if create and size:
                    os.ftruncate(self._fd, size)
                stats = os.fstat(self._fd)
                size = stats.st_size
                self._mmap = mmap.mmap(self._fd, size)
            except OSError:
                self.unlink()
                raise

            from .resource_tracker import register

            register(self._name, "shared_memory")

        else:

            # Windows Named Shared Memory

            if create:
                while True:
                    temp_name = _make_filename() if name is None else name
                    # Create and reserve shared memory block with this name
                    # until it can be attached to by mmap.
                    h_map = _winapi.CreateFileMapping(
                        _winapi.INVALID_HANDLE_VALUE,
                        _winapi.NULL,
                        _winapi.PAGE_READWRITE,
                        (size >> 32) & 0xFFFFFFFF,
                        size & 0xFFFFFFFF,
                        temp_name,
                    )
                    try:
                        last_error_code = _winapi.GetLastError()
                        if last_error_code == _winapi.ERROR_ALREADY_EXISTS:
                            if name is not None:
                                raise FileExistsError(
                                    errno.EEXIST,
                                    os.strerror(errno.EEXIST),
                                    name,
                                    _winapi.ERROR_ALREADY_EXISTS,
                                )
                            else:
                                continue
                        self._mmap = mmap.mmap(-1, size, tagname=temp_name)
                    finally:
                        _winapi.CloseHandle(h_map)
                    self._name = temp_name
                    break

            else:
                self._name = name
                # Dynamically determine the existing named shared memory
                # block's size which is likely a multiple of mmap.PAGESIZE.
                h_map = _winapi.OpenFileMapping(_winapi.FILE_MAP_READ, False, name)
                try:
                    p_buf = _winapi.MapViewOfFile(h_map, _winapi.FILE_MAP_READ, 0, 0, 0)
                finally:
                    _winapi.CloseHandle(h_map)

                try:
                    size = _winapi.VirtualQuerySize(p_buf)
                    self._mmap = mmap.mmap(-1, size, tagname=name)
                finally:
                    UnmapViewOfFile(p_buf)

        self._size = size
        self._buf = memoryview(self._mmap)


multiprocessing.shared_memory.SharedMemory = _SharedMemory

```


## 参考链接
- [multiprocessing.shared_memory --- 可从进程直接访问的共享内存](https://docs.python.org/zh-cn/3.8/library/multiprocessing.shared_memory.html)
- ['unlink()' does not work in Python's shared_memory on Windows](https://stackoverflow.com/questions/65968882/unlink-does-not-work-in-pythons-shared-memory-on-windows)
- [memory leak in multiprocessing.shared_memory.SharedMemory in Windows](https://bugs.python.org/issue40882)
- [SharedMemory.close() destroys memory](https://github.com/python/cpython/issues/91044)
- [Bug on multiprocessing.shared_memory](https://github.com/python/cpython/issues/84140)
- [blakeblackshear/frigate](https://github.com/blakeblackshear/frigate)