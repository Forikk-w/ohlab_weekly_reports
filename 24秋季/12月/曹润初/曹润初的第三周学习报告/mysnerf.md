#### 这段代码是针对论文写成的，但由于我能力有限，简化了模型，比如说将三通道的RGB图像改为了单通道，同时还直接忽略了深度图depths。饶是如此，这仍然耗费了我四个小时的时间调试这段代码，因为总是出现张量维度不匹配和损失函数计算错误的情况。
代码逻辑如下：
1.随机生成卫星图像，并用噪声进行模糊处理。
2.定义snerf神经网络，输入为一张图的像素点的坐标和阳光方向，视角默认为从上向下；输出为图像的颜色（单通道），天空颜色，密度和天空颜色占比
3.计算损失函数，具体公式见论文。（代码可能有错）
4.梯度下降，对模型进行优化。


```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim

# 生成合成的卫星图像数据（模拟）
def generate_synthetic_data(num_images, image_size, true_shape):
    images = []
    depths = []
    for _ in range(num_images):
        # 模拟不同的光照条件（这里简化为随机强度）
        light_intensity = np.random.uniform(0.5, 1.5)
        # 模拟图像（这里简化为与真实形状和光照相关的随机噪声）
        image = true_shape + np.random.normal(0, 0.1, image_size) * light_intensity
        images.append(image)
        # 模拟深度图（这里简化为与真实形状相关的随机噪声）
        depth = true_shape + np.random.normal(0, 0.5, image_size)
        depths.append(depth)
    return np.array(images), np.array(depths)

# 定义真实形状（模拟地球表面的简单地形）
true_shape = np.zeros((100, 100))
true_shape[30:70, 30:70] = 1.0

# 生成合成数据
num_images = 20
image_size = (100, 100)
images, depths = generate_synthetic_data(num_images, image_size, true_shape)
```


```python
images = torch.from_numpy(images).float().unsqueeze(1)
depths = torch.from_numpy(depths).float().unsqueeze(1)
```


```python
images.shape
```




    torch.Size([20, 1, 100, 100])




```python
depths.shape
```




    torch.Size([20, 1, 100, 100])




```python
true_shape.shape
```




    (100, 100)




```python
def get_snerf_input(image):
    h, w = image.shape[-2:]
    x_coords, y_coords = torch.meshgrid(torch.arange(w), torch.arange(h), indexing='ij')
    coords = torch.stack([x_coords, y_coords, torch.zeros_like(x_coords)], dim=-1).float()
    return coords.view(-1, 3)
```


```python
x=get_snerf_input(images[0])
print(x)
```

    tensor([[ 0.,  0.,  0.],
            [ 0.,  1.,  0.],
            [ 0.,  2.,  0.],
            ...,
            [99., 97.,  0.],
            [99., 98.,  0.],
            [99., 99.,  0.]])
    


```python
x.shape
```




    torch.Size([10000, 3])




```python


# 定义S-NeRF模型
class SNeRF(nn.Module):
    def __init__(self):
        super(SNeRF, self).__init__()
        # 定义密度网络
        self.density_layers = nn.Sequential(
            nn.Linear(3, 64),  # 输入 3D 坐标
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 1)
        )
        # 定义反照率网络
        self.albedo_layers = nn.Sequential(
            nn.Linear(3, 64),  # 输入 3D 坐标
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 1),
            nn.Sigmoid()
        )
        # 定义太阳可见性网络
        self.sun_visibility_layers = nn.Sequential(
            nn.Linear(3 + 2, 64),  # 输入 3D 坐标 + 太阳方向（合成后的5维输入）
            nn.ReLU(),
            nn.Linear(64, 64),
            nn.ReLU(),
            nn.Linear(64, 1),
            nn.Sigmoid()
        )
        # 定义天空颜色估计层
        self.sky_color_layers = nn.Sequential(
            nn.Linear(2, 64),  # 输入太阳方向（2维）
            nn.ReLU(),
            nn.Linear(64, 1)
        )

    def forward(self, x, sun_direction):
        # 修正：拼接 x 和 sun_direction 时，确保维度正确
        density_input = x  # 只使用 3D 坐标输入到密度层
        density = self.density_layers(density_input)
        
        albedo_input = x  # 只使用 3D 坐标输入到反照率层
        albedo = self.albedo_layers(albedo_input)
        
        # 太阳可见性和天空颜色的输入：拼接 3D 坐标和 太阳方向
        # 计算天空颜色：仅使用太阳方向
        
        sky_color = self.sky_color_layers(sun_direction)  
        sun_direction = sun_direction.unsqueeze(0).expand(x.shape[0], -1)   # 扩展为与 x 的第0维相同
        
        visibility_input = torch.cat([x, sun_direction],dim=-1)  # 拼接
        
        sun_visibility = self.sun_visibility_layers(visibility_input)
        
          
        
        return density, albedo, sun_visibility, sky_color
```


