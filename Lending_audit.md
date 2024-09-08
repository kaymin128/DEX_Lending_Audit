# Lending 버그 리포트

## 1. `liquidate` 함수의 잘못된 돈 전송 로직

### 설명
`liquidate` 함수에서 `msg.sender`에게서 돈을 가져오지 않고 `msg.sender`에게 토큰을 전송한다. 즉 부채를 갚지 않고 담보를 전송하는 것이므로, 공격자 입장에서 큰 돈을 `borrow`하면 `liquidate`함수에서 `max_liquidation`이 커져 큰 `amount`만큼 돈을 인출해갈 수 있다.

- **파일명:** [`DreamAcademyLending.sol`](https://github.com/kaymin128/lending_solidity/blob/main/src/DreamAcademyLending.sol)
- **함수명:** `liquidate`
- **줄 번호:** 136

### 파급력
**심각도:** High  
빚을 청산하는 것이므로 `msg.sender`에게서 돈을 가져오고 돈을 받아야 하는 것인데, 돈을 가져오는 행위가 없으므로 잘못된 시스템이다. 큰 돈을 인출하는 공격이 가능하기에 매우 심각한 문제이다.

### 해결방안
`transferFrom`을 사용하여 `msg.sender`에게서 돈을 가져와 부채를 갚고, `reward`를 그 보상으로 `msg.sender`에게 전송해야 한다.

---

## 2. `liquidate` 함수의 잘못된 청산 로직

### 설명
`liquidate` 함수에서 사용자의 부채가 100 ETH 이상일 경우 빚의 1/4만 청산할 수 있으며, 그 이하일 경우에는 모두 청산된다. 하지만 그렇기에 빚이 작을 때 과한 청산을 유발할 수 있으며, 이는 사용자에게 불리한 결과를 가져올 수 있다.

- **파일명:** [`DreamAcademyLending.sol`](https://github.com/kaymin128/lending_solidity/blob/main/src/DreamAcademyLending.sol)
- **함수명:** `liquidate`
- **줄 번호:** 125-130

### 파급력
**심각도:** Medium  
이 문제로 인해 빚이 적은 사용자가 과도한 청산을 겪을 수 있으며, 이로 인해 사용자는 예기치 않은 자산 손실을 입을 수 있다.

### 해결방안
빚에 관계없이 항상 `max_liquidation=users[user].borrowed_usdc/4`로 바꿔주면 된다.

---