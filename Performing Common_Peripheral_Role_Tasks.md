# Performing Common Peripheral Role Tasks

지난 챕터에서 우리는 Central을 어떻게 사용하는지 익혔다. 이번 챕터에서는 Core Bluetooth framework에서 Peripheral 중심의 사용법에 대해서 알아보겠다. 

이번 챕터에서 배울 것은 아래와 같다.

- Start up a peripheral manager object
- Set up services and characteristics on your local peripheral
- Publish your services and characteristics to your device’s local database
- Advertise your services
- Respond to read and write requests from a connected central
- Send updated characteristic values to subscribed centrals

이번 챕터에서 간단하고 추성적인 코드들을 볼 것이다. 구체적으로 알고 싶다면 추후에 다루게 될 [Core Bluetooth Background Processing for iOS Apps](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html#//apple_ref/doc/uid/TP40013257-CH7-SW1), [Best Practices for Setting Up Your Local Device as a Peripheral](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/BestPracticesForSettingUpYourIOSDeviceAsAPeripheral/BestPracticesForSettingUpYourIOSDeviceAsAPeripheral.html#//apple_ref/doc/uid/TP40013257-CH5-SW1). 를 보면 된다. 

## Starting Up a Peripheral Manager

우선 peripheral의 역할을 구현하려면 peripheral 매니저를 할당과 초기화를 해야한다. 할당 할 때, [CBPeripheralManager](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager) 클래스의 [initWithDelegate:queue:options:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393295-init) 메소드를 호출한다. 

~~~objective-c
    myPeripheralManager =
        [[CBPeripheralManager alloc] initWithDelegate:self queue:nil options:nil];
~~~

위 코드에서 self는 peripheral의 이벤트를 delegate로 받기위해 넘긴다. queue는 nil로 설정하였는데, 이는 main queue를 사용하겠다는 의미이다. peripheral 할당이 완료 되면, [peripheralManagerDidUpdateState:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393271-peripheralmanagerdidupdatestate) delegate 메소드가 실행된다. 당신의 앱이 peripheral로서 역할을 하고자 한다면 이 delegate 메소드를 구현해야한다. 

## Setting Up Your Services and Characteristics

peripheral의 service와 characteristic는 트리 구조 형식으로 되어있다. peripheral을 구성 할 때, 반드시 트리 구조로 service와 characteristic으로 만들어야 한다. 이를 위해서는 service와 characteristic을 식별하는 방법에 대해 알아야 한다. 

### Services and Characteristics Are Identified by UUIDs

peripheral의 service와 characteristic은 128bit의 UUID로 이루어져 있다. Core Bluetooth framework는 [CBUUID](https://developer.apple.com/documentation/corebluetooth/cbuuid) 객체로 표현한다. service와 characteristic을 식별하는 UUID들은 Bluetooth Special Interest Group(SIG)에 사전정의되어 있지는 않다. Bluetooth SIG는 16비트로 단축 된 UUID를 정의하고 게시(published)한다. 

예를 들면 Bluetooth SIG는 심박수 service의 16bit UUID인 180D로 사전 정의한다. 이 UUID는 128-bit의 0000180D-0000-1000-8000-00805F9B34FB와 같다. 128 bit UUID는 Bluetooth 기반이다. 

CBUUID 클래스는 128bit의 긴 UUID를 쉽게 다루도록 도와준다. 예를들어 128bit의 긴 UUID를 코드로 표현하고자 할 때, [CBUUID](https://developer.apple.com/documentation/corebluetooth/cbuuid) 의 UUIDWithString로 초기화 하여 간단하게 사용할 수 있다. 

~~~objective-c
    CBUUID *heartRateServiceUUID = [CBUUID UUIDWithString: @"180D"];
~~~

위 코드처럼 16비트로 정의하여 초기화 하면 Core Bluetooth는 내부적으로 128비트로 저장한다. 

### Create Your Own UUIDs for Custom Services and Characteristics

사전 정의된 bluetooth UUID로 service와 characteristic을 식별하지 않을 수도 있다. 그럴 경우, 고유한 128bit UUID를 생성하여 식별해야한다. 

command-line 유틸리티에서 uuidgen 명령어를 사용하여 128bit UUID를 쉽게 생성 할 수 있다. 우선 터미널을 열고 uuidgen을 입력한다. 그런 다음 아래와 같이 하이픈으로 구분되는 ASCII 문자열 형식으로 고유 한 128 비트 값을 수신하면 된다. 

~~~
$ uuidgen
71DA3FD1-7E10-41C1-B16F-4430B506CDE7
~~~

이를 통해 얻은 UUID로 UUIDWithString로 초기화 하여 CBUUID를 얻을 수 있다.

~~~objective-c
CBUUID *myCustomServiceUUID =
        [CBUUID UUIDWithString:@"71DA3FD1-7E10-41C1-B16F-4430B506CDE7"];
~~~

### Build Your Tree of Services and Characteristics

services와 characteristics의 UUID를 얻었다면, mutable한 services와 characteristics을 만들 수 있다. 이를 통해 트리 구조의 peripheral을 구성 할 수 있다. 예를들어 characteristic UUID를 만들었다면, [CBMutableCharacteristic](https://developer.apple.com/documentation/corebluetooth/cbmutablecharacteristic) 메소드의 [initWithType:properties:value:permissions:](https://developer.apple.com/documentation/corebluetooth/cbmutablecharacteristic/1519073-init) 를 호출하여 mutable characteristic을 만들 수 있다.

~~~objective-c
myCharacteristic =
        [[CBMutableCharacteristic alloc] initWithType:myCharacteristicUUID
         properties:CBCharacteristicPropertyRead
         value:myValue permissions:CBAttributePermissionsReadable];
~~~

mutable characteristic을 만들었다면, 그것의 properties와 value 그리고 권한을 설정 해야한다. 설정한 properties와 permission에 따라 characteristic의 value는 readble할 지, writable할 지, value를 구독할 수 있는지 결정 된다. 

예를 들면 연결된 central에 의해 characteristic의 value가 읽을 수 있는지 결정 된다. 

> Characteristic의 value를 지정하면, value가 캐시되고 해당 properties와 permission이 읽을 수 있게 설정 된다. 따라서 characteristic의 값을 쓸 수 있다. 혹은 characteristic 속해있는 service의 lifetime 동안 값이 변경 될 것이 예상 된다면, nil로 지정해야 한다. 그러면 characteristic의 값이 동적으로 변하고, 연결된 cental로 부터 write나 read 같은 동작이 요청 된다는 것을 보장한다.

위에서 mutable characteristic를 생성해보았다. 이젠 mutable service을 생성해보자. [CBMutableService](https://developer.apple.com/documentation/corebluetooth/cbmutableservice) 클래스의 [initWithType:primary:](https://developer.apple.com/documentation/corebluetooth/cbmutableservice/1434330-init) 메소드를  호출해야한다. 

~~~objective-c
    myService = [[CBMutableService alloc] initWithType:myServiceUUID primary:YES];
~~~

Primary 파라미터에 YES로 지정했는데, 이는 보조 Service가 아닌 메인 service임을 뜻한다. 메인(primary) service는 메인(primary) 기능을 설명한다. 그리고 다른 service의 참조 값을 가지고 있을 수 있다. 보조 Service는 참조 되어 있는 다른 service에 대해서만 설명한다. 

예를 들면, 심박 측정기의 메인 service는 심박 센서를 통해 심박률을 나타내주지만, 보조 서비스는 센서의 배터리 정보를 보여준다. 

service를 만들었다면, 만들어뒀던 characteristic을 조합해야한다. 

~~~objective-c
    myService.characteristics = @[myCharacteristic];
~~~

<br/>

## Publishing Your Services and Characteristics

services와 characteristics의 구성을 마쳤다면, peripheral 역할을 하고 있는 device의 services와 characteristics을 publish 해보도록 하자. Core Bluetooth Framework를 이용한다면 쉽게 할 수 있다. 아래처럼 [CBPeripheralManager](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager) 클래스의 [addService:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393255-add) 메서드를 호출해보자.

~~~objective-c
    [myPeripheralManager addService:myService];
~~~

위와 같은 코드는 myPeripheralManager가 myService를 publish하는 것이다. 이를 통해 [peripheralManager:didAddService:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393279-peripheralmanager) delegate 메서드가 실행된다. 만약 에러가 발생한다면, publish가 되지 않으니 delegate를 구현하여 error를 살펴 볼 수도 있다.

~~~objective-c
- (void)peripheralManager:(CBPeripheralManager *)peripheral
            didAddService:(CBService *)service
                    error:(NSError *)error {
 
    if (error) {
        NSLog(@"Error publishing service: %@", [error localizedDescription]);
    }
    ...
~~~

> Service 및 이와 관련 characteristics을 publish하면, service가 캐시되어 더 이상 변경 할 수 없다. 

<br/>

## Advertising Your Services

services와 characteristics의 publish를 모두 마쳤다면, listening하고있는 central에게 광고할 준비가 되었다는 것이다. 아래의 코드는 [CBPeripheralManager](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager) 클래스의 [startAdvertising:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393252-startadvertising) 메서드를 호출한 후 광고 할 data들을 담아서 service를 광고 할 수 있다. 

~~~objective-c
    [myPeripheralManager startAdvertising:@{ CBAdvertisementDataServiceUUIDsKey :
        @[myFirstService.UUID, mySecondService.UUID] }];
~~~

위 코드에서 사용 된 [CBAdvertisementDataServiceUUIDsKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdataserviceuuidskey) 는 광고 하고 싶은 service의 CBUUID의 배열을 넘기겠다는 의미다.  Key로 사용되어지는 것은 [CBAdvertisementDataLocalNameKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdatalocalnamekey), [CBAdvertisementDataServiceUUIDsKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdataserviceuuidskey) 두가지 뿐이다. 

local peripheral에서 광고를 시작하게 되면, [peripheralManagerDidStartAdvertising:error:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393321-peripheralmanagerdidstartadverti) delegate 메서드가 호출 된다. 만약 에러가 방출 되면 광고에 실패가 났다는 뜻이기에 해당 delegate를 구현하여 원인을 알아봐야 한다.

~~~objective-c
- (void)peripheralManagerDidStartAdvertising:(CBPeripheralManager *)peripheral
                                       error:(NSError *)error {
 
    if (error) {
        NSLog(@"Error advertising: %@", [error localizedDescription]);
    }
    ...
~~~

> Data 광고는 제한 된 장소와 여러 기기가 동시에 광고를 하고 있기에 최선의 노력이 필요하다. 광고 행위는 백그라운드에서도 영향을 받는데 이는 다음 챕터([Core Bluetooth Background Processing for iOS Apps](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html#//apple_ref/doc/uid/TP40013257-CH7-SW1))에서 알아보자.

peripheral이 광고를 시작하면, central은 발견할 수 있고 연결 할 수 있다.

<br/>

## Responding to Read and Write Requests from a Central

한개 또는 여러 central과 연결이 되었다면, 그것들로 부터 read나 write와 같은 요청을 받을 수 있다. 아래 예제는 이러한 요청들을 처리하는 거에 대해 말해주고 있다.

characteristics중 하나의 value에 대해 **read** 요청을 받았다면, peripheral manager는 [peripheralManager:didReceiveReadRequest:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393257-peripheralmanager) delegate 메서드를 호출한다. 이 메서드는 [CBATTRequest](https://developer.apple.com/documentation/corebluetooth/cbattrequest) 객체를 통해 request를 전달한다. 이 객체는 요청을 수행 할 수 있는 여러가지 property들을 담고 있다. 

예를 들면, characteristic value를 읽기 위한 요청이 들어왔다고 가정해보자. 그렇다면 delegate 메서드를 통해 받은 [CBATTRequest](https://developer.apple.com/documentation/corebluetooth/cbattrequest) 객체의 property를 통해 central이 request 보낸 characteristic UUID와 너가 갖고 있는 characteristic UUID를 대조 할 수 있다. 

~~~objective-c
- (void)peripheralManager:(CBPeripheralManager *)peripheral
    didReceiveReadRequest:(CBATTRequest *)request {
 
    if ([request.characteristic.UUID isEqual:myCharacteristic.UUID]) {
        ...
~~~

다음으로 characteristic UUID가 일치한다면, 다음 단계는 읽기 요청이 characteristic의 value 범위를 벗어난 인덱스 위치에서 읽기를 요청하지 않도록 하는 것이다.

~~~objective-c
    if (request.offset > myCharacteristic.value.length) {
        [myPeripheralManager respondToRequest:request
            withResult:CBATTErrorInvalidOffset];
        return;
    }
~~~

request의 offset이 범위에 벗어나지 않았다고 가정하고 다음으로 넘어가면, request의 characteristic property의 value는 기본적으로 nil이 default다. 그렇기에 요청 받은 value에 너의 local peripheral이 갖고 있는 characteristic의 value를 담아주면 된다. 

~~~objective-c
    request.value = [myCharacteristic.value
        subdataWithRange:NSMakeRange(request.offset,
        myCharacteristic.value.length - request.offset)];
~~~

value 값을 세팅 했다면, remote central에 데이터를 잘 담았다고 응답해야한다. 위에서 담은 value가 있는 request와 요청에 대한 성공 여부를 [CBPeripheralManager](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager) 클래스의 메서드 [respondToRequest:withResult:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393293-respond) 에 담아서 호출한다.

~~~objective-c
    [myPeripheralManager respondToRequest:request withResult:CBATTErrorSuccess];
    ...
~~~

[peripheralManager:didReceiveReadRequest:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393257-peripheralmanager) 메서드가 호출 할 때, [respondToRequest:withResult:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393293-respond) 메서드를 한번 씩만 호출한다. 

> 만약 characteristic UUID가 매치되지 않거나 혹은 어떠한 이유로 read에 실패 했다면, request에 value를 담을 필요가 없다. 대신에 [respondToRequest:withResult:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393293-respond) 메서드를 즉시 호출하여 실패 이유를 알려야 한다. 가능하면 [CBATTError Constants](https://developer.apple.com/documentation/corebluetooth/cbatterror) 를 통해 실패의 원인을 제공하는 것도 좋다. 

central로 부터 **write** 요청에 대해 다루는 것은 쉽다. central이 너의 characteristic에 write 요청을 보냈다면, peripheral manager는 [peripheralManager:didReceiveWriteRequests:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393315-peripheralmanager) delegate 메서드를 호출한다. 이때 delegate는 write 요청에 대한 [CBATTRequest](https://developer.apple.com/documentation/corebluetooth/cbattrequest) 객체가 담긴 배열을 전달한다. 그리고 나면 request value를 받을 수 있다. 

~~~objective-c
    myCharacteristic.value = request.value;
~~~

위의 코드에서는 offset이 범위에 벗어나는지에 대한 코드는 작성하지 않았지만, characteristic value를 write 할 때는 고려해야한다. 

read 요청에 대해 응답하려면 [respondToRequest:withResult:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393293-respond) 메서드를 호출 하면 된다. 단 read request때와 마찬가지로 [peripheralManager:didReceiveWriteRequests:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393315-peripheralmanager) 메서드가 호출 될 때 한번씩만 호출해야 한다. 해당 메서드를 호출 할 때, 첫번째 파라미터에 [CBATTRequest](https://developer.apple.com/documentation/corebluetooth/cbattrequest) 객체를 한개만 넘겨야한다. 하지만 didReceiveWriteRequest delegate메서드를 통해 여러개의 request 객체를 받았겠지만, 그 중 첫번째 객체만 넘기면 된다.

~~~objective-c
    [myPeripheralManager respondToRequest:[requests objectAtIndex:0]
        withResult:CBATTErrorSuccess];
~~~

<br/>

## Sending Updated Characteristic Values to Subscribed Centrals

이전 [Subscribing to a Characteristic’s Value](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/PerformingCommonCentralRoleTasks/PerformingCommonCentralRoleTasks.html#//apple_ref/doc/uid/TP40013257-CH3-SW16)에서 말했듯이 central은 characteristic value들을 한개 이상씩 구독 할 수 있다. 그들이 구독한 characteristic value가 값이 변한다면, 알려야 할 필요가 있다. 밑에서 조금 더 살펴보자.

central이 너의 characteristic value를 구독한다면, peripheral manager는 [peripheralManager:central:didSubscribeToCharacteristic:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393261-peripheralmanager) delegate 메서드를 호출한다. 

~~~objective-c
- (void)peripheralManager:(CBPeripheralManager *)peripheral
                  central:(CBCentral *)central
didSubscribeToCharacteristic:(CBCharacteristic *)characteristic {
 
    NSLog(@"Central subscribed to characteristic %@", characteristic);
    ...
~~~

위의 delegate 메서드를 통해 변경 된 값을 central에 보낼 수 있다.

[updateValue:forCharacteristic:onSubscribedCentrals:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393281-updatevalue) 메서드를 호출하여 central에 변경 된 value를 전송 할 수 있다. 

~~~objective-c
    NSData *updatedValue = // fetch the characteristic's new value
    BOOL didSendValue = [myPeripheralManager updateValue:updatedValue
        forCharacteristic:characteristic onSubscribedCentrals:nil];
~~~

위의 메서드를 호출하여 구독한 central에 변경 된 값을 보낸다면, 너는 마지막 파라미터에 어떤 central에 넘길지 명시해야 한다. 하지만 위의 코드에서는 nil을 넘기는데, 이는 해당 peripheral에 구독한 모든 central에 값을 넘기는 것이다. [updateValue:forCharacteristic:onSubscribedCentrals:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393281-updatevalue) 메서드는 Bool값을 리턴하는데 이는 성공적으로 값을 넘겼는지에 대한 값이다. 만약 전송하고자 하는 값의 queue가 모두 찼다면 False를 리턴한다. 그러면 peripheral manager는 전송 큐가 여유가 생겨 사용 가능해지면 [peripheralManagerIsReadyToUpdateSubscribers:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanagerdelegate/1393248-peripheralmanagerisreadytoupdate) delegate 메서드를 호출한다. 그때 [updateValue:forCharacteristic:onSubscribedCentrals:](https://developer.apple.com/documentation/corebluetooth/cbperipheralmanager/1393281-updatevalue) 메서드를 다시 사용하여 값을 다시 전송하면 된다. 

