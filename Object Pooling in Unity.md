# Unity 内存优化 和 内存池使用实践

![ObjectPooling-feature](http://img.tairan.com/2016-12-19-ObjectPooling-feature-1-250x250.png)

> 想象一下：你正在打飞机！！哦，不，测试你最新的和最棒的一个射击游戏。敌人在以你能掌握的最快速度来回飞行，然后，砰！卡了一帧之后，你就被凶神恶煞的外星人手打成了翔。

这可是场横扫千军的战斗，不应该由于莫名其妙的内存尖峰左右战斗的结果。你是不是也曾经因为这个问题输掉？来来来，搬个小马扎，听我来扒一扒 *对象池技术* 吧。

在这篇Unity教程中，你将学到：

* 所有关于对象池技术的内容

* 如何将一个game object入池

* 如何在运行时按需扩展对象池

* 如何扩展对象池以适应不同的对象

在教程的最后，你会得到一个可以得到新游戏的全部代码。而且，你会懂得如何为现有的游戏改进这个代码。

*预备知识：*你需要熟悉C#基础并且知道如何操作Unity的开发环境。如果你需要帮助，查看这里[Unity教程(暂未翻译)](https://www.raywenderlich.com/unity-tutorials "Unity tutorials")

## 什么是对象池技术？

`Instantiate()` 和 `Destroy()` 是在游戏流程中好用而必备的方法。（通常情况下，单独调用这两个方法只占用CPU相当微小的时间。）

然而，对于在游戏流程中生命周期短暂而且每秒大量摧毁的对象群而言，CPU进行内存分配的时间占用十分显著。

![子弹就是一个适合入池的GameObject的好例子](http://img.tairan.com/2016-12-19-whatthe-650x480.png)

子弹就是一个适合入池的GameObject的好例子。


并且，Unity使用*垃圾收集(Garbage Collection)*技术来释放不需要继续使用的内存。不断调用`Destroy()`会频繁的激活收集，而这会拖慢CPU导致游戏流程的卡顿。


这一行为在移动设备和网页等资源受限的环境下中尤为致命。

*对象池技术*将尚未用到的对象放在游戏流程之前——比如在读取界面的时候——进行预先实例化。这样游戏可以从『池子』中重用对象，而不需在游戏流程中不断地创建和销毁。

![是否使用池的比对](http://img.tairan.com/2016-12-19-NonVsPooling.png)

## 开始

如果还没有Unity5或者更新版本，从 [Unity官网](https://unity3d.com/get-unity/download "Unity官网")下载。


然后下载 [起始项目](https://koenig-media.raywenderlich.com/uploads/2016/06/SuperRetroShooter_Starter.zip "起始项目")，解压并在Unity里打开 `SuperRetroShooter_Starter`项目——这是一个预先建好的纵向卷轴射击项目。

*注意*：美术资源的版权属于 [OpenGameArt](http://opengameart.org)的 Master484, Marcus, Luis Zuno 和 Skorpio。免授权音乐则来自于杰出的[Bensound](http://www.bensound.com "Bensound")

随意看看里面的脚本吧，比如*Barrier*；他们都非常实用，不过这篇教程就不对它们详细阐述了。


在浏览这些时，注意在游戏流程中Hierarchy（栏）里都发生了些什么是很有帮助的。因此我建议取消Game标签（*Game Tab*）工具栏（* toolbar *）里的*Maximize on Play*选项。

![最大化](http://img.tairan.com/2016-12-19-maximise.png)

点击*play*按钮看看。 :]


注意当发射时，在Hierarchy（栏）里实例化了大量的`PlayerBullet(Clone)`。当击中敌人或者离开屏幕时，他们会被销毁。


更糟的是收集那些掉落的强化道具，他们迅速用子弹的复制体填满了Hierarchy（栏），随后又在下一秒里立刻销毁它们。

![你需要修复这里](http://img.tairan.com/2016-12-19-Youaregoingtofixthat.gif)

在现在这个状态下，Super Retro Shooter这个游戏就像一个『内存的搅屎棍』，而你要做个英雄，让这个游戏全速工作并且更加严谨地使用资源。

###是时候试试手了

在Hierarchy里点击*Game Controller*。这个对象（GameObject）会在场景里持续存在，适合在它上面添加对象池脚本。

在Inspector中，点击*Add Component*按钮，选择*New C# Script*。起名为*ObjectPooler*。


*双击*新的脚本在MonoDevelop中打开它，并在 *类* 里添加以下代码：

 

public static ObjectPooler SharedInstance;
void Awake() {
		SharedInstance = this;
}



有一些脚本会需要在游戏流程中访问对象池，因此用`public static instance`令各个脚本可以访问它而不需要从GameObject上获取Component的操作。

在脚本顶端，增加以下`using`声明：

  

using System.Collections.Generic;

 


你将会`使用(using)`一个泛型(generic)列表来存储入池了的对象。这一声明允许你访问泛型数据结构，这样就可以在脚本里用到`List`类。

*注意*:普通(泛型的原文Generic也有普通的意思)？*没人*想要普通！每个人都希望独特！

在程序语言，比如C#里，泛型(普通)能让你的代码可以被很多不同的类型所使用，同时保持类型安全(type safety)。


一个典型的应用就是在使用集合(collection)的时候使用泛型。这个方法限定了在数组(array)中存放对象的类型，以防比如在猫数组里放了个狗，尽管这会显得很神奇。:]

说到列表(list)，添加对象池列表以及两个新的公共变量：

 

public List <GameObject> pooledObjects;
public GameObject objectToPool;
public int amountToPool;



这些变量的命名是很自明的(self-explanatory)。


通过Unity的Inspector，你将能够指定GameObject入池，并且设置预实例化的数量。一会儿会做这个。

然后，把这些代码加入*Start()*:



pooledObjects = [new](http://www.google.com/search?q=new+msdn.microsoft.com) 	ListGameObject>();
for (int i = 0; i  amountToPool; i++) {
GameObject obj = (GameObject)Instantiate(objectToPool);
		obj.SetActive(false); 
		pooledObjects.Add(obj);
}

  


`for`循环会实例化`numberToPool`所指定的数量的`objectToPool`GameObject。然后这些GameObject会被设置为inactive状态，之后再加入到`pooledObjects`列表中。

回到Unity然后在Inspector中添加*Player Bullet Prefab*到*objectToPool*变量上。在*numberToPool*字段上填入*20*。

![对象池设置](http://img.tairan.com/2016-12-19-ObjectPoolSetup.gif)

再次运行游戏。现在场景（视图）里应该有20个预实例化好但无家可归的子弹了。

![Well done! You now have an object pool :]](http://img.tairan.com/2016-12-19-ScreenCap1-208x500.png)

干得好！你现在拥有一个对象池了 :]

### 深入对象池

Jump back into the *ObjectPooler* script and add the following new method:

回到*ObjectPooler*脚本然后添加以下代码：

   
	public GameObject GetPooledObject() {
	//1
	for (int i = 0; i  pooledObjects.Count; i++) {
	//2
		if (!pooledObjects[i].activeInHierarchy) {
			return pooledObjects[i];
		}  
	}
	//3 
	return null;
	}

  


首先要注意的是这个方法现在的返回值类型是`GameObject`而非`void`。这是指别的脚本可以向`GetPooledObject`请求一个池中的对象，而他会返回一个GameObject作为回应。其他会发生的事情如下：


1. 这个方法使用`for`循环遍历你的`pooledObjects`列表。

2. 检查其中一个对象是否`并非`激活(active)状态。如果是激活的，循环到下一个。如果是非激活的，退出方法并把这个对象提交给对`GetPooledObject`的调用者。

3. 如果目前没有非激活状态的对象，退出方法并且不返回任何东西。

现在，你可以向池子请求对象了，这需要替换掉原有的子弹的实例化和销毁代码，以对象池替代。

玩家的子弹会在`ShipController`脚本里的两个方法中（被）实例化。

* `Shoot()`

* `Shoot()`

* `ActivateScatterShotTurret()`

* `ActivateScatterShotTurret()`

在MonoDevelop里打开*ShipController*脚本（并找到下列代码）：

   

Instantiate(playerBullet, turret.transform.position, turret.transform.rotation);

  

代替 *并*用以下代码替换：

   

GameObject bullet = ObjectPooler.SharedInstance.GetPooledObject(); 
	if (bullet != null) {
		bullet.transform.position = turret.transform.position;
		bullet.transform.rotation = turret.transform.rotation;
		bullet.SetActive(true);
	}

  


*注意*: 在继续之前，确保在`Shoot()`和`ActivateScatterShotTurret()`方法中也做好了这样的替换。


在之前，这些方法遍历玩家飞船上的激活着的子弹列表（（基于加成道具）），然后在枪口位置与角度上实例化子弹。

而现在改成了询问`ObjectPooler`脚本获取一个池中的对象。如果拿到了，将它设置到枪口的位置与角度，然后设为`active`状态来向敌人倾泻下你的弹火吧：]

### 归还对象池

当你不再需要使用子弹的时候，将它们归还到池中，而不是将对象销毁。

有两个方法可以销毁不再需要的玩家子弹：

* net当子弹移出屏幕时，调用`DestroyByBoundary`中的`OnTriggerExit2D()` 方法移除它们。

* `OnTriggerEnter2D()` in the `EnemyDroneController` script removes the bullet when it collides and destroys an enemy.

*当子弹碰撞并摧毁敌人时，调用`EnemyDroneController`中的`OnTriggerEnter2D()`方法移除它们。

在MonoDevelop中打开*DestroyByBoundary*，并将`OnTriggerExit2D`方法替换成如下代码：


   
if (other.gameObject.tag == "Boundary") {
	if (gameObject.tag == "Player Bullet") {
			gameObject.SetActive(false);
	} else {
		Destroy(gameObject);
		}
}

  



那些*离开屏幕范围时需要被移除的对象*可以使用这一段通用脚本加以控制。这段代码可以判断这些触发了碰撞的对象是否拥有`Player Bullet`标签——如果有，则将其设置为非活动状态，而不是直接销毁。

类似地，打开*EnemyDroneController*并找到`OnTriggerEnter2D()`方法，将`Destroy(other.gameObject);`替换成这一行代码：


   
other.gameObject.SetActive(false);

  



等等，刀下留人！

没错，我几乎都能*看见*你的鼠标指在了运行按钮上。在进行了这么一大段编程之后，你肯定对于测试对象池跃跃欲试了吧。先别忙！我们还要再改一个脚本——别担心，只是一点小改动。^_^

在MonoDevelop中打开*BasicMovement*脚本，并将`Start`方法重命名为`OnEnable`。

![One gotcha when you use the object pool pattern is remembering the lifecycle of your pooled object is a little different.](http://img.tairan.com/2016-12-19-GameLoop-Diagram.png)

这里有个坑：使用对象池模式时，注意池中对象的生命周期会和之前有所不同。

好了，你*可以*点击运行了。^_^

当你射击时，在Hierarchy中的玩家子弹自动从未激活状态变为激活状态。它们会在移出屏幕、或击中敌机的时候，以优雅代码应有的方式变成回未激活的状态。

干的漂亮！

![ScreenRec4Gif](http://img.tairan.com/2016-12-19-ScreenRec4Gif.gif)

但假如你收集了所有的加成道具，会怎样？

“啊呀，子弹不够用了”？

![sgd_16_whathaveyoudone](http://img.tairan.com/2016-12-19-sgd_16_whathaveyoudone-433x320.jpg)

至高无上的游戏设计者当然可以控制一切！比如可以控制玩家的火力，玩家就会更加专注于射击目标，也可以反过来鼓励玩家随时随地射个痛快。


若要实现这样的控制，只要修改最初的池中设定过的子弹数量就可以了。


相反的，你也可以走另一条路线，给池中设置一大堆的子弹，直接满足收集了全部加成道具的场景。但这会产生一个问题：若你在百分之九十的情况下只需要50发弹药，为什么要为这难得一见的终极力量而给池中设计100一百发的容量呢？


毕竟这样做，内存中就会有50发子弹几乎派不上用场了。

![Fairtome](http://img.tairan.com/2016-12-19-Fairtome-432x320.png)

### 超级扩展对象池

现在你将学习如何修改对象池，以在运行时按需增加池中的对象数量。

在MonoDevelop中打开*ObjectPooler*脚本，并将以下代码加入到公有变量中：


   
public bool shouldExpand = true;

  


这个代码会在Inspector中创建一个复选框，用来决定是否可以增加池中对象的数量。

在`GetPooledObject()`中，将`return null;`修改为如下代码：


   
if (shouldExpand) {
		GameObject obj = (GameObject)Instantiate(objectToPool);
		obj.SetActive(false);
		pooledObjects.Add(obj);
		return obj;
} else {
		return null;
}

  




当向池中请求玩家子弹时，若找不到未激活的子弹可以使用，这段代码就会设法扩展对象池，而不是简单的退出。如果这样做，你将实例化一个新子弹，设置其为未激活状态，将其加入到池中，并作为返回值交给请求者。


在Unity里点击*运行*，然后试试看吧。搞点儿加成道具，射个痛快！你的20发子弹的对象池是可以根据需要自行扩展的！

![ScreenRec6Gif](http://img.tairan.com/2016-12-19-ScreenRec6Gif.gif)

![seriously](http://img.tairan.com/2016-12-19-seriously-650x481.png)

###对象池群

Invariably, lots of bullets mean lots of enemies, lots of explosions, lots of enemy bullets and so on.

通常情况下，大量的子弹就意味着大量的敌人，不停的爆炸效果，敌人们大量发射的子弹，以及其他方方面面开销。

要为这场疯狂屠戮做准备，你需要扩展你的对象池以便于处理复杂的对象类型。接下来的一步，将实现在Inspector中的相同位置调节 不同类型自带的参数。

在*ObjectPooler*类的*前面*加入以下代码：



  
[System.Serializable]
public class ObjectPoolItem {
}

 



`[System.Serializable]` allows you to make instances of this class editable from within the Inspector.

`[System.Serializable]`允许你在Inspector里实例化这个类的对象。

接下来将`objectToPool`，`amountToPool` 和 `shouldExpand`这三个变量移动到新的`ObjectPoolItem`类里。注意：在移动过程中会遇到报错，不过我们稍后会搞定它。

更新之后的`ObjectPoolItem`类应该是这样的：


  
[System.Serializable]
public class ObjectPoolItem {
public int amountToPool;
public GameObject objectToPool;
public bool shouldExpand;
}

 



现在一些`ObjectPoolItem`的实例就能修改自己成员数据与行为了。

当你添加了上面的共有变量（public variables），你需要确保从`ObjectPooler`中*删除*它们。将下列代码加入`ObjectPooler`中：


  

public List<ObjectPoolItem> itemsToPool;

 




新的变量列表可以让你控制`ObjectPooler`的实例

接下来你需要调整`ObjectPooler`里的`Start()`以保证`ObjectPooler`的所有实例都在你的池列表中。将`Start()`的代码修改为如下：


  


```void Start () {
pooledObjects = [new](http://www.google.com/search?q=new+msdn.microsoft.com) 	ListGameObject>();
foreach (ObjectPoolItem item in itemsToPool) {
for (int i = 0; i item.amountToPool; i++) {
GameObject obj = (GameObject)Instantiate(item.objectToPool);
obj.SetActive(false);
pooledObjects.Add(obj);
}
}
}
```

 


在这里，通过一个`foreach`循环来遍历*ObjectPoolItem*的所有实例，并把合适的对象添加到对象池中。

You might be wondering how to request a particular object from the object pool – sometimes you need a bullet and sometimes you need more Neptunians, you know?

你可能会想知道怎样从对象池里请求一个特定的对象——有时候你需要一枚子弹，有时候你需要多来几颗海王星，诸如此类。

Tweak the code in `GetPooledObject` so it matches the following:

将`GetPooledObject`里的代码调整为如下：



  

```public GameObject GetPooledObject(string tag) {
for (int i = 0; i pooledObjects.Count; i++) {
if (!pooledObjects[i].activeInHierarchy && pooledObjects[i].tag == tag) {
return pooledObjects[i];
}
}
foreach (ObjectPoolItem item in itemsToPool) {
if (item.objectToPool.tag == tag) {
if (item.shouldExpand) {
GameObject obj = (GameObject)Instantiate(item.objectToPool);
obj.SetActive(false);
pooledObjects.Add(obj);
return obj;
}
}
}
return null;
}
```

 



现在你的`GetPooledObject`多了一个`string`参数，这样游戏就可以按照相应的`tag`标签来进行索引。这个方法会在对象池里检索一个具有此标签的非活动对象，然后将其返回。


此外，在找不到相应的对象的情况下，它会根据标签来检索相关的`ObjectPoolItem`实例并检查是否可以将其扩展。

###增加Tag标签

这里先拿子弹来举例，之后你可以加入其他的对象。

在MonoDevelop里打开*ShipController*脚本。在`Shoot()`和`ActivateScatterShotTurret()`里找到如下代码：



  
GameObject bullet = ObjectPooler.SharedInstance.GetPooledObject();

 



在代码里添加`Player Bullet`这一参数作为Tag标签。



  
GameObject bullet = ObjectPooler.SharedInstance.GetPooledObject(“Player Bullet”);

 




回到Unity里点击*GameController* ，在Inspector打开它。

在*ItemsToPool* 中增加一个新的道具，然后在其里面加入20发玩家的子弹。

![multiobjectpool1](http://img.tairan.com/2016-12-19-multiobjectpool1-405x500.png)


点击*Play*来确保别的东西没有被乱动过。:]

好！*现在*你已经学会怎么向对象池里添加新的对象了。

将*ItemsToPool* 的大小改为*3*，然后新加入两类敌人的舰船。按照如下内容来调整*ItemsToPool* 的实例参数。

Element 1:

*Object to Pool: EnemyDroneType1
Amount To Pool: 6
Should Expand: Unchecked*

Element 2

*Object to Pool: EnemyDroneType2
Amount to Pool: 6
Should Expand: Unchecked*

![multiobjectpool2](http://img.tairan.com/2016-12-19-multiobjectpool2-341x500.png)

如处理子弹时一样，你需要改变这两类敌舰`instantiate`和`destroy`的方法。

敌舰在执行`GameController`脚本时进行实例化，并在`EnemyDroneController`这一脚本中销毁。

之前已经做过一次了，所以这里我们会快一点。-w-

打开*GameController*脚本。在*SpawnEnemyWaves()*中找到*enemyType1 instantiation code*：



  
Instantiate(enemyType1, spawnPosition, spawnRotation);

 


将其替换为如下代码：



  
GameObject enemy1 = ObjectPooler.SharedInstance.GetPooledObject("Enemy Ship 1"); 
if (enemy1 != null) {
enemy1.transform.position = spawnPosition;
enemy1.transform.rotation = spawnRotation;
enemy1.SetActive(true);
}

 


找到*enemyType2 instantiation*这段代码：



  
Instantiate(enemyType2, spawnPosition, spawnRotation);

 


将其替换为：



  
GameObject enemy2 = ObjectPooler.SharedInstance.GetPooledObject("Enemy Ship 2"); 
if (enemy2 != null) {
enemy2.transform.position = spawnPosition;
enemy2.transform.rotation = spawnRotation;
enemy2.SetActive(true);
}

 



最后，打开*EnemyDroneController* 脚本，目前`OnTriggerExit2D()`在敌舰离开屏幕之后就会销毁这个敌舰对象，这可不太行。（这太浪费了！）

找到此行代码：



  
Destroy(gameObject);

 


将其替换为如下代码以确保敌人还会回到对象池：



  
gameObject.SetActive(false);

 



与此类似，在`OnTriggerEnter2D()`中敌人碰到玩家子弹也会被销毁。找到*Destroy()*：



  
Destroy(gameObject);

 


替换为如下：



  
gameObject.SetActive(false);

 



按*play*按钮，观察所有子弹和敌人的实例在活动与非活动状态之间的切换，以及他们在屏幕上出现的时机。

![Elon Musk would be very proud of your reusable space ships!](http://img.tairan.com/2016-12-19-ScreenRec8Gif.gif)


你的可再生船很棒棒哦~




![Finished](http://img.tairan.com/2016-12-19-Finished-650x480.png)

###何去何从

感谢你跑完这段教程。这里是完整工程的[链接](https://koenig-media.raywenderlich.com/uploads/2016/06/SuperRetroShooter_Complete.zip)。


在这个教程里，你了改写一个现有的游戏，通过给其加入对象池的方式，从可以预期的过载中拯救玩家的CPU，并解决了因此而会产生的跳帧和电池过热问题。


你在脚本的切换和内容整合上熟练度+5。

如果想了解更多关于对象池的信息，可以检阅关于这一内容的[Unity在线教程](https://www.youtube.com/watch?v=9-zDOJllPZ8 "Unity's live training")。

希望这份教程对你有用！我想知道这份教程是如何帮助你做出一个屌屌的东西，或者将一个现有的app变得屌屌的。


可以在评论里附上你的作品的链接，同样欢迎问题与关于改进的讨论！我很期待与你聊聊关于对象池，Unity以及干掉外星人的事情。





