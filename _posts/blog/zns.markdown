---
layout:     post
title:      "[NVMe 2.0] ZNS(Zoned Namespace)"
date:       2021-07-16
author:     정 현호 (jhh9977@gluesys.com)
categories: blog
tags:       [NVMe, SSD, ZNS, PCIe, LBA]
cover:      ""
main:       ""
---

## 왜 ZNS가 도입되었을까요?

NVMe 2.0 버전이 출시되면서 ZNS(Zoned Namespace)라는 새로운 개념이 도입되었습니다. NVMe 입장에서만 새로운 개념이지, 비슷한 개념으로 이미 ZAC(Zoned Device ATA Command Set) 및 ZBC(Zoned Block Command)의 형태로 SMR(Shingled Magnetic Recording) HDD에 도입되어 있는 구조를 바탕으로 NVMe 2.0 스펙에 추가되었습니다. 그렇다면 왜 NVMe에 ZNS 개념이 도입된 것일까요? 

Random Write 데이터를 저장하는 방식을 사용하는 기존 SSD는 저장되는 데이터를 관리하기 위해 가비지컬렉션 방식을 사용해왔습니다. 문제는 이러한 가비지컬렉션 과정에서 유효 데이터를 찾기 위한 Read, 그리고 다른 블록으로 옮겨 쓰기 위한 Write의 불필요한 I/O가 발생한다는 것입니다. 그만큼 SSD의 수명이 짧아질 수 밖에 없는데, ZNS는 이러한 단점을 보완하기 위해 도입된 기술입니다. 

그럼 ZNS가 무엇인지, 그리고 ZNS가 어떻게 기존 SSD의 단점을 보완할 수 있는지에 대해 ZNS 세부 스펙을 통해 한 번 알아볼까요?

&nbsp;

## ZNS란?

ZNS는 Zoned Namespace의 약자로 연관성 있는 데이터 그룹을 Zone이라는 Namespace로 추상화한 것입니다. 즉 기존 데이터 저장 단위인 블록들을 Random Write 방식이 아닌, 특정 Zone에 유사한 데이터 스트림을 순차적으로 Write를 하는 것이 특징입니다.

&nbsp;

### 장점

Zone은 호스트에 의해 관리되기 때문에, 기존 SSD의 약점인 가비지 컬렉션으로 인한 불필요한 I/O가 발생하지 않습니다. 이로 인해 쓰기 증폭의 발생이 줄어들며, SSD 내부 저장 공간 관리의 필요성이 적어지므로 OP(Over Provisioning)의 비율을 줄일 수 있습니다.

&nbsp;

### Zone State Machine

아래 사진은 데이터가 I/O가 되면서 시스템에 의해 변화하는 각 Zone의 상태들에 대한 관계도입니다.

![Alt text](/assets/ZNS_FIG3.png){: width="900"}  
[출처1]

Zone의 상태를 크게 활성 Zone, 비활성 Zone의 2 가지 상태집합으로 분류하여 정리해봤습니다.

#### 1. Inactive Zone State  

모든 Zone의 초기 상태는 ZSE이며, 나머지는 Zone의 활동에 따라 변하는 고유 상태입니다.

> * ZSE  
>   Zone에 데이터가 비어있음을 의미하고 WP는 해당 Zone의 첫 번째 LBA입니다.
> * ZSF  
>   Zone에 데이터가 가득 찼음을 의미하고, WP는 해당 Zone의 마지막 LBA입니다.
> * ZSRO  
>   수명이 다 한 상태로, Zone 용량 중 일부가 작동을 중지한 후에도 호스트가 해당 Zone에 대한 네임스페이스를 계속 사용할 수 있습니다(ex: copy, ZSO로의 상태전환)
> * ZSO  
>   수명이 다 한 상태로, 더이상의 상태전환이 발생할 수 없습니다.

#### 2. Active Zone State 

최대 활성화될 수 있는 리소스는 제한되며, 다음과 같은 상태가 있습니다.

