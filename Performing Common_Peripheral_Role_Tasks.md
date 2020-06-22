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

> Characteristic의 value를 지정하면, value가 캐시되고 해당 properties와 permission이 읽을 수 있게 설정 된다. 따라서 characteristic의 값을 쓸 수 있거나 characteristic 속해있는 service의 lifetime 동안 값이 변경 될 것이 예상 된다면, nil로 지정해야 한다. 그러면 characteristic의 값이 동적으로 변하고, 연결된 cental로 부터 write나 read 같은 동작이 요청 된다는 것을 보장한다.

위에서 mutable characteristic를 생성해보았다. 이젠 mutable service을 생성해보자. [CBMutableService](https://developer.apple.com/documentation/corebluetooth/cbmutableservice) 클래스의 [initWithType:primary:](https://developer.apple.com/documentation/corebluetooth/cbmutableservice/1434330-init) 메소드를  호출해야한다. 

~~~objective-c
    myService = [[CBMutableService alloc] initWithType:myServiceUUID primary:YES];
~~~

Primary 파라미터에 YES로 지정했는데, 이는 