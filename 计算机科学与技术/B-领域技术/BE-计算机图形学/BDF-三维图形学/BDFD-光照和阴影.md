# 计算机图形学光照与阴影复习笔记

## 一、光照模型基础

### 1.1 光照的物理基础
- **光线传播**：直线传播，遵循反射和折射定律
- **能量守恒**：入射光 = 反射光 + 吸收光 + 透射光
- **双向反射分布函数（BRDF）**：描述表面如何反射光线

### 1.2 光的基本类型
#### 1.2.1 按光源类型
- **方向光（Directional Light）**：平行光，模拟太阳
- **点光源（Point Light）**：从一点向所有方向发射
- **聚光灯（Spot Light）**：圆锥形发射区域
- **环境光（Ambient Light）**：均匀来自所有方向

#### 1.2.2 按光照成分（Phong模型）
- **环境光（Ambient）**：模拟间接光照
- **漫反射（Diffuse）**：Lambertian反射
- **镜面反射（Specular）**：高光反射

## 二、局部光照模型

### 2.1 Lambert漫反射模型
- **公式**：I = k_d × I_l × max(0, n·l)
  - k_d：漫反射系数（表面颜色）
  - I_l：光源强度
  - n：表面法线
  - l：光线方向（指向光源）
- **特点**：与观察方向无关

### 2.2 Phong反射模型（1975）
- **光照方程**：
  ```
  I = I_ambient + I_diffuse + I_specular
      = k_a × I_a + k_d × I_l × (n·l) + k_s × I_l × (r·v)^α
  ```
  - k_a：环境光系数
  - k_s：镜面反射系数
  - r：反射光方向，r = 2(n·l)n - l
  - v：视线方向（指向观察者）
  - α：高光指数（控制高光大小）

### 2.3 Blinn-Phong模型（1977）
- **改进**：使用半角向量代替反射向量
- **公式**：I_specular = k_s × I_l × (n·h)^α
  - h：半角向量，h = (l + v) / ||l + v||
- **优点**：计算效率高，视觉效果相似

### 2.4 实现代码示例（GLSL风格）
```glsl
// Phong光照模型实现
vec3 phongLighting(vec3 position, vec3 normal, vec3 viewDir) {
    vec3 lightDir = normalize(lightPos - position);
    vec3 reflectDir = reflect(-lightDir, normal);
    
    // 漫反射
    float diff = max(dot(normal, lightDir), 0.0);
    vec3 diffuse = lightColor * diff * material.diffuse;
    
    // 镜面反射
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    vec3 specular = lightColor * spec * material.specular;
    
    // 环境光
    vec3 ambient = lightColor * material.ambient;
    
    return ambient + diffuse + specular;
}
```

## 三、高级光照模型

### 3.1 Cook-Torrance模型（物理基础）
- **基于微表面理论**：表面由微小镜面组成
- **公式**：f(l,v) = D × F × G / (4(n·l)(n·v))
  - D：法线分布函数（NDF）
  - F：菲涅尔项（Fresnel）
  - G：几何衰减项（Geometry）
- **适用于**：金属、粗糙表面

### 3.2 Oren-Nayar模型
- **改进Lambert模型**：考虑表面粗糙度
- **适用于**：粗糙漫反射表面（布料、泥土）

### 3.3 Minnaert模型
- **模拟天鹅绒效果**：边缘变暗
- **公式**：I ∝ (n·l)^k × (n·v)^{1-k}

## 四、材质与纹理

### 4.1 材质属性
- **漫反射颜色**：表面基本颜色
- **镜面反射颜色**：高光颜色
- **光泽度**：控制高光大小
- **法线**：表面朝向
- **粗糙度**：表面光滑程度
- **金属度**：金属/非金属

### 4.2 纹理映射
- **漫反射纹理**：表面颜色
- **法线纹理**：存储表面法线扰动
- **高光纹理**：控制镜面反射强度
- **置换纹理**：真正改变几何形状

