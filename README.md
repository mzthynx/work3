# <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">基于 Taichi 的 Phong 光照模型交互式渲染实验报告</font>
## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">一、实验目标</font>
1. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">理论理解：深入掌握局部光照的核心原理，精准区分环境光（Ambient）、漫反射（Diffuse）与镜面高光（Specular）三个光照分量的物理意义与计算逻辑。</font>
2. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">数学基础</font><font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：熟练运用三维空间向量运算，包括法向量求解、光线方向 / 视线方向 / 反射方向的计算，为光照模型实现提供数学支撑。</font>
3. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">工程实践：掌握利用 Taichi 编程语言实现交互式三维渲染的方法，通过 UI 控件实时调节材质参数，直观观察参数变化对渲染结果的影响，完成光线求交、深度测试与 Phong 着色器的全流程开发。</font>

## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">二、实验原理</font>
<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">Phong 光照模型是计算机图形学中经典的局部光照经验模型，其核心思想是将物体表面反射的光分解为环境光、漫反射、镜面高光三个独立分量，最终通过分量叠加得到像素的最终 RGB 颜色值，数学表达式为：</font>

$I=I_{ambient}+I_{diffuse}+I_{specular}$

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（一）各分量计算原理</font>
<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">环境光（</font>$I_{ambient}$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">）：模拟场景中经多次反射后均匀分布的背景光，消除物体表面因未直接接收光源而呈现全黑的现象，计算公式为：</font>

$I_{ambient}=K_a\times C_{light}\times C_{object}$

其中，$K_a$为环境光系数，$C_{light}$为光源颜色，$C_{object}$为物体表面基础颜色。

<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">漫反射（</font>$I_{diffuse}$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">）</font>：模拟粗糙表面向各个方向均匀散射的光，其强度与光线入射角的余弦值成正比（遵循 Lambert 定律），计算公式为：

$I_{diffuse}=K_d\times max(0,N·L)\times C_{light}\times C_{object}$

其中，$K_d$为漫反射系数,$N$为物体表面法向量，$L$为指向光源的方向向量，$max(0,N·L)$保证光线从背面照射时漫反射强度为 0。

<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">镜面高光（</font>$I_{specular}$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">）：模拟光滑表面反射的强光，强度与观察方向和理想反射方向的夹角相关，夹角越小高光越强，计算公式为：</font>

$I_{specular}=K_s\times max(0,R·V)^n\times C_{light}$

<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">其中，</font>$K_s$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为镜面高光系数，</font>$R$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为光线的理想反射向量，</font>$V$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为指向摄像机的方向向量，</font>$n$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为高光指数（Shininess），控制高光区域的大小。</font>

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（二）核心技术支撑</font>
1. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">光线投射（Ray Casting）：为屏幕每个像素发射一条射线，通过计算射线与三维几何体的交点，确定像素对应的三维空间位置。</font>
2. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">深度测试</font><font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：当射线同时击中多个物体时，选择距离摄像机最近的交点（最小的正数</font><font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">t</font><font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">）进行着色，保证正确的遮挡关系。</font>
3. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">Taichi 并行计算：利用 Taichi 的 GPU 并行加速能力，实现高效的光线追踪与渲染，提升交互帧率。</font>

## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">三、实验环境与工具</font>
1. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">开发环境：Python 3.12.12，Taichi 1.6+（基于 GPU 架构加速）</font>
2. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">核心工具：Taichi 内置的ti.ui.Window实现 UI 交互，ti.Vector.field存储像素数据，ti.func定义内核函数实现并行计算</font>
3. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">三维场景：通过数学公式隐式定义红色球体与紫色圆锥，无需外部模型文件。</font>

## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">四、实验步骤与代码实现</font>
### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（一）环境初始化与参数定义</font>
1. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">初始化 Taichi 并指定 GPU 架构，定义窗口分辨率，创建用于存储像素颜色的向量场。</font>
2. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">定义全局交互参数（环境光系数$K_a$、漫反射系数$K_d$、高光系数$K_s$、高光指数$n$），通过 Taichi 字段实现跨内核参数共享。</font>

```plain
import taichi as ti

# 初始化Taichi
ti.init(arch=ti.gpu)

# 窗口分辨率
res_x, res_y = 800, 600
pixels = ti.Vector.field(3, dtype=ti.f32, shape=(res_x, res_y))

# 定义全局交互参数
Ka = ti.field(ti.f32, shape=())
Kd = ti.field(ti.f32, shape=())
Ks = ti.field(ti.f32, shape=())
shininess = ti.field(ti.f32, shape=())
```

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（二）工具函数实现</font>
<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">实现向量归一化、向量反射等基础工具函数，为几何相交测试与光照计算提供支撑。</font>

