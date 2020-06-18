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
[myCentralManager connectPeripheral:peripheral options:nil];
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

peripheral을 연결한 후에 그들의 데이터를 explore 한다.  explore의 첫 단계는 peripheral이 제공하는 service 중 사용 가능한 것들만 찾는다. peripheral이 advertising 할 수 있는 양이 제한 되어 있음으로 찾고자 하는 service가 실제로는 더 많을 수 있다.

하지만 [discoverServices:](https://developer.apple.com/documentation/corebluetooth/cbperipheral/1518706-discoverservices) 메소드를 사용하여 모든 service를 찾을 수 있다. 

 ~~~objective-c
    [peripheral discoverServices:nil];
 ~~~

> 실제 사용할 때는 위의 메소드 파라미터에 nil을 자주 쓰지는 않을 것이다. nil을 넘기면 모든 service를 return하기 때문이다. peripheral은 너가 사용하고자 하는 service보다 더 많은 양을 갖고 있다. 이를 모두 찾으면 배터리 소모와 시간 낭비가 되버린다. 그렇기에 찾고자 하는 서비스의 UUID를 지정하여 찾고는 한다.

service를 발견 했을 때, peripheral(연결 되어진)은 [peripheral:didDiscoverServices:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518744-peripheral) delegate 메소드를 콜한다. peripheral에 의해 발견 된 service의 객체인 CBService 배열들이 만들어진다. 

아래의 코드는 발견한 service들에 접근하는 코드다.

~~~objective-c
- (void)peripheral:(CBPeripheral *)peripheral
didDiscoverServices:(NSError *)error {
 
    for (CBService *service in peripheral.services) {
        NSLog(@"Discovered service %@", service);
        ...
    }
    ...

~~~

<br/>

## Discovering the Characteristics of a Service

원하는 service를 찾았으면, service의 characteristic을 찾아야한다. 해당 service의 모든 characteristic을 찾는 것은 쉽다. peripheral의 [discoverCharacteristics:forService:](https://developer.apple.com/documentation/corebluetooth/cbperipheral/1518797-discovercharacteristics) 메소드를 호출하면 된다.

~~~objective-c
    NSLog(@"Discovering characteristics for service %@", interestingService);
    [peripheral discoverCharacteristics:nil forService:interestingService];
~~~

> 일반적으로 실제 개발할 때는 첫번째 파라미터에 nil을 넘기지 않는다. 모든 characteristic들을 return 받기 때문이다. 찾고자 하는 characteristics보다 훨씬 더 많기 때문에 이를 모두 return 받게 되면 배터리 소모와 시간 낭비가 되버린다. 찾고자 하는 UUID를 넘겨서 사용하는 것이 일반적이다.

명시된 service의 characteristic을 찾으면, peripheral은 [peripheral:didDiscoverCharacteristicsForService:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518821-peripheral) delegate method를 호출한다. Core Bluetooth는 CBCharacteristic 객체로 이루어진 배열을 만든다. 아래의 코드는 발견된 characteristic을 log로 찍는 delegate를 구현한 것이다.

~~~objective-c
- (void)peripheral:(CBPeripheral *)peripheral
didDiscoverCharacteristicsForService:(CBService *)service
             error:(NSError *)error {
 
    for (CBCharacteristic *characteristic in service.characteristics) {
        NSLog(@"Discovered characteristic %@", characteristic);
        ...
    }
    ...
~~~

<br/>

## Retrieving the Value of a Characteristic

characteristic은 peripheral의 service 정보가 담겨져 있는 한개의 value를 가지고 있다. 예를 들면 체온계의 온도 측정 characteristic은 섭씨 온도에 대한 정보를 나타낸다. characteristic에 대한 정보는 직접 reading 하거나 구독하여 검색 할 수 있다. 

### Reading the Value of a Characteristic

찾고자 하는 service의 characteristic을 찾았다면, peripheral의 readValueForCharacteristic: 메소드를 호출하여 characteristic을 읽을 수 있다. 

~~~objective-c
NSLog(@"Reading value for characteristic %@", interestingCharacteristic);
[peripheral readValueForCharacteristic:interestingCharacteristic];
~~~

위의 메소드를 실행하면, [peripheral:didUpdateValueForCharacteristic:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518708-peripheral) delegate 메소드가 호출된다. value 검색이 성공적으로 됐다면, 아래의 코드처럼 characteristic value 프로퍼티에 접근 할 수 있다. 

~~~objective-c
- (void)peripheral:(CBPeripheral *)peripheral
didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic
             error:(NSError *)error {
 
    NSData *data = characteristic.value;
    // parse the data as needed
    ...

~~~

> 모든 characteristic들이 읽을 수 있는 것들이 아니다. [properties](https://developer.apple.com/documentation/corebluetooth/cbcharacteristic/1519010-properties)에 [CBCharacteristicPropertyRead](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties/cbcharacteristicpropertyread) 상수를 넣으면 읽을 수 있는 characteristic인지 아닌지 확인 가능하다. 만약 읽을 수 없는 characteristic이라면 [peripheral:didUpdateValueForCharacteristic:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518708-peripheral)은 그에 해당하는 error를 뱉을 것이다.

<br/>

### Subscribing to a Characteristic’s Value

[readValueForCharacteristic:](https://developer.apple.com/documentation/corebluetooth/cbperipheral/1518759-readvalue) 메소드를 사용하여 static한 characteristic을 읽을 수는 있지만, 동적인 characteristic에는 적절하지 않다. 변동이 심한 characteristic(예를 들면 심박수 측정기)을 검색하려면 그들을 구독해야 한다. characteristic을 구독하면 value가 변동이 있을때마다 알림을 받을 수 있다. 

peripheral의 [setNotifyValue:forCharacteristic:](https://developer.apple.com/documentation/corebluetooth/cbperipheral/1518949-setnotifyvalue) 메소드를 사용하면 characteristic의 값을 구독할 수 있다. 

~~~objective-c
[peripheral setNotifyValue:YES forCharacteristic:interestingCharacteristic];
~~~

구독을 시행하면, peripheral은 [peripheral:didUpdateNotificationStateForCharacteristic:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518768-peripheral) delegate 메소드를 호출한다. 만약 구독이 실패 됐다면, 위의 delegate에서 error의 원인을 찾을 수 있다. 

~~~objective-c
- (void)peripheral:(CBPeripheral *)peripheral
didUpdateNotificationStateForCharacteristic:(CBCharacteristic *)characteristic
             error:(NSError *)error {
 
    if (error) {
        NSLog(@"Error changing notification state: %@",
           [error localizedDescription]);
    }
    ...
~~~

> 모든 characteristic들이 구독 기능을 제공하지 않는다. [properties](https://developer.apple.com/documentation/corebluetooth/cbcharacteristic/1519010-properties)에 [CBCharacteristicPropertyNotify](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties/1518976-notify) 혹은 [CBCharacteristicPropertyIndicate](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties/1519085-indicate) 상수를 포함하여 구독이 되는지 체크 할 수 있다. 

구독이 성공하면, peripheral 기기는 characteristic 값이 변동 될 때마다 알림을 준다. 값이 변경 될 때 마다 [peripheral:didUpdateNotificationStateForCharacteristic:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518768-peripheral)를 실행하는데, 변경된 값을 찾기 위해서는 위에서 **Reading the Value of a Characteristic** 챕터처럼 똑같이 검색하면 된다. 

## Writing the Value of a Characteristic

characteristic value를 write 해야할 때도 있다. 만약 BLE 온도 조절기에 조절하고자 하는 실내 온도 값을 전송해야 할 때다. 만약 characteristic value이 write 할 수 있다면, [writeValue:forCharacteristic:type:](https://developer.apple.com/documentation/corebluetooth/cbperipheral/1518747-writevalue) 메소드를 사용하여 write 할 수 있다. 

~~~objective-c
NSLog(@"Writing value for characteristic %@", interestingCharacteristic);
[peripheral writeValue:dataToWrite forCharacteristic:interestingCharacteristic type:CBCharacteristicWriteWithResponse];
~~~

characteristic value를 write하려고 할 때, 사용하고자 하는 write의 타입을 주입한다. 위의 코드에서 보자면 [CBCharacteristicWriteWithResponse](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicwritetype/cbcharacteristicwritewithresponse) 를 write type으로 넘겨주고 있다. 이 타입은 write의 성공 유무의 값을 [peripheral:didWriteValueForCharacteristic:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate/1518823-peripheral) delegate 메소드로 받을지 말지 결정한다. 

~~~objective-c
- (void)peripheral:(CBPeripheral *)peripheral
didWriteValueForCharacteristic:(CBCharacteristic *)characteristic
             error:(NSError *)error {
 
    if (error) {
        NSLog(@"Error writing characteristic value: %@",
            [error localizedDescription]);
    }
    ...
~~~

이와 반대로 [CBCharacteristicWriteWithoutResponse](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicwritetype/withoutresponse) write type을 넘기면, 우선 write operation 성능은 올라가지만 전달 되는 것이 보장되거나 보고되지 않는다. 그리고 위의 delegate 메소드 또한 호출하지 않는다. Core bluetooth에서 더욱 많은 write type이 보고 싶다면 [CBCharacteristicWriteType](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicwritetype) 을 참고하라. 

> 어떤 Characteristic은 write 만 제공하거나 혹은 아예 지원 안할수도 있다. Characteristic이 write를 지원하는지 체크하려면 [properties](https://developer.apple.com/documentation/corebluetooth/cbcharacteristic/1519010-properties) 에 [CBCharacteristicPropertyWriteWithoutResponse](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties/cbcharacteristicpropertywritewithoutresponse) 혹은 [CBCharacteristicPropertyWrite](https://developer.apple.com/documentation/corebluetooth/cbcharacteristicproperties/1519089-write) 상수를 넣으면 된다.

