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

당신의 앱이 백그라운드에서도 블루투스와 관련된 일을 수행하고 싶다면, `Info.plist` 에서 지원한다는 것을 명시해야한다. 그러면 suspend 상태에서 블루투스 이벤트가 발생하면 시스템에서 앱에게 전달해준다. 이러한 지원은 BLE 통신에서 굉장히 중요하다. 예를 들면 심박수 앱처럼 정기적인 데이터 전달을 위해서 말이다.

블루투스 백그라운드 모드는 명시해야 할 것이 두가지다. 하나는 Central 모드, 나머지는 Peripheral 모드다. 만약 두가지 모두 구현했다면, 백그라운드에서 두가지를 모두 지원한다고 명시해야 한다. `Info.plish` 에서 블루투스 백그라운드 지원을 명시하려면 [UIBackgroundModes](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/plist/info/UIBackgroundModes) 를 이용하여 아래의 키에서 해당되는 것을 사용하면 된다.

- bluetooth-central: CoreBluetooth 프레임워크를 사용하여, Peripheral과 통신한다. 
- bluetooth-central: Core Bluetooth 프레임워크를 사용하여, 데이터를 앱과 공유한다. 

### The bluetooth-central Background Execution Mode

Central을 구현하여 bluetooth-central 키를 이용하여`Info.plist` 에 명시했다면, Core bluetooth 프레임워크는 백그라운드에서도 수행 될 수 있다. 즉, 백그라운드에서 여전히 peripheral과 discover, connect, explore, interact이 가능하다. 게다가 [CBCentralManagerDelegate](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate) 혹은 [CBPeripheralDelegate](https://developer.apple.com/documentation/corebluetooth/cbperipheraldelegate) delegate 메서드를 통해 알림을 받아, central 이벤트를 받을 수 있다. 예를 들면, 연결에 성공 됐거나 끊기거나 혹은 peripheral이 characteristic value를 변경하거나 central 상태가 변경 됐을 때 말이다.

앱이 백그라운드에 있는 동안 많은 블루투스 일을 수행 할 수 있지만, peripheral을 검색 할 때는 포어그라운드에 있을 때와 다르게 동작 한다는 점을 유의해야한다. 

- [CBCentralManagerScanOptionAllowDuplicatesKey](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerscanoptionallowduplicateskey) 스캔 옵션 키는 무시되어지고, 광고하고 있는 여러가지  peripheral을 발견한다 해도, 하나의 단일 이벤트로 통합된다. 
- 만약 백그라운드에서 peripheral을 모두 스캔하면, Central이 광고 된 패킷을 검색하는 간격이 늘어난다. 결과적으로 광고하고 있는 peripheral의 발견하는 시간이 더 길어질 것이다.

이러한 사항들은 전파 사용을 최소화하고, 배터리 수명을 향상 시키는데 도움이 된다.

### The bluetooth-peripheral Background Execution Mode

백그라운드에서 peripheral로서 수행하기 위해서는 `Info.plist` 에서 bluetooth-peripheral을 사용해야한다. `Info.plist` 에 이를 명시하면 시스템은 read, write, subscription 이벤트를 알려준다. 

추가로 CoreBluetooth 프레임워크는 백그라운드에서도 자신 peripheral을 광고 할 수 있도록 해준다. 하지만 알아야 할 것은 본인을 광고 할 때, 포어그라운드 있을 때와 다르게 백그라운드에서 동작한다는 것을 알아야한다. 

- [CBAdvertisementDataLocalNameKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdatalocalnamekey) 광고 키는 사용되지 않고, peripheral의 로컬 이름은 광고 되지 않는다.
- [CBAdvertisementDataServiceUUIDsKey](https://developer.apple.com/documentation/corebluetooth/cbadvertisementdataserviceuuidskey) 를 갖고 있는 모든 service UUID들은 `overflow` 영역에 위치해있다. 명확하게 검색하는 iOS 디바이스에서만 찾을 수 있다.
- 만약 백그라운드에서 모든 앱이 광고를 한다면, peripheral이 광고 패킷을 보내는 빈도가 현저하게 줄어들 것이다.

## Use Background Execution Modes Wisely

비록 백그라운드 지원을 위해 백그라운드 실행 모드를 명시했다 하더라도, 특이 케이스를 사용하기 위해서는 백그라운드 절차를 수행해줘야 한다. 많은 블루투스 관련 작업을 수행하기 위해서는 iOS 기기의 무선 사용을 적극적으로 사용해야하는데 이러한 처리는 배터리 수명에 부정적인 영향을 끼친다. 그렇기에 백그라운드에서 수행하는 작업량을 최소한으로 해야한다. 블루투스 관련 작업은 최대한 빠르게 처리하고 가능한 빨리 suspend로 다시 돌아가야 한다. 

어떠한 앱들은 CoreBluetooth 백그라운드 수행을 위해 아래의 몇가지 가이드 라인을 명시한다. 

- 앱은 세션 기반이어야 하며, 블루투스 관련 이벤트 전달을 시작하고 중지할 시기를 결정할 수 있는 인터페이스를 유저에게 제공해야한다.  
- 앱이 깨어난 후 작업을 완료하는데 10초 정도 걸린다. 이상적으로는 가능한 빨리 작업을 수행하고, 다시 suspend 되도록 해야한다. 백그라운드에서 실행하는데 너무 많은 시간을 사용하는 앱은 시스템에 의해 제한되거나 종료 될 수 있다.
- 시스템이 앱을 깨운 이유와 관련 없는 작업을 수행하지 않도록 해야한다. 

## Performing Long-Term Actions in the Background

몇몇 앱들은 백그라운드에서도 Core bluetooth 프레임워크를 이용하여 오랫동안 작업을 수행해야 할 수도 있다. 블루투스로 동작하는 집 보안 앱을 예시로 들어보자면, 앱이 백그라운드에 있는 동안 사용자가 집을 떠나면 문을 자동적으로 잠궈야 하고 사용자가 돌아오면 문을 열어야 한다. 이를 위해 앱과 잠금장치는 상호작용을 해야한다. 사용자가 집을 나갈 때, iOS기기는 잠금 범위를 벗어나 잠금에 대한 연결이 끊어질 수 있다. 이 시점에서 단순히 앱은 connectPeripheral:options 메소드를 호출 할 수 있으며, 연결 요청 시간이 초과되지 않기 때문에 사용자가 집에 돌아 올 때, iOS 장치가 다시 연결 된다. 

그런데, 사용자가 몇일 동안 집을 떠나있는다고 가정해보자. 만약 사용자가 집을 나왔을 때, 앱이 시스템에 의해 정지가 된다면 유저가 집으로 돌아온다 해도 앱이 다시 연결되진 않을 것이고 문을 열 수 없게 될 것이다. 이러한 경우를 위해 Core bluetooth를 계속 사용하여 활성 및 보류중인 연결 모니터링과 같은 장기적인 작업을 수행할 수 있어야 한다. 

### State Preservation and Restoration

State 유지와 복구가 Core bluetooth에 내장되어 있기 때문에, 앱이 더 이상 실행되지 않아도 시스템에 앱의 Central과 peripheral 매니저의 상태를 유지하고 특정 블루투스 관련 작업을 수행하도록 요청 할 수 있다. 하나의 관련 작업을 완료 했다면, 백그라운드에 있는 앱을 재실행하여 이벤트를 다루기 위해 state를 복구할 기회를 준다. 위에서 설명한 집 보안 관련 앱의 경우로 설명하자면, 시스템은 connection 요청을 모니터링 할 것이고 사용자가 집으로 돌아오고 connection 작업이 완료 됐을 때 [centralManager:didConnectPeripheral:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518969-centralmanager) delegate가 호출되어져 앱이 재실행 될 것이다. 

Core bluetooth 프레임워크는 Central과 Peripheral의 상태 관리를 지원한다. Central 앱을 구현했고 state 보존과 복구를 지원한다면, 앱이 메모리 이슈로 중지 될 때 시스템은 Central 매니저 객체의 state를 저장한다. (만약 여러개의 Central 매니저를 가지고 있다면, 한가지의 매니저만 선택 할 수 있다.)  특히 [CBCentralManager](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager) 객체는 아래의 절차를 따른다.

- Central 매니저의 Service는 