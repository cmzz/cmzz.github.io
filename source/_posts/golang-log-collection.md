---
title: Go 实时日志收集
tags: 
- 技术
- golang
- 并发编程
date: 2022-08-12 08:00:00
---

通常基础系统都会自动处理与收集日志，并不太需要应用来收集。但利用 go 的并发与携程能力，要实现实时的日志收集也是非常简单。

功能设计

![日志收集程序设计](https://tva1.sinaimg.cn/large/e6c9d24egy1h54ajve0rlj20qf0d3q4e.jpg)

- 主程序向指定的日志文件记录日志
- 收集程序独立于主程序之外，通过实时监控程序日志的方式即时读取新写入的日志
- 在读取到日志行之后，将其通过 channel 传送给日志处理进程
- 处理完成后视需要存储到 `es / logstash / influxdb`

优点

日志收集程序独立于主程序之外运行，对程序无侵入性。

实现

```golang
package logCollection

import (
   "bufio"
   "encoding/json"
   "errors"
   "io"
   "os"
   "strings"
   "syscall"
   "time"
)

var logIno uint64 = 0
var logCount int

type LogLine struct {
   Level      string  `json:"level"`
   TS         float64 `json:"ts"`
   Caller     string  `json:"caller"`
   Msg        string  `json:"msg"`
   StackTrace string  `json:"stacktrace"`
}

func getFileIno(path string) uint64 {
   fileinfo, _ := os.Stat(path)
   stat, ok := fileinfo.Sys().(*syscall.Stat_t)
   if !ok {
      return 0
   }

   return stat.Ino
}

func openLog(path string, rc chan []byte) {
   var f *os.File
   var err error

   // create file if not exists
   if _, err = os.Stat(path); errors.Is(err, os.ErrNotExist) {
      f, err = os.Create(path)
      if err != nil {
         panic(err)
      }
   } else {
      f, err = os.Open(path)
      if err != nil {
         panic(err)
      }
   }

   logIno = getFileIno(path)
   if logIno == 0 {
      panic(errors.New("open log file failed"))
   }

   defer func(f *os.File) {
      _ = f.Close()
   }(f)

   f.Seek(0, 2)
   buf := bufio.NewReader(f)

   for {
      // check file ino
      // if changed, reopen file
      if logIno != getFileIno(path) {
         variable.Logger.Error("openLog logIno != new : ", logIno)
         _ = f.Close()

         f, err = os.Open(path)
         if err != nil {
            variable.Logger.ErrorF("open log file failed: %s\n", err)
            panic(err)
         }

         logIno = getFileIno(path)
         continue
      }

      line, err := buf.ReadBytes('\n')
      switch {
      case err == io.EOF:
         time.Sleep(time.Second)
      case err != nil:
         break
      default:
         rc <- line[:len(line)-1]
      }
   }
}

func Write(path string) {
   go func() {
      for {
         time.Sleep(time.Second)
      }
   }()

   c := make(chan []byte)
   go openLog(path, c)

   for {
      line := <-c
      process(line)
   }
}

func process(line []byte) {
   var logLine LogLine
   err := json.Unmarshal(line, &logLine)
   if err != nil {
      return
   }

   if logLine.Level == "debug" || logLine.Level == "info" {
      return
   }

   // 精简 trace
   traces := strings.Split(logLine.StackTrace, "\n")
   if len(traces) > 2 {
      traces = traces[2:]
      if len(traces) > 3 {
         traces = traces[:3]
      }
   }
   
   write(&logLine)
}

func write(log *LogLine) {
   // write to es
}
```

注意上面代码中的一个小细节：

在我们系统中，主程只负责向 `app.log` 写日志，但 `app.log` 会在每天凌晨被重命名为 `app_2022-08-10.log` 以实现日志分割。

那么如何检测文件被移动呢？

上面代码中的实现是通过 `file index no, fio` 来判断，如果打开的 `app.log` fio 与磁盘上 `app.log` 文件的 fio 不一致，则认为是创建了新的日志文件，此时再重新打开日志即可。