> * ZSIO  
>   Write 명령어를 실행하여 암시적으로 오픈된 상태이며, 리소스 부족 시, 호스트에 의한 Open에 의해 닫힐 수 있습니다.
> * ZSEO  
>   open 명령어를 명시적으로 실행하여 오픈된 상태이며, 호스트 소프트웨어에 의한 Close 명령어를 통해서만 닫힐 수 있습니다.
> * ZSC  
>   암시적 오픈을 통한 해당 Zone의 쓰기 용량이 모두 채워졌거나, Close 명령어를 통한 리소스 해제되었음을 의미합니다. 

&nbsp;

위에 상태도에 따라, 실제로 드라이브에 데이터가 쓰여질 때 Zone의 상태가 어떻게 변할까요?

### 상태 변환 과정의 예시

> 1. 일반적으로 새 드라이브의 초기 상태는 ZSE:Empty 상태입니다.
> 2. ZSE 상태에서 Write 명령어를 통해 쓰기를 시작하는 경우, 자동으로 ZSIO:Implicitly 상태로 전환됩니다.  
>    호스트에 의한 Open 명령어를 통해, ZSE:Empty, ZSIO:Implicitly 상태는 ZSEO:Explicitly 상태로 전환될 수도 있습니다.  
>    Open 명령어를 통한 명시 오픈 상태인 ZSEO:Explicitly는 호스트에 의한 Close Zone 명령어를 통해서만 ZSC:Close 상태로 전환될 수 있습니다.  
> 3. 용량이 모두 채워지면, 자동으로 ZSC:Close 상태로 전환됩니다.
> 4. ZSC 상태는 Zone Send Action의 Finish Zone 명령 or Zone Active Excursion의 Finish Zone Recommended 속성에 의해 ZSF:Full 상태로 전환되어 최종 완료 처리됩니다.

&nbsp;

## Zone Specification

### I/O Command Interpretation

ZNS 스펙에 대해 알아보기에 앞서, NVMe 프로토콜은 적합한 I/O Command Set을 어떻게 식별하여 사용할까요? 

![Alt text](/assets/ZNS_FIG2.png){: width="700"}  
[출처3]

CID(Command Identifier) 필드를 통해 어떤 Command Set을 사용할 것인지 식별합니다. 컨트롤러는 NSID 필드를 통해 원하는 Namespace에 대한 Command Set을 사용할 수 있습니다. 최종 식별된 Command Set의 Command는 Opcode 필드로 구분하여 사용할 수 있습니다.

### ZNS Command Set

ZNS Command Set에 대해 알아보기에 앞서, ZNS Command Set이 어떤 속성을 사용하여 Zone을 핸들링 하는지 알아보겠습니다.

#### Zone Attributes

Zone Attributes는 Zone Command Set 필드에 사용되는 속성들입니다. 

> * Zone Type  
>   Zone에 데이터 I/O를 위한 룰로, **2h** 값이면 순차 쓰기가 요구됨을 의미합니다.
> * Zone State  
>   각 Zone은 상태 집합으로 구성된 상태 시스템과 연결되어 있으며, 해당 Zone의 작동 특성을 정의합니다.
> * WP(Write Pointer)  
>   각 Zone의 writable한 다음 논리 블록 주소를 정의합니다.
> * ZSLBA(Zone Start Logical Block Address)  
>   각 Zone의 가장 낮은 논리 블록 주소를 정의합니다.
> * Zone Capacity  
>   각 헤Zone의 writable한 논리 블록의 총량을 정의합니다.
> * Zone Descriptor Extension Valid  
>   해당 Zone과 연결된 유효 데이터가 있는지를 정의합니다.
> * Reset Zone Recommended  
>   컨트롤러가 해당 Zone을 리셋하도록 정의합니다.
> * Finish Zone Recommended  
>   컨트롤러가 해당 Zone을 완료하도록 권장함을 정의합니다.
> * Zone Finished by Controller  
>   해당 Zone이 완료되었음을 정의합니다.

ZNS Command는 위의 속성들을 사용하여 Zone을 핸들링합니다. 그렇다면 어떤 ZNS Command가 있는지 알아보겠습니다.
ZNS Command의 종류는 **Zone Append, Zone Management Send, Zone Management Recieve**의 총 3가지가 있습니다.

