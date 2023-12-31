# High_Level_Synthesis

어쩌면 디지털 회로설계의 차세대 설계방법이 될 HLS에 대하여 고찰한다.

아래 첨부된 논문을 참고하여 HLS에 대한 기본적인 이해를 설명하고자 한다.

논문이 31장이기 때문에 다 설명할 생각은 추호도 없고 영어가 재미도 없으므로 가장 많이 사용되는 Vivado HLS를 위주로 서술해 나간다.

시간이 많고 할께 없다면 본 논문을 정독해서 모든 내용을 정리해보는걸 추천한다.

물론 난 안할꺼다.


![image](https://github.com/dylee0907/High_Level_Synthesis/assets/79738681/2b2e3c80-cfa3-4b2c-9f45-df8b3af3225b)

#Abstract

하드웨어의 궁극적인 목표는 본 기기에서 돌아가는 소프트웨어 실행 속도의 가속화이다. 

소프트웨어 실행의 가속화라는 것은 하드웨어를 효율적으로 활용하여 경우에 따른 데이터 call in과 call back 및 computation을 적은 파워 소모를 통해 행하는 것을 지칭한다. 이런 요구를 충족하면서 값이 저렴한 반도체 칩인 ASIC은 아키텍처가 고정되어 있기 때문에 다양한 소프트웨어를 실행하는데 한계가 있다. 

또한 ASIC을 만들어본 사람들은 알겠지만 그 과정이 정말 주옥같기 때문에 다른 방법으로 효율적인 하드웨어를 구현하는 것이 더 이득일 수 있다. (하지만 아직은 ASIC을 만드는게 최선이긴 하다. 이유는 뒤에서 설명).

따라서, 엔지니어들은 Reconfigurable한 FPGA와 기존의 CPU를 결합한 Heterogeneous computing system을 활용하려고 한다. 

대부분의 프로그램이 실행되는 CPU를 사용하면서 computation weight가 큰 연산은 FPGA로 가속하는 하드웨어를 사용하면 어떤 프로그램이든 빠르게 돌릴 수 있는 칩이 생성되는 것이다.

CPU는 보통 C 또는 C++로 동작시키며 hard coding을 통해 조작이 가능하므로 HLS 또한 C, C++를 사용하여 동작을 시킨다.

하지만 논문에도 나와있듯 아직 HLS를 활용하여 High Level coding을 하는것은 여러모로 제약이 많다. 
C언어를 하면서 우리를 괴롭혔던 동적 할당, 포인터 등 해결해야 할 문제들이 존재한다.
Code partitioning, optimization, design space exploration 또한 아직 미해결된 상태이다.(이쯤되면 안쓰는게 맞지않을까 싶다.)

#Intro

무어의 법칙이 위배되면서 연산 능력을 향상시키기 위해 Heterogeneous computing이 시작되었다.

" Heterogeneous computing is a promising approach in which a group of processing nodes execute a workload in parallel. Given different kinds of nodes including multi-core CPUs, real-time processors, DSPs, GPUs, and accelerators on FPGAs or ASICs, the computing workload can be partitioned such that each part is executed on a processor that is well-matched to its requirements and the performance optimisation goals".


이 문장이 본 논문의 내용을 정확히 요약해준다. 

사실 우리가 수동으로 프로세서의 종류에 맞춰 프로그램 수행 역할을 부여해도 효율적인 하드웨어 운용과 빠른 소프트웨어 실행을 할 수 있다.

FPGA에서 돌리는건 verilog로 코딩해서 넣고, CPU는 C로 코딩해서 넣고, GPU에는 CUDA로 넣으면 되지만 이게 매우 귀찮은 작업이기 때문에 이 모든걸 HLS로 한번에 해버리겠다는 생각이 논문의 내용이다.(근데 HLS가 안되면 저걸 다 할줄 아는 엔지니어가 떼돈을 버는거라 개꿀이긴 하다.)

HLS를 사용하는 방법에는 크게 3가지가 있다.
1. 합성가능한 C로 내가 코딩을 한다.
2. 특정 부분만 직접 C로 코딩한걸 Tool을 이용해서 합성 가능한 RTL로 변환을 해준다.
3. MaxJ나 CAL같은 언어를 써서 Tool을 써서 RTL로 변환을 해준다.
(아래 그림을 참고)

![image](https://github.com/dylee0907/High_Level_Synthesis/assets/79738681/13027849-80ea-4548-a8a6-99888825c7c0)

그럼 수동으로 HLL function을 RTL로 변환하는데 어떤 어려운점이 있는지 확인해보자.
1. HLL 코드를 이해하고 HDL 코드로 바꾸러면 시간과 돈이 많이 든다. (삼성은 돈이 많아서 그냥 하는듯 하다. 참고로 걔네는 python으로 하기 때문에 여러분의 S22 발열이 미쳤던 이유가 사실 여기 있다. python을 사용해본 사람들은 알겠지만 bit수를 고려한 코딩을 하지 않기 때문에 python을 기반으로 하드웨어를 제작하게 되면 margin을 두고 하드웨어 optimization을 하기 때문에 power 소모가 크다.)
2. HLL(C, C++)도 이해하고 RTL도 이해하는 엔지니어가 많지 않다. 소프트웨어와 하드웨어를 둘다 할 줄 아는 사람은 의사보다 수가 적다.
3. Floating point 연산, 동적 할당, function call by reference와 같은 기능을 하드웨어로 구현하기는 정말 어렵다.
4. 소프트웨어 코드를 조금 바꾸면 하드웨어도 다시 고쳐야한다.
5. 이미 제작된 CPU에서 사용되는 코드를 병렬처리하기가 어렵다. (병렬처리 안할꺼면 FPGA로 가속하는게 의미가 없음)

이런 문제들로 인해 현재 가장 많이 사용되는 방법은 가장 복잡한 수준의 연산을 처리해야하는 부분만 HLL로 처리하여 FPGA에서 합성하는 방식이다.
위 그림의 2번에 해당하는 방식으로 다시 3가지 방법으로 나뉜다.
1. Behavioural High Level Synthesis(2a): 하드웨어 엔지니어가 복잡한 연산만 직접 합성 가능한 형태로 코드를 수정한다. 이후 HLS tool을 사용해 RTL로 변환한다.
2. Synthesisable Behavioural code generation(2b): Tool을 사용하여 HLL로 합성 가능한 코드를 생성한다. 이 tool을 사용할때 엔지니어의 개입을 통해 일부 코드를 수정 해야한다. 이후 HLS tool을 사용하여 RTL로 변환한다.
3. Domain Specific Language(DSL) for HLS(2c): 하드웨어 엔지니어가 HLL로 작성된 복잡한 연산을 DSL로 변환하고 DSL 컴파일러가 RTL로 변환한다.

마지막 3번째는 Dataflow grah를 이용한 방식이다. DFG는 연산을 위한 node와 데이터 이동을 위한 edge로 구성되어 있으며, 계산시 synch 및 asynch 방식으로 실행이 가능하기 때문에 FPGA로 구현하기에 적합하다.
1. Dataflow HLS(3a): MaxJ 또는 CAL과 같은 언어를 이용하여 엔지니어가 직접 합성 가능하도록 코딩한다. 이후 HLS tool을 이용하여 RTL로 변환하여 FPGA에 올린다.
2. High-level DFG representation(3b): DFG를 더 높은 수준에서 구현한 후 합성 가능한 형태로 변환한다. 이후 dataflow HLS를 사용하여 RTL로 변환한다.

앞서 말했듯이 사실상 2a 방법을 제외하고 나머지 방식은 기능적으로 한계가 많다.
따라서, 2a 방법을 위주로 HLS를 설명하고자 한다.


![image](https://github.com/dylee0907/High_Level_Synthesis/assets/79738681/8be9fd14-fd86-49a0-bff8-d419f56a1c7e)

위 그림은 HLS design flow를 나타낸다.
참고로 모든 HLL이 합성 가능한 형태는 아니다.

현재 HLS tool은 pointer, recursion, 동적할당과 같은 기능을 하드웨어로 구현할 수 없기 때문에 이와 같은 기능은 지양하며 코드를 구현해야 합성이 가능하다.
Parsing~CDFG는 tool 알아서 해주는 과정이므로 설명은 생략한다.

Flow에서 Allocation은 design library에서 해당 코드의 동작에 대응되는 cell을 불러오는 과정이다.
(해당 방법은 Primelib을 통해 새로운 cell을 제작할때 characterization 단계에서 cell을 import하는 방식과 매우 유사하다. PTSD오는 과정이다.)

Scheduling은 시스템의 clk cycle에 맞춰 control step을 정의하고
Resource Binding은 시스템에서 연산 수행 타이밍과 데이터 전송 타이밍을 결정하는 과정이다.

마지막으로 RTL generate 단계를 통해 엔지니어가 요구하는 RTL을 출력하게 된다.


FPGA synthesis를 위한 Behavioural Approach 종류와 제공되는 기능을 아래 사진으로 확인할 수 있다.
![image](https://github.com/dylee0907/High_Level_Synthesis/assets/79738681/913a92c7-1b34-4b59-85c5-a3295f267995)

보이는바와 같이 현재는 Vivado HLS가 가장 다양한 기능을 제공하며 그 다음으로는 Intel FPGA SDK for OpenCL과 Intel HLS compiler이 뒤따르는 것이 보인다.
(FPGA의 양대 산맥인 두 회사가 가장 훌륭한 tool을 제공한다.)

#Vivado HLS

Vivado HLS는 합성 가능한 C, C++, SystemC, OpenCL HLL code로부터 Verilog와 VHDL을 통한 RTL 구현이 가능한 tool이다.
아무래도 회사에서 제공하는 툴이기 때문에 tool 내부의 작동 방식은 기밀이지만 Xilinx에서 AutoPilot tool을 사버린 후 나온 tool이기 때문에 동작 방식은 AutoPilot과 유사할 것을 추축된다.

논문에 그 방식이 설명되어있지만 딱히 필요 없으므로 생략한다.
아래 그림을 참고하면 된다.
![image](https://github.com/dylee0907/High_Level_Synthesis/assets/79738681/259997ba-2da0-4fa5-a045-1b3ff4e03e2f)


#INTEL FPGA SDK FOR OpenCL

이 tool은 OpenCL을 verilog RTL로 변환해준다.
OpenCL은 프로세서, GPU, FPGA를 포함한 이종 플랫폼을 위한 병렬 애플리케이션 프로그래밍 프레임워크이다.
OpenCl SDK는 멀티코어를 이용하여 여러개의 thread를 통해 병렬 연산을 실행하는 대신 FPGA에서 파이프라이닝을 통해 연산을 진행한다.

#INTEL HLS COMPILER

이 tool은 C++로 작성된 코드를 RTL로 변환한다.
컴파일러가 loop unrolling, data dependency, pipelining을 엔지니어가 제공한 코드에 맞춰 구현한다.
Vivado HLS와 같이 사용 가능한 IP를 제공하며 생성된 RTL에 맞춰 testbench 검증 또한 제공된다.

#Thoughts

아무리 봐도 아직은 verilog를 이용한 하드웨어 코딩이 HLS를 사용하는 방식보다 편한 부분이 있다.
결국 우리가 HLS로 돌아서려는 이유도 컴퓨터 소프트웨어 코딩으로 쉽게 구현이 가능한 AI의 backpropagation, DFT 등의 알고리즘을
하드웨어로 구현하기 어렵기 때문(정확히 말하면 귀찮기 때문)이지만 tool을 사용해 변환하는 과정이 너무 귀찮다.

실제로 이런 이유로 앞서 말했듯이 ASIC을 사용할 수 밖에 없을 것이다. 
기능적 제약도 너무 많고 코드 변환 과정을 거치는게 또 굉장히 귀찮은 과정이기 때문에 HLS의 성능이 대폭 향상되어 동적할당과 같은 문제가 해결된다면
그떄 비로서 SoC 형태로 Heterogenous Computing을 하는게 의미 있을것 같다.

차라리 머리를 좀 더 써서 하드웨어 아키텍처를 구현하는게 낫지 않을까 싶다. 
새로 아키텍처 만들면 저널급 논문도 하나 나올텐데...

물론 tool을 세팅하는 과정만 지나면 쓸만하겠지만 대부분 제공되는 user manual이 너무 개판이고 기본적으로 600장이 넘어가기 때문에 그걸 다 읽어보기가 거부감이 든다.
Primelib, Design Compiler, IC Compiler 때문에 읽은 매뉴얼을 생각하면 아찔하다.

따라서, HLS 공부는...내년의 내가 알아서 할꺼라고 믿는다.

쓰면서 생각이 든게 verilog로 만든 아키텍처와 HLS로 만든 아키텍처를 DC compiler로 비교하는 논문을 쓰면 컨퍼런스급 논문은 나올것 같은데 이것도 내년에 해봐야지....
