# Review motion architecture

> Hello Seulki,
> Please find the example project ​[here](https://sigmatek-my.sharepoint.com/:u:/g/personal/jonas_draeger_sigmatek_at/IQDOCCT022DjTpqfyBTCAZH0AX1i_efF-1Tdcx5silosWX4)​. It is kept relatively simple. Below, I would like to explain the ideas behind the implementation.
> Regarding Motion:
>
> how the TCP communication object, API command handling, motion objects, and axis objects should be separated and connected,
> When issuing Motion commands, it is important to ensure that the call is executed on the same core on which the RtWork is running. If necessary, a buffer or flag mechanism should be used to transfer the command and switch tasks accordingly.
> which functions should be assigned to real-time, cyclic, or background processing,
> The command execution and, more importantly, the monitoring/control should take place in RtWork to keep the latency as low as possible.
> whether commands and responses should be exchanged through an intermediate gateway, queue, shared buffer, or another recommended structure,
> The data exchange can be implemented in whichever way you prefer. However, be careful with data access, as communication may take place between different tasks.
> how position and status data from 8 to 12 axes should be collected and provided efficiently,
> The data can either be read individually via clients or through pointers in a loop. From a timing perspective, there is no difference.
> how to minimize the effect of increasing axis and motion-processing load on TCP response time and jitter,
> The task responsible for TCP communication can be assigned to the other core (by default this is already the case, although it runs on same prio as the background task). This minimizes the impact of motion calculations on communication performance.
> and how the architecture should be designed so that different axes and groups can be selected flexibly without creating unnecessary processing load.
> This depends on whether a profiled motion is required, or whether it is sufficient to define the motion via a master axis and have the other axes follow accordingly. Using a master axis generally provides better performance. With LMCProfile, the axes are already synchronized to the path velocity according to their individual speeds, and a motion buffer is available.
> Regarding TCP Communication:
>
> Is incoming TCP data received and buffered independently by the operating system or runtime, or is it processed only when the LASAL TCP object is executed?
> The TCP communication library classes use their own thread, which is started during Init(). This makes processing independent of the class call itself. However, this thread has a relatively low priority. Performance may potentially be improved by implementing the handling in a dedicated class running in CyWork. RtWork is not a viable option, since retrieving data from Linux does not provide real-time guarantees. In the example project, I have included a UDP network implementation where the data is retrieved directly in CyWork.
> What part of the communication processing is handled by the operating system, the runtime, and the LASAL application respectively?
> In LASAL, the check for available data is triggered and the message acknowledgment is generated. The actual receiving, checking, and sending of data takes place on the Linux side.
> How does the execution cycle assigned to the TCP object affect the actual request-to-response time?
> It cannot be influenced, as this runs in a UserTask with a fixed priority. In contrast, the UDP classes retrieve the data directly in CyWork, meaning that the CyWork cycle time also corresponds to the response time.
> Why does the current TCP object become unstable when assigned to a 1 ms cycle, while it operates at approximately 10 ms?
> I was not able to reproduce this issue. I successfully operated all classes with a 1 ms cycle time on both a CP 312 and a CP 111. RtWork is not suitable because the Linux part does not provide real-time capabilities.
> What architecture does SIGMATEK recommend when TCP communication must interact with motion objects running in a faster real-time cycle?
> General Recommendations:
> Decouple communication and motion execution by running them on different cores. Commands should be transferred to RtWork using a buffer or a similar mechanism to ensure the required core switch.
> Reading data can take place directly within the communication thread. For large amounts of data (such as log files), it is advisable to decouple processing from the TCP thread as well, in order to maintain communication performance.
> Whenever possible, combine data into a single message to keep communication overhead low relative to the actual payload.
>
> Best regards,
>
> M.Sc. Jonas Dräger
>
> Applikation
>
> 메일 원문 번역해주고 , 이친구가 Sigmatek 프로그램으로 뭘하고 싶은건지 분석해봐

<details><summary>1 previous message</summary>

> 메일 내용 자체로도 답할 수 있지만, 이 저장소의 Elmo/SIGMATEK 맥락과 맞물린 요청이라 이전 메모리에서 관련 사실만 빠르게 확인하겠습니다.
</details>

**원문 번역**

Seulki님 안녕하세요.

예제 프로젝트는 여기에서 확인해 주세요. 비교적 단순하게 유지했습니다. 아래에서는 구현의 의도를 설명드리고자 합니다.

**Motion 관련**

TCP 통신 객체, API 명령 처리, 모션 객체, 축 객체를 어떻게 분리하고 연결해야 하는지에 대해:

Motion 명령을 실행할 때는 해당 호출이 `RtWork`가 실행 중인 동일한 코어에서 실행되도록 보장하는 것이 중요합니다. 필요하다면 명령을 전달하고 작업을 전환하기 위해 버퍼나 플래그 메커니즘을 사용해야 합니다.

어떤 기능을 real-time, cyclic, background 처리에 배치해야 하는지에 대해:

명령 실행, 그리고 더 중요하게는 모니터링/제어가 가능한 한 낮은 latency를 유지하도록 `RtWork`에서 수행되어야 합니다.

명령과 응답을 중간 gateway, queue, shared buffer 또는 다른 권장 구조를 통해 교환해야 하는지에 대해:

데이터 교환은 원하는 방식으로 구현해도 됩니다. 다만 통신이 서로 다른 task 사이에서 이루어질 수 있으므로 데이터 접근에 주의해야 합니다.

8~12축의 위치와 상태 데이터를 효율적으로 수집하고 제공하는 방법에 대해:

데이터는 client를 통해 개별적으로 읽어도 되고, pointer를 사용해 loop에서 읽어도 됩니다. 시간 성능 관점에서는 차이가 없습니다.

축 수와 motion 처리 부하가 증가할 때 TCP 응답 시간과 jitter에 미치는 영향을 최소화하는 방법에 대해:

TCP 통신을 담당하는 task를 다른 코어에 할당할 수 있습니다. 기본적으로 이미 그렇게 되어 있지만, background task와 같은 priority로 실행됩니다. 이렇게 하면 motion 계산이 통신 성능에 미치는 영향을 줄일 수 있습니다.

불필요한 처리 부하 없이 여러 축과 group을 유연하게 선택할 수 있도록 architecture를 어떻게 설계해야 하는지에 대해:

이것은 profiled motion이 필요한지, 아니면 master axis를 정의하고 다른 축들이 그 master axis를 따라가게 하는 방식으로 충분한지에 따라 달라집니다. 일반적으로 master axis를 사용하는 방식이 더 나은 성능을 제공합니다. `LMCProfile`을 사용하면 축들이 각자의 속도에 따라 path velocity에 이미 동기화되어 있고, motion buffer도 사용할 수 있습니다.

**TCP Communication 관련**

수신 TCP 데이터는 OS 또는 runtime에 의해 독립적으로 수신/버퍼링되는지, 아니면 LASAL TCP 객체가 실행될 때만 처리되는지에 대해:

TCP 통신 library class들은 `Init()` 중에 시작되는 자체 thread를 사용합니다. 따라서 처리는 class call 자체와 독립적입니다. 다만 이 thread는 priority가 비교적 낮습니다. 성능은 `CyWork`에서 실행되는 전용 class 안에 처리를 구현하면 개선될 가능성이 있습니다. `RtWork`는 적합하지 않습니다. Linux에서 데이터를 가져오는 동작은 real-time 보장을 제공하지 않기 때문입니다. 예제 프로젝트에는 데이터를 `CyWork`에서 직접 가져오는 UDP network 구현을 포함했습니다.

통신 처리 중 OS, runtime, LASAL application이 각각 담당하는 부분에 대해:

LASAL에서는 사용 가능한 데이터가 있는지 확인하는 동작이 trigger되고, message acknowledgment가 생성됩니다. 실제 데이터 수신, 확인, 송신은 Linux 쪽에서 수행됩니다.

TCP 객체에 할당된 execution cycle이 실제 request-to-response 시간에 어떤 영향을 주는지에 대해:

영향을 줄 수 없습니다. 이것은 고정 priority를 가진 `UserTask`에서 실행되기 때문입니다. 반대로 UDP class들은 데이터를 `CyWork`에서 직접 가져오므로, `CyWork` cycle time이 response time에 해당합니다.

현재 TCP 객체가 1 ms cycle에 할당되면 불안정해지고, 약 10 ms에서는 동작하는 이유에 대해:

저는 이 문제를 재현할 수 없었습니다. CP 312와 CP 111 양쪽에서 모든 class를 1 ms cycle time으로 성공적으로 동작시켰습니다. `RtWork`는 적합하지 않습니다. Linux 부분이 real-time capability를 제공하지 않기 때문입니다.

빠른 real-time cycle에서 실행되는 motion object와 TCP 통신이 상호작용해야 할 때 SIGMATEK이 권장하는 architecture에 대해:

일반 권장 사항:

통신과 motion 실행을 분리하고, 서로 다른 코어에서 실행하십시오. 필요한 core switch가 이루어지도록 명령은 buffer 또는 유사한 메커니즘을 사용해 `RtWork`로 전달해야 합니다.

데이터 읽기는 communication thread 안에서 직접 수행할 수 있습니다. log file처럼 많은 양의 데이터를 처리해야 하는 경우에는 통신 성능을 유지하기 위해 그 처리도 TCP thread에서 분리하는 것이 좋습니다.

가능하면 데이터를 하나의 message로 합쳐 실제 payload 대비 통신 overhead가 낮게 유지되도록 하십시오.

감사합니다.

M.Sc. Jonas Dräger
Application

**분석**

확인된 사실 기준으로 말하면, Jonas가 하고 싶은 것은 “SIGMATEK PLC 안에서 TCP 통신과 motion 제어를 한 덩어리로 돌리지 말고, 역할을 분리한 예제 architecture를 보여주는 것”입니다.

핵심은 이겁니다.

1. PC 또는 외부 장비에서 TCP/UDP로 motion API 명령을 보낸다.
2. SIGMATEK 쪽 통신 객체가 명령을 받는다.
3. 통신 task가 직접 motion을 때리는 것이 아니라, buffer/flag/shared data 같은 구조로 `RtWork` 쪽에 명령을 넘긴다.
4. 실제 motion command 실행, 상태 감시, 제어는 `RtWork`에서 한다.
5. 응답에 필요한 위치/status 데이터는 통신 쪽에서 읽어 packet으로 묶어 다시 보낸다.
6. 축이 8~12개로 늘어나도 motion 계산 부하와 TCP 응답 jitter가 서로 덜 간섭하도록 task/core를 분리한다.

즉, 그는 “TCP 객체를 1 ms로 돌리면 실시간 통신이 된다”는 식으로 접근하지 말라고 말하고 있습니다. TCP는 Linux 쪽 처리와 낮은 priority thread가 끼어 있어서 real-time 보장을 못 합니다. 빠르고 결정적인 motion 처리는 `RtWork`로 보내고, 통신은 별도 task/core에서 처리하라는 뜻입니다.

또 하나 중요한 점은, Jonas는 queue나 gateway를 반드시 쓰라고 강제하지 않습니다. 방식은 자유지만, 서로 다른 task 사이에서 데이터를 공유하므로 race condition이나 잘못된 동시 접근을 조심하라는 말입니다.

우리 Elmo/SIGMATEK 맥락에 대입하면, 이 메일은 “PMAS/MMCLib 스타일 TCP API를 SIGMATEK에서 그대로 흉내 내되, 실제 motion 실행은 LASAL `RtWork` 구조에 맞게 분리해서 구현하라”는 방향에 가깝습니다. 특히 TCP response time과 motion latency를 같은 문제로 보지 말라는 게 핵심입니다.

> 지금 분석한 내용을 토대로 C:\work\Elmo\Elmo_Master\MotionTCPDemo 프로젝트를 분석해서 구조를 파악한 결과 파일을 만들어봐. 그리고 우리가 만든 C:\work\Elmo\Elmo_Master\Elmo_EtherCAT_Test_4Axis 프로젝트랑 비교해서 구조가 어떠한 차이가 있는지 분석해봐

<details><summary>17 previous messages</summary>

> 먼저 두 프로젝트의 실제 파일 구조와 이전 Elmo 작업 맥락을 확인하겠습니다. 결과 파일은 repo 안에 남기고, 분석은 메일 내용에서 확인된 구조 원칙과 실제 LASAL 소스/네트워크 파일을 분리해서 보겠습니다.
>
> 파일 목록상 `MotionTCPDemo`는 단순 예제가 아니라 통신 계층(`TCPCommunication`, `_SigTCPComReceive`, `TCPRT_DataManager`, `UDPTransmission`)과 motion 계층(`MotionControlIF`, `TCP_MotionIF_Dummy`)이 분리된 LASAL 프로젝트입니다. 이제 네트워크 연결도와 주요 `.st` 구현부를 직접 대조하겠습니다.
>
> `Elmo_EtherCAT_Test_4Axis` 쪽 기존 규칙 문서도 확인했습니다. 기존 프로젝트는 이미 “TCP callback에서 바로 motion을 오래 기다리지 말고, 수신/큐잉/실행/응답을 분리하라”는 방향으로 문서화되어 있어서, 이번 비교는 그 원칙이 실제 코드에 얼마나 반영됐는지를 기준으로 잡겠습니다.
>
> 중요한 사실이 하나 확인됐습니다. `MotionTCPDemo`의 핵심 사용자 구현은 일부러 비워둔 예제에 가깝고, `TCP_MotionIF_Dummy` 주석에 “BG user thread에서 받은 명령을 직접 처리하거나 RT class로 버퍼 전달하라, LMCAxis 직접 명령은 같은 코어 확인하라”는 Jonas 메일과 같은 지침이 코드로 박혀 있습니다.
>
> `MotionTCPDemo`의 TCP 경로는 `TCPCommunication`이 별도 parallel task를 만들고, `DataManager`가 10단계 priority FIFO로 송신 job을 관리하며, 수신 데이터는 DestinationID 기반 callback으로 해당 receiver에 넘기는 구조입니다. 반대로 `DataManager::SetData`는 RT task 호출을 기본적으로 막고 있어서, 실시간 motion 실행부에서 바로 TCP 송신 queue에 넣는 구조가 아니라는 점도 확인됐습니다.
>
> `MotionTCPDemo`의 TCP 수신은 payload parser 예제까지 완성된 상태가 아닙니다. 구조 예제입니다. `ReceiveMsg`/`TCP_MotionIF_Dummy`의 callback은 데이터를 받을 수 있는 hook만 열어 두고 실제 motion command 해석은 사용자가 채워 넣도록 되어 있습니다.
>
> 기존 `Elmo_EtherCAT_Test_4Axis`는 `TCPMotionInterface.Response()`가 받은 프레임을 즉시 `MsgPaser()`로 넘기고, 그 안에서 `LMCAxis.MoveAbsolute`, `LMCRobot.MoveLinearCoord`, `SendData`까지 바로 호출합니다. 이 부분이 Jonas 예제의 “통신 callback과 RT motion 실행을 버퍼/flag로 분리”한 구조와 가장 큰 차이입니다.
>
> 또 하나 확인된 차이: 기존 프로젝트는 `_TCPIPServer_RT` wrapper가 `RtWork`에서 `_TCPIPServer.CyclicCall()`을 직접 호출하게 되어 있습니다. Jonas 메일 기준으로는 TCP/Linux 수신 처리를 `RtWork`로 가져오는 방식은 적합하지 않다고 했으므로, 이 설계는 예제 프로젝트의 권장 방향과 정면으로 다릅니다.
>
> 이제 결과 파일을 만들기 전에 현재 worktree 상태를 확인하겠습니다. 기존 변경과 섞지 않도록 새 문서만 추가하는 방향으로 처리합니다.
>
> 문서는 `docs/architecture` 아래에 새 분석 파일로 만들겠습니다. 기존 C#/WPF 변경과 LASAL 프로젝트 자체는 건드리지 않고, 확인한 소스/네트워크 구조를 근거로 정리만 추가합니다.
>
> `MotionTCPDemo`는 축 12개와 master axis, `_LMCProfile` 기반 robot/profile 객체를 전부 simulation 값으로 둔 예제입니다. 기존 Elmo 프로젝트는 4개 `_LMCAxis`가 실제 `Elmo_1..4` EtherCAT/DS402 drive class와 `PosController`를 통해 연결되어 있어, motion layer 자체의 목적이 다릅니다.
>
> 정정합니다. `DataManager::SetData()` 본문에는 RT task 호출 차단 guard가 있지만, 이 프로젝트의 `DataManager.h`에서 `DataManager_AcceptRtJob`가 정의되어 있어 그 차단은 컴파일에서 빠질 가능성이 큽니다. 문서에는 “코드 구조상 guard가 있지만 현재 헤더 설정은 RT job 허용”으로 정확히 적겠습니다.
>
> 이제 분석 결과 문서를 추가합니다. 내용은 구조 요약, 두 프로젝트의 네트워크/태스크/통신/모션 계층 비교, 그리고 기존 Elmo 프로젝트에 적용할 때의 구체적 변경 방향으로 구성하겠습니다.
>
> <details><summary>Created a file</summary>
>
> - Wrote `./docs/architecture/MotionTCPDemo_vs_Elmo_EtherCAT_Test_4Axis_Analysis_2026-07-03.md` (+361 -0)
>
> </details>
>
> 분석 문서를 추가했습니다. 이제 파일 내용을 빠르게 확인하고, 새 문서에 공백 오류가 없는지 별도로 검사하겠습니다.
>
> 검사에서 새 문서 끝에 불필요한 빈 줄이 하나 잡혔습니다. 내용 문제는 아니지만 `git diff --check` 기준에 맞추기 위해 EOF를 정리하겠습니다.
>
> <details><summary>Edited a file</summary>
>
> - Wrote `./docs/architecture/MotionTCPDemo_vs_Elmo_EtherCAT_Test_4Axis_Analysis_2026-07-03.md` (+0 -1)
>
> </details>
</details>

분석 결과 파일 만들었습니다.

[MotionTCPDemo_vs_Elmo_EtherCAT_Test_4Axis_Analysis_2026-07-03.md](./docs/architecture/MotionTCPDemo_vs_Elmo_EtherCAT_Test_4Axis_Analysis_2026-07-03.md)

핵심 결론은 이겁니다. `MotionTCPDemo`는 완성된 TCP motion API가 아니라, 통신 task와 `RtWork` motion task를 분리하는 SIGMATEK 구조 예제입니다. 반면 우리 `Elmo_EtherCAT_Test_4Axis`는 실제 Elmo EtherCAT 4축에 PMAS-like TCP API를 직접 붙인 통합형 구조라서, `TCPMotionInterface.Response()`에서 parsing, response, `LMCAxis`/`LMCRobot` 호출이 바로 일어납니다.

가장 큰 차이는 `_TCPIPServer_RT`입니다. 기존 Elmo 프로젝트는 `RtWork`에서 TCP server `CyclicCall()`을 돌리는데, Jonas 메일 기준으로는 Linux/TCP receive를 `RtWork`에 넣는 방식이 권장 구조와 맞지 않습니다. 문서에는 이 부분을 다음 구조 변경 우선순위로 정리했습니다.

검증은 새 문서에 대해 `git diff --check --no-index`로 공백 오류까지 확인했습니다. 기존 worktree에는 이미 수정/미추적 파일이 있었고, 이번 작업은 새 문서 추가만 했습니다.
