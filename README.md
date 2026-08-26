# Non-blocking-TCP-client-server-communication-between-two-ABB-AC500-V3-PLCs-over-Ethernet
This project demonstrates reliable Ethernet-based TCP/IP socket communication between two ABB AC500 V3 PLCs.  One PLC operates as a TCP client and the second PLC operates as a TCP server. The project provides a practical industrial example of direct, non-blocking TCP client/server communication over standard ethernet. 

Applications were developped in ABB Automation Builder 2.9, which can be downloaded for free from the official site given below.
```iecst
https://www.abb.com/global/en/areas/motion/digital-tools/automation-builder/software-download
```
In the following TCP-Client application, the instructions are given:

--> build socket server address:
```iecst
SOCKETADDRESS.sin_family := syssocket.GVL.SOCKET_AF_INET;
SOCKETADDRESS.sin_port   := syssocket.SysSockHtons(usHost := wPort);
eInetAddr                := syssocket.SysSockInetAddr(
    szIPAddress := ServerIP,
    pInAddr     := ADR(SOCKETADDRESS.sin_addr)
);
```
, then success if returns 0.
--> Create non-blocking socket meaning, if no data is received, socket is kept open:
```iecst
hSocket := syssocket.SysSockCreate(
	iAddressFamily := syssocket.GVL.SOCKET_AF_INET,
	diType     	   := syssocket.GVL.SOCKET_STREAM,
	diProtocol 	   := syssocket.GVL.SOCKET_IPPROTO_IP,
	pResult    	   := ADR(eCreate)
);					 

IF (eCreate = 0) AND (hSocket <> SysSocket.SysSocket_Interfaces.RTS_INVALID_HANDLE) THEN
	diNonBlocking := 1;
	
	eNonBlocking := syssocket.SysSockIoctl(
		hSocket := hSocket,
		diCommand := syssocket.GVL.SOCKET_FIONBIO,
		pdiParameter := ADR(diNonBlocking)
	);
END_IF
```
--> Connect to the socket:
```iecst
eConnect := syssocket.SysSockConnect(hSocket        := hSocket,
									 pSockAddr      := ADR(SOCKETADDRESS),
									 diSockAddrSize := SIZEOF(SOCKETADDRESS));
```
--> Send data to server socket:
```iecst
xiSent := syssocket.SysSockSend(
    hSocket       := hSocket,
    pbyBuffer     := ADR(Data) + udiSendOffset,
    diBufferSize  := TO_DINT(udiTelegramSize - udiSendOffset),
    diFlags       := syssocket.GVL.SOCKET_MSG_DONTWAIT,
    pResult       := ADR(eSend)
);
```

The TCP-Server application follows its given instructions as given shortly-below:

--> Create listening socket and create it non-blocking:
```iecst
hListen := syssocket.SysSockCreate(
    iAddressFamily := syssocket.GVL.SOCKET_AF_INET,
    diType 		   := syssocket.GVL.SOCKET_STREAM,
    diProtocol 	   := syssocket.GVL.SOCKET_IPPROTO_IP,
    pResult 	   := ADR(eCreate)
);

diNonBlocking := 1;

eNonBlocking := SysSocket.SysSockIoctl(
    hSocket   := hListen,
    diCommand := syssocket.GVL.SOCKET_FIONBIO,
    pdiParameter := ADR(diNonBlocking)
				);
```
--> Build local address and assign it to socket level:
```iecst
wPort := 5000;

LocalAddress.sin_family      := syssocket.GVL.SOCKET_AF_INET;
LocalAddress.sin_port        := syssocket.SysSockHtons(usHost := wPort);
LocalAddress.sin_addr.ulAddr := 0;

eBind := syssocket.SysSockBind(
            hSocket   	   := hListen,
            pSockAddr 	   := ADR(LocalAddress),
            diSockAddrSize := SIZEOF(LocalAddress)
);
```
--> Create listening handle:
```iecst
eListen := syssocket.SysSockListen(
            hSocket 		 := hListen,
            diMaxConnections := 1
		);
```
--> Accept incoming client request
```iecst
hClient := syssocket.SysSockAccept(
    hSocket 		:= hListen,
    pSockAddr 		:= ADR(ClientAddress),
    pdiSockAddrSize := ADR(diClientAddrSize),
    pResult 		:= ADR(eAccept)
);
```
--> Receive incoming data:
```iecst
xiRecv := syssocket.SysSockRecv(
	hSocket   	 := hClient,
	pbyBuffer 	 := ADR(ArrayRecv) + udiRecvOffset,
	diBufferSize := TO_DINT(udiTelegramSize - udiRecvOffset),
	diFlags 	 := 0,
	pResult 	 := ADR(eRecv)
);
```
--> Close accepted client when commanded to:
```iecst
eCloseClient := syssocket.SysSockClose(
            	hSocket := hClient
            );
```

# Conclusion
In the end, both applications together formed a strong-basis of industrial-PLC communication by implementing:
1) connection timeout and polling
2) non-blocking Accept/Send/Receive TCP connection
3) partial-transfer handling
4) automatic reconnect/retry
5) disconnect detection
6) socket lifecycle cleanup on Stop/Reset/Download/Online Change

I would describe the project as a robust reusable industrial TCP transport layer, but not yet a fully fault-tolerant application protocol: it still lacks features such as Application-level acknowledgments (ACKs), sequence numbers/duplicate detection, heartbeat/keepalive supervision, and guaranteed end-to-end delivery confirmation.
