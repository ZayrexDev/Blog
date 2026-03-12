---
title: CLI程序美化教程（一）：使用\r制作动态进度条
date: 2026-03-12 18:17:16
tags:
---

## 0x00 引言

> 本部分没有任何干货，可以直接跳过

在我刚学习写代码的时候，经常做一些命令行的小程序玩（因为GUI还不会做（）），
虽然能够实现我想要的功能，但这极致简约风格的黑底白字我是实在看得不得劲。
再加上之后跑各种程序的时候又见识到了各种大佬的命令行程序，才知道CLI程序
做的不好看原来触及到的是我的上限而非终端的上限😭

<div style="display: flex; justify-content: center; text-align: center;">
  <figure>
    <figcaption>我做的：</figcaption>
    <img src="https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/cli-beautify-01-using-return-asserts/ugly.png" width="100%" alt="有点弱" />
    <figcaption><i>有点弱</i></figcaption>
  </figure>
  
  <figure>
    <figcaption>别人做的：</figcaption>
    <img src="https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/cli-beautify-01-using-return-asserts/gemini.png" width="100%" alt="Description 2" />
    <figcaption><b>！？强强？！</b></figcaption>
  </figure>
</div>

于是我就去了解了一下CLI程序到底该怎么做得美观，自己也写了一些好看一点的小程序，
便于此留下学习经验供各位参考。

本文章作为CLI程序美化系列（可能会存在）的第一篇文章，从最基础的`\r`开始介绍，
初步实现行刷新功能。在之后的文章中，我会介绍使用`ANSI`实现彩色、样式输出和
全屏渲染控制，和使用`JLine`读取键盘输入实现交互的方法。

## 0x01 \r 是什么

在大部分情况下，`\r`指的是`回车`。这里的`回车`和`换行`不同，前者是将光标移动到行首，后者是将光标移动到下一行。而我们知道，当光标位于一行的行首时，再输出文字，就可以将已经输出的文字覆盖掉。如果根据程序运行情况实时覆盖，就能起到动态进度条、状态栏等的效果。

## 0x02 \r 的简单使用

要使用`\r`的话，直接输出即可。下面是一个例子：

```java
System.out.print("正在加载中..."); // 不能使用println()
// Doing something ...
Thread.sleep(2000);
System.out.print("\r加载完成！   "); // 加空格补齐长度
```

这里有几点需要注意：

首先部分IDE内置的运行调试窗口可能不支持光标移动的操作，所以可能需要进行
额外设置，或是手动编译运行。

其次在输出待覆盖文字时**不能换行**，即不能调用`println()`或者加`\n`，
否则光标就会移动到下一行，而无法覆盖上一行的文字。

最后如果要覆盖的文字的长度比被覆盖的文字短，则不会被完全覆盖，所以必要
时需要**使用空格补齐**。

## 0x03 使用\r制作带进度条的下载器

以下程序实现了一个命令行下载器，并将下载进度动态更新至进度条。

```java
import java.io.FileOutputStream;
import java.io.InputStream;
import java.net.URL;
import java.net.URLConnection;
import java.util.Scanner;

public class CLIDownloader {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        System.out.print("请输入下载地址: ");
        String fileURL = scanner.nextLine();
        System.out.print("请输入保存的文件名: ");
        String saveFileName = scanner.nextLine();
        System.out.println("连接中，准备开始下载...");

        try {
            URL url = new URL(fileURL);
            URLConnection conn = url.openConnection();
            long totalBytes = conn.getContentLengthLong();

            InputStream in = conn.getInputStream();
            FileOutputStream out = new FileOutputStream(saveFileName);

            byte[] buffer = new byte[64 * 1024];
            int bytesRead;
            long downloadedBytes = 0;

            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
                downloadedBytes += bytesRead;
                // 借助 \r 打印动态进度条
                if (totalBytes > 0) {
                    int percent = (int) (downloadedBytes * 100 / totalBytes);
                    StringBuilder bar = new StringBuilder("[");
                    for (int i = 0; i < 50; i++) {
                        bar.append(i < percent / 2 ? "=" : " ");
                    }
                    bar.append("]");
                    // \r 让光标回到行首，实现覆盖刷新
                    System.out.print("\r" + bar + " " + percent + "%");
                }
            }

            in.close();
            out.close();
            System.out.println("\n下载完成！文件已保存为: " + saveFileName);
        } catch (Exception e) {
            System.out.println("\n下载出错: " + e.getMessage());
            e.printStackTrace();
        } finally {
            scanner.close();
        }
    }
}
```

> 虽然本文章使用`Java`进行演示、但大部分编程语言理论上都支持使用`\r`进行光标移动操作🤔

运行演示：

![demo](https://zcraftasserts-1302810751.cos.ap-shanghai.myqcloud.com/blog/cli-beautify-01-using-return-asserts/demo.gif)

## 0x04 总结

相比于其他美化控制台程序的方法来说，使用`\r`是最简单直接的一种。
虽然只能实现单行文本的刷新，但已经能够满足大部分轻量程序的要求。
而如果想要进一步美化CLI程序（颜色输出、全屏渲染控制、接收键盘控制等）
则需要使用更高级的一些方法，例如`ANSI`和`JLine`库（Java），之后可能
会写一些教程qw

> 其实当程序使用上述技术后，就从CLI（Command Line Interface）
> 程序进化为TUI（Text User Interface）程序了😋
