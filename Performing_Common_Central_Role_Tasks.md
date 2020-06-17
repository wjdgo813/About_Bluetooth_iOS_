# Performing Common Central Role Tasks

BLE에서 Central의 역할을 구현한 Device는 공통적인 task를 수행한다. Peripheral을 발견하고 연결하고, Peripheral이 제공하는 데이터와 통신하는 것들 말이다.

Peripheral의 역할을 구현한 Device들도 여러가지 task들을 수행한다. service 자신의 존재를 알리고, 연결된 central의 read, write 요청에 응답하는 역할을 한다.

이 챕터에서는 Core Bluetooth framework의 Central쪽에서의 사용 방법에 대해 알아 볼 것이다. Code 베이스로 Cetnral 역할을 개발하는 앱에 대해 알아 볼 것이다. 배울 것은 아래와 같다.

- Start up a central manager object
- Discover and connect to peripheral devices that are advertising
- Explore the data on a peripheral device after you’ve connected to it
- Send read and write requests to a characteristic value of a peripheral’s service
- Subscribe to a characteristic’s value to be notified when it is updated

## Starting Up a Central Manager

CBCentralManager은 객체 지향이기 때문에, BLE를 사용하기 전에 이 객체를 할당하고 초기화 해야한다. 초기화는 아래와 같이 진행한다. 

~~~objective-c
myCentralManager =
        [[CBCentralManager alloc] initWithDelegate:self queue:nil options:nil];
~~~

위의 예제에서 self는 delegate를 넘겨주는데, 이는 central 역할로서의 event를 받기 위함이다. dispatch queue는 nil로 할당했는데, central 역할을 main queue에서 사용하기 위해서다. 

Central manager를 만들었을 때, delegate 객체의 method인 [centralManagerDidUpdateState:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518888-centralmanagerdidupdatestate)가 호출된다. BLE가 central 디바이스를 지원하고 사용가능하기 위해 이 delegate 메소드를 구현해야 한다.

## Discovering Peripheral Devices That Are Advertising

초기화 된 central 매니저의 첫번째 task는 이전에 언급했었던  [Centrals Discover and Connect to Peripherals That Are Advertising](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothOverview/CoreBluetoothOverview.html#//apple_ref/doc/uid/TP40013257-CH2-SW3) 처럼 peripheral들은 그들의 존재를 주변에 알린다. [scanForPeripheralsWithServices:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518986-scanforperipheralswithservices) 메소드 호출을 함으로서 주변 advertising하고 있는 peripheral들을 발견할 수 있다. 

```objective-c
`    [myCentralManager scanForPeripheralsWithServices:nil options:nil];`
```

> 위의 코드에서 첫번째 파라미터에 nil을 넘겼다는 것은 지원 되고 있는 service와 상관 없이 주변의 모든 peripheral들을 검색하겠다는 의미다. 실제 앱에서 사용 할 때, [CBUUID](https://developer.apple.com/documentation/corebluetooth/cbuuid) array를 지정한다. peripheral의 service들은 UUID를 advertising 한다. UUID 배열을 파라미터로 넘기면 거기에 해당하는 하나의 peripheral을 return 받는다. 

Central manager가 peripheral을 발견할 때 마다, [centralManager:didDiscoverPeripheral:advertisementData:RSSI:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518937-centralmanager) delegate의 메소드가 호출된다. 새롭게 발견 된 peripheral은 [CBPeripheral](https://developer.apple.com/documentation/corebluetooth/cbperipheral) 객체를 return 한다. 발견된 peripheral과 연결을 할 것이라면, strong reference를 유지해라.  이 상태는 deallocate 되지 않을 것이다. 

아래의 코드는 프로퍼티에 발견된 peripheral을 할당하여 reference를 유지하는 예제다.

~~~objective-c

- (void)centralManager:(CBCentralManager *)central
 didDiscoverPeripheral:(CBPeripheral *)peripheral
     advertisementData:(NSDictionary *)advertisementData
                  RSSI:(NSNumber *)RSSI {
 
    NSLog(@"Discovered %@", peripheral.name);
    self.discoveredPeripheral = peripheral;
    ...

~~~

여러 개의 device와 연결을 할 것이라면, peripherals의 array에 저장하면 된다. 연결하고자 하는 peripheral을 모두 찾으면, 전원 절약을 하기 위해 scan 동작을 멈춰야 한다. 

```objective-c
`    [myCentralManager stopScan];`
```

<br/>

## Connecting to a Peripheral Device After You’ve Discovered It

연결하고자 하는 peripheral을 발견 했다면, central manager의 [connectPeripheral:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518766-connect) 메소드를 호출하면 된다. 

```objective-c
`    [myCentralManager connectPeripheral:peripheral options:nil];`
```

연결이 성공적으로 되었다면, central manager delegate의 [centralManager:didConnectPeripheral:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518969-centralmanager) 가 호출된다. Peripheral과 통신을 시작하기 전에 peripheral의 delegate를 설정 할 수 있다. 

~~~objective-c
- (void)centralManager:(CBCentralManager *)central
  didConnectPeripheral:(CBPeripheral *)peripheral {
 
    NSLog(@"Peripheral connected");
    peripheral.delegate = self;
    ...

~~~

<br/>

## Discovering the Services of a Peripheral That You’re Connected To

peripheral을 연결한 후에 그들의 데이터를 볼 수 있다. 