[개요] 2024.07~2024.10 / 방제기 통신 모듈과 485 통신으로 상태 조회·분사 제어.

[사용기술]
 - Java, Android (minSdk 24, compileSdk/targetSdk 34)
 - 시리얼 통신: android_serialport_api + JNI(libSerialPort.so, armeabi-v7a /
   arm64-v8a / x86 / x86_64 포함), 장치 /dev/ttyS1, 19200 baud
 - 아키텍처: ViewModel + LiveData, ViewPager2 + Fragment 3개, Dialog 기반 설정 화면
 - 로깅: orhanobut Logger + 파일 로그(/cleaner/Log.txt)
 - 설정 저장: SharedPreferences(PrefsHelper/KeyControl)
 - 단말 운영: BOOT_COMPLETED 수신 시 자동 실행, DeviceAdminReceiver

[주요기능]
 - 하우스(01~04)별 온도·습도 조회, 정보 요청
 - 방제 분사: 하우스·노즐(최대 8개 구역) 선택, 분사 강도(노즐 단계) 조절
 - 컴프레서/펌프 시작·정지, 물청소·에어청소 동작
 - 사이클 반복(반복 횟수, 반복 휴지시간), 시작/종료 시간 예약, 정시 실행
 - 물 부족(Stortage_Water) 감지, 전체 정지(House_All_Stop)
 - 온습도 기준값 설정, 시간 설정, 노즐 설정, 조정(Adjustment) 다이얼로그

[핵심 시스템 구조와 구현 방식]
 - 프레임 프로토콜: STX(0x02) + 명령코드 + 길이 + 데이터 + CRC16 + ETX(0x03).
   CRC16(CRC-CCITT, 다항식 0x1021, 초기값 0xFFFF)은 CRC16Util로 계산.
 - SerialPortManager(싱글톤): 포트 열기/닫기, 송신(sendBytes, 지연 전송), 수신 스레드
   (SerialPortReadThread)와 리스너 콜백. SerialModel에 명령/응답 코드를 상수화
   (예: 하우스 01·02 컴프 시작/정지, 03·04 물청소, 에어청소 등 송신-응답 쌍).
 - 수신 파싱(SerialDataUtil): 헤더 확인 → 고정 길이 패킷 조립(분할 수신 시 캐시에
   이어붙임, 타임아웃 시 폐기) → CRC 검증 후 반환.
 - MainActivity는 Timer 여러 개(정보요청, 온습도, 컴프/펌프/청소 단계, 시작시간)로
   동작 시퀀스를 진행하고, 결과를 MainViewModel의 LiveData(진행상황, 반복 주기,
   하우스별 온·습도 등)로 UI에 반영.

[데이터 흐름]
 사용자 설정/예약(Dialog, SharedPreferences) → MainActivity 타이머 시퀀스
   → SerialPortManager.SendSerialCom(명령 + CRC) → RS-485 방제기 컨트롤러
 컨트롤러 응답 → 수신 스레드 → SerialDataUtil(조립·CRC 검증) → SerialModel 코드 분기
   → MainViewModel(LiveData) → 화면 / 파일 로그(LogFile)