```python
model=snerf_model = SNeRF()
density, albedo, sun_visibility, sky_color = model(x, torch.tensor([np.pi/4,np.pi/4]))
print(density)
print(albedo)
print(sun_visibility)
print(sky_color)
```

    tensor([[ 0.0990],
            [ 0.0976],
            [ 0.0687],
            ...,
            [-9.5540],
            [-9.5782],
            [-9.6023]], grad_fn=<AddmmBackward0>)
    tensor([[0.4897],
            [0.4790],
            [0.4663],
            ...,
            [0.0151],
            [0.0147],
            [0.0141]], grad_fn=<SigmoidBackward0>)
    tensor([[0.4600],
            [0.4751],
            [0.4879],
            ...,
            [0.9611],
            [0.9616],
            [0.9620]], grad_fn=<SigmoidBackward0>)
    tensor([0.0389], grad_fn=<ViewBackward0>)
    


```python

irradiance = sun_visibility * torch.ones(1) + (1 - sun_visibility) * sky_color
predicted_color = albedo * irradiance
print(images[0][0].shape)
print(predicted_color.shape)
loss = nn.MSELoss()(predicted_color,images[0][0].view(10000,-1))
print(loss)
```

    torch.Size([100, 100])
    torch.Size([10000, 1])
    tensor(0.1542, grad_fn=<MseLossBackward0>)
    


```python
print(sun_visibility.shape)
print(images[0].shape)
```

    torch.Size([10000, 1])
    torch.Size([1, 100, 100])
    


```python
model=snerf_model = SNeRF()
sun_directions = [torch.tensor([np.random.uniform(0, 2 * np.pi), np.random.uniform(0, np.pi)]).float() for _ in range(len(images))]
# 定义S-NeRF损失函数
def snerf_loss(snerf_model, images,sun_directions,lambda_s=0.0005):
    total_loss = 0.0
    batch_size = images.shape[0]

    # 计算像素损失
    for i in range(batch_size):
        x = get_snerf_input(images[i])  # 获取S-NeRF模型的输入
        density, albedo, sun_visibility, sky_color = snerf_model(x, sun_directions[i])
        irradiance = sun_visibility * torch.ones(3) + (1 - sun_visibility) * sky_color
        predicted_color = albedo[i] * irradiance
        total_loss += nn.MSELoss()(predicted_color, images[i][0].view(10000,-1))
    # 计算太阳校正损失
    for i in range(batch_size):
        x = get_snerf_input(images[i])
        density, albedo, sun_visibility, sky_color = snerf_model(x, sun_directions[i])
        with torch.no_grad():
            transparency = compute_transparency(density)  # 假设已经定义了计算透明度的函数
        solar_correction_loss = lambda_s * (torch.sum(transparency - sun_visibility[i]) ** 2) +1 - torch.sum(sun_visibility[i] * transparency)
        total_loss += solar_correction_loss
    return total_loss / batch_size
# 计算透明度的函数（根据论文中的公式）
def compute_transparency(density):
    alpha = 1 - torch.exp(-density)
    alpha = torch.clamp(alpha, min=1e-6, max=0.99)  # 限制 alpha 的范围
    transparency = torch.cumprod(1 - alpha, dim=0)
    return transparency

print(snerf_loss(snerf_model, images,sun_directions))
```

    tensor(7756.1079, grad_fn=<DivBackward0>)
    

    D:\anaconda\envs\torchdamage\Lib\site-packages\torch\nn\modules\loss.py:535: UserWarning: Using a target size (torch.Size([10000, 1])) that is different to the input size (torch.Size([10000, 3])). This will likely lead to incorrect results due to broadcasting. Please ensure they have the same size.
      return F.mse_loss(input, target, reduction=self.reduction)
    


```python
# 初始化S-NeRF模型
snerf_model = SNeRF()

# 训练S-NeRF模型
num_epochs = 100
learning_rate = 0.01
optimizer = optim.Adam(snerf_model.parameters(), lr=learning_rate)

for epoch in range(num_epochs):
    optimizer.zero_grad()
    sun_directions = [torch.tensor([np.random.uniform(0, 2 * np.pi), np.random.uniform(0, np.pi)]).float() for _ in range(len(images))]
    loss = snerf_loss(snerf_model, images, sun_directions)
    loss.backward()
    optimizer.step()
    if epoch%10==0:
        print(f'Epoch {epoch + 1}/{num_epochs}, S-NeRF Loss: {loss.item()}')

```

    D:\anaconda\envs\torchdamage\Lib\site-packages\torch\nn\modules\loss.py:535: UserWarning: Using a target size (torch.Size([10000, 1])) that is different to the input size (torch.Size([10000, 3])). This will likely lead to incorrect results due to broadcasting. Please ensure they have the same size.
      return F.mse_loss(input, target, reduction=self.reduction)
    

    Epoch 1/100, S-NeRF Loss: 20100.837890625
    Epoch 11/100, S-NeRF Loss: 5.481657981872559
    Epoch 21/100, S-NeRF Loss: 1.604288101196289
    Epoch 31/100, S-NeRF Loss: 1.2277047634124756
    Epoch 41/100, S-NeRF Loss: 1.187923789024353
    Epoch 51/100, S-NeRF Loss: 1.1886228322982788
    Epoch 61/100, S-NeRF Loss: 1.1879128217697144
    Epoch 71/100, S-NeRF Loss: 1.1873924732208252
    Epoch 81/100, S-NeRF Loss: 1.227446436882019
    Epoch 91/100, S-NeRF Loss: 1.1871311664581299
    


```python


```


```python

```