```plain
@ti.func
def intersect_sphere(ro, rd, center, radius):
    t = -1.0
    normal = ti.Vector([0.0, 0.0, 0.0])
    oc = ro - center
    b = 2.0 * oc.dot(rd)
    c = oc.dot(oc) - radius * radius
    delta = b * b - 4.0 * c
    if delta > 0:
        t1 = (-b - ti.sqrt(delta)) / 2.0
        if t1 > 0:
            t = t1
            p = ro + rd * t
            normal = normalize(p - center)
    return t, normal
```

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（三）几何体相交测试</font>
1. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">球体相交测试</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：通过一元二次方程求解光线与球体的交点，计算交点处的法向量。</font>

```plain
@ti.func
def intersect_sphere(ro, rd, center, radius):
    t = -1.0
    normal = ti.Vector([0.0, 0.0, 0.0])
    oc = ro - center
    b = 2.0 * oc.dot(rd)
    c = oc.dot(oc) - radius * radius
    delta = b * b - 4.0 * c
    if delta > 0:
        t1 = (-b - ti.sqrt(delta)) / 2.0
        if t1 > 0:
            t = t1
            p = ro + rd * t
            normal = normalize(p - center)
    return t, normal
```

2. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">圆锥相交测试</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：转换到局部坐标系构建一元二次方程，求解光线与圆锥的交点，验证交点在圆锥高度范围内并计算法向量。  </font>

```plain
@ti.func
def intersect_cone(ro, rd, apex, base_y, radius):
    t = -1.0
    normal = ti.Vector([0.0, 0.0, 0.0])
    H = apex.y - base_y
    k = (radius / H) ** 2
    
    ro_local = ro - apex  # 转换到以顶点为原点的局部坐标系
    A = rd.x**2 + rd.z**2 - k * rd.y**2
    B = 2.0 * (ro_local.x * rd.x + ro_local.z * rd.z - k * ro_local.y * rd.y)
    C = ro_local.x**2 + ro_local.z**2 - k * ro_local.y**2
    
    if ti.abs(A) > 1e-5:  # 避免除零
        delta = B**2 - 4.0 * A * C
        if delta > 0:
            t1, t2 = (-B - ti.sqrt(delta)) / (2.0 * A), (-B + ti.sqrt(delta)) / (2.0 * A)
            if t1 > t2:
                t1, t2 = t2, t1  # 保证t1为更近交点
            
            # 验证交点在圆锥高度范围[-H, 0]内
            y1 = ro_local.y + t1 * rd.y
            if t1 > 0 and -H <= y1 <= 0:
                t = t1
            else:
                y2 = ro_local.y + t2 * rd.y
                if t2 > 0 and -H <= y2 <= 0:
                    t = t2
                    
            if t > 0:
                p_local = ro_local + rd * t
                normal = normalize(ti.Vector([p_local.x, -k * p_local.y, p_local.z]))  # 圆锥法向量计算
    return t, normal
```

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（四）渲染内核实现</font>
1. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为每个像素生成光线，计算光线起点（摄像机位置）与方向。</font>
2. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">依次测试光线与球体、圆锥的相交，通过深度筛选最近交点。</font>
3. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">基于 Phong 光照模型计算像素最终颜色，结合环境光、漫反射、镜面高光分量叠加。</font>

```plain
@ti.kernel
def render():
    for i, j in pixels:
        u = (i - res_x / 2.0) / res_y * 2.0
        v = (j - res_y / 2.0) / res_y * 2.0
        
        ro = ti.Vector([0.0, 0.0, 5.0])  # 摄像机位置
        rd = normalize(ti.Vector([u, v, -1.0]))  # 光线方向

        min_t = 1e10  # 最近交点距离
        hit_normal = ti.Vector([0.0, 0.0, 0.0])  # 交点法向量
        hit_color = ti.Vector([0.0, 0.0, 0.0])  # 物体基础颜色
        
        # 测试球体相交
        t_sph, n_sph = intersect_sphere(ro, rd, ti.Vector([-1.2, -0.2, 0.0]), 1.2)
        if 0 < t_sph < min_t:
            min_t = t_sph
            hit_normal = n_sph
            hit_color = ti.Vector([0.8, 0.1, 0.1])  # 红色球体
            
        # 测试圆锥相交
        t_cone, n_cone = intersect_cone(ro, rd, ti.Vector([1.2, 1.2, 0.0]), -1.4, 1.2)
        if 0 < t_cone < min_t:
            min_t = t_cone
            hit_normal = n_cone
            hit_color = ti.Vector([0.6, 0.2, 0.8])  # 紫色圆锥

        color = ti.Vector([0.05, 0.15, 0.15])  # 背景色

        # 若击中物体，计算Phong光照
        if min_t < 1e9:
            p = ro + rd * min_t
            N = hit_normal
            
            light_pos = ti.Vector([2.0, 3.0, 4.0])  # 点光源位置
            light_color = ti.Vector([1.0, 1.0, 1.0])  # 白光
            
            L = normalize(light_pos - p)  # 光源方向向量
            V = normalize(ro - p)  # 视线方向向量

            # Phong光照分量计算
            ambient = Ka[None] * light_color * hit_color
            diff = ti.max(0.0, N.dot(L))
            diffuse = Kd[None] * diff * light_color * hit_color
            
            R = normalize(reflect(-L, N))  # 反射向量
            spec = ti.max(0.0, R.dot(V)) ** shininess[None]
            specular = Ks[None] * spec * light_color 
            
            color = ambient + diffuse + specular  # 分量叠加
                
        pixels[i, j] = ti.math.clamp(color, 0.0, 1.0)  # 限制颜色在[0,1]范围
```

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（五）交互式 UI 与主函数</font>
<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">创建ti.ui.Window实现交互窗口，通过滑块控件实时绑定材质参数，循环执行渲染与窗口显示。</font>

