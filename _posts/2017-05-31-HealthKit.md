
晚上看了下HealthKit，发现还挺有意思的，特别是苹果归纳了基本上现实生活中所有的健康和运动相关单位类型…

#### 一、什么是HealthKit

 HealthKit是Apple在iOS8推出的一款与健康有关的框架。通过集成HealthKit，iOS系统上不同的APP之间可以互享用户**体征**和**运动**相关的数据，比如身高、体重、血型以及运动步数、跑步距离等。
 
各APP都可以按照一定的规范向HealthKit提供数据，HealthKit会根据用户的偏好设置，自动将不同来源的所有数据整合。各APP也可以从HealthKit中获取到其他APP提供的数据。

另外，HealthKit也支持一些硬件外设，比如蓝牙手环。iPhone手机自身可以监测到的运动信息，也会自动导入到HealthKit中。
 
 
#### 二、HealthKit基本结构

简而言之，HealthKit整合了不同APP的数据并共享。为了达到这个目的，框架预先定义了一组描述用户健康信息的数据类型和单位，其他APP需要按照这种统一的规范提供或获取数据。

##### 1.对象

HealthKit中的数据对象主要分为2类：

* Characteristic Data（特征数据），表示一些基本不会变化的数据。比如用户的出生日期、血型和性别。我们可以通过调用 `dateOfBirth()`, `bloodType()`等函数从HealthKit Store中读取数据。特征数据只能通过iOS系统的**健康（Health）**应用填写或修改。

* Sample Data（样本数据），表示一些与时间相关的数据，用户的某些信息可能会随着时间的变化而变化，比如身高、运动步数。所有的样本对象都是`HKSample`的子类。它们都有3个属性：

名称 | 含义
------------ | -------------
Type | 数据的类型
Start date | 数据开始的时间
End date | 数据结束的时间

##### 2.数据

为了让数据具有实际物理意义，首先需要为数据中的值指定单位。在创建之前介绍的样本数据的时候，就需要为数据添加对应的单位。HealthKit中通过`HKUnit`来创建HealthKit支持的基本单位，比如：``` [HKUnit mileUnit] ```就实例化了现实中的单位**米**。

有了基本单位，还需要加上具体的数值才能构成一个具有现实意义的数据，```HKQuantity```就是一个由单位和数值构成的数据类，具体实例化方式为：``` [HKQuantity quantityWithUnit:[HKUnit mileUnit] doubleValue:10]```这样就表达了现实中的10米。
##### 3. 存储

HealthKit所有的数据都存储在一个名为HealthKit Store的加密数据库中。

保存数据比较简单，通过Sava函数即可：

```
- (void)saveObject:(HKObject *)object withCompletion:(void(^)(BOOL success, NSError * _Nullable error))completion;
```
查询数据比较麻烦，需要先通过数据类型（HKSampleType）、过滤条件（NSPredicate）、排序方式（NSSortDescriptor）构建一个查询条件（HKSampleQuery）对象交给HealthKit Store执行。

#### 三、隐私说明

由于用户的健康数据比较敏感，用户在使用时，必须明确设置每个APP对HealthKit读写的权限。用户甚至被要求单独为每种数据类型设置允许或拒绝的权限。比如，用户可以允许我们的APP读取计步数据，但是拒绝读取血糖水平。为了防止信息泄露，APP是不知道它是否被禁止读取数据的。 
 
HealthKit的数据不会保存在iCloud中，也不会在多设备间同步。这些数据只会保存在用户的本地设备中。
 
另外，如果我们的APP提供健康或运动相关的服务，需要要在AppStore相关页面和APP的相关界面上有明确的说明，否则不能调用HealthKit的API。

#### 四、如何集成

1.XCode中开启HealthKit

![](http://7xi7el.com1.z0.glb.clouddn.com/YH_Mall.xcodeproj%20Xcode,%20Today%20at%2010.27.50%20PM.png)

2.在需要用到的类中导入头文件

```
#import <HealthKit/HealthKit.h>
```

3.申请授权

```
if ([HKHealthStore isHealthDataAvailable]) {
        self.healthStore = [[HKHealthStore alloc] init];
        NSSet *writeDataTypes = [self dataTypesToWrite];
        NSSet *readDataTypes = [self dataTypesToRead];
        [self.healthStore requestAuthorizationToShareTypes:writeDataTypes readTypes:readDataTypes completion:^(BOOL success, NSError *error) {
            
            if (!success) {
                NSLog(@"You didn't allow HealthKit to access these read/write data types. In your app, try to handle this error gracefully when a user decides not to provide access. The error was: %@. If you're using a simulator, try it on a device.", error);
                return; }
            
            // Handle success here.         
        }];
    }
```

获得用户的授权后，就可以使用相关API了。