#### 1. Zone Append

![Alt text](/assets/ZNS_FIG4.png){: width="700"}  
[출처2]

Zone Append 명령어는 Zone의 ZSLBA 필드에 표시된 I/O 컨트롤러에 데이터 및 메타데이터를 기록하는 명령어입니다. 즉 데이터를 기록할 때,  해당 Zone의 가장 낮은 논리 블록 주소를 WP로 지정하는 방식입니다. WP는 자동으로 조정되며, 이는 여러 개의 쓰기 작업이 동시에 실행될 수 있음을 의미합니다. 쓰기가 완료되면 컨트롤러는 명령 CQE(Completion Queue Entry)를 I/O 완료 대기열에 게시합니다.  명령이 성공적으로 완료되면, 데이터가 포함된 가장 낮은 LBA가 반환됩니다.

기존 NVMe SSD 방식에서 활용된 데이터를 기록 방법인 Write Command 방식은 WP를 명시적으로 알려줘야 했습니다. 쓰기 명령 제출 시, WP의 싱크가 맞지 않으면 순차 쓰기 요구사항 위반으로 오류가 발생할 수 있습니다. Zone Append Command를 통해 이러한 문제점을 보완할 수 있습니다.

#### 2. Zone Management Send

여러 Zone의 action 수행에 대한 제어를 요청하는 작업을 정의합니다. 여기서 action은 Zone Send Action 필드가 담당합니다.

* ZSA(Zone Send Action)  
  Zone의 실질적인 action 수행을 위한 작업으로, Zone 상태 전환을 위해 직접적으로 사용되는 Byte 필드입니다.
  ZSA는 Zone의 상태를 어떻게 변환시킬까요? ZSA가 수행하는 Zone의 첫 번째 논리 블록 주소(SLBA)이 명시된 경우만 한 번 살펴보겠습니다.
> * Close Zone  
    해당 Zone이 Open 상태라면, Zone을 ZSC:Close 상태로 전환시킵니다.
> * Finish Zone  
    수명이 남아 있다면, 해당 Zone을 ZSF:Full 상태로 전환시킵니다.
> * Open Zone  
    명시 Open 명령어로, 수명이 남아 있다면 ZSEO:Explicitly 상태로 전환시킵니다.
> * Reset Zone 
    수명이 남아있다면, 해당 Zone은 ZSE:Empty 상태로 전환
> * Offline Zone  
    ZSRO 상태는 ZSO 상태로 전환
> * Set Zone Descriptor Extension Zone  
    ZSE 상태만 ZSC 상태로 전환

#### 3. Zone Management Receive  

Zone Management Receive 명령어는 Zone에 대한 정보(Zone Type, Zone State, Zone Attributes 등)가 포함된 데이터 버퍼를 반환합니다. 반환된 데이터를 통해 해당 Zone에 대한 정보를 알 수 있습니다.  

&nbsp;

마치며
-----

지금까지 ZNS Command Set에 대하여 자세히 알아보았는데요, NVMe ZNS 기술은 우리가 사용해왔던 SSD에 안정성 및 효율성을 제공합니다. 앞으로 더 좋은 기술이 나오기를 고대하며 포스팅을 마치겠습니다.

&nbsp;

출처
---
* <a href=https://nvmexpress.org/wp-content/uploads/NVM-Express-Zoned-Namespace-Command-Set-Specification-1.1-2021.06.02-Ratified-1.pdf>출처 1</a>
* <a href=https://zonedstorage.io/introduction/zns/>출처 2</a>
* <a href=https://snia.org/sites/default/files/SDC/2020/154-Martin-Hands-NVMe-2.0-Specification-Preview.pdf >출처 3</a>

[출처1]: https://nvmexpress.org/wp-content/uploads/NVM-Express-Zoned-Namespace-Command-Set-Specification-1.1-2021.06.02-Ratified-1.pdf
[출처2]: https://zonedstorage.io/introduction/zns/
[출처3]: https://snia.org/sites/default/files/SDC/2020/154-Martin-Hands-NVMe-2.0-Specification-Preview.pdf 