### 4.3 法线贴图（Normal Mapping）
- **目的**：模拟表面细节而不增加几何复杂度
- **切线空间**：相对于表面的局部坐标系
- **转换**：将法线从切线空间转换到世界空间

## 五、阴影技术

### 5.1 阴影的基本原理
- **阴影体（Shadow Volume）**：几何方法
- **阴影映射（Shadow Mapping）**：图像空间方法（最常用）

### 5.2 阴影映射算法
#### 5.2.1 基本步骤
1. **从光源视角渲染**：生成深度图（Shadow Map）
2. **从相机视角渲染**：将像素转换到光源空间
3. **深度比较**：判断是否在阴影中

#### 5.2.2 实现伪代码
```glsl
// 阴影映射关键步骤
float calculateShadow(vec4 fragPosLightSpace) {
    // 透视除法
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    
    // 变换到[0,1]范围
    projCoords = projCoords * 0.5 + 0.5;
    
    // 获取最近深度（从光源视角）
    float closestDepth = texture(shadowMap, projCoords.xy).r;
    
    // 当前片段深度
    float currentDepth = projCoords.z;
    
    // 阴影测试
    float shadow = currentDepth > closestDepth ? 1.0 : 0.0;
    
    return shadow;
}
```

### 5.3 阴影映射的问题与改进

#### 5.3.1 阴影锯齿（Aliasing）
- **原因**：深度图分辨率不足
- **解决方法**：
  - 提高深度图分辨率
  - 百分比渐进滤波（PCF）
  - 方差阴影映射（VSM）

#### 5.3.2 自遮挡（Self-Shadowing）
- **原因**：深度精度问题
- **解决方法**：添加阴影偏移（Shadow Bias）
  ```glsl
  float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);
  shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
  ```

#### 5.3.3 Peter Panning
- **原因**：阴影偏移过大
- **解决方法**：调整偏移量，使用斜坡偏移（Slope Scale Bias）

### 5.4 高级阴影技术

#### 5.4.1 百分比渐进滤波（PCF）
- **原理**：对周围多个纹素采样并平均
- **效果**：柔化阴影边缘
- **代码**：
  ```glsl
  float pcfShadow(vec4 fragPosLightSpace) {
      vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
      projCoords = projCoords * 0.5 + 0.5;
      
      float shadow = 0.0;
      vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
      
      for(int x = -2; x <= 2; ++x) {
          for(int y = -2; y <= 2; ++y) {
              float depth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r;
              shadow += (projCoords.z - bias) > depth ? 1.0 : 0.0;
          }
      }
      return shadow / 25.0;
  }
  ```

#### 5.4.2 方差阴影映射（VSM）
- **原理**：存储深度均值和方差
- **优点**：软阴影，减少采样次数
- **公式**：使用切比雪夫不等式估计阴影概率

#### 5.4.3 级联阴影映射（CSM）
- **针对**：大场景、方向光
- **原理**：将视锥体分为多个区域，每个区域使用不同分辨率的阴影图
- **优点**：近处高精度，远处低精度

#### 5.4.4 屏幕空间阴影（SSS）
- **原理**：在屏幕空间计算阴影
- **优点**：分辨率与屏幕一致，无透视锯齿
- **缺点**：需要深度缓冲，有屏幕空间限制

## 六、全局光照（Global Illumination）

### 6.1 基本概念
- **直接光照**：直接从光源到达表面的光照
- **间接光照**：经过其他表面反射后到达的光照
- **全局光照**：直接光照 + 间接光照

### 6.2 全局光照算法

#### 6.2.1 光线追踪（Ray Tracing）
- **原理**：反向追踪从相机出发的光线
- **优点**：精确，物理正确
- **缺点**：计算量大
- **伪代码**：
  ```
  function rayTrace(ray, depth):
      if depth > maxDepth: return backgroundColor
      
      hit = findIntersection(ray)
      if not hit: return backgroundColor
      
      color = calculateLocalIllumination(hit)
      
      // 反射
      if material is reflective:
          reflectionRay = computeReflectionRay(ray, hit)
          color += rayTrace(reflectionRay, depth+1) * material.reflectivity
      
      // 折射
      if material is transparent:
          refractionRay = computeRefractionRay(ray, hit)
          color += rayTrace(refractionRay, depth+1) * material.transparency
      
      return color
  ```

