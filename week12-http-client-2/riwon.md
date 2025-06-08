## TCP 3-Way Handshake

### TCP & 3-Way Handshake

> Transmission Control Protocol

인터넷 상에서 데이터를 메세지의 형태로 보내기 위해 IP와 함께 사용하는 프로토콜을 말합니다.
다시 말해 서버와 클라이언트 간 데이터를 신뢰성 있게 전달하기 위해 만들어졌으며, 논리적인 접속을 성립하기 위해서 '3-Way Handshake'를 사용합니다.

> 3-Way Handshake

데이터 전송이 시작되기 전, 클라이언트와 서버 간에 신뢰할 수 있는 연결을 설정하기 위해 TCP(전송 제어 프로토콜)에서 사용되는 기본적인 과정입니다.
이를 통해 클라이언트와 서버 양쪽이 서로 동기화 되었고 통신할 준비가 되었음을 보장하게 됩니다.<br>
즉, 두 장치 간에 신뢰할 수 있는 연결을 설정하는 데 사용되는 기본적인 과정이며, 이러한 과정은 세 단계로 이루어져 있습니다.
SYN(연결 설정), SYN-ACK(연결 설정-응답 확인), ACK(응답 확인) 단계를 거치며 클라이언트와 서버는 서로 연결이 성공적으로 설정되었음을 확인합니다.

### TCP Flag Information

> Flag Bit

`TCP Header`에는 CONTROL BIT 혹은 FLAG BIT가 존재합니다. 총 6비트로 구성되며, 각각의 비트는 `URG/ACK/PSH/RST/SYN/FIN`의 의미를 가집니다.
특정 위치의 비트가 1로 설정됨에 따라, 해당 패킷이 어떠한 내용을 담고 있는 패킷인지를 표현할 수 있게 됩니다.

3-Way Handshake에서는 SYN과 ACK 플래그가 사용되며, 4-Way Handshake의 경우 FIN 플래그가 함께 사용됩니다.

> SYN(Synchronize Sequence Number)

연결 설정. Connection을 생성할때 사용하는 flag입니다. 
Sequence Number를 랜덤으로 설정하여 세션을 연결하는 데 사용하며, 초기에 Sequence Number를 전송합니다.

> ACK(Acknowledgement)

응답 확인. 패킷을 받았다는 것을 의미하는 flag입니다. 
Acknowledgement Number 필드가 유효한지를 나타냅니다.<br>
만약 프로세스가 쉬지 않고 데이터를 전송한다고 가정하면, 최초의 연결을 설정하는 과정에서 전송되는 첫 번째 세그먼트를 제한 
모든 세그먼트의 ACK 비트는 1로 지정된다고 볼 수 있습니다.

> FIN(Finish)

연결 해제. 세션 연결을 종료시킬 때 사용되며, 더 이상 전송할 데이터가 없음을 의미합니다. `4-Way Handshake`에서 사용합니다.

참고로 RST, URG, PSH 플래그는 아래와 같은 의미로 사용됩니다.

- Reset (RST):
TCP 연결에 문제가 있거나, 해당 통신이 존재해서는 안 된다고 판단되는 경우 연결을 즉시 종료하기 위해 사용됩니다.
예를 들어, 예상치 못한 패킷이 특정 호스트로 전송되었을 때, 수신 측에서 RST를 보낼 수 있습니다.

- Urgent (URG):
패킷에 포함된 데이터가 긴급하게 처리되어야 함을 나타냅니다. 
이 플래그는 긴급 포인터(Urgent Pointer) 필드와 함께 사용되어, 패킷 내에서 긴급 데이터의 위치를 지정합니다.

- Push (PSH):
송신 측에서 데이터를 버퍼에 더 쌓지 않고 즉시 수신 측에 전달하라고 요청할 때 사용됩니다. 
이 플래그는 일반적으로 실시간 오디오나 영상 스트리밍과 같은 애플리케이션에서 사용됩니다.

### 3-Way Handshake

> 1단계 : Client -> SYN -> Server

클라이언트가 서버에게 접속을 요청하는 SYN 플래그를 보냅니다.

> 2단계 : Server -> SYN + ACK -> Client

서버는 Listen 상태에서 SYN이 들어온 것을 확인하고, Syn-Received 상태로 바뀌어 SYN + ACK 플래그를 클라이언트에게 전송합니다.
그 후 서버는 다시 ACK 플래그를 받기 위해 대기 상태로 변경됩니다.

> 3단계 : Client -> ACK -> Server

