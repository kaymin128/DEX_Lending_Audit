# DEX 버그 리포트

## 1. `removeLiquidity` 함수의 LP 토큰 소각 후 자산 전송 문제

### 설명
`removeLiquidity` 함수에서 LP 토큰이 소각된 후에야 사용자가 자산을 수령하기 때문에 자산 수령이 실패할 경우, 이미 소각된 LP 토큰을 복구할 방법이 없기에 사용자가 손실을 입을 수 있다.

- **파일명:** [DEX.sol](https://github.com/pluto1011/-Lending-DEX-_solidity/blob/main/src/DEX.sol)
- **함수명:** `removeLiquidity`
- **줄 번호:** 77-79

### 파급력
**심각도:** High  
이 버그는 사용자가 유동성 제거 시 자산을 수령하지 못하면, LP 토큰만 소각되어 자산 손실로 이어질 수 있기 때문에 심각한 문제가 된다.

### 해결방안
LP 토큰 소각을 자산 수령 후로 이동시켜, 자산이 성공적으로 전송된 후에만 LP 토큰이 소각되도록 수정하여 해결할 수 있다. 즉 `require(IERC20(token).transfer(msg.sender, amount), "transfer fail");` 로 require 문을 먼저 작성하고, 그 이후에 `_burn(msg.sender, lpAmount);` 로 소각을 실행하는 것이 안전하다.
마찬가지로, 101-102 줄의 `transferFrom` 함수와 `transfer` 함수를 사용하는 부분에서도 require 문 안에 작성하여 성공했을때만 트랜잭션이 계속돼서 실행되도록 하는게 안전해 보인다.

---

## 2. 불필요한 `IERC120` 인터페이스 사용

### 설명
이미 6번 줄에 `IERC20` 인터페이스를 import 하였기 때문에 굳이 인터페이스 내용을 같이 적을 필요가 없다.

- **파일명:** [Dex.sol](https://github.com/pluto1011/-Lending-DEX-_solidity/blob/main/src/DEX.sol)
- **인터페이스명:** `IERC120`
- **줄 번호:** 133-147

### 파급력
**심각도:** Low
문제가 되지 않기에 심각도는 낮다. 효율성이 떨어지고 가스비용이 증가할 뿐이다.

### 해결방안
아래에 적은 `IERC120` 인터페이스 부분만 삭제해도 효율성이 증가할 것 같다.

---

## 3. 불필요한 `SafeMath` 및 `Math` 라이브러리 사용

### 설명
코드에서 `mul`를 위해 `SafeMath` 라이브러리를 사용했는데, `mul`은 사실 `*` 기호로 간단하게 연산 가능하였기에, `SafeMath` 라이브러리 사용이 불필요해 보인다. 코드가 길지 않은 `sqrt` 함수 하나때문에 라이브러리를 사용하는 것보다, 그냥 이 함수만 따로 가져오거나 하는게 더 합리적으로 보였다. 추가로 `Math` 라이브러리 또한 import 할 필요가 없어 보였다.

- **파일명:** [Dex.sol](https://github.com/pluto1011/-Lending-DEX-_solidity/blob/main/src/DEX.sol)
- **라이브러리명:** `SafeMath`
- **줄 번호:** 4, 7, `SafeMath` 사용 부분

### 파급력
**심각도:** Low  
코드의 효율성이 저하된다.

### 해결방안
`SafeMath` 라이브러리의 코드가 한 파일에 작성되어 있는데, 이를 제거하여 코드를 간소화하고 가스 비용을 줄일 수 있다. `mul`은 `*` 기호로 대체하고 `sqrt` 함수만 따로 가져와 해당 라이브러리를 import 하여 사용하지 않고, `Math.sol`을 아예 쓰지 않는 것이 더 효율적으로 보인다.

---

## 4. 불필요한 `_update` 함수의 사용

### 설명
`removeLiquidity` 함수에서 유동성을 제거할 때, `_update` 함수가 두 번 호출되고 있다. 첫 번째 호출은 LP 토큰 소각 후, 두 번째 호출은 토큰 전송 후 이루어진다. 불필요하게 두 번 유동성 풀 상태를 업데이트하게 되어 가스 비용이 증가할 수 있다. 그리고 `transferFrom`과 `transfer` 함수가 정상 동작한다면 `_update` 함수의 사용이 불필요해 보인다.

- **파일명:** [Dex.sol](https://github.com/skskgus/Dex_solidity/blob/main/src/Dex.sol)
- **함수명:** `_update`
- **줄 번호:** 32-35, 103, 108

### 파급력
**심각도:** Low  
이 문제는 유동성 풀 상태를 불필요하게 두 번 업데이트하여 불필요한 가스 비용을 만든다. 만약 `transfer`과 `transferFrom` 함수를 사용시 `require` 문을 사용하여 전송이 성공했는지 확인한다면, `_update`함수의 사용이 불필요해 보인다.

### 해결방안
`_update` 함수를 한 번만 호출하도록 변경하여, LP 토큰 소각과 자산 전송 후에 유동성 풀 상태를 업데이트하도록 하면 가스 비용을 절감할 수 있다. 더 나아가 `transfer`과 `transferFrom` 함수를 사용시 `require` 문을 사용하여 전송이 성공했는지 확인하는 로직으로 변경한다면, 더 안전하게 트랜잭션을 실행시킬 수 있고 `_update` 함수도 사용하지 않아도 되기에 코드의 효율성과 가스비 절감 측면에서 바람직할 것 같다.

---

## 5. `swap` 함수에서 잘못된 입력값 검사

### 설명
`swap` 함수에서 `amount0Out`과 `amount1Out` 중 하나만 0이어야 하지만, 이 조건을 `require(amount0Out == 0 || amount1Out == 0, "Invalid input")`로 검증하고 있다. 그러나, 이 조건은 두 값이 모두 0인 경우에도 통과하게 되기에, 이는 스왑이 올바르게 작동하지 않을 수 있는 상황을 발생시킬 수 있다.

- **파일명:** [Dex.sol](https://github.com/skskgus/Dex_solidity/blob/main/src/Dex.sol)
- **함수명:** `swap`
- **줄 번호:** 112

### 파급력
**심각도:** Medium  
이 문제로 인해 잘못된 스왑 요청이 허용될 수 있으며, 이는 유동성 풀의 상태를 의도치 않게 변경시킬 수 있다.

### 해결방안
입력 검증 로직을 수정하여 `amount0Out`과 `amount1Out`이 동시에 0이 아닌지, 그리고 두 값 중 하나만 0인지를 정확하게 검증할 수 있다. 예를 들면 이전 줄에 `require(!(amount0Out==0 && amount1Out==0), "both are zero");` 라고 작성하여 둘중 하나만 0의 값을 갖도록 할 수 있다.

---

## 6. 불필요한 라이브러리의 함수들 가져오기

### 설명
`_mint`, `_burn`, `_update`의 함수들을 그대로 가져왔는데, 이 함수들이 사실 `ERC20.sol`에 적혀 있기에 또 작성하는 것이 불필요해 보인다.

- **파일명:** [Dex.sol](https://github.com/skskgus/Dex_solidity/blob/main/src/Dex.sol)
- **함수명:** `_mint`, `_burn`, `_update`
- **줄 번호:** 22-35

### 파급력
**심각도:** Low  
코드의 양이 늘어나는 문제이고 효율성 문제이기에, 심각하지 않은 문제이다.

### 해결방안
`_mint`, `_burn`, `_update`의 함수들을 그대로 가져왔는데, 이 함수들이 사실 `ERC20.sol`에 적혀 있기에 그냥 이 파일을 임포트하고 함수를 적은 부분을 삭제하는 것이 코드 크기를 줄이고 효율성을 증가시킬 것이다.

---

## 7. `removeLiquidity` 함수에서의 `_burn` 함수 사용 시 언더플로우 가능성

### 설명
`_burn` 함수에서 LP 토큰을 소각할 때, 소각량이 계정의 잔액보다 클 경우 언더플로우가 발생할 수 있다.

- **파일명:** [Dex.sol](https://github.com/skskgus/Dex_solidity/blob/main/src/Dex.sol)
- **함수명:** `_burn`
- **줄 번호:** 27-30

### 파급력
**심각도:** High
계정의 잔액이 적거나 소각량이 커 언더플로우가 발생하는 경우, 계정의 잔액이 매우 큰 값으로 저장될 수 있기에 치명적인 취약점이라고 생각된다.

### 해결방안
LP 토큰 소각 전에 소각량이 계정 잔액보다 크지 않은지 확인하는 로직을 추가적으로 작성해야 한다. `require` 문을 사용하여 `shares`가 `msg.sender`의 잔액 이하인지 확인하는 로직을 추가하는게 좋겠다.

---

## 8. 불필요한 `IERC20` 라이브러리 사용

### 설명
이미 `ERC20` 라이브러리를 import 해서 사용하고 있기에 인터페이스 사용이 불필요해 보인다. 불필요한 가스 비용을 야기할 수 있다.

- **파일명:** [Dex.sol](https://github.com/hakid29/Dex_solidity/blob/main/src/Dex.sol)
- **라이브러리명:** `IERC20`
- **줄 번호:** 4

### 파급력
**심각도:** Low
심각한 문제가 아니라, 그냥 불필요한 import 문제이다.

### 해결방안
이 코드는 다른 코드들과 달리 인터페이스가 아닌 `ERC20`을 import 하고 `Dex` 컨트랙트에서 상속하여 그점이 가독성 면에서도, 효율성 면에서도 좋아 보였다. 그냥 4번 줄만 주석 처리하거나 삭제하면 되는 일이다. 