# Introduction

CoreBluetooth framework는 iOS와 Mac이 블루투스 장비와 통신하기 위한 Class들을 제공해준다. 이를 통해 너의 앱은 peripheral devices를 발견, 탐구, 상호작용이 가능하다. 

<img src="./img/introduction.png"/>

#### At a Glance

Bluetooth low energy wireless technology은 BLE 4.0 기반이다. Low-Energy들과 통신하도록 프로토콜이 정의되어 있다. The Core Bluetooth framework는 Bluetooth low energy protocol stack을 추상화했다. 개발자들이 블루투스 기기와 통신을 더욱 쉽게 하도록 말이다.

### Centrals and Peripherals Are the Key Players in Core Bluetooth

Bluetooth low energy 통신에서는 **central**과 **peripheral** 이 두가지가 핵심이다. 서로 다른 역할을 가지고 있는데, peripheral은 일반적으로 다른 device들이 필요로 하는 데이터들을 가지고 있다. central은 peripheral이 제공해주는 정보들을 some task들을 수행하기 위해 사용한다.  

예를 들면, 블루투스 디지털 온도계는 iOS앱에 방의 온도를 전달해준다. 

각각 역할을 수행할 때, task 설정을 다르게 한다. Peripherals은 무선으로 데이터를 알려 존재를 알린다. Centrals은 근처에 찾고자하는 Peripherals를 찾는다. Centrals이 원하는 Peripheral를 발견하면, connect를 시도한다. 그리고 Peripheral의 데이터를 exploring하고 interacting하기 시작한다. 

**Relevalnt Chapter: [Core Bluetooth Overview](<https://github.com/wjdgo813/About_CoreBluetooth/blob/master/CoreBluetooth_Overview.md>)**

### Core Bluetooth Simplifies Common Bluetooth Tasks

Core Bluetooth framework은 Bluetooth 4.0의 구체화 된 내용을 추상화 하여 제공한다. 만약 central 역할 하는 앱을 만든다면, Core Bluetooth는 Peripheral들을 발견하고 연결하는데 쉽게 만들어 준다. 또한 Peripheral을 explore하고 통신하는데 도와준다. 거기에 CoreBluetooth는 peripheral의 역할을 하는데에도 도와준다.

**Relevant Chapters:** [Performing Common Central Role Tasks]()









