---
layout: post
title: Understand Distributed and Whitted Ray tracing.
description: Understanding Monte Carlo Methods(3)
date: 2025-11-24
tags:
  - Rendering
  - Monte-Carlo
  - Whitten
comments: true
use_math: true
---
2025/11/24 written.
2025/11/30 updated.
# Reference

- pbrt v3, https://www.pbrt.org/
- Nori Renderer.

# Whitted Ray tracing
This method is designed to simulate refraction and reflection. Suppose some object has perfectly reflection, such as mirror. So, We can know reflection vector $r$ is computed by

![[Pasted image 20251128151232.png]]

$$\begin{align}
\vec{r} = 2\vec{n}(\vec{n} \cdot \vec{l}) - \vec{l}.
\end{align}$$

In world coordinates, we just insert value into this formula as it is. However, let us consider local coordinates. We know normal vector is $(0, 0, 1)$ in local coordinates. Therefore, we can compute

$$\begin{align}
\textbf{n} \cdot \textbf{l} = (0, 0, 1) \cdot (x, y, z) = z \\
\textbf{r}=2(0,0,z)-(x,y,z) = (-x,-y,z)
\end{align}$$

Not only dielectric materials(like diamonds, glass, water, ...) have perfectly reflection but also has refraction. How we compute dielectric materials that have transparent. The refraction is based on `snell's Law` and more different than reflection. 

![[Pasted image 20251128200915.png]]

Snell's law means $n_i sin\theta_i=n_o sin\theta_o$. $n_i$ and $n_o$ are constant ratio of refraction about object(air, glass). That is, $\frac{n_i}{n_o} = \mathfrak{n}(\mathsf{refraction\ ratio})$. 

$$\begin{align}
n_i sin\theta_i=n_o sin\theta_o \rightarrow n_i^2 sin^2\theta_i=n_o^2 sin^2\theta_o \\
n_i^2 (1-cos^2\theta_i)=n_o^2 (1-cos^2\theta_o) \\
cos^2\theta_o = 1 - \frac{n^2_i(1-cos^2\theta_i)}{n_o^2} = 1 - \frac{n^2_i(1-(\textbf{d} \cdot \textbf{n})^2)}{n_o^2}
\end{align}$$

![[Pasted image 20251128204829.png]]

We may define $\textbf{t} = \sin\theta_o \textbf{b}+cos\theta_o(\textbf{-n})$. We wonder that how we define $\textbf{t}$? You can think `vector sum`.

$$\mathbf{t} = (\text{수평 길이}) \times \mathbf{b} + (\text{수직 길이}) \times (-\mathbf{n})$$

But, they($\textbf{t},\ \textbf{b}$) are unknown. Hence, We can define $\textbf{b}$ using $\textbf{d} = sin\theta_i \textbf{b} - cos\theta_i (\textbf{n})$. Thus, $\textbf{b} = \frac{\textbf{d} + \textbf{n} cos\theta_i}{sin\theta_i}$ and $\textbf{t} = sin\theta_o (\frac{\textbf{d} + \textbf{n} cos\theta_i}{sin\theta_i}) - cos\theta_o \textbf{n}$. In the above figure, we can found $cos^2 \theta_o$ using $cos^2 \theta_i$. Finally, we know that how we define $\textbf{t}$..!

You may take a question, saying that there's still a problem $sim\theta_o$. However, if you look at the formula that I described at the Snell's Law, we can know $sin\theta_o = \frac{n_i}{n_o}sin\theta_i$.

$$\begin{align}
\textbf{t} =  \frac{n_i}{n_o}(\textbf{d} + \textbf{n} cos\theta_i) - \textbf{n} \sqrt{1 - \frac{n^2_i(1-(\textbf{d} \cdot \textbf{n})^2)}{n_o^2}}
\end{align}$$

The ray is defined by $R = o + \textbf{t}d$. Thus, we can compute which direction the rays will move in. On the other hand, if you're going from the inside to the outside, we just change $i$ and $o$. In the code, 

```cpp
if (cosThetaI < 0.0f) { 
	// 1. change n_i and n_o (1.0/1.5 -> 1.5/1.0) 
	std::swap(etaI, etaT); 
	// 2. -wi.dot(n) = cosThetaI -> -wi.dot(-n) = -cosThetaI
	cosThetaI = -cosThetaI; 
}
```

So far, it's been an explanation in the world coordinates. Now let us consider the situation in the local coordinates. It's so simple. Because normal vector is `(0, 0, 1)`. We insert it into $\textbf{t}$.

$$\begin{align}
\textbf{t} =  \frac{n_i}{n_o}(\textbf{d} + \textbf{n} cos\theta_i) - \textbf{n} \sqrt{1 - \frac{n^2_i(1-(\textbf{d} \cdot \textbf{n})^2)}{n_o^2}} \\
=  \frac{n_i}{n_o}((x, y,z) + \textbf{n}cos\theta_i) - \textbf{n} \sqrt{1 - \frac{n^2_i(1-(\textbf{d} \cdot \textbf{n})^2)}{n_o^2}} \\
= \frac{n_i}{n_o}(x, y, z)- \frac{n_i}{n_o}\textbf{n}cos\theta_i- \textbf{n}cos\theta_o 
\end{align}$$

We can compute $x'$, $y'$, $z'$ in local coordinate.

$$\begin{align}
x' = \frac{n_i}{n_o}x \\
y' = \frac{n_i}{n_o}y \\
z' = \frac{n_i}{n_o}z - \frac{n_i}{n_o}cos\theta_i - cos\theta_o = \frac{n_i}{n_o}z - \frac{n_i}{n_o}((x,y,z) \cdot (0,0,1)) - cos\theta_o = -cos\theta_o
\end{align}$$


![[cbox-whitted.png]]