---
title: games101_assignment_2[WIP]
date: 2024-02-02 16:17:02
tags:
---

本篇记录games101作业二的各种解法以及常见错误，比如MSAA的黑边。

<!--more-->

比如现在遍历到一个三角形 $t_1$，MSAA_COUNT=4，一个像素点 $p$ 中的四个采样点中有3个命中了这个三角形，那么这种方法会让3/4三角形色与1/4底色混合。但是如果之后又遍历到一个三角形 $t_2$，完全覆盖了该像素点 $p$ ，也就是命中四个采样点，那么无论 $t_1$ 是否在 $t_2$的上方，都会导致之前的1/4底色被混入了最终的颜色里面，这就是黑边的出现。



如何解决，先看看MSAA_OFF的代码。正常流程，当前点在较前就直接设置颜色。

```cpp
if (!insideTriangle(x, y, t.v)) continue;
auto z_interpolated = interp(x, y, t);

if (depth_buf[frame_idx] <= z_interpolated) continue;
depth_buf[frame_idx] = z_interpolated; // Min(near)
set_pixel(p, color);
```

但是当多采样后，每个采样点都有自己的z值，这样直接set_pixel导致frame_buffer的数量没和depth_buf数量对上，。所以需要多一个与depth_buf对应的sample_frame_buf来储存中间值，再把sample_frame_buffer映射到frame_buffer上，aka卷积。



但是想一想，之前
