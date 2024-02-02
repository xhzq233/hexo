---
title: games101_assignment_2[WIP]
date: 2024-02-02 16:17:02
tags:
---

本篇记录games101作业二的各种解法以及常见错误，比如MSAA的黑边。

![image-20240202193421384](../../../Library/Application%20Support/typora-user-images/image-20240202193421384.png) to ![image-20240202193444350](../../../Library/Application%20Support/typora-user-images/image-20240202193444350.png)

<!--more-->



先看看MSAA_OFF的代码。正常流程，当前点在较前就直接设置颜色。

```cpp
if (!insideTriangle(x, y, t.v)) continue;
auto z_interpolated = interp(x, y, t);

if (depth_buf[frame_idx] <= z_interpolated) continue;
depth_buf[frame_idx] = z_interpolated; // Min(near)
set_pixel(p, color);
```

但是当多采样后，每个采样点都有自己的z值，如果不考虑增加frame_buf，这样set_pixel就导致frame_buf的数量没和depth_buf数量对上，就会混入不期待的颜色。比如下面的代码

```cpp
constexpr const float sample_points_step = 1.0f / (MSAA_COUNT * 2);
int cnt = 0;
float x_sample, y_sample;
for (int i = 0; i < MSAA_X; ++i) {
    x_sample = x + sample_points_step * (float) (i % MSAA_COUNT * 2 + 1);
    y_sample = y + sample_points_step * (float) ((int) (i / MSAA_COUNT) * 2 + 1);

    if (!insideTriangle(x_sample, y_sample, t.v)) continue;

    auto z_interpolated = interp(x_sample, y_sample, t);

    int interp_idx = frame_idx * MSAA_X + i;
    if (depth_buf[interp_idx] <= z_interpolated) continue;
    depth_buf[interp_idx] = z_interpolated; // Min(near)
    cnt++;
}

if (!cnt) continue;

color = color * (float(cnt) / 4.0) + frame_buf[frame_idx] * (float(4 - cnt) / 4.0);
```


比如现在遍历到一个三角形 $t_1$，MSAA 2x，一个像素点 $p$ 中的四个采样点中有3个命中了这个三角形，那么这种方法会让3/4三角形色与1/4底色混合。但是如果之后又遍历到一个三角形 $t_2$，完全覆盖了该像素点 $p$ ，也就是命中四个采样点，那么无论 $t_1$ 是否在 $t_2$的上方，都会导致之前的1/4底色被混入了最终的颜色里面，这就是黑边的出现。


所以需要多一个与depth_buf对应的sample_frame_buf来储存中间值，再把sample_frame_buf映射到frame_buf上，aka卷积。

```cpp
// ... 上同
    if (depth_buf[interp_idx] <= z_interpolated) continue;
    depth_buf[interp_idx] = z_interpolated; // Min(near)
    frame_buf_sample[interp_idx] = color;
    cnt++;
}

if (!cnt) continue;
auto start = frame_buf_sample.begin() + frame_idx * MSAA_X;
set_pixel(p, std::reduce(start, start + MSAA_X) / MSAA_X);
```







但是想一想，之前
