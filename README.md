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
