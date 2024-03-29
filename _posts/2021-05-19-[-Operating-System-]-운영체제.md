---
categories:
  - OS
tags:
  - OS
  - programming
---
# 운영체제
___
<br>
## 1. 역할

- 컴퓨터의 하드웨어 제어

- 컴퓨터 프로그램 제어

- 사용자와 컴퓨터 간 커뮤니케이션 지원

<br>

## 2. 인터페이스

- 사용자 인터페이스
	- Shell(애플리케이션) -> 터미널, GUI 제공

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/dd60decb-f7e0-4917-b751-40e3a474c49a" alt width= 400>
<em>Shell</em>
</center>
<br>

- 응용프로그램 인터페이스 제공
	- API(Application programming interface) : 함수로 제공 (ex) 'open()' 함수 등)
	- 라이브러리(일반적 경우) : ex) C 라이브러리 

<br>
- 시스템 콜 (시스템 호출 인터페이스)
	- 운영체제가 운영체제의 각 기능을 사용할 수 있도록 '시스템 콜'이라는 명령(함수)를 제공(Application이 API라는 요청서를 사용, O.S에 명령)
	- API 내부는 시스템 콜 호출하는 형태로 대부분 구성

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/919fc68e-6926-408e-ba18-3ab3427e92b1" alt width=400>
<em></em>
</center>

<br>

- CPU 사용 권한

	- 사용자 모드(Application 사용 시 CPU 사용)

	- 커널 모드(O.S가 CPU 사용: 특수 자원 접근)
		- 시스템 콜은 커널 모드로만 실행하여, 응용프로그램이 컴퓨터 시스템 무단 접근 방지
		- 커널 모드로 실행하려면 반드시 시스템 콜을 거쳐야하고 시스템 콜은 운영체제가 제공

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/7ab8c64c-2134-4484-b61d-9c00208f337e" alt width=500>
<em></em>
</center>

<br>
## 3. 프로세스 스케쥴링
<br>

- 배치 프로세싱 : 자동으로 다음 응용프로그램이 이어서 실행될 수 있도록 하는 시스템 (FIFO(선입선출)형태)

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/40b520f7-8cdd-4fb5-bfd7-a506bc52c3d8" alt width=300>
<em>어떤 프로그램의 실행 시간이 길면 뒤의 프로그램의 기다림 증가, 멀티테스킹X
</em>
</center>

<br>
- 시분할 시스템: 응용프로그램이 CPU를 사용하는 시간을 잘게 쪼개어 실행될 수 있도록 하는 시스템

- 멀티테스킹: 단일 CPU에서 여러 응용프로그램이 동시에 실행되는 것처럼 보이도록 하는 시스템(시분할 시스템과 동일)

- 멀티프로그래밍: 최대한 CPU를 많이 활용하는 시스템
	- 시간 대비 CPU활용도를 높임
	- 응용프로그램을 짧은 시간 안에 완료할 수 있음

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/ae4826d0-2b89-4312-8343-e4508a337fd6" alt width= 500>
<em>멀티테스킹 VS 멀티프로세싱</em>
</center>

<br>

※ 멀티테스킹 VS 멀티프로세싱
- 멀티테스킹: 단일 CPU
- 멀티프로세싱: 여러 CPU에 하나의 프로그램을 병렬로 실행시켜 실행 속도 극대화

<br>

※ 실제로는 시분할 시스템, 멀티프로그래밍, 멀티테스킹이 유사한 의미로 통용

<br>

## 4. 스케쥴링 알고리즘
<br>
### 1) 프로세스

= 작업, task, job

- 응용프로그램 != 프로세스
	- 응용프로그램은 여러 개의 프로세스로 이루어질 수 있음
	- C/C++로 프로그램 제작 -> 1개 프로세스
	- 여러 프로그램끼리 통신 -> 다중 프로세스

- 스케쥴러가 프로세스를 관리
<br>

### 2) 스케쥴링

어떤 순서로 프로세스 실행할지 (목표 : 시분할 시스템, 멀티프로그래밍)

- FIFO 스케쥴러 : 먼저 들어온 프로세스부터 CPU에서 처리(배치 프로세싱과 같음,queue)

- SJF(Shortest job First) 스케쥴러 : 가장 프로세스 실행시간이 짧은것부터 실행

- 우선순위 기반 스케쥴러
	- 정적 우선순위 : 프로세스마다 우선순위를 미리 지정
	- 동적 우선순위 : 스케쥴러가 상황에 따라 우선순위를 동적으로 변경

- RoundRObin 스케쥴러

<br>

<center><img src="https://github.com/limbsoo/limbsoo.github.io/assets/96706760/099f5814-490b-4eca-80ff-28a7f411c618" alt width="80%">
<em>RoundRObin 스케쥴러</em>
</center>






<br>
<br>









