---
title: MultiToken contract
sidebar_position: 1
---
# MultiToken Contract Explanation

## Introduction

The AElf Fungible Token (FT) concept is implemented through smart contracts on the AElf blockchain platform, utilizing 
a MultiToken contract. This contract facilitates core functionalities, including `creation`, `issuance`, `transfer`, 
`transferFrom`, `approval`, `burning`, and `locking` of tokens. Aligned with industry practices, AElf FT adheres to the 
fungibility definition, ensuring each unit is interchangeable, possessing equal value and functionality.

In the AElf ecosystem, FTs represent digital assets, benefiting from the versatility and programmability offered by 
smart contracts. The MultiToken contract empowers developers to create customized fungible tokens tailored to specific 
business requirements. Users can seamlessly issue new FTs, conduct secure transfers, provide approvals, and perform 
actions like burning or locking tokens when needed. This modular design ensures adaptability to diverse scenarios, 
meeting various user and enterprise needs.

## Functionalities of AElf MultiToken Contract

### 1. Create

The `Create` functionality initiates the introduction of new fungible tokens to the AElf blockchain. This pivotal feature 
generates a key-value pair in the state.TokenInfo hashmap, ensuring symbol uniqueness and allowing customization based 
on specific business requirements.

### 2. Issue

The `Issue` functionality adds a specified quantity of tokens to a designated user, facilitating initial distribution and 
ongoing issuance within the AElf blockchain ecosystem. User balances are updated to reflect the issued amount, 
contributing to the total token supply.

### 3. Transfer

The `Transfer` function in the AElf MultiToken contract facilitates the secure and efficient transfer of a specified 
quantity of tokens from one user to another. This crucial functionality involves a meticulous implementation that 
adjusts the balances of both the sender and recipient addresses. By ensuring consistency and accuracy in token 
transfers, this function plays a fundamental role in fostering reliable peer-to-peer transactions within the AElf 
blockchain ecosystem.

```csharp
public override Empty Transfer(TransferInput input)
{
    AssertValidToken(input.Symbol, input.Amount);
    DoTransfer(Context.Sender, input.To, input.Symbol, input.Amount, input.Memo);
    DealWithExternalInfoDuringTransfer(new TransferFromInput
    {
        From = Context.Sender,
        To = input.To,
        Amount = input.Amount,
        Symbol = input.Symbol,
        Memo = input.Memo
    });
    return new Empty();
}
```

### 4. TransferFrom

The AElf MultiToken contract's `TransferFrom` function enhances the token transfer capability by introducing the concept 
of an allowance, setting it apart from the Transfer operation. For each user, spender, and token combination, the 
contract maintains an allowance in the state, organized as a Hashmap. This allowance specifies the maximum quantity of 
a particular token that the spender is authorized to transfer on behalf of the user. During the successful execution of 
TransferFrom, the contract deducts the transferred token amount from the corresponding allowance.

This functionality proves valuable in scenarios where users delegate permission to a third party for managing their 
tokens. The TransferFrom operation meticulously validates transaction parameters, encompassing details such as the 
spender, sender, recipient, and token amount. It ensures that the spender possesses the requisite authorization to 
execute the transfer, subsequently updating the balances of the sender and recipient. This refined feature significantly 
augments the flexibility and utility of the AElf MultiToken contract, providing robust management of token transfers 
within the blockchain ecosystem.

```csharp
public override Empty TransferFrom(TransferFromInput input)
{
    AssertValidToken(input.Symbol, input.Amount);
    // First check allowance.
    var allowance = State.Allowances[input.From][Context.Sender][input.Symbol];
    if (allowance < input.Amount)
    {
        if (IsInWhiteList(new IsInWhiteListInput { Symbol = input.Symbol, Address = Context.Sender }).Value)
        {
            DoTransfer(input.From, input.To, input.Symbol, input.Amount, input.Memo);
            DealWithExternalInfoDuringTransfer(input);
            return new Empty();
        }

        Assert(false,
            $"[TransferFrom]Insufficient allowance. Token: {input.Symbol}; {allowance}/{input.Amount}.\n" +
            $"From:{input.From}\tSpender:{Context.Sender}\tTo:{input.To}");
    }

    DoTransfer(input.From, input.To, input.Symbol, input.Amount, input.Memo);
    DealWithExternalInfoDuringTransfer(input);
    State.Allowances[input.From][Context.Sender][input.Symbol] = allowance.Sub(input.Amount);
    return new Empty();
}
```

### 5. Approval

The `Approval` function in AElf MultiToken is a pivotal feature that empowers users to grant permission to a designated 
spender, defining the maximum amount of tokens the spender is authorized to transfer on their behalf. This functionality 
proves particularly valuable in scenarios where users aim to delegate token management to third parties, such as 
decentralized applications (DApps) or smart contracts.

