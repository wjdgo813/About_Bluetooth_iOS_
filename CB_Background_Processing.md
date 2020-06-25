# Core Bluetooth Background Processing for iOS Apps

iOS 앱들은 실행되는 것이 백그라운드 상태인지 포어그라운드 상태인지 알아야 한다. 보통, 포어그라운드 보다 백그라운드에서 제약들이 더 많다. 앱 디바이스의 시스템 리소스들이 더욱 제한되어 있기 때문이다. 

일반적으로 Core Bluetooth가 가진 공통 업무들은(peripheral이던 central이던) 백그라운드나 suspended 상태에서 비활성화 되어있다. 그래서 개발자는 백그라운드에서 동작하도록 지원하는 것을 명시해야 한다. 블루투스와 연관된 이벤트가 발생하면 suspend 상태라 하더라도 앱을 깨울 수 있도록 허락해야한다. 앱이 광범위한 백그라운드 업무를 수행하지 않더라도, 중요한 이벤트가 발생할 때 시스템에 알람을 요청 할 수 있다. 

그러나 당신이 백그라운드를 지원하도록 선언하였어도 영원히 백그라운드에서 동작하지는 않는다. 어떤 시점에서 시스템은 현재 포그라운드 메모리를 확보하기 위해 앱을 종료 할 수 있기 때문이다. 

다행히도 iOS7부터 CoreBluetooth는 central과 peripheral state 정보를 저장하고, 앱이 다시 실행 될 때, 그 상태 그대로 복구한다. 이 기능을 통해 블루투스와 관련된 기능을 오랫동안 사용 할 수 있다.

## Foreground-Only Apps

대부분의 앱들은 특정 백그라운드 일을 위한 권한을 요청하지 않는다면, 백그라운드 상태로 진입 후 빠르게 suspend 상태로 진입한다. 그렇게 되면 suspend 상태에서 당신의 앱은 블루투스 관련된 일은 할 수 없고, 다시 포어그라운드로 돌아오지 않는 이상 이벤트도 받을 수 없다. 

foreground만 지원하는 앱의 central은 백그라운드 상태에서 광고하고 있는 peripheral을 scan 할 수도 발견도 할 수 없다. 마찬가지로 Peripheral도 광고는 불가능하고, characteristic value에 접근하려고 하면 에러를 받게 된다. 

사용사례에 따라 당신의 앱의 동작들은 여러가지로 영향을 받을 수 있다. 예를 들면, 연결 되어있는 peripheral의 데이터와 통신하려고 하는데, 그때 너의 앱이 suspend 상태로 들어갔다고(다른 앱을 키게 되서) 가정해보자. 이때 peripheral과의 연결이 끊긴다면, 당신의 앱은 다시 실행 될 때까지 연결이 끊겼다는 사실을 알 수 없다. 

### Take Advantage of Peripheral Connection Options

백그라운드나 suspend 상태로 들어간 상태에서 블루투스와 관련된 이벤트가 발생하면 시스템에 의해 큐에 축적되고, 앱이 포어그라운드로 올라 올 때, 그 이벤트를 전달한다. 즉, Core Bluetooth는 central 이벤트가 발생할 때 유저에게 전달할 방법을 제공한다. 그런 다음 유저는 이런 특정 이벤트를 통해 앱을 포그라운드로 가져와야하는지 여부를 결정한다. 

다음은 [CBCentralManager](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager) 클래스의 메서드인 [connectPeripheral:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518766-connect) 를 peripheral에 연결 할 때, peripheral의 옵션에 포함하여 이러한 알림을 제공 할 수 있다. 

- [CBConnectPeripheralOptionNotifyOnConnectionKey](https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionnotifyonconnectionkey) : 연결이 이미 되어있는 상태에서 앱이 suspend 모드로 진입하면 시스템이 peripheral에 알린다. 
- [CBConnectPeripheralOptionNotifyOnDisconnectionKey](https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionnotifyondisconnectionkey): 앱이 suspend 상태에서 disconnect가 되버리면, peripheral에 disconnect 이벤트를 전달한다.
- [CBConnectPeripheralOptionNotifyOnNotificationKey](https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionnotifyonnotificationkey) : 앱이 suspend 된 상태에서 받은 모든 알림을 peripheral에 전달한다.

<br/>

## Core Bluetooth Background Execution Modes

