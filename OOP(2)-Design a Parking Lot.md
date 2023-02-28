# OOP面向对象编程（二）-停车场设计 Design a parking lot

![OOP(2)-Parking Lot](https://github.com/WhosthatAoli/OOP-Design/blob/main/images/Parking%20Lot/OOP(2)-Parking%20Lot.png)

## 设计背景

### 常问问题

+ 停车场是否有多层？
+ 有什么样的汽车类型，它们所占用的空间相同吗？
+ 是否有特殊停车位保留给特殊车辆？
+ 停车场是否有收费系统，收费系统如何工作？
+ 停车位是由系统分配还是由车主自己选择？
+ 是否还有没提到的停车场可能拥有的功能

### 假设以下使用条件

+ 停车场有多层

+ 车辆拥有不同的大小，占用不同的停车位空间，比如轿车占用1个标准车位，豪华汽车占用2个，卡车占用3个
+ 停车场拥有收费系统，只有一个出口与入口
+ 车主会被自动分配车位，顺序是由车位编号由小到大，层数由底到高
+ 如果没有车位了，车主会被告知无法停车
+ 车主在离开停车场时被收取停车费，费用= 时间（hours）*单价（hourly rate）

## OOP设计

### 上层设计

+ 我们需要创建基础的**车辆**（Vehicle）类，而轿车（Car），豪华汽车（Limo），卡车（Truck）继承车辆类

+ 每种车辆拥有特定占用车位大小
+ **车主**（建立一个class）拥有车辆，账单（需要支付）
+ **停车场**（建立class）拥有多个**楼层**（建立class）
+ 每个楼层拥有多个**停车位**(spot)，使用标志位 = 1表示已占用，0表示空
+ **停车系统**（建立class）作为Controller，负责追踪停车与付款

### Code

**Vehicle** Class是车辆的基础类，拥有一个size attribute，决定占用空间的大小。

```python
class Vehicle:
    def __init__(self, spot_size):
        self._spot_size = spot_size

    def get_spot_size(self):
        return self._spot_size

```



**Driver** Class 拥有车辆，账单等属性

```python
class Driver:
    def __init__(self, id, vehicle):
        self._id = id
        self._vehicle = vehicle
        self._payment_due = 0

    def get_vehicle(self):
        return self._vehicle

    def get_id(self):
        return self._id

    def charge(self, amount):
        self._payment_due += amount

```



**轿车**（Car），**豪华汽车**（Limo），**卡车**（SemiTruck）继承Vehicle类

```python
class Car(Vehicle):
    def __init__(self):
        super().__init__(1)

class Limo(Vehicle):
    def __init__(self):
        super().__init__(2)

class SemiTruck(Vehicle):
    def __init__(self):
        super().__init__(3)

```



一个停车楼层（ParkingFloor）是停车位（parking spots）的容器，停车位用一个数组表示（这里以一维数组为例）。

使用`l`,`r`代表一个车辆占用的空间范围。

```python
class ParkingFloor:
    def __init__(self, spot_count):
        self._spots = [0] * spot_count
        self._vehicle_map = {}

    def park_vehicle(self, vehicle):
        size = vehicle.get_spot_size()
        l, r = 0, 0
        while r < len(self._spots):
            if self._spots[r] != 0:
                l = r + 1
            if r - l + 1 == size:
                # we found enough spots, park the vehicle
                for k in range(l, r+1):
                    self._spots[k] = 1
                self._vehicle_map[vehicle] = [l, r]
                return True
            r += 1
        return False

    def remove_vehicle(self, vehicle):
        start, end = self._vehicle_map[vehicle]
        for i in range(start, end + 1):
            self._spots[i] = 0
        del self._vehicle_map[vehicle]

    def get_parking_spots(self):
        return self._spots

    def get_vehicle_spots(self, vehicle):
        return self._vehicle_map.get(vehicle)

```



一个车库拥有一定数量的楼层，注意在init时，每一层可能拥有不同的停车位数量。

```python
class ParkingGarage:
    def __init__(self, floor_count, spots_per_floor):
        self._parking_floors = [ParkingFloor(spots_per_floor) for _ in range(floor_count)]

    def park_vehicle(self, vehicle):
        for floor in self._parking_floors:
            if floor.park_vehicle(vehicle):
                return True
        return False

    def remove_vehicle(self, vehicle):
        for floor in self._parking_floors:
            if floor.get_vehicle_spots(vehicle):
                floor.remove_vehicle(vehicle)
                return True
        return False

```



停车系统类作为Controller，负责记录停车时间和收费

```python
import datetime
import math

class ParkingSystem:
    def __init__(self, parkingGarage, hourlyRate):
        self._parkingGarage = parkingGarage
        self._hourlyRate = hourlyRate
        self._timeParked = {} # map driverId to time that they parked

    def park_vehicle(self, driver):
        currentHour = datetime.datetime.now().hour
        isParked = self._parkingGarage.park_vehicle(driver.get_vehicle())
        if isParked:
            self._timeParked[driver.get_id()] = currentHour
        return isParked
    
    def remove_vehicle(self, driver):
        if driver.get_id() not in self._timeParked:
            return False
        currentHour = datetime.datetime.now().hour
        timeParked = math.ceil(currentHour - self._timeParked[driver.get_id()])
        driver.charge(timeParked * self._hourlyRate)

        del self._timeParked[driver.get_id()]
        return self._parkingGarage.remove_vehicle(driver.get_vehicle())

```



这样就完成了parking lot！让我们测试一下

```python
parkingGarage = ParkingGarage(3, 2)
parkingSystem = ParkingSystem(parkingGarage, 5)

driver1 = Driver(1, Car())
driver2 = Driver(2, Limo())
driver3 = Driver(3, SemiTruck())

print(parkingSystem.park_vehicle(driver1))      # true
print(parkingSystem.park_vehicle(driver2))      # true
print(parkingSystem.park_vehicle(driver3))      # false

print(parkingSystem.remove_vehicle(driver1))    # true
print(parkingSystem.remove_vehicle(driver2))    # true
print(parkingSystem.remove_vehicle(driver3))    # false

```

