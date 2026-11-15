---
title: 'Godot实践:俯视角2D角色间物理推挤碰撞实现'
published: 2026-11-15
description: '关于如何实现CharacterBody2D节点间推挤碰撞的学习经验'
image: ''
tags: ['Godot', '2D', '实践']
category: 'Godot'
draft: false
lang: 'zh-CN'
---
<center><a href='#最终方案-运用物理知识'>[省流]一键查看最终方案</a></center>

# 前言

最近心血来潮想做游戏，于是我再次打开了Godot，想了一下决定做个类似以撒的俯视角2D打怪游戏

为了让角色能够移动还具有物理效果，一般选用**CharacterBody2D** 或者 **RigidBody2D** ，RigidBody2D需要依靠施加力来让角色进行移动，实际用下来感觉手感并不好，于是我选择了CharacterBody2D(以下简称Character)

> **CharacterBody2D** 是针对用户控制的物理体的特化类。它们不会受到物理的影响，但会影响路径上的其他物理体。除了由 [PhysicsBody2D.move_and_collide()](https://docs.godotengine.org/zh-cn/4.x/classes/class_physicsbody2d.html#class-physicsbody2d-method-move-and-collide) 提供的常见的碰撞检测之外，它们主要用于提供移动对象的高阶 API，能够检测墙壁和斜坡（[move_and_slide()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-move-and-slide) 方法）。因此适用于需要高度可配置的物理体，因为通常是用户控制的角色，所以必须按照特定的方式移动、与世界发生碰撞。
> <footer> —— <a href=https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html>Godot Engine 4.7 简体中文文档 - CharacterBody2D</a></footer>

> `CharacterBody2D` 用于实现通过代码控制的物体。Character bodies 在移动时可以检测到与其他物体的碰撞，但不受引擎物理属性（如重力、摩擦力等）的影响。虽然这意味着你必须编写一些代码来创建它们的行为，但这也意味着你可以更精确地控制它们如何移动和反应。
> <footer> —— <a href=https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html](https://docs.godotengine.org/zh-cn/4.x/tutorials/physics/using_character_body_2d.html>Godot Engine 4.7 简体中文文档 - 使用 CharacterBody2D/3D</a></footer>


但是问题来了：Character在使用`move_and_slide`方法进行移动时，如果与其他Character发生碰撞会立刻停止移动，贴着对方移动只会纹丝不动，这并不符合真实的物理规律。正如上面所说，关于推挤/碰撞的行为逻辑需要我们自己来编写，于是我对如何实现这一功能开始了探索。

<hr />

# 现有代码
仅为方便理解接下去的内容而列出，实际运用时可按需取舍和修改。
玩家和怪物都属于`Mob`类，拥有以下属性：

```gdscript title='mob.gd'
@export_group("Movement Settings")
@export var speed: float
@export var acceleration: float
@export var mass: float
var friction: float
var target_velocity: Vector2
```
移动相关代码：
```gdscript title='mob.gd'
func _physics_process(delta: float) -> void:
	if target_velocity != Vector2.ZERO:
		velocity = velocity.move_toward(target_velocity, acceleration * delta)
	else:
		velocity = velocity.move_toward(Vector2.ZERO, friction * delta)
	
	if velocity.length() < 1.0:
		velocity = Vector2.ZERO
	
	move_and_slide()
	
	[COLLISION]
```
玩家移动代码：
```gdscript title="qu.gd"
func _physics_process(delta: float) -> void:
	var input_dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")
	
	if input_dir != Vector2.ZERO:
		target_velocity = speed * input_dir
	else:
		target_velocity = Vector2.ZERO
	
	super._physics_process(delta)
```
target_velocity代表实体预期达到的速度。在运行时，如果target_velocity不为0，则以加速度acceleration使当前速度向预期速度增加，否则以摩擦力friction减少至0。  
玩家移动时，根据speed和输入的方向input_dir计算预期速度，如果未按下移动键，将预期速度设为0。  
此外，我还添加了 **质量(mass)** 这一属性，用于计算摩擦力和处理碰撞，friction = mass * 500   

在游戏中，玩家是一条蛆(qu)，怪物是苍蝇(fly)，以下是他们各自的属性：
```text title=qu
speed = 100.0
acceleration = 500.0
mass = 1.0
```

```text title=fly
speed = 50.0
acceleration = 100.0
mass = 0.1
```

因为应该先调用`move_and_slide`方法来判断与哪些实体发生了碰撞，我们应该在mob.gd的`[CODE]`处添加碰撞相关代码。  
由于可能与多个实体发生碰撞，我们应该遍历每一个碰撞的实体，以下是代码实现：
```gdscript title='mob.gd'
func _physics_process(delta: float) -> void:
	...
	
	for i in get_slide_collision_count():
			var collision = get_slide_collision(i)
			var collider = collision.get_collider()
			
			if collider is Mob:
				var normal = collision.get_normal()
				var total_mass = mass + collider.mass
				[CODE 2]
```

> [int](https://docs.godotengine.org/zh-cn/4.x/classes/class_int.html#class-int) [**get_slide_collision_count**](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-get-slide-collision-count)() const  
> 返回最近一次调用 [move_and_slide()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-move-and-slide) 时，该物体发生碰撞并改变方向的次数。
> 
> [KinematicCollision2D](https://docs.godotengine.org/zh-cn/4.x/classes/class_kinematiccollision2d.html#class-kinematiccollision2d)  [**get_slide_collision**](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-get-slide-collision) (slide_idx: [int](https://docs.godotengine.org/zh-cn/4.x/classes/class_int.html#class-int))  
> 返回 [KinematicCollision2D](https://docs.godotengine.org/zh-cn/4.x/classes/class_kinematiccollision2d.html#class-kinematiccollision2d)，包含最近一次调用 [move_and_slide()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-move-and-slide) 时发生的碰撞信息。因为单次调用 [move_and_slide()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-move-and-slide) 可能发生多次碰撞，所以你必须指定碰撞索引，范围为 0 到 ([get_slide_collision_count()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-get-slide-collision-count) - 1)。另见 [get_last_slide_collision()](https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html#class-characterbody2d-method-get-last-slide-collision)。
> <footer> —— <a href=https://docs.godotengine.org/zh-cn/4.x/classes/class_characterbody2d.html>Godot Engine 4.7 简体中文文档 - CharacterBody2D</a></footer>

`collider`：碰撞的实体，准确来说是撞向自身的实体，自身作为被动方  
`normal`：“碰撞物体的形状在碰撞点的法线“，是一个2D方向向量  
`total_mass`：碰撞的两个实体的总质量。  
接下来，我们就可以在`[CODE 2]`处编写代码了。  

<hr />

# 第一步尝试-被动方角度
既然有了法线normal，那么很自然就能想到将当前实体的速度增加上碰撞实体在法线方向上的速度。这是从被动方的角度出发的。
```gdscript
velocity += normal * collision.get_collider_velocity().length()
```
效果如下：
<img src="/images/posts/godot/2d-character-push-learning-experience/first_try.webp" loading="lazy">
可以看到玩家难以推动怪物，且碰撞很生硬。  
在每次碰撞时使用`printt(collider.name, collider.velocity, collision.get_collider_velocity())`打印出信息，可以得到  
```
Fly	(0.0, 0.0)	(0.0, 0.0)
Fly	(0.0, 0.0)	(0.0, 0.0)
Qu	(0.0, -100.0)	(0.0, -5.371398)
Fly	(0.0, -100.0)	(0.0, -3.857116)

```
可以看出`collision.get_collider_velocity()`返回的是碰撞后的速度，将其直接换成`collider.velocity`，并乘上`(collider.mass/total_mass)`，试试效果。
```gdscript
velocity += normal * collider.velocity.length() * (collider.mass/total_mass)
```
<img src="/images/posts/godot/2d-character-push-learning-experience/second_try.webp" loading="lazy">

发现推动时有段落感，使用 `printt(name, collider.name, normal, collider.velocity.length())` 打印出信息，发现以下片段  

```text
Fly_1	Qu	(0.0, -1.0)	99.8492660522461
Qu	Fly_1	(0.0, 1.0)	90.7720642089844
Qu	Fly_1	(0.0, 1.0)	89.9387283325195
Qu	Fly_1	(0.0, 1.0)	89.1053924560547
Qu	Fly_1	(0.0, 1.0)	88.2720565795898
Qu	Fly_1	(0.0, 1.0)	87.438720703125
Qu	Fly_1	(0.0, 1.0)	86.6053848266602
Qu	Fly_1	(0.0, 1.0)	85.7720489501953
```

原来玩家撞向怪物后，在怪物移动、玩家贴着移动的过程中，只在玩家脚本中检测到了**怪物对玩家的碰撞**。由于我们的代码是从被动方的角度向自身施加速度，就导致了**玩家只在碰撞的一瞬间向怪物施加一个接近100的速度，接下来都是怪物对玩家施加阻碍玩家移动的速度**。  

那么，是不是只要从主动方的角度对被撞者施加速度就好了呢？

<hr />

# 第二步尝试-主动方角度
主动方角度只需要把原来的代码都反过来就好了
```gdscript
#velocity += normal * collider.velocity.length() * (collider.mass/total_mass)
collider.velocity -= normal * velocity.length() * (mass/total_mass)
```
<img src="/images/posts/godot/2d-character-push-learning-experience/third_try.webp" loading="lazy">  
发现碰撞时施加的速度明显变大了，而且还出现了类似爆炸的效果，这种效果在多个怪物追逐玩家时更为明显:  
<img src="/images/posts/godot/2d-character-push-learning-experience/third_try_2.webp" loading="lazy">  
原因其实很简单，两个实体的碰撞体积重合时双方都会不断向对方施加速度，叠加起来就形成了一个巨大的值；多个怪物挤压玩家时，每个怪物施加的速度都会叠加在玩家身上，导致了“爆炸”的现象。  

<hr />

# 原因剖析
被动方同样也是从各个怪物上获取速度施加于自身，为什么就不会爆炸呢？
这又要回到第一步尝试发现的内容： **玩家撞向怪物后，在怪物移动、玩家贴着移动的过程中，只在玩家脚本中检测到了怪物对玩家的碰撞**   
也就是说，当**玩家不动，怪物向玩家持续移动**时，实际上是**怪物检测到了玩家的碰撞**，当使用**被动方**的角度时，这会导致**大部分时间都是玩家阻碍怪物的运动**，换成**主动方**时，就导致**大部分时间是怪物对玩家施加运动**，从而导致了爆炸。  
乍一看，这似乎十分地反直觉，明明是A推B，为什么不是B检测到A对自己的碰撞，而是A检测到B？

其实答案早就藏在文章一开始关于`get_slide_collision`的定义里：  “**包含最近一次调用 move_and_slide() 时发生的碰撞信息**”  
这意味着只有实体确实通过move_and_slide方法移动了，才能检测到碰撞。对于上面的情况，B是静止的，A是运动的，自然是A检测到碰撞而不是B了。
那么真正的解决方法也就呼之欲出了：每次碰撞对双方都施加对应速度

<hr />

# 第三步尝试-双向作用
将第一二步的两行代码结合起来，看看效果：
> <img src="/images/posts/godot/2d-character-push-learning-experience/fourth_try.webp" loading="lazy">
> <center>怪物静止时</center>

> <img src="/images/posts/godot/2d-character-push-learning-experience/fourth_try_2.webp" loading="lazy">
> <center>玩家静止时</center>

发现碰撞力度有问题，只要利用 **神圣的物理学** 即可解决这个问题

<hr />

# 最终方案-运用物理知识
根据 **基于恢复系数的二维碰撞冲量模型** 可得：
## 1. 计算相对速度在法线上的投影
$v_{rel} = (\vec{v}_A - \vec{v}_B) \cdot \vec{n}$

## 2. 计算碰撞冲量大小
$J = \frac{-(1 + e) \cdot v_{rel}}{\frac{1}{m_A} + \frac{1}{m_B}}$

## 3. 计算碰撞后双方的速度
$\vec{v}_A' = \vec{v}_A + \frac{J}{m_A} \vec{n}$

$\vec{v}_B' = \vec{v}_B - \frac{J}{m_B} \vec{n}$

其中，  
$m_A, m_B$：物体 A 和 B 的质量  
$\vec{v}_A, \vec{v}_B$：碰撞前的速度向量  
$\vec{n}$：碰撞法线单位向量  
$e$：恢复系数（Restitution，0 表示完全非弹性碰撞，1 表示完全弹性碰撞）  

最终得到以下代码：
```gdscript title=mob.gd
func _physics_process(delta: float) -> void:
	...
	
	for i in get_slide_collision_count():
		var collision = get_slide_collision(i)
		var collider = collision.get_collider()
		
		if collider is Mob:
			var normal = collision.get_normal()
			var total_mass = mass + collider.mass
			var rev = (velocity - collider.velocity).dot(normal)
			var e = 1
			var j = -(1+e) * rev/(1/collider.mass+1/mass)
			velocity += j/mass * normal
			collider.velocity -= j/collider.mass * normal
```

其中`e`可以根据自己需要进行调节。
> <img src="/images/posts/godot/2d-character-push-learning-experience/final.webp" loading="lazy">
> <center>e=1 完全弹性碰撞</center>

> <img src="/images/posts/godot/2d-character-push-learning-experience/final_2.webp" loading="lazy">
> <center>e=0 完全非弹性碰撞</center>

**历尽千辛万苦，终于实现了丝滑的物理碰撞。**

<hr />

# 附-完整代码
<details>
<summary>点击展开...</summary>

```gdscript title='mob.gd'
@abstract
class_name Mob
extends CharacterBody2D

@export_group("Movement Settings")
@export var speed: float
@export var acceleration: float
@export var mass: float
var friction: float
var target_velocity: Vector2

func _ready() -> void:
	friction = mass * 500

func _physics_process(delta: float) -> void:
	if target_velocity != Vector2.ZERO:
		velocity = velocity.move_toward(target_velocity, acceleration * delta)
	else:
		velocity = velocity.move_toward(Vector2.ZERO, friction * delta)
	
	if velocity.length() < 1.0:
		velocity = Vector2.ZERO
	
	move_and_slide()
	
	for i in get_slide_collision_count():
		var collision = get_slide_collision(i)
		var collider = collision.get_collider()
		
		if collider is Mob:
			var normal = collision.get_normal()
			var total_mass = mass + collider.mass
			var rev = (velocity - collider.velocity).dot(normal)
			var e = 0
			var j = -(1+e) * rev/(1/collider.mass+1/mass)
			velocity += j/mass * normal
			collider.velocity -= j/collider.mass * normal
```

```gdscript title='qu.gd'
extends Mob

func _physics_process(delta: float) -> void:
	var input_dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")
	
	if input_dir != Vector2.ZERO:
		target_velocity = speed * input_dir
	else:
		target_velocity = Vector2.ZERO
	
	super._physics_process(delta)

```

</details>

