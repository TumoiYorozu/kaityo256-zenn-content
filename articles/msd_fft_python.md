---
title: "平均二乗変位をフーリエ変換で求める"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python"]
published: false
---

## 概要

ランダムな力を受けて動く粒子がいるとします。その運動方程式は以下のようなランジュバン方程式に従うでしょう。

$$
\frac{dx}{dt} = \hat{R}
$$

ただし、$\hat{R}$は白色雑音です。この時、時刻$t$と時刻$t+\tau$の座標の差の二乗の平均は、その時間$\tau$に比例します。

$$
\left< \left(x(t) - x(t+\tau) \right)^2 \right> \propto Dt
$$

この$\left< \left(x(t) - x(t+\tau) \right)^2 \right>$を、平均二乗変位(Mean Square Displacement, MSD)と呼びます。いま、時間間隔$h$で座標を観測し、$x_k = x(hk)$と書きましょう。この離散的な時系列$\{x_k\}$が$N$点得られた時(つまり$k=0,1,\cdots,N-1$)、平均二乗変位$D(m)$は

$$
D(m) = \frac{1}{N-m} \sum_{k=0}^{N-m-1}
\left(x_{k+m} - x_k\right)^2
$$

で定義されます。これはナイーブに計算すると$O(N^2)$となりますが、フーリエ変換を使うと$O(N\ln N)$になって、特に$N$が大きい時に高速になります。それを見てましょう、というのが本稿の趣旨です。

## 周期境界補正

平均二乗変位を求める前に、周期境界補正について考えます。多くの場合、シミュレーションでは周期境界を用いて計算を行います。すると、注目する粒子が境界を跨ぐ時に、それをちゃんと考慮してやらないと変位がおかしなことになります。まずは周期境界下で座標の時系列`x`が与えられた時に、周期境界条件の補正をしてやることを考えましょう。

システムサイズを$L=10$とします。後で必要となるライブラリもインポートしておきます。

```py
import numpy as np
from matplotlib import pyplot as plt
import random
L = 10
```

粒子が$x$の正方向に等速直線運動をしており、境界を2回またいだ状態を考えます。こんな感じです。

```py
L = 10
x = [i - (i // L)*L  for i in range(L*3)]
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/x1.png)

これが、あたかも周期境界がなく、無限の領域を運動していたらどんな軌跡だったかを考えるのが周期境界条件補正です。単純にはこんなことをしてやりたくなります。

```py
def adjust_periodic_wrong(x):
    for i in range(len(x)-1):
        if x[i+1] - x[i] > L/2:
            x[i+1] -= L
        if x[i+1] - x[i] < -L/2:
            x[i+1] += L
```

要するに、一つ前の座標と$L/2$以上離れていたら$L$だけ補正するものです。しかし、これはうまくいきません。

```py
x = [i - (i // L)*L  for i in range(L*3)]
adjust_periodic_wrong(x)
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/x1_wrong.png)

このように、二回境界をまたぐケースでおかしくなります。

正しくはこうしてやる必要があります。

```py
def adjust_periodic(x):
    for i in range(len(x)-1):
        if x[i+1] - x[i] > L/2:
            x[i+1] -= (x[i+1] - x[i]+L/2)//L*L
        if x[i+1] - x[i] < -L/2:
            x[i+1] += (x[i] - x[i+1]+L/2)//L*L
```

何回境界をまたいだかを計算し、その分だけ補正してやります。結果はこうなります。

```py
x = [i - (i // L)*L  for i in range(L*3)]
adjust_periodic(x)
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/x1_correct.png)

ちゃんとまっすぐになりましたね。

念のため逆方向も確認しておきましょう(基本)。

```py
x = [-i - (-i // L)*L  for i in range(L*3)]
plt.plot(x)
plt.show()
adjust_periodic(x)
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/x2.png)
![fig](msd_fft_python/x2_correct.png)

大丈夫そうです。

## ランダムウォーク

### 時系列の作成

それでは、ランダムウォークのデータを作りましょう。単に毎ステップ、-1か+1にランダムで進むだけのコードです。ただし、周期境界条件により$0<x<L$となるようにしています。

```py
def diffusion(step):
    x = np.zeros(step)
    pos = 0.0
    for i in range(step):
        pos += random.choice([-1,1])
        if pos < 0:
            pos += L
        if pos > L:
            pos -= L
        x[i] = pos
    return x
```

実行結果はこんな感じです。

```py
random.seed(12345)
x = diffusion(2**12)
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/random_walk.png)

周期境界によりよくわからなくなっています。補正してやりましょう。

```py
adjust_periodic(x)
plt.plot(x)
plt.show()
```

![fig](msd_fft_python/random_walk_correct.png)

それっぽくなりました。以後、この軌跡について平均二乗変位を計算します。

### シンプルな計算

まずは、定義式

$$
D(m) = \frac{1}{N-m} \sum_{k=0}^{N-m-1}
\left(x_{k+m} - x_k\right)^2
$$

をそのまま計算してやりましょう。こうなるでしょうか。

```py
def calc_msd_simple(x):
    n = len(x)
    msd = []
    for s in range(1,n//4):
        x2 = 0.0
        for i in range(n-s):
            x2 += (x[i+s]-x[i])**2
        x2 /= (n-s)
        msd.append(x2)
    return msd
```

そのままなので、特に難しくはないと思います。ただし、$m$が$N$に近いとサンプル数が少なくなってしまうため、$0<m<N/4$としました。

平均二乗変位をプロットしてやりましょう。ついでに時間を測ってみます。

```py
%%time
msd_simple = calc_msd_simple(x)
plt.plot(msd_simple)
plt.show()
```

![fig](msd_fft_python/msd_simple.png)

```txt
CPU times: user 3.32 s, sys: 16.7 ms, total: 3.34 s
Wall time: 3.34 s
```

3.34秒かかりました。

### NumPyっぽく計算

シンプルな計算では遅すぎるので、ちょっとPythonっぽく計算してみましょう。欲しいのは、インデックスが`s`だけずれた座標の差です。PythonはN個の要素を持つ配列`a`にたいして`a[s:]`とすると`a[s]`.`a[s+1]`,...`a[N-1]`の部分配列を、$a[:-s]$とすると、`a[0]`,`a[1]`,...,`a[N-s-1]`の部分配列を返します。

```py
a = np.arange(10)
a[2:]  # => array([2, 3, 4, 5, 6, 7, 8, 9])
a[:-2] # => array([0, 1, 2, 3, 4, 5, 6, 7])
```

なので、`a[s:] - a[:-s]`により、`s`だけ離れたインデックスの差の配列を得ることができます。差を計算するのも二乗を計算するのもNumPy配列の演算になるので、高速化が期待できます。コードに落とすとこんな感じになるでしょう。

```py
def calc_msd_np(x):
    n = len(x)
    msd = []
    for s in range(1,n//4):
        dx = x[s:] - x[:-s]
        msd.append(np.average(dx**2))
    return msd
```

実行して時間を測ってみます。

```py
%%time
msd_np = calc_msd_np(x)
plt.plot(msd_np)
plt.show()
```

![fig](msd_fft_python/msd_np.png)

```txt
CPU times: user 151 ms, sys: 0 ns, total: 151 ms
Wall time: 156 ms
```

実行時間は156 msと、かなり高速化されました。

念のために二つを重ねてみましょう。

```py
fig, ax = plt.subplots()
ax.plot(msd_simple,label="Simple")
ax.plot(msd_np,label="NumPy")
ax.legend()
```
![fig](msd_fft_python/msd_simple_np.png)

完全に一致していますね。