#### 6.2.2 辐射度算法（Radiosity）
- **原理**：基于热辐射理论，表面间能量交换
- **优点**：适合漫反射主导场景
- **缺点**：存储量大，不适合动态场景

#### 6.2.3 光子映射（Photon Mapping）
- **两步法**：
  1. **光子追踪**：从光源发射光子，存储在光子图中
  2. **密度估计**：渲染时收集附近光子计算辐射度

### 6.3 实时全局光照技术

#### 6.3.1 环境光遮蔽（Ambient Occlusion）
- **原理**：模拟角落和缝隙处的阴影
- **实现方法**：
  - SSAO（屏幕空间环境光遮蔽）
  - HBAO（水平基准环境光遮蔽）
  - GTAO（基于地面的真实环境光遮蔽）

#### 6.3.2 反射探针（Reflection Probes）
- **原理**：预计算场景反射信息
- **类型**：立方体贴图、球谐函数

#### 6.3.3 光照探针（Light Probes）
- **原理**：在场景中放置采样点，预计算间接光照
- **存储**：球谐系数（SH）

#### 6.3.4 光照贴图（Lightmaps）
- **原理**：预计算静态光照，存储在纹理中
- **优点**：高质量，实时性能好
- **缺点**：内存占用大，不支持动态物体

## 七、基于物理的渲染（PBR）

### 7.1 PBR核心原则
1. **能量守恒**：反射光 + 折射光 ≤ 入射光
2. **菲涅尔效应**：入射角越大反射越强
3. **微表面理论**：表面由微小镜面组成

### 7.2 常见PBR模型
- **Disney BRDF**：艺术友好，参数直观
- **GGX/Trowbridge-Reitz**：流行的法线分布函数
- **Smith几何项**：常用的几何衰减函数

### 7.3 PBR材质参数
- **基础色（Albedo）**：表面颜色
- **金属度（Metallic）**：0.0（非金属）到1.0（金属）
- **粗糙度（Roughness）**：0.0（光滑）到1.0（粗糙）
- **环境光遮蔽（AO）**：预计算的环境遮蔽

### 7.4 PBR渲染方程
```
L_o(p, ω_o) = ∫_Ω f_r(p, ω_i, ω_o) L_i(p, ω_i) (n·ω_i) dω_i
```
其中：
- L_o：出射辐射度
- f_r：BRDF
- L_i：入射辐射度
- n·ω_i：余弦项

## 八、实现注意事项

### 8.1 性能优化
- **光照裁剪**：只计算影响当前像素的光源
- **延迟渲染**：将光照计算延迟到G-Buffer之后
- **光照烘焙**：预计算静态光照
- **重要性采样**：蒙特卡洛积分加速

### 8.2 视觉质量
- **色调映射**：将HDR转换为显示范围
- **伽马校正**：纠正显示器的非线性响应
- **抗锯齿**：MSAA、FXAA、TAA

### 8.3 常见问题
1. **Mach带效应**：在明暗边界出现虚假条纹
   - 解决方法：增加采样，使用抖动
   
2. **光渗（Light Bleeding）**：光线穿过遮挡物
   - 解决方法：调整阴影偏移，使用接触硬化阴影
   
3. **闪烁（Flickering）**：阴影随时间变化
   - 解决方法：使用稳定的阴影映射技术

## 九、现代趋势

### 9.1 实时光线追踪（RTX）
- **混合渲染**：光栅化 + 光线追踪
- **应用**：精确反射、软阴影、全局光照

### 9.2 神经网络渲染
- **神经辐射场（NeRF）**：使用神经网络表示3D场景
- **深度学习超级采样（DLSS）**：AI增强图像质量

### 9.3 基于距离场的全局光照（DFGI）
- **原理**：使用距离场快速计算可见性
- **优点**：动态场景，实时性能