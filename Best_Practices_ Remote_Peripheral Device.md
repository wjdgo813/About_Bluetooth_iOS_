# Best Practices for Interacting with a Remote Peripheral Device

Core Bluetooth framework은 Central 역할로서 많은 것들을 지원해준다. 즉, Central로서 Peripheral을 discover, 연결, peripheral의 데이터와 통신하도록 지원한다. 이번 챕터는 개발하는데에 있어서 가이드라인을 제공할 것이다. 

## Be Mindful of Radio Usage and Power Consumption

블루투스 통신 관련 개발을 할 때, 기억해야 할 것이 있다. 블루투스 통신은 디바이스의 라디오를 공유하여 무선통신을 한다. 다른 무선 통신들도 기기의 라디오를 이용하기 때문에, 라디오 사용을 최소화하며 개발해야 한다. 

라디오 사용을 최소화 하는 것은 iOS 개발을 하는 것이라면 더욱 중요하다. 라디오 사용은 배터리 수명을 단축 시킬 수 있기 때문이다. 밑에서 설명할 가이드라인은 기기의 라디오를 잘 사용하는데에 도움이 될 것이다. 

### Scan for Devices Only When You Need To

advertising하고 있는 Peripheral를 찾기 위해 [scanForPeripheralsWithServices:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518986-scanforperipheralswithservices) 메서드를 호출하면, Central 기기는 advertising 디바이스를 찾기 위해 명시적으로 정지 시킬 때까지 라디오를 계속 사용한다.  

찾고자 하는 Peripheral 기기를 찾았다면, scanning 과정을 중단해야 한다. [CBCentralManager](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager) 클래스의 [stopScan](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518984-stopscan) 메서드를 호출하면 scan 동작을 중지한다. 

### Specify the CBCentralManagerScanOptionAllowDuplicatesKey Option Only When Necessary

Peripheral은 Central들에게 본인의 존재를 알리기 위해 초당 다수의 advertising 패킷을 보낸다. [scanForPeripheralsWithServices:options:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/1518986-scanforperipheralswithservices) 메서드로 scan을 시작하는데, 이 메서드의 기본 동작은 여러 개의 advertising 데이터들을 하나의 이벤트로 합치는 것이다. 즉, Central 매니저가 [centralManager:didDiscoverPeripheral:advertisementData:RSSI:](https://developer.apple.com/documentation/corebluetooth/cbcentralmanagerdelegate/1518937-centralmanager) 메서드를 호출하면 