```plain
def main():
    window = ti.ui.Window("Phong Shading Demo", (res_x, res_y))
    canvas = window.get_canvas()
    gui = window.get_gui()
    
    # 初始化材质参数
    Ka[None] = 0.2
    Kd[None] = 0.7
    Ks[None] = 0.5
    shininess[None] = 32.0

    while window.running:
        render()
        canvas.set_image(pixels)  # 绘制渲染结果
        
        # 交互式UI面板
        with gui.sub_window("Material Parameters", 0.7, 0.05, 0.28, 0.22):
            Ka[None] = gui.slider_float('Ka (Ambient)', Ka[None], 0.0, 1.0)
            Kd[None] = gui.slider_float('Kd (Diffuse)', Kd[None], 0.0, 1.0)
            Ks[None] = gui.slider_float('Ks (Specular)', Ks[None], 0.0, 1.0)
            shininess[None] = gui.slider_float('N (Shininess)', shininess[None], 1.0, 128.0)

        window.show()  # 刷新窗口

if __name__ == '__main__':
    main()
```

## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">五、实验结果与分析</font>
### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（一）渲染结果展示</font>
<img src="https://cdn.nlark.com/yuque/0/2026/gif/62188406/1776339257645-e3c9083a-9544-4d98-b959-da311d80eee9.gif" width="804" title="" crop="0,0,1,1" id="ue9e56e2a" class="ne-image">

<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">运行程序后，窗口左侧显示深红色球体，右侧显示紫色圆锥，背景为深青色。通过右侧 UI 面板的滑块控件，可实时调节</font>$K_a,K_d,K_s$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">高光指数</font>$n$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">，渲染结果随参数变化呈现不同视觉效果：</font>

1. $K_a$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（环境光系数）：增大</font>$K_a$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">时，物体整体亮度提升，暗部细节更清晰；减小</font>$K_a$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">时，物体暗部逐渐变暗，甚至失去层次感。</font>
2. $K_d$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（漫反射系数）：</font>$K_d$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">越大，物体漫反射效果越显著，表面粗糙感越强；</font>$K_d$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">趋近于 0 时，漫反射几乎消失，物体仅保留环境光与高光效果。</font>
3. $K_s$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（高光系数）：</font>$K_s$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">增大时，镜面高光区域更亮、更明显；</font>$K_s$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">为 0 时，高光效果完全消失，物体呈现哑光质感。</font>
4. <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">高光指数</font>$n$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：</font>$n$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">越大，高光区域越集中、越锐利，物体表面光滑感越强；</font>$n$<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">越小时，高光区域越弥散，表面粗糙感越明显。</font>

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（二）结果分析</font>
1. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">光照模型验证</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：Phong 模型的三个分量叠加效果符合理论预期，环境光保证了物体整体亮度，漫反射模拟了粗糙表面的散射特性，镜面高光还原了光滑表面的反光效果。</font>
2. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">交互性与实用性</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：通过 UI 滑块实时调节参数，可直观理解各材质参数对视觉效果的影响，验证了参数设计与绑定逻辑的正确性。</font>
3. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">性能表现</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：基于 Taichi 的 GPU 并行加速，800×600 分辨率下渲染帧率稳定，交互操作无明显卡顿，体现了 Taichi 在实时渲染中的高效性。</font>

## <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">六、实验总结与思考</font>
### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（一）实验总结</font>
<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">本次实验成功实现了基于 Taichi 的 Phong 光照模型交互式渲染，完成了三维场景构建、光线求交、深度测试、Phong 着色器编写与 UI 交互五大核心任务。通过实验，深入理解了局部光照的物理机制，掌握了 Taichi 并行计算与 UI 交互的开发方法，实现了材质参数的实时调节与渲染效果的动态反馈。</font>

### <font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">（二）实验思考</font>
1. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">模型优化方向</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：Phong 模型存在高光 “过亮” 的视觉缺陷，可尝试改进为 Blinn-Phong 模型（通过半程向量H计算高光），提升高光效果的真实感。</font>
2. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">扩展功能</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：可增加更多几何体（如立方体、圆柱）的相交测试，或添加多光源、纹理映射功能，丰富渲染场景。</font>
3. **<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">性能优化</font>**<font style="color:rgb(0, 0, 0);background-color:rgba(0, 0, 0, 0);">：针对复杂场景，可引入空间划分（如八叉树、BVH）减少光线求交的计算量，进一步提升渲染效率。</font>
