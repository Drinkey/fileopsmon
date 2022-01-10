# fileopsmon
Monitor file operations in specified directories

INotify 类主体 copy 自 https://github.com/chrisjbillington/inotify_simple 。由于使用场景和环境限制，只能将其 copy 后并稍作修改以适配环境。

该代码只在 Py 2.7环境下调试通过，能够按需完成工作。由于运行环境不支持 Python 3.x，所以没有调试，同时 Python 3.8的环境可以使用 asyncio 来实现多目录同时监控的调度，而不是使用多线程。

使用inotify接口实现对文件系统事件进行监控，通过加载 `libc.so`，对其接口进行调用。

inotify 只会监控指定目录下文件的文件操作，而不会监控其子目录中的文件操作。举例来说
```sh
automation@u1804-kvm-14-13:~$ tree .
.
└── level_1_dir
    ├── 1.txt
    ├── 2.txt
    └── level_2_dir
        ├── 2_1.txt
        └── 2_2.txt

2 directories, 4 files
```

如果我们监控的目标目录是`level_1_dir`，那么
- `1.txt` 和 `2.txt`的文件操作，以及对`level_2_dir`本身的目录操作会被监控
- 对`level_2_dir`目录中的文件`2_1.txt` 和`2_2.txt`的文件操作不会被监控

如果需要监控`level_2_dir`目录中的文件`2_1.txt`和`2_2.txt`的文件操作，需要将`level_2_dir`也加入监控目标中，此时，监控目标为`level_1_dir` 和 `level_1_dir/level_2_dir`。此时，level_1_dir及其子目录中的文件操作都会被监控。

## Usage

```sh
support@Appliance:/tmp/inotify# python fileops.py -h
usage: fileops.py [-h] [--recursive] [--watchlist WATCHLIST_FILE]
              [--flags {read,write,all}]
              [N [N ...]]

Monitor file operations in specified dirs.

positional arguments:
  N                     Directories to watch

optional arguments:
  -h, --help            show this help message and exit
  --recursive, -r       [Danger!]Whether recursively look for sub directories
  --watchlist WATCHLIST_FILE, -i WATCHLIST_FILE
                        Where read the list of dir to watch
  --flags {read,write,all}, -f {read,write,all}
                        What file operations wants to watch
```

## Examples

监控`/etc/watch_dir` and `/etc/watch_dir/certs`两个目录的写操作

```sh
support@Appliance:/tmp/inotify# python fileops.py /etc/watch_dir /etc/watch_dir/certs -f write
```

监控`/etc/watch_dir`及其子目录的读操作
```sh
support@Appliance:/tmp/inotify# python fileops.py /etc/watch_dir -f read -r
```

请注意，`--recursive` 参数可能会带来麻烦，当指定目录下有很多子目录时，系统可能无法同时监控所有子目录，系统会报如下错误：`thread.error: can't start new thread`，导致一部分目录没有被真正加入监控列表中，使程序处于异常状态，无法通过 Ctrl+C结束，必须通过`kill -9 <pid>`来结束。


Assume you have a file contains a list of dir or files you want to monitor
```sh
support@Appliance:/tmp/inotify# cat watchlist
/etc
/etc/watch_dir
/etc/watch_dir/certs
/tmp/debug
support@Appliance:
```

Start the script
```sh
support@Appliance:/tmp/inotify# python fileops.py -i watchlist -f read
```
- `-f` or `--flags` specifies what flags you want to monitor, options could be `read`, `write` or `all`
- `-i` or `--watchlist` specifies the watchlist file. The watch list can be specified in arguments like the following:

