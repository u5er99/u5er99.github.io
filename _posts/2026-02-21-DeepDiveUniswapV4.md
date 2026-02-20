---
title: DeepDive Uniswap V4  ## 글 제목
date: 2026-02-21 00:00:00 +0900 ## 작성 시간
categories: [web3, DEX] ## 카테고리
tags: [web3, uniswap]     ## 태그
math: true
---

## Uniswap V4 아키텍처 딥다이브

## Overview

본 아티클에서는 Uniswap V4의 아키텍처 변화와 새롭게 도입된 핵심 기능들을 상세히 다룬다. 단순한 기능 나열을 넘어, V4가 왜 이러한 구조로 변경되었는지에 대한 설계 의도를 중점적으로 분석한다. 또한, 새로운 기능 구현 시 개발자들이 반드시 고려해야 할 보안 사항과 주의점을 짚어본다.

논의는 실제 **Solidity 코드 베이스**와 **수학적 모델**을 기반으로 진행된다. 따라서 Solidity와 V3 AMM의 기본 원리에 대한 이해가 있는 독자를 대상으로 한다.

본문에서 다룰 핵심 키워드는 다음과 같다.

- **Architecture:** Singleton Pattern, Flash Accounting
- **Extensibility:** Hook, NoOp
- **Mechanism:** Tick Bitmap, Liquidity Provision, Swap

## Introduction: V3의 한계와 V4의 등장 배경

Mr.Juno와 Minbuu가 말씀하신 것처럼 Why가 제일 중요하기 때문에

이번 단락에서는 Uniswap V3가 가졌던 3가지 구조적 한계점과, 이를 V4가 어떤 기술적 혁신으로 극복했는지 살펴본다.

#### **1. 아키텍처의 비효율성**

![alt text](/assets/img/image-3.png)

Uniswap V3는 `PoolFactory`를 통해 새로운 Pool을 생성할 때마다 매번 별도의 스마트 컨트랙트를 배포하는 방식을 사용한다. 사실 이 방식은 Pool을 만드는 것 자체만으로도 가스비 부담이 크다.

특히 멀티홉(Multi-hop) 스왑에서 비효율적인 면이 두드러진다. 예를 들어 사용자가 `ETH -> USDC -> DAI` 경로로 스왑한다고 치자. 기존 구조에서는 `ETH/USDC` 컨트랙트와 `USDC/DAI` 컨트랙트를 각각 따로 호출해야 하고, 각 단계를 거칠 때마다 실제로 토큰 전송(Transfer)이 일어나야 했다.

결국 **1) Pool 생성 비용**, **2) 여러 컨트랙트를 오가는 호출 비용**, **3) 단계마다 발생하는 중복된 토큰 전송 비용**. 이 세 가지 측면에서 불필요한 가스 낭비가 발생하고 있었던 것이다.

- **V4의 해답: Singleton과 Flash Accounting**

V4는 이 문제를 Singleton 패턴과 Flash Accounting 도입으로 해결했다.

먼저 Singleton 패턴이다. 모든 Pool의 상태(State)를 `PoolManager`라는 하나의 컨트랙트에서 mapping으로 관리하는 방식이다. 새로운 Pool을 만들 때마다 무겁게 컨트랙트를 배포할 필요가 없으니 생성 비용을 99% 가까이 아낄 수 있다. 당연히 Pool 간 이동 시 컨트랙트를 갈아타는 호출 비용도 사라진다.

여기에 Flash Accounting이 더해지면서 토큰 전송 비용도 획기적으로 줄었다. 기존과 달리 V4의 스왑 과정에서는 토큰이 단계별로 이동하지 않는다. 대신 `PoolManager` 내부 장부에서 `delta` 값(받을 돈/줄 돈)만 계산하고, 모든 로직이 끝난 뒤 최종 결과물만 딱 정산한다. 덕분에 아무리 복잡한 경로로 스왑하더라도 중간 과정에서 나가는 토큰 전송 비용은 완벽하게 '0'이 된다.

#### **2. 실행 흐름의 제약**

V3의 코어 로직은 불변(Immutable)하며, 모든 Pool에 동일한 연산 과정이 강제된다. 이는 다양한 파생 기능을 구현하는 데 있어 다음과 같은 기술적 제약을 야기했다.

- **첫째는 고정된 수수료이다.**

V3는 Pool 생성 시 0.05%, 0.3% 등 미리 정의된 수수료 티어 중 하나를 선택해야 하며, 생성 후에는 이를 변경할 수 없다. 이는 급변하는 변동성 장세에서 유동성 제공자(LP)가 적절한 리스크 프리미엄을 수취하지 못하게 하거나, 반대로 안정적인 장세에서 경쟁력 없는 높은 수수료를 유지하게 만드는 비효율적인 제약이다.

- **둘째, 기능 구현으로 인한 Overhead**

V3에서 TWAMM(시간 가중 평균 가격)이나 지정가 주문(Limit Order) 같은 기능을 구현하려면 반드시 `User -> Logic Contract(Wrapper) -> V3 Pool` 형태의 호출 구조를 가져야한다.

이러한 구조는 기능 수행에 필요한 본질적인 연산 외에도, 로직을 연결하는 과정에서 추가적인 비용을 발생시킨다. 대표적으로 서로 다른 컨트랙트를 오가며 발생하는 Context Switching, 그리고 래퍼와 Pool 사이에서 토큰을 주고받기 위한 중복된 token Transfer 등이 있다.

- **V4의 해답 : Hooks**

V4는 이러한 제약을 'Hook'이라는 콜백(Callback) 인터페이스로 해결했다. Hook은 Pool의 주요 Lifecycle 시점에 개발자가 작성한 외부 로직을 직접 주입할 수 있게 해준다.

- `beforeInitialize` / `afterInitialize`
- `beforeAddLiquidity` / `afterAddLiquidity`
- `beforeRemoveLiquidity` / `afterRemoveLiquidity`
- `beforeSwap` / `afterSwap`
- `beforeDonate` / `afterDonate`

이제 개발자는 무거운 래퍼 컨트랙트를 거치지 않고, `beforeSwap`이나 `afterAddLiquidity` 시점에 직접 커스텀 로직을 실행할 수 있다. 이를 통해 시장 상황에 따라 실시간으로 변하는 Dynamic Fee를 구현하거나, 불필요한 중개 과정을 제거하여 실행 흐름을 완전히 제어할 수 있게 된 것이다.

#### 3. 자산 관리의 비효율

- **ERC-6909: Native Claims**

기존의 ERC-20은 토큰마다 별도의 컨트랙트가 존재하며, 각 컨트랙트가 `mapping(address => uint256) balance`를 관리한다. 따라서 서로 다른 토큰을 교환하려면 복수의 컨트랙트에 Call하여 각각의 상태를 변경해야 하여 비효율적이었다.

V4의 `PoolManager`는 ERC-6909를 상속받아 구현되었다. 이는 단일 컨트랙트 내에서 `id`(토큰 식별자)를 통해 여러 종류의 토큰 잔고를 한 번에 관리하는 표준이다.

기술적으로 `PoolManager`는 다음과 같은 중첩 맵핑(Nested Mapping)을 가진다.

```solidity
mapping(address owner => mapping(uint256 id => uint256 balance)) public balanceOf;
```

스왑이 완료되었을 때, 사용자가 `take()`를 호출하여 외부로 토큰을 인출하지 않으면 `PoolManager`는 해당 토큰(예: USDC)의 `id`에 해당하는 사용자의 `balanceOf` 값을 증가시킨다.
이는 값비싼 `ERC20.transfer` 호출 없이, `PoolManager`의 스토리지 슬롯(Storage Slot) 하나만 업데이트(`SSTORE`)하면 소유권 이전이 완료됨을 의미한다.

이 'Claim' 토큰은 V4 생태계 내에서 실제 토큰과 1:1로 동일하게 취급되며, 언제든 `burn()`을 통해 실제 ERC-20 토큰으로 Redeem할 수 있다.

- **Donate: 수익의 재분배 (단순 전송과의 차이)**

Hooks를 활용하면 MEV(Maximal Extractable Value)를 포착하거나, 커스텀 수수료를 징수하여 추가 수익을 낼 수 있다. 문제는 이 수익을 LP들에게 나눠주는 방법이다.

하지만 단순히 토큰을 Pool 컨트랙트로 전송(`transfer`)하는 것만으로는 분배가 이루어지지 않는다. 프로토콜 입장에서 그 토큰은 주인이 없는 잉여 자산(Reserves)일 뿐, LP들의 몫이라는 기록이 없기 때문이다.

이 문제를 해결하기 위해 **Donate** 기능이 도입되었다.

`donate` 함수는 토큰을 Pool에 주입함과 동시에, 프로토콜의 핵심 변수인 `feeGrowthGlobal` (전역 수수료 누적 변수)을 강제로 업데이트한다. 즉, 수학적으로 주입된 토큰은 LP에게 가는 거래 수수료라고 만들어준다.

이 함수가 호출되면 기부된 금액만큼 `feeGrowthGlobal` 값이 증가하고, 결과적으로 현재 활성화된(Active) 유동성 포지션을 가진 모든 LP는 자신의 지분에 비례하여 이 수익을 즉시 수령할 수 있게 된다. Donate는 Hooks가 만들어낸 부가 가치를 기존의 수수료 정산 시스템에 태워 LP들에게 환원시키는 핵심 파이프라인이다.

## Body: **코드 베이스로 보는 Uniswap V4**

Uniswap V4에 존재하는 여러 흐름들을 통해서 확인해보자.

### Part 1. Create Pool

Uniswap V4에서 유동성을 공급하거나 스왑을 하기 위해서는 가장 먼저 Pool이 존재해야 한다. Singleton 구조인 V4에서는 새로운 컨트렉트를 배포하지 않고 `initialize()` 함수를 호출을 통해 `PoolManager`에 Pool의 정보를 등록하여 Pool을 만들 수 있다.

#### 1. Pool의 정의: `PoolKey`

V4는 **`PoolKey`** 구조체를 통해 Pool의 정보를 담는다.

![](/assets/img/image-4.png)

- **Currency 정렬:** `currency0`과 `currency1`은 주소값 크기 순으로 자동 정렬된다. 이는 `ETH/USDC`와 `USDC/ETH`가 서로 다른 풀로 중복 생성되는 것을 방지하기 위함이다.
- **Hooks 포함:** V3와 가장 큰 차이점은 `hooks` 주소가 Key에 포함된다는 점이다. 즉, 동일한 토큰 쌍이라도 어떤 Hook을 장착했느냐에 따라 아예 다른 Pool로 취급된다.
    
    (Hook은 뒤에서 자세히 다뤄보자.)
    

#### 2. Pool의 초기화: `initialize()`

`PoolKey`가 준비되었다면, `initialize` 로직을 통해 실제 풀을 생성한다.

![](/assets/img/image-5.png)


`PoolKey`와 초기 가격인 `sqrtPriceX96`을 파라미터로 전달하면, 먼저 키 속성에 대한 유효성 검증을 수행한다. 이후 `beforeInitialize` 훅을 실행하고, `PoolId`를 생성하여 내부 저장소(`_pools`)에 초기 가격을 기록한다. (hooks.isValidHookAddress에 대해서는 뒷 파트에서 보자)

#### 3. Pool의 식별: `PoolId`

`PoolManager`는 전달받은 `PoolKey`값을 그대로 저장하지 않고, 이를 Hash하여 고유한 식별자인 `PoolId`를 생성한다.

![](/assets/img/image-6.png)



`PoolKey`의 5개 필드(`currency0, currency1, fee, tickSpacing, hooks`)는 각각 32바이트를 차지하므로, 총 160바이트(`0xa0`)를 읽어서 해시값을 만든다. 

이 `PoolId`는 V4의 Singleton 컨트랙트 내에서 특정 Pool을 찾기 위한 유일한 주소 역할을 한다. Pool이 생성된 후에는 `PoolKey`의 어떤 속성도 변경할 수 없다.

#### 4. 초기 가격 설정: Slot0와 State

다음으로 `PoolManager`는 `PoolId`를 키(Key)로 하여 초기 상태를 저장한다.
`PoolManager`가 라이브러리 함수를 호출하면, 실제 데이터 저장은 `Pool.sol` 내부 로직에 의해 수행된다.

```solidity
// PoolManager.sol (Caller)
tick = _pools[id].initialize(sqrtPriceX96, lpFee);

// Pool.sol (Callee)
function initialize(State storage self, uint160 sqrtPriceX96, uint24 lpFee) internal returns (int24 tick) {
    if (self.slot0.sqrtPriceX96() != 0) PoolAlreadyInitialized.selector.revertWith();

    tick = TickMath.getTickAtSqrtPrice(sqrtPriceX96);

    // the initial protocolFee is 0 so doesn't need to be set
    self.slot0 = Slot0.wrap(bytes32(0)).setSqrtPriceX96(sqrtPriceX96).setTick(tick).setLpFee(lpFee);
}
```

위 코드를 보면 `State` 구조체의 많은 필드 중 오직 `Slot0`만 업데이트한다. 나머지도 살펴보자.


![](/assets/img/image-7.png)


1. **`Slot0 slot0`**

- Pool의 기준점이 되는 `sqrtPriceX96`과 현재 `tick` 등을 하나의 슬롯에 Packing하여 저장한다. 가격은 0이 될 수 없으므로 반드시 초기화해야 한다.

2. **`uint128 liquidity`**

- 현재 활성화된 유동성의 총량을 보여주는 변수로 0의 값을 지닌다. 이 변수는 뒤에 유동성 공급파트에서 다룰 예정이다.

3. **`feeGrowthGlobal0X128`,`feeGrowthGlobal1X128`** 

- 풀 생성 이후 누적된 수수료이다. 거래가 발생한 적 없어 0을 지닌다. 뒤에 Swap 파트에서 다룰 예정이다.

4. **`mapping ticks / bitmap / positions`**

- 유동성 공급자에 대한 정보나 Tick 정보는 아직 존재하지 않아 값이 없다.

이후 마지막으로 `afterInitialize`훅을 실행하고 끝나게 된다.

이로써 Pool이 생성되어 거래를 시작할 준비가 완료되었다.

### Part 2. Hooks

Part 1에서 `PoolKey`에 `hooks` 주소를 담아 Pool을 생성하는 구조를 살펴보았다. Hook은 개발자가 Pool의 Lifecycle에 개입할 수 있게 해주지만, 빈번한 호출로 인한 가스비 문제가 발생할 수 있다. V4는 이를 해결하기 위해 Bitmasking, Assembly Call, 그리고 Delta 연산이라는 세 가지 핵심 기술을 도입했다.

이번 파트에서는 Hook이 어떻게 설계되었는지와 호출 흐름을 살펴보자. 

#### 1. Address Bitmasking & Mining

일반적인 DeFi 프로토콜은 설정값을 Storage에 저장하고 조회한다. 하지만 V4는 `SLOAD` 가스를 절감하기 위해 주소 자체에 설정을 내장하는 방식을 택했다.

**Flag Constants**

`Hooks` 라이브러리에는 14개의 기능 플래그가 정의되어 있다.

```solidity
uint160 internal constant ALL_HOOK_MASK = uint160((1 << 14) - 1);

...
uint160 internal constant BEFORE_SWAP_FLAG = 1 << 7;
uint160 internal constant AFTER_SWAP_FLAG = 1 << 6;
...

uint160 internal constant BEFORE_SWAP_RETURNS_DELTA_FLAG = 1 << 3;
uint160 internal constant AFTER_SWAP_RETURNS_DELTA_FLAG = 1 << 2;
uint160 internal constant AFTER_ADD_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 1;
uint160 internal constant AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG = 1 << 0;
```

코드를 보면 `1 << 7`과 같이 시프트 연산을 사용한다. 이는 주소의 뒤의 14 bit가 각 기능에 매핑된다는 뜻이다. 예를 들어 `BEFORE_SWAP_FLAG`(`1 << 7`)를 활성화하려면, 훅 주소의 뒤에서 8번째 bit가 반드시 `1`이어야 한다. 16진수로 표현 시 `0x...0080` 형태를 가진다.

이 때문에 개발자는 `CREATE2`와 `Salt`를 조합하여, 원하는 flag bit가 설정된 주소를 Mining해야만 훅을 등록할 수 있다.

또한, 아래를 보면 단순히 함수 실행여부를 결정하는 Flag 외에도 Delta 값을 리턴받을지 말지를 결정하는 Flag가 별도로 존재한다는 것이다.

- **함수 실행 Flag:** `BEFORE_SWAP_FLAG`
- **Delta 변경 Flag:** `BEFORE_SWAP_RETURNS_DELTA_FLAG`

Return Delta를 활성화한 주소를 채굴하면, Hook은 단순히 로직을 수행하는 것을 넘어 스왑 수량이나 유동성 수량을 직접 변경할 수 있는 권한을 갖게 된다.

#### 2. Permissions Validation

비트마스크는 효율적이지만, 개발자의 실수를 유발하기 쉽다. 주소 채굴이 잘못되었을 경우를 대비해 V4는 `Permissions` 구조체를 통한 앞의 14 bit의 검증 절차를 제공한다.

```solidity
struct Permissions {
    bool beforeInitialize;
    bool afterInitialize;
        ...
    bool afterAddLiquidityReturnDelta;
    bool afterRemoveLiquidityReturnDelta;
}
```

Hook 컨트랙트가 배포될 때, `validateHookPermissions` 함수를 통해 의도한 bit들이 제대로 설정 되었는지 채굴된 주소가 일치하는지 배포 시점에 강제로 검증된다. (`BaseHook.sol`에 구현되어 있어 상속받아 처리)

#### **3. Hook Execution Flow**

`PoolManager`가 Hook을 실행하고 결과를 반영하는 과정은 크게 Entry → Call → Apply 3단계로 이루어진다.

**3-1. Entry Point: `beforeSwap`**

Hook들 중에서 `beforeSwap`의 흐름을 살펴보자. 여기서 외부 호출과 로직 개입이 모두 관리된다.

![](/assets/img/image-8.png)



코드에서 보듯이 `BEFORE_SWAP_FLAG`, `BEFORE_SWAP_RETURNS_DELTA_FLAG` 두 값을 확인하여 flag가 활성화되어 있을때만 로직을 수행한다.

**3-2. Low-Level Call: Assembly `callHook`**

`callHook`은 실제 외부 컨트랙트를 호출하는 엔진이다. 가스 최적화와 동적 리턴 데이터 처리를 위해 Inline Assembly를 사용한다.


![](/assets/img/image-9.png)


일반 call 호출은 리턴값의 크기를 모르기 때문에 넉넉하게 메모리를 잡거나 불필요한 복사를 수행한다. EVM은 메모리를 쓸수록 가스 비용이 제곱으로 증가한다.
따라서, V4는 `retSize`를 0으로 설정하여 데이터를 일단 별도 버퍼에 둔 뒤, `returndatasize()`로 정확한 크기를 확인하고 딱 필요한 만큼만 메모리를 할당하여 가스비를 절감한다.

**3-3. Data Parsing**

`callHook`이 반환한 `result`는 컴퓨터만 아는 `bytes` 덩어리다. 이 `bytes` 값에 존재하는 delta, fee를 얻기 위해선 Parsing 해야한다. 

아래는 `beforeSwap` 과정에서 delta 값을 처리하는 로직이다.

```solidity
BeforeSwapDelta hookReturn = BeforeSwapDelta.wrap(result.parseReturnDelta());

int128 deltaAmount = hookReturn.getSpecifiedDelta();

// 3. [Application] 꺼낸 숫자를 실제 거래량에 반영
// "원래 100개 스왑하려 했는데, 훅이 -10개 하라니까 90개만 해!"
if (deltaAmount != 0) {
    amountToSwap += deltaAmount; 
}
```

V4는 효율성을 위해 `int256`라는 하나에 두 개의 숫자(입력/출력 토큰 변동량)를 담는 Bit Packing 방식을 사용한다. 

여기서 Specified Delta와 Unspecified Delta 차이는 다음과 같다.

- **상위 128비트 (Specified Delta):** 사용자가 "이만큼 바꿀래요"라고 지정한 토큰의 변동량.
- **하위 128비트 (Unspecified Delta):** 그 결과로 계산되어 나오는 반대편 토큰의 변동량.

`beforeSwap` 단계에서는 아직 스왑 결과가 계산되기 전이므로, Hook은 사용자가 제시한 입력값을 수정하기 위해 Specified Delta를 추출하여 `amountToSwap`에 반영한다.

#### 4. Recursion Protection: `noSelfCall`

Hook이 실행 도중 다시 `PoolManager`를 호출하여 무한 재진입에 빠지는 것을 방지하기 위해 Hook 함수들에 제어자로 `noSelfCall`가 사용된다.

![](/assets/img/image-10.png)


참고로 앞서 본 `beforeSwap`처럼 값을 리턴해야 하는 함수는 modifier 대신 내부 `if`문으로 수동 구현하며, 그 외 함수들은 이 modifier를 사용하여 재진입을 차단한다. (`afterSwap`, `afterModifyLiquidity` 함수도 수동 구현이다)

#### 5. Practical Implementation: BaseHook 활용

실무에서는 `v4-periphery` 패키지의 `BaseHook`을 상속받아 더 안전하고 쉽게 훅을 개발한다.

**(1) Hook Contract (Standard Style)**

우리가 만들 CounterHook은 스왑이 발생할 때마다 횟수를 세는 간단한 로직을 담고 있다.

![](/assets/img/image-11.png)

**(2) Integration: PoolKey와의 연결**

이제 이 훅을 Part 1에서 배운 `PoolKey`에 장착하여 풀을 생성해 보자.

![](/assets/img/image-12.png)

이제 누군가 이 풀에서 `swap()`을 호출하면:

1. `PoolManager`는 `key.hooks`에 주소가 있는지 확인한다.
2. 주소의 비트마스크를 확인하고(`0x...80...`), `beforeSwap`을 실행해야 함을 인지한다.
3. `CounterHook.beforeSwap`을 호출하여 카운트를 1 증가시킨다.
4. 스왑 로직을 계속 진행한다.

### Part 3. Lock Mechanism

Uniswap V4의 상호작용은 `PoolManager.unlock()` 함수를 통해 시작된다. V3와 달리 V4는 Singleton 구조와 Flash Accounting을 도입했기 때문에, 트랜잭션의 Context을 관리하는 **Lock Mechanism**이 필수적이다.

#### 1. Lock 라이브러리와 Transient Storage

V4는 락 상태 관리를 위해 `Lock` 라이브러리를 사용한다. 이 라이브러리는 이더리움 Dencun 업그레이드에 포함된 EIP-1153 (Transient Storage)을 활용하여 가스 효율성을 극대화했다.

아래는 `Lock` 라이브러리의 Core 이다.

```solidity
/// @notice This is a temporary library that allows us to use transient storage (tstore/tload)
/// TODO: This library can be deleted when we have the transient keyword support in solidity.
library Lock {
    // The slot holding the unlocked state, transiently. bytes32(uint256(keccak256("Unlocked")) - 1)
    bytes32 internal constant IS_UNLOCKED_SLOT = 0xc090fc4683624cfc3884e9d8de5eca132f2d0ec062aff75d43c0465d5ceeab23;

    function unlock() internal {
        assembly ("memory-safe") {
            // unlock
            tstore(IS_UNLOCKED_SLOT, true)
        }
    }

    function lock() internal {
        assembly ("memory-safe") {
            tstore(IS_UNLOCKED_SLOT, false)
        }
    }

    function isUnlocked() internal view returns (bool unlocked) {
        assembly ("memory-safe") {
            unlocked := tload(IS_UNLOCKED_SLOT)
        }
    }
}
```

`IS_UNLOCKED_SLOT`에 저장된 데이터는 트랜잭션이 종료되면 자동으로 소멸하므로, 기존 `SSTORE/SLOAD` 방식(약 20,000 gas) 대비 매우 저렴한 비용(100 gas)으로 상태를 관리할 수 있다.

#### 2. Execution Flow

`PoolManager`는 위 라이브러리를 사용하여 Unlock → Callback → Check 순서로 트랜잭션을 처리한다.

```solidity
/// @inheritdoc IPoolManager
function unlock(bytes calldata data) external override returns (bytes memory result) {
    if (Lock.isUnlocked()) AlreadyUnlocked.selector.revertWith();

    Lock.unlock();

    // the caller does everything in this callback, including paying what they owe via calls to settle
    result = IUnlockCallback(msg.sender).unlockCallback(data);

    if (NonzeroDeltaCount.read() != 0) CurrencyNotSettled.selector.revertWith();
    Lock.lock();
}
```

1. **`if (Lock.isUnlocked())`**
- `unlock` 함수가 이미 실행 중인지 확인한다. 이는 `unlock` 함수 자체의 Re-entrancy를 방지하기 위함이다.
1. **`Lock.unlock();`**
- `Lock.unlock()`을 호출하여 상태를 `true`로 변경한다. 이때부터 `PoolManager`의 장부(Delta) 기록이 시작된다.
1. **`IUnlockCallback(msg.sender).unlockCallback(data)`**
- `msg.sender`의 `unlockCallback`을 호출한다. 사용자는 이 함수 내부에서 실질적인 로직(`swap`, `modifyLiquidity`)을 수행한다.
1. **`if (NonzeroDeltaCount.read() != 0)`**
- 콜백이 종료된 후, `NonzeroDeltaCount`를 조회한다. 이 값이 `0`이 아니라면(즉, Pool에게 줄 돈을 안 줬거나 받을 돈을 안 받아갔다면), 트랜잭션은 `CurrencyNotSettled` 에러와 함께 실패한다.
1. **`Lock.lock()`**
- 모든 검증이 통과되면 `Lock.lock()`을 호출하여 상태를 다시 `false`로 되돌리고 트랜잭션을 종료한다.

#### 3. Access Control: **`onlyWhenUnlocked`**

`PoolManager`는 `unlock` 함수를 통해서만 상태 변경이 시작되도록 설계되었다. 이를 코드 레벨에서 강제하는 것이 바로 `onlyWhenUnlocked` Modifier다.

```solidity
/// @notice This will revert if the contract is locked
modifier onlyWhenUnlocked() {
    if (!Lock.isUnlocked()) ManagerLocked.selector.revertWith();
    _;
}
```

V4의 핵심 로직인 `swap`, `modifyLiquidity`, `donate`, `take`, `settle`, `mint` 등은 모두 이 모디파이어를 달고 있다.

따라서 누군가 `PoolManager.unlock()`을 호출하지 않고 외부에서 곧바로 `modifyLiquidity()`를 호출하려고 시도하면, `Lock.isUnlocked()` 검사에서 걸려 트랜잭션은 즉시 실패(`ManagerLocked`)한다.

즉, 우리가 다음 파트에서 다룰 유동성 공급 또한 반드시 이 `unlock` 컨텍스트 안에서, `unlockCallback`을 통해 실행되어야 한다.

### Part 4. Liquidity Provision

이번 파트에서는 유동성 공급하는 과정을 코드 레벨에서 상세히 따라갈 예정이다. V4는 V3의 Concentrated Liquidity 구조를 계승한다. 따라서 본문의 이해를 돕기 위해 Tick, Price, Liquidity의 기본 개념을 나오기 때문에 아래 링크를 통해 먼저 숙지하시는 것을 추천합니다.

https://uniswapv3book.com/milestone_0/uniswap-v3.html

#### 1. Liquidity Provision Work flow

V4의 유동성 공급은 단순히 함수 하나를 호출하는 것으로 끝나지 않는다. Flash Accounting을 위해 [요청] → [Lock] → [Callback] → [실행] → [정산] 흐름을 따른다.


![](/assets/img/image-13.png)

1. user가 Periphery 컨트랙트에 `modifyLiquidity()` 함수를 호출한다.
2. Periphery 컨트랙트가 PoolManager 컨트랙트의 `unlock()` 함수를 호출하면, PoolManager는 `unlockCallback()` 호출한다.
3. callback 로직에서 PoolManager의 `modifyLiquidity()`를 다시 호출해주게 된다.
4. Manager에서는 Pool이 유효한지 `checkPoolInitialized()` 함수를 통해 확인한다.
5. Manager 컨트랙트에서 Hook 주소 bit를 확인하여 before Hook이 필요한지 확인한다. 필요하다면 hook을 호출한다.
6. pool에 있는 유동성 position 수수료를 정산하고 balance delta 값을 계산한다.
7. ModifyLiquidity 이벤트를 발생시킨다.
8. Manager 컨트랙트에서 after Hook이 필요한지 확인한다. 필요하다면 hook을 호출한다.
9. balance delta 값을 Periphery 컨트랙트에게 반환한다.
10. Periphery 컨트랙트에서 balance 값을 settle한다.
11. `unlockCallback` 로직이 끝나면 Manager 쪽에서 non-zero delta 값을 확인하고 컨트랙트를 다시 lock 한다.

이 과정을 이제 따라가 보자.

#### 2. Setup

현재 price와 공급할 범위를 다음처럼 설정해보자.

- **Current price**: 1 ETH = $5000 USDC
- **Liquidity range**: $4545 (lower) to $5500 (upper)
- **Pool Setting:** Fee Tier 0.3% (`tickSpacing = 60`)

**1. Price to Tick 변환**

아래 공식을 통해 변환된다.

$$
tick = log_{1.001}(Price)
$$

```solidity
tickCurrent = log₁.₀₀₀₁(5000) ≈ 85176
tickLower = log₁.₀₀₀₁(4545) ≈ 84222
tickUpper = log₁.₀₀₀₁(5500) ≈ 8602
```

V4는 가스비 최적화를 위해 모든 Tick을 사용하지 않고, `tickSpacing`(여기서는 60) 단위로 Tick을 사용한다. 따라서 계산된 Tick은 가장 가까운 유효한 Tick으로 보정(Rounding)되어 범위가 약간 변할 수 있다.

```solidity
**Upper Tick**
84222 ÷ 60 = 1403.7 → floor(1403.7) = 1403
1403 × 60 = 84180

**Lower Tick**
86029 ÷ 60 = 1433.8 → ceil(1433.8) = 1434
1434 × 60 = 86040

tickLower = 84180  (~$4543)
tickUpper = 86040  (~$5502)
```

따라서 유동성이 공급될 때, ~$4543과~$5502 범위로 들어가게 된다.

#### 3. Entry Point

유동성 공급의 진입점은 `PoolManager` 컨트랙트이다. `modifyLiquidity()` 함수에 대해 알아보자. 이곳은 복잡한 계산보다는 전체적인 흐름 관리를 담당한다.


![](/assets/img/image-14.png)

이 코드에서 주목할 점은 다음과 같다.

- `pool.modifyLiquidity` 호출 전후로 `before`와 `after` 훅을 실행하여 커스텀 로직이 개입할 시점을 제공한다.
- 복잡한 Tick 계산과 상태 변경 로직은 `Pool` Library 함수로 분리하여 처리한다.
- Library가 계산해준 결과를 `_accountPoolBalanceDelta` 함수로 넘겨, 유저의 장부에 빚 또는 자산을 기록한다.

이제 `pool.modifyLiquidity` 내부로 들어가, 실제 유동성이 어떻게 계산되고 저장되는지 몇가지 부분으로 나눠서 살펴보자.

#### **4. Code Execution Flow**

![](/assets/img/image-15.png)


Pool 라이브러리의 `modifyLiquidity` 함수는 Tick 상태 갱신, 수수료 계산, 비트맵 관리, 그리고 전역 유동성 업데이트를 담당합니다.

주요 로직을 기능별로 나누어 분석해보자.

**4-1. Tick Update**

유동성 변화가 있는 경우(`liquidityDelta != 0`), Lower/Upper Tick의 상태를 갱신한다.


![](/assets/img/image-16.png)

tick update하는 과정을 보자.

`liquidityDelta != 0` : 유동성을 추가하던 제거하든 delta 값이 0이 아닐 때를 의미한다.

`updateTick(self, ticklower, liquidityDelta, false);`

- lower position의 tick을 업데이트 시켜준다.
- false : lower tick을 의미한다.
- tick이 flpped 여부를 반환해준다. (상태 : 초기화 → 초기화X or 초기화X → 초기화)
- 이 tick의 새로운 gross liquidity 값도 반환해준다.

`updateTick(self, tickupper, liquidityDelta, true);`

- upper position의 tick을 업데이트 시켜준다.
- true : upper tick을 의미한다.
- 반환값은 lower과 같다.
    

**4-2. Safety Guard**

V4는 단일 Tick에 과도한 유동성이 집중되어 발생할 수 있는 수학적 오류를 방지하기 위해 안전장치를 둔다.

![](/assets/img/image-17.png)


유동성을 추가할 때, tickSpacing을 기반으로 단일 tick에서 저장할 수 있는 최대 유동성 값을 확인한다.

그런 다음 프로토콜은 제안된 업데이트 이후 총 유동성에 대해 두 가지 검사를 진행한다. 첫째, `tickLower`이 최대 유동성 한도를 초과하지 않는지 `state.liquidityGrossAfterLower`와 `maxLiquidityPerTick`를 비교하여 확인한다. 

이 검사에 실패하면 거래는 `TickLiquidityOverflow` 오류와 함께 되돌려져 유동성 추가가 revert 된다. 마찬가지로 `tickUpper`에 대해서도 `tickUpperstate.liquidityGrossAfterUpper`를 확인하여 동일한 검증을 수행한다.

위 검증을 통해 추가할 유동성 값으로 인해 오버플로우를 방지할 수 있다.

**4-3. Bitmap Optimization**

앞에서 계산한 유동성에 따라 나온`flippedLower`, `flippedUpper` 값을 확인해서 lower, upper tick의 bitmap을 변경시킬 수 있다.

![](/assets/img/image-18.png)


해당 tick의 Gross liquidity의 값에 따라 다음과 같이 변한다.

- 유동성이 0이었는데 증가한 경우 : ex) 0 → 100
    - 비활성 상태에서 활성화되서 `flipped = true`
    - `flipTick` 실행 O
- 유동성이 100이었는데 더 증가한 경우 : ex) 100 → 150
    - 활성 상태에서 그대로 활성화되서 `flipped = false`
    - `flipTick` 실행 X
- 유동성이 있었는데 0이 된 경우 : ex) 100 → 0
    - 활성 상태에서 그대로 활성화되서 `flipped = true`
    - `flipTick` 실행 O

이 방식은 첫 공급 시에는 가스비가 들지만, 이미 유동성이 있는 Tick에 추가 공급할 때는 비싼 스토리지 연산을 건너뛰어 가스 비용을 현저히 줄여준다.

**4-4. Fee & Position Management**

![](/assets/img/image-19.png)



1. 수수료 증가분을 추적
    - token0, token1에 대한 LP들의 수수료는 각각 `feeGrowthInside0X128`, `feeGrowthInside1X128`에 들어있다.
    - 이 값들은 Q128.128 고정 소수점 형식으로 저장된다.(따라서 X128이 뒤에 붙는다)
2. position 관리
    - self.position.get 명령어를 통해서 특정 유동성 position을 검색한다.
        - owner
        - tickLower
        - tickUpper
        - salt
3. 수수료 계산
    - position.update() 명령어로 이 position의 수수료를 계산한다.
        - **`liquidityDelta`** - 유동성의 변화(can be positive or negative)
        - position tick 범위 내에서 발생하는 수수료 증가 값
4. 반환 값
    - 미납된 수수료 값을 delta 값으로 변환시킨다.
    - toBalanceDelta()를 통해 fee를 struct로 변환시켜 반환한다.

`position.update`를 호출하면 내 포지션의 유동성을 변경함과 동시에, 그동안 쌓인 수수료를 계산하여 반환해 준다. 이 값은 즉시 지급되는 것이 아니라 `feeDelta`에 기록되어 나중에 정산된다.
**4-5. Cleanup**

![](/assets/img/image-20.png)


유동성을 제거할 때 위 로직이 진행된다. (`liquidityDelta < 0`)

앞에서 말했듯 유동성 제거시 해당 tick의 유동성이 0이 되어서 flipped가 되었다면, 더 이상 해당 Tick의 정보를 저장하고 있을 필요가 없으므로 저장된 데이터를 삭제한다.

또 이더리움에서는 사용하던 Storage를 지우고 0으로 만들면 가스비 일부를 환불 받아, 저장 부담과 가스 비용을 줄일 수 있다.

**4-6. Global Liquidity Update**

![](/assets/img/image-21.png)


1. **Price와 Position의 관계**에 따라 3가지로 나뉘게 된다.
    - Price가 Position 범위보다 아래에 있을 때 → token0만 필요하다.
    - Price가 Position 범위 안에 있을 때 → 두 토큰 다 필요하다.
    - Price가 Position 범위보다 위에 있을 때 → token1만 필요하다.
2. **token Amount 계산**
    - 정밀도를 위해 SqrtPriceMath로 계산하였다. (Q64.96)
    - `getAmount0Delta()` : token0 amount 계산
    - `getAmount1Delta()` : token1 amount 계산
3. **Liquidity Update**
    - `LiquidityMath.addDelta()`를 사용해서 양수 음수 계산을 안전하게 한다.
    - Price가 Position 범위 안에 있을 때는 현재 넣는 유동성이 바로 사용될 수 있도록 추가한다.
4. Gas Optimization
    - `liquidityDelta == 0` 으로 유동성 변화가 0이면 skip 시켰다.
    - tick 구조를 가짐으로써 가격 계산을 할 때, 무거운 지수 연산을 하지 않고 가벼운 tick 조회 함수로 빠르게 계산할 수 있게 되었다.

이 과정을 통해 최종적으로 계산된 `delta`와 `feeDelta`가 `PoolManager`로 반환되고, 서두에 언급한 `_accountPoolBalanceDelta`를 통해 기록된다.

### Part 5. Swap Mechanism

이번 파트에서는 Swap 메커니즘을 코드 레벨에서 상세히 따라갈 예정이다. 

#### 1. Swap Work Flow

![](/assets/img/image-22.png)


1. User는 periphery contract를 통해서 swap 과정을 진입한다.
2. Periphery contract가 PoolManager를 `unlock`하고, periphery contract의 `unlockCallback()`을 trigger 한다.
3. `Callback()` 함수 안에서 manager contract의 `swap()` 함수를 호춣한다.
4. manager contract에서는 파라미터로 들어온 pool이 유효한지 초기화가 되었는지 확인한다.
5. manager contract는 pool에 연결된 hook이 존재하는지 확인하고, 필요하다면 `beforeSwap()`을 실행한다.
6. 실제 swap 과정이 Pool Library에서 일어난다.
7. manager contract는 있다면 protocol fee를 charge하고 Swap() 이벤트를 발생시킨다.
8. 다시 `afterSwap()`이 존재하는지 확인하고, 있다면 실행한다.
9. 마지막으로 user가 "내야 할 토큰"과 "받을 토큰"의 최종 계산값인 `BalanceDelta`를 Periphery contract로 반환한다.
10. Periphery contract는 `BalanceDelta` 값을 바탕으로 Settle된다.
11. Callback이 종료되면, PoolManager에서 모든 Delta 값이 0이 되었는지 확인하고 Lock을 걸어 트랜잭션을 종료한다.

여기서 중요한 점이 `BalanceDelta` 값으로 정산이 완료되는데 어떤 역할을 하는지 알아보자.

`BalanceDelta`는 token0, 1을 나타내는 변수이다. int256으로 된 type으로 상위, 하위 128bit씩 한 token의 변동량을 나타낸다. 이는 양수, 음수 둘 다 가질 수 있다.

`BalanceDelta` 값이 양수이면 PoolManager가 User에게 빛을 진 것이고 음수는 User가 PoolManager에게 빛진 것을 말한다.

**`BalanceDelta`가 settle되는데는 2가지 방법이 있다.**

1. token을 양쪽 방향으로 Delta 값만큼 transfer하는 것이다. 
2. PoolManager에 locked 되어있는 user의 ERC 6909 token을 Minting/Burning한다.

따라서 중간에 transfer하는 과정없이 마지막에만 토큰 이동(또는 6909 Minting/Burning)이 되게 된다.

#### 2. **Conceptual knowledge**

**2-1 방향성과 가격 이동: `zeroForOne`**
Swap 함수는 내부적으로 어떤 토큰을 팔고 사는지에 따라 연산 방식이 완전히 두 갈래로 나뉜다. 이를 결정하는 boolean 변수가 `zeroForOne`이다.

- `zeroForOne == true` (token0 매도 / token1 매수):
    - 풀에 token0이 많아지고 token1이 감소한다.
    - Tick 이동: 현재 Tick에서 왼쪽(하락 방향)으로 이동
- `zeroForOne == false` (token1 매도 / token0 매수):
    - 풀에 token1이 많아지고 token0이 감소한다.
    - Tick 이동: 현재 Tick에서 오른쪽(상승 방향)으로 이동

**2-2 스왑 방식: Exact Input vs Exact Output**

- 유저가 스왑을 요청할 때 입력하는 수량인 `amountSpecified`의 부호에 따라 연산의 목표가 달라진다.
- Exact Output (`amountSpecified > 0`):
    ◦ **의미:** "내가 정확히 1000개의 토큰을 받고 싶으니, 필요한 만큼만 내 지갑에서 빼가."
    ◦ 연산 기준: Output이 목표치에 도달할 때까지 역산하여 필요한 입력량과 수수료를 계산한다.
- Exact Input (`amountSpecified < 0`):
    - **의미:** "내가 정확히 1000개의 토큰을 넣을  테니, 나오는 만큼 줘."
    - 연산 기준**:** input에서 수수료를 선공제한 후, 남은 수량이 0이 될 때까지 스왑을 진행한다.

**2-3. Swap Loop와 단계별 실행**
Swap은 한 번의 수식 계산으로 끝나지 않는다. 현재 가격에서 다음 활성화된 Tick까지의 구간을 하나의 Step으로 정의하고, 이 단계를 반복하는 `while` 루프로 구현되어 있다.

- 목표 Tick 찾기: 현재 가격 기준으로 `zeroForOne` 방향에 있는 가장 가까운 활성 Tick을 찾는다.
- Step 계산: `SwapMath.computeSwapStep()`을 호출하여 해당 Tick까지 도달하는 데 필요한 토큰 입력량, 출력량, 수수료를 계산한다.
- Tick 교차: 만약 목표 Tick에 도달했다면 Tick을 넘어가며(Crossing), 해당 Tick에 걸려있던 유동성을 전체 활성 유동성에 더하거나 뺀다.
- 반복: `amountSpecified`가 모두 소진되거나 지정한 가격 제한(`sqrtPriceLimitX96`)에 도달할 때까지 1~3을 반복한다.

#### 3. Entry Point

Swap의 진입점은 `PoolManager` 컨트랙트로 `swap()` 함수를 통해 시작된다. 이곳은 복잡한 계산보다는 전체적인 흐름 관리를 담당한다.

![](/assets/img/image-23.png)


이 코드에서 주목할 점은 다음과 같다.

- `_swap()` 호출 전후로 `before`와 `after` 훅을 실행하여 커스텀 로직이 개입할 시점을 제공한다.
- 복잡한 Tick 계산과 상태 변경 로직은 `Pool` Library 함수로 분리하여 처리한다. (`_swap` 안에서 호출함)
- Library가 계산해준 결과를 `_accountPoolBalanceDelta` 함수로 넘겨, 유저의 장부에 빚 또는 자산을 기록한다.

이제 `pool.swap` 내부로 들어가, 실제 유동성이 어떻게 계산되고 저장되는지 몇가지 부분으로 나눠서 살펴보자.

#### 4. Code Execution Flow

![](/assets/img/image-24.png)


**Initial State Variables**

- `Slot0 slot0Start = self.slot0;` **:** Pool 상태를 가져온다. (sqrtPriceX96, tick, fee settings, etc.)
- `bool zeroForOne = params.zeroForOne;`: swap 방향을 설정한다.

**Protocol Fee Calculation**

- `uint256 protocolFee = zeroForOne ? slot0Start.protocolFee().getZeroForOneFee() : slot0Start.protocolFee().getOneForZeroFee();` : swap 방향에 따라 protocol fee 값을 가져온다. 이는 LP들을 위한게 아니라 protocol 운영을 위한 fee로 들어간다.

**Amount Tracking**

- `int256 amountSpecifiedRemaining = params.amountSpecified;`: user가 swap하고 싶은 금액으로 swap 과정중에 감소된다.
- `int256 amountCalculated = 0;`: 이 값은 output 금액이 swap이 진행되면서 누적된다.

**Initializing Swap State**

- `result.sqrtPriceX96 = slot0Start.sqrtPriceX96();`: 현재 금액을 가져와서 swap을 진행 과정에 사용한다.
- `result.tick = slot0Start.tick();`: 현재 Tick 값
- `result.liquidity = self.liquidity;`: pool에 현재 유동성 값

![](/assets/img/image-25.png)


**Fee Configuration**

- Hook에서 fee가 override되었는지 확인하고 아니면 slot0꺼를 그대로 사용한다.
- `params↑.lpFeeOverride.isOverride()` : hook에서 fee를 override했는지 확인한다.
- `params.lpFeeOverride.removeOverrideFlagAndValidate()`: override가 되었다면 flag를 지우고 fee가 유효한지 검증한 다음 fee 값을 반환해준다.
- `swapFee = protocolFee == 0 ? lpFee : uint16(protocolFee).calculateSwapFee(lpFee);` : protocolFee가 없으면 lpFee가 swapFee가 되고, 만약 있다면 lpFee와 protocolFee를 합쳐서 fee를 만든다.

**Swap Validation Safeguard**

- `swapFee >= SwapMath.MAX_SWAP_FEE` : `amountSpecified > 0` 이면 Exact Output이다. 만약 swapFee가 100% 넘을때,  user가 넣어야 할 input 값을 계산하는 과정에서 수학적으로 오류가 생겨 revert 난다. 아래가 input 값 계산 공식이다.

$$
Total\ Input = \frac{Net\ Input}{1 - Fee\ Rate}
$$

- 마지막으로 `amountSpecified == 0`이라면 swapFee를 없애서 오류가 발생하지 않게 만든다.

![](/assets/img/image-26.png)


방향에 따라 price 검증을 한다.

1. zeroForOne (Swapping token0 → token1)
    - Price가 감소한다.
    - `Limit >= sqrtPriceX96`  → PriceLimitAlreadyExceeded
    - `Limit <= MIN_SQRT_PRICE` → PriceLimitOutOfBounds
2. !zeroForOne (Swapping token1 → token0)
    - Price가 증가한다.
    - `Limit <= sqrtPriceX96` → PriceLimitAlreadyExceeded
    - **`limit ≥ MAX_SQRT_PRICE`** → PriceLimitOutOfBounds

```solidity
// continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
while (!(amountSpecifiedRemaining == 0 || result.sqrtPriceX96 == params.sqrtPriceLimitX96)) {
    step.sqrtPriceStartX96 = result.sqrtPriceX96;

    (step.tickNext, step.initialized) =
        self.tickBitmap.nextInitializedTickWithinOneWord(result.tick, params.tickSpacing, zeroForOne);

    // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
    if (step.tickNext <= TickMath.MIN_TICK) {
        step.tickNext = TickMath.MIN_TICK;
    }
    if (step.tickNext >= TickMath.MAX_TICK) {
        step.tickNext = TickMath.MAX_TICK;
    }

    // get the price for the next tick
    step.sqrtPriceNextX96 = TickMath.getSqrtPriceAtTick(step.tickNext);

    // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
    (result.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
        result.sqrtPriceX96,
        SwapMath.getSqrtPriceTarget(zeroForOne, step.sqrtPriceNextX96, params.sqrtPriceLimitX96),
        result.liquidity,
        amountSpecifiedRemaining,
        swapFee
    );

    // if exactOutput
    if (params.amountSpecified > 0) {
        unchecked {
            amountSpecifiedRemaining -= step.amountOut.toInt256();
        }
        amountCalculated -= (step.amountIn + step.feeAmount).toInt256();
    } else {
        // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
        unchecked {
            amountSpecifiedRemaining += (step.amountIn + step.feeAmount).toInt256();
        }
        amountCalculated += step.amountOut.toInt256();
    }

    // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
    if (protocolFee > 0) {
        unchecked {
            // step.amountIn does not include the swap fee, as it's already been taken from it,
            // so add it back to get the total amountIn and use that to calculate the amount of fees owed to the protocol
            // cannot overflow due to limits on the size of protocolFee and params.amountSpecified
            // this rounds down to favor LPs over the protocol
            uint256 delta = (swapFee == protocolFee)
                ? step.feeAmount // lp fee is 0, so the entire fee is owed to the protocol instead
                : (step.amountIn + step.feeAmount) * protocolFee / ProtocolFeeLibrary.PIPS_DENOMINATOR;
            // subtract it from the total fee and add it to the protocol fee
            step.feeAmount -= delta;
            amountToProtocol += delta;
        }
    }

    // update global fee tracker
    if (result.liquidity > 0) {
        unchecked {
            // FullMath.mulDiv isn't needed as the numerator can't overflow uint256 since tokens have a max supply of type(uint128).max
            step.feeGrowthGlobalX128 +=
                UnsafeMath.simpleMulDiv(step.feeAmount, FixedPoint128.Q128, result.liquidity);
        }
    }

    // Shift tick if we reached the next price, and preemptively decrement for zeroForOne swaps to tickNext - 1.
    // If the swap doesn't continue (if amountRemaining == 0 or sqrtPriceLimit is met), slot0.tick will be 1 less
    // than getTickAtSqrtPrice(slot0.sqrtPrice). This doesn't affect swaps, but donation calls should verify both
    // price and tick to reward the correct LPs.
    if (result.sqrtPriceX96 == step.sqrtPriceNextX96) {
        // if the tick is initialized, run the tick transition
        if (step.initialized) {
            (uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128) = zeroForOne
                ? (step.feeGrowthGlobalX128, self.feeGrowthGlobal1X128)
                : (self.feeGrowthGlobal0X128, step.feeGrowthGlobalX128);
            int128 liquidityNet =
                Pool.crossTick(self, step.tickNext, feeGrowthGlobal0X128, feeGrowthGlobal1X128);
            // if we're moving leftward, we interpret liquidityNet as the opposite sign
            // safe because liquidityNet cannot be type(int128).min
            unchecked {
                if (zeroForOne) liquidityNet = -liquidityNet;
            }

            result.liquidity = LiquidityMath.addDelta(result.liquidity, liquidityNet);
        }

        unchecked {
            result.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
        }
    } else if (result.sqrtPriceX96 != step.sqrtPriceStartX96) {
        // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
        result.tick = TickMath.getTickAtSqrtPrice(result.sqrtPriceX96);
    }
}
```

**Swap While Loop code 설명**

1. Loop 조건
    - `amountSpecifiedRemaining != 0` : 아직 swap 할 amount가 남아있지 않으면 Loop 끝
    - `result.sqrtPriceX96 == params.sqrtPriceLimitX96` : user가 정한 limit과 같으면 Loop 끝
2. Step 마다 초기화
- `step.sqrtPriceStartX96` 값을 현재 price인 `result.sqrtPriceX96`로 초기화
1. 다음 Tick 찾기
    - `nextInitializedTickWithinOneWord()` 함수를 통해서 유동성있는 다음 Tick을 찾아주고 초기화되었는지 flag를 준다.
    - 다음 Tick이 `MIN_TICK`과 `MAX_TICK` 범위 안에 있는지 검사한다.
2. 다음 Tick으로 Price 변경
    - `step.sqrtPriceNextX96` 값을 `TickMath.getSqrtPriceAtTick` 값으로 변경
3. Swap Step 계산
    - `SwapMath.computeSwapStep()` 함수를 통해서 swap step 계산을 하게 된다.
    - 현재 price (`result.sqrtPriceX96`)
    - Target price (다음 tick price 또는 user의 price limit 중에 swap 방향으로 더 가까운 곳)
    - 현재 liquidity (`result.liquidity`)
    - swap으로 바꿔야할 남은 amount (`amountSpecifiedRemaining`)
    - swap fee (`swapFee`)
    
    반환값
    
    - step이후 현재 price (`result.sqrtPriceX96`)
    - 이번 step에서 사용될 Input amount (`step.amountIn`)
    - 이번 step에 나올 amount (`step.amountOut`)
    - Fee amount (`step.feeAmount`)
4. Update Amounts
    - `amountSpecifiedRemaining`**:** 유저가 처음에 고정한 목표 수량 중, 스왑 단계를 거치면서 아직 채우지 못하고 '남은 수량'을 추적하는 변수. (최종적으로 0을 향해 수렴함)
    - `amountCalculated`**:** 유저가 고정하지 않은 '반대쪽 토큰'의 수량이 스왑 단계를 거치며 얼마나 발생했는지 누적해서 합산하는 변수.
    
    여기에선 2가지 경우에 대해 따져본다.
    
    1. Exact Output **(**`params.amountSpecified > 0`)
        
        user가 특정 토큰을 "정확히 이만큼 받겠다"고 출력량을 고정한 상태이다.
        
    - `amountSpecifiedRemaining` (남은 목표 출력량 갱신)
        - 초기 상태: 목표 출력량인 양수로 시작한다.
        - `amountSpecifiedRemaining -= step.amountOut`
        - 이번 step에서 pool로부터 확보한 출력량 값인 `step.amountOut`을 빼줌으로써 점점 0으로 간다.
    - `amountCalculated` (필요한 총 입력량 누적)
        - 초기 상태: 0으로 시작해.
        - `amountCalculated -= (step.amountIn + step.feeAmount)`
        - 해당 출력량을 얻기 위해 유저가 pool에 지불해야 할 input과 수수료를 뺌으로써 기록한다. 결과적으로 이 변수는 점점 더 큰 음수가 된다.
    1. **Exact Input(**`params.amountSpecified <= 0`)
        
        user가 특정 토큰을 "정확히 이만큼 지불하겠다"고 입력량을 고정한 상태이다.
        
    - `amountSpecifiedRemaining` (남은 목표 입력량 갱신)
        - 초기 상태: 내 지갑에서 빠져나가는 수량이므로 음수로 시작한다.
        - `amountSpecifiedRemaining += (step.amountIn + step.feeAmount)`
        - 이번 단계에서 스왑과 수수료로 소비된 수량을 기존의 음수 값에 더해줌으로써, 남은 지불 목표치를 점점 0에 가깝게 줄여나간다.
    - `amountCalculated` (확보한 총 출력량 누적)
        - 초기 상태: 0으로 시작한다.
        - `amountCalculated += step.amountOut`
        - 입력량을 지불하고 풀에서 받게 될 출력량을 더함으로써 기록한다. 결과적으로 이 변수는 점점 더 큰 양수가 된다.
5. Protocol Fee Handling
    
    ProtocolFee가 있을 때, LP Fee와 ProtocolFee를 나누는 과정이다.
    
    - `swapFee == protocolFee` : protocolFee만 있으므로 `step.feeAmount` 모두 protocol에게 fee가 돌아간다.
    - `!(swapFee == protocolFee)` : step.feeAmount에서 LP 몫과 protocol 몫을 나누어준다. `amountToProtocol`에 protocolFee가 들어가고, `step.feeAmount`에 LP Fee가 들어간다.
6. Update Fee Growth Global
    - `step.feeGrowthGlobalX128 += UnsafeMath.simpleMulDiv(step.feeAmount, FixedPoint128.Q128, result.liquidity);`
    - 앞에서 구분한 LP Fee인 `step.feeAmount`를 유동성 1단위당 얼마나 나눠 가질지 `step.feeGrowthGlobalX128`에 누적시켜두는 과정이다.
7. Tick Transition
    
    step 하나를 끝내고 다음 tick으로 넘어갈지와 수수료와 상태 갱신하는 과정이다.
    
    1. 만약 `result.sqrtPriceX96 == step.sqrtPriceNextX96` swap 연산 결과로 도달한 tick과 목표했던 다음 tick과 같으면 아래 내용을 수행한다.
    - 그런데 만약 `step.initialized` 활성화된 tick이라면
        - 아래 코드는 앞에서 구한 수수료가 token0인지 token1인지 `zeroForOne`에 따라 할당해준다.
        
        ```
        (uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128) = zeroForOne
            ? (step.feeGrowthGlobalX128, self.feeGrowthGlobal1X128)
            : (self.feeGrowthGlobal0X128, step.feeGrowthGlobalX128);
        ```
        
        - `zeroForOne` 방향에 따라 유동성을 update 해준다.
        
        ```
        unchecked {
            if (zeroForOne) liquidityNet = -liquidityNet;
        }
        result.liquidity = LiquidityMath.addDelta(result.liquidity, liquidityNet);
        ```
        
    - tick 위치 보정을 해준다.
        - `zeroForOne == true` : `result.tick = step.tickNext - 1`
            - 가격이 하락하여 특정 tick 경계선을 왼쪽으로 뚫고 나갔기 때문에, 현재 위치는 해당 틱의 바로 아래 구간에 속하게 된다. 따라서 인덱스에서 `-1`을 해준다.
        - `zeroForOne == false` : `result.tick = step.tickNext`
            - 가격이 상승하여 오른쪽(상방)으로 경계선을 뚫었으므로, 해당 틱의 인덱스를 그대로 현재 위치 구간으로 사용한다.
    1. 만약 swap 연산 결과 목표했던 다음 tick에 도달하지 못하면?
    - `amountSpecified`이 먼저 고갈되어, 목표 tick 가격에 도달하지 못하고 중간에 멈춘 경우이다.
    - tick을 crossing 할 필요 없으므로 `crossTick`은 실행되지 않는다. 대신 최종적으로 멈춘 가격(`result.sqrtPriceX96`)을 기준으로 `TickMath.getTickAtSqrtPrice` 수학 라이브러리를 돌려, 현재 속해있는 tick 인덱스를 재계산하여 업데이트한다.

![](/assets/img/image-27.png)


**Swap 후 상태 업데이트**

1. slot0 업데이트
    
    새로운 tick, sqrtPriceX96을 pool에 저장해놓는다.
    
2. Liquidity 조정
    
    swap 중에 유동성이 변경되면, 반영한다.
    
3. Fee Accounting
    
    `zeroForOne` 방향에 따라 수수료 업데이트 한다.
    
4. Balance Delta Calculation
    
    먼저 2가지의 수량 데이터가 있다.
    
    - `params.amountSpecified - amountSpecifiedRemaining` : 처음에 고정한 목표량과 남은 수량의 차이로 swap loop를 돌며 사용되거나 채워진 **지정 토큰 수량**이다.
    - `amountCalculated` : swap loop를 돌며 계산된 반대 토큰의 수량이다.
    
    이곳에서는 4가지 경우를 하나의 조건문으로 처리했다.
    
    `zeroForOne` : `true`면 token0 매도, `false`면 token1 매도
    
    `params.amountSpecified` < 0 : `true`면 ExactInput, `false`면 ExactOutput
    
    - 조건식이 false일 때, else 로직 실행
        
        user가 수량을 지정한 토큰이 token0인 경우
        
        - `true != true` (Exact Input, token0 매도)
        user가 token0을 정확히 지불하겠다고 지정했다. 따라서 token0 자리에 지정 토큰 처리량을 넣고, token1 자리에 엔진이 계산한 Output 수량을 넣는다.
        - `false != false` (Exact Output, token1 매도)
        user가 token0을 정확히 받겠다고 지정했다. 따라서 token0 자리에 지정 토큰 처리량을 넣고, token1 자리에 엔진이 계산한 Input 수량을 넣는다.
    - 조건식이 ture일 때, if 로직 실행
        - user가 수량을 지정한 토큰이 token1인 경우
        - `false != true` (Exact Input, token1 매도)
        user가 token1을 정확히 지불하겠다고 지정했다. 따라서 token1 자리에 지정 토큰 처리량을 넣고, token0 자리에 계산된 Output 수량을 넣는다.
        - `true != false` (Exact Output, token0 매도)
        user가 token1을 정확히 받겠다고 지정했다. 따라서 token1 자리에 지정 토큰 처리량(Output)을 넣고, token0 자리에 계산된 Input 수량을 넣는다.

## Conclusion

이번 Uniswap V4 코드를 분석하며 체감한 변화의 핵심은 가스비 최적화에 대한 고민과 플랫폼으로의 아키텍처 전환이다.

먼저, 연산 로직 전반에서 V3보다 가스 소모를 줄이기 위해 구조적으로 고민한 흔적들이 계속해서 확인된다. 다양한 기능이 추가되었음에도 최적화를 통해 비용을 낮추려 노력한 점이 코드 곳곳에서 돋보인다.

가장 눈여겨볼 부분은 새롭게 도입된 Hook 아키텍처다. 이는 외부 개발자들이 Uniswap을 핵심 Core로 삼아 새로운 금융 기능을 손쉽게 개발하고 연동할 수 있는 환경을 만들어 주었다. 일반 유저의 유동성과 서드파티 개발자의 로직을 직접 연결해 준다는 점에서, 이제 Uniswap은 단순한 dApp을 넘어 하나의 생태계 플랫폼으로 자리 잡으려는 것으로 보인다.

결과적으로 이번 V4 업데이트는 Uniswap이 앞으로 DEX 시장의 중심 인프라로서 확고한 자리를 잡기 위한 행보라고 생각한다.

## References

https://www.quillaudits.com/research/uniswap-development/uniswap-v4/liquidity-mechanics-in-uniswap-v4-core

https://uniswapv3book.com/index.html

https://hyun-jeong.medium.com/uniswap-series-2-cpmm-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-4a82de8aba9

https://hyun-jeong.medium.com/uniswap-series-3-%EC%9C%A0%EB%8B%88%EC%8A%A4%EC%99%91-v3-%ED%86%BA%EC%95%84%EB%B3%B4%EA%B8%B0-a058235823e3

https://medium.com/decipher-media/uniswap-v4-deep-dive-1-%EC%BD%94%EB%93%9C%EB%A5%BC-%ED%86%B5%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EB%8A%94-v4-2881fa0358aa

https://docs.uniswap.org/

https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive

https://medium.com/@organmo/uniswap-v4-deep-dive-2-uniswap-v4%EC%9D%98-%ED%98%84-%EC%83%81%ED%99%A9%EA%B3%BC-%EC%A0%84%EB%A7%9D-7205d4bcdabb