SYN + ACK 상태를 확인한 클라이언트는 서버에게 ACK를 보내며, 최종적으로 연결이 성립하여 Established 상태가 됩니다.

### 4-Way Handshake

3-Way Handshake가 연결 확립을 위해 진행되는 절차라고 볼 수 있다면, 4-Way Handshake는 연결을 종료하기 위해 수행되는 절차를 의미합니다.

> 1단계 : Client -> FIN -> Server

우선 클라이언트가 연결을 종료하겠다는 FIN 플래그를 전송합니다. 보낸 후에 클라이언트는 FIN-WAIT-1 상태로 변하게 됩니다.

> 2단계 : Server -> ACK -> Client

클라이언트로부터 FIN 플래그를 받은 서버는 이를 확인하고 ACK를 클라이언트에게 보내주게 됩니다. 그 후 CLOSE-WAIT 상태로 변하게 됩니다. 
마찬가지로 클라이언트도 서버에서 종료될 준비가 됐다는 FIN 플래그를 받기 위해 FIN-WAIT-2 상태가 됩니다.

> 3단계 : Server -> FIN -> Client

Close 준비가 다 된 후 서버는 클라이언트에게 FIN 플래그를 전송하게 됩니다.

> 4단계 : Client -> ACK -> Server

이에 클라이언트는 해지 준비가 되었다는 정상 응답인 ACK를 서버에게 보냅니다. 또한 이때 클라이언트는 TIME-WAIT 상태로 변경되게 됩니다.

### TCP State

|상태| 설명                                                         |
|--|------------------------------------------------------------|
|CLOSED| 포트가 닫힌 상태                                                  |
|LISTEN| 포트가 열린 상태, 연결 요청 대기 중                                      |
|SYN_RECV| SYN 요청을 받고 상대방의 응답을 기다리는 중                                 |
|ESTABLISHED| 포트 연결 상태                                                   |
|TIME-WAIT| 서버로부터 FIN 요청을 받고 일정 시간(기본값은 240초)동안 연결을 남겨서 잉여 패킷을 기다리는 과정 |

- 참고로 TIME-WAIT 상태는 의도치 않은 에러로 인해 연결이 데드락으로 빠지는 것을 방지하기 위해 사용됩니다.
만약 에러로 인해 종료가 지연되고 지정한 시간이 초과되면 CLOSED 상태로 변경된다고 합니다.

### Sequence Number & Acknowledgement Number

> Sequence Number

일련 번호는 송신 측에서 각 세그먼트에 고유하게 부여하는 번호로, 수신 측이 패킷을 올바른 순서로 재조립할 수 있도록 도와줍니다. 
아래의 Acknowledgement Number와 함께 사용되어 신뢰성 있는 데이터 전송을 보장하고, 중복 패킷을 방지하는 데 중요한 역할을 합니다.

> Acknowledgement Number

확인 응답 번호는 TCP 세그먼트를 정상적으로 수신했음을 알리기 위해 사용되며, 수신 측은 해당 번호를 사용해 다음으로 기대하는 시퀀스 번호를 송신 측에게 알려줍니다. 

- 호스트 A가 호스트 B로 Sequence Number 3001번과 함께 100 바이트 데이터를 전송합니다.
- 호스트 B는 100 바이트를 받고, 다음으로 받고자 하는 데이터 번호를 Acknowledgement Number에 넣습니다. (여기서는 3101번이 됩니다.)
- 호스트 A는 그럼 이제 호스트 B에게 Sequence Number 3101번 부터 다시 데이터를 전송하게 됩니다.

### Connection Timeout

서버는 `Connection Timeout`을 설정해서 클라이언트가 리소스를 점유하지 않도록 주의해야 합니다. 
다시 말해 일부 클라이언트에 의해 전체적인 처리 성능이 저하되지 않도록 해야 합니다.

예를 들어, 악의적 공격자가 서버에게 SYN 패킷을 보낸 후, SYN + ACK를 받고 나서 ACK를 보내지 않는 여러 클라이언트를 구동했다면 어떻게 될까요?

이 경우 Timeout이 없다면 서버는 ACK를 기다리는 연결 대기 상태(Syn-Received)로 머물게 되고, 영영 오지 않을 클라이언트의 ACK를 기다리게 될 것입니다.<br>
따라서, 서버는 자신의 리소스가 불필요하게 낭비되는 것을 방지하기 위해 적절한 Connection Timeout 값을 설정해야 합니다.

---