In the execution of the `Approval` operation, the contract dynamically updates the allowance mapping in its state. This 
process establishes a transparent record of authorized spenders along with their corresponding token limits. By doing 
so, the AElf MultiToken contract enhances the flexibility and security of token management within the broader AElf 
blockchain ecosystem, catering to diverse user needs and fostering a more robust and adaptable token management system.

```csharp
public override Empty Approve(ApproveInput input)
{
    AssertValidToken(input.Symbol, input.Amount);
    State.Allowances[Context.Sender][input.Spender][input.Symbol] = input.Amount;
    Context.Fire(new Approved
    {
        Owner = Context.Sender,
        Spender = input.Spender,
        Symbol = input.Symbol,
        Amount = input.Amount
    });
    return new Empty();
}
```

### 6. Burning


The `burning` functionality in the AElf MultiToken contract empowers users to permanently eliminate a specified quantity 
of tokens from circulation. The Burn function diligently validates parameters, ensuring that the sender possesses a 
sufficient balance for the burn operation. Upon successful execution, the contract deducts the burned tokens from the 
sender's balance and meticulously updates the total token supply.

`Burning` serves as a versatile feature, applied for various purposes such as reducing token supply, adjusting economic 
parameters, or implementing deflationary mechanisms. This capability significantly enhances the versatility and economic 
management aspects of the AElf MultiToken contract within the blockchain ecosystem, providing users and developers with 
a powerful tool for shaping the token dynamics in alignment with diverse scenarios and requirements.

```csharp
public override Empty Burn(BurnInput input)
{
    var tokenInfo = AssertValidToken(input.Symbol, input.Amount);
    Assert(tokenInfo.IsBurnable, "The token is not burnable.");
    ModifyBalance(Context.Sender, input.Symbol, -input.Amount);
    tokenInfo.Supply = tokenInfo.Supply.Sub(input.Amount);
    Context.Fire(new Burned
    {
        Burner = Context.Sender,
        Symbol = input.Symbol,
        Amount = input.Amount
    });
    return new Empty();
}
```

### 7. Locking

The locking functionality in AElf distinguishes itself from traditional time-based locks prevalent in other systems. 
Upon invocation of the `Lock` function, a designated amount of tokens is seamlessly transferred to a virtual address. 
Unlike conventional models, this locking mechanism operates without the imposition of a predefined time duration. The 
tokens persist at the virtual address until the execution of the `Unlock` function, wherein the locked tokens are 
gracefully returned to the user's address.

This innovative approach bestows users with a distinctive capability – the freedom to lock and unlock tokens devoid of 
the constraints imposed by a predetermined time period. This design choice enhances the flexibility in managing token 
access within the AElf blockchain ecosystem, providing users with a powerful and adaptable tool for tailored token 
utilization.

```csharp
public override Empty Lock(LockInput input)
{
    AssertSystemContractOrLockWhiteListAddress(input.Symbol);
    Assert(Context.Origin == input.Address, "Lock behaviour should be initialed by origin address.");
    var allowance = State.Allowances[input.Address][Context.Sender][input.Symbol];
    if (allowance >= input.Amount)
        State.Allowances[input.Address][Context.Sender][input.Symbol] = allowance.Sub(input.Amount);
    AssertValidToken(input.Symbol, input.Amount);
    var fromVirtualAddress = HashHelper.ComputeFrom(Context.Sender.Value.Concat(input.Address.Value)
        .Concat(input.LockId.Value).ToArray());
    var virtualAddress = Context.ConvertVirtualAddressToContractAddress(fromVirtualAddress);
    // Transfer token to virtual address.
    DoTransfer(input.Address, virtualAddress, input.Symbol, input.Amount, input.Usage);
    DealWithExternalInfoDuringLocking(new TransferFromInput
    {
        From = input.Address,
        To = virtualAddress,
        Symbol = input.Symbol,
        Amount = input.Amount,
        Memo = input.Usage
    });
    return new Empty();
}
```

## AElf MultiToken Contract V.S. ETH ERC-20 Contract

1. **Divergent Locking Mechanism**: 
- AElf's lock function transfers tokens to a virtual address without a time lock.
- ERC-20 typically implements time locks using smart contracts to restrict transfers for a specified period.

2. **Advanced Allowance Management with TransferFrom**: 
- AElf introduces the Allowance concept and TransferFrom function for flexible token transfers and permission management.
- ERC-20 achieves authorization through the approval function, with TransferFrom implementation varying between contracts.

3. **Modularity and Customization**: 
- AElf's modular design allows adaptability to various scenarios, supporting user-defined features and business logic.
- ERC-20 contracts have more standardized interfaces, potentially limiting flexibility compared to AElf contracts.

4. **Smart Contract Programming Language**: 
- AElf smart contracts are written in C#. 
- ERC-20 contracts are usually coded in Solidity.