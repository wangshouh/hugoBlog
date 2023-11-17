---
title: "Cairo 实战入门:Starknet Foundry 与组件语法"
date: 2023-11-16T11:47:33Z
tags: [cario,ERC-20]
---

## 概述

随着 [Starknet Foundry](https://github.com/foundry-rs/starknet-foundry) 的进一步更新，使用 `Starknet Foundry` 进行 Cairo 智能合约开发可能会逐渐成为未来主流。

本文的主要内容实际上是介绍 `cairo v2.3` 引入的 [Components](https://community.starknet.io/t/cairo-components/101136) 重大更新，但考虑 `Starknet Foundry` 的活跃开发，所以本文使用了 `Starknet Foundry` 作为开发框架，而不是与之前的文章一样使用 Cairo 自带的框架。

## 前置准备

读者可以在 Linux 或者 Mac 系统终端内运行以下命令:

```bash
curl -L https://raw.githubusercontent.com/foundry-rs/starknet-foundry/master/scripts/install.sh | sh
```

运行完成后，即可安装 `snforge` 和 `sncast` 两大工具。其中 `snforge` 用于合约项目的管理，比如触发测试等，而 `sncast` 则负责与 starknet 区块链进行通信，实现合约部署等功能。

读者应当在阅读后文前，对于常规的 ERC20 代币合约也有所了解，如果读者不了解，请自行阅读我之前编写的 [Cairo 实战入门:编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/cairo1-with-erc20/) 文章。

## 第一个组件

读者可以运行以下命令初始化并测试初始化项目:

```bash
snforge init erc20_component
cd erc20_component/
snforge test
```

测试应该可以正常通过。

接下来，我们编写第一个组件，在 `src` 文件夹下创建 `components/ownable.cairo` 文件，同时在 `src` 文件夹下创建 `components.cairo` 并在 `components.cairo` 内写入以下内容:

```rust
mod ownable;
```

在 `lib.cairo` 文件头部加入 `mod components;` ，将其导入根目录。

在 `src` 文件夹下创建 `contracts/hello_starknet.cairo` 文件，并将 `lib.cairo` 的以下内容剪切至 `hello_starknet.cairo` 文件内:

```rust
#[starknet::interface]
trait IHelloStarknet<TContractState> {
    fn increase_balance(ref self: TContractState, amount: felt252);
    fn get_balance(self: @TContractState) -> felt252;
}

#[starknet::contract]
mod HelloStarknet {
    #[storage]
    struct Storage {
        balance: felt252,
    }

    #[external(v0)]
    impl HelloStarknetImpl of super::IHelloStarknet<ContractState> {
        fn increase_balance(ref self: ContractState, amount: felt252) {
            assert(amount != 0, 'Amount cannot be 0');
            self.balance.write(self.balance.read() + amount);
        }

        fn get_balance(self: @ContractState) -> felt252 {
            self.balance.read()
        }
    }
}
```

创建 `src/contracts.cairo` 文件写入以下内容:

```rust
mod hello_starknet;
```

并在 `lib.cairo` 内写入以下内容:

```rust
mod contracts;
```

最后，修复 `tests/test_contract.cairo` 中的导入问题:

```rust
use erc20_component::contracts::hello_starknet::IHelloStarknetSafeDispatcher;
use erc20_component::contracts::hello_starknet::IHelloStarknetSafeDispatcherTrait;
```

至此我们完成所有的准备工作。

接下来，我们开始编写人生第一个组件 `ownable` 组件。在进行编码之前，我们首先介绍一下组件系统的作用。众所周知，在 solidity 语言中，借助面向对象的力量，我们可以通过继承实现良好的代码可组合性。我们可以编写一个基础的 ERC20 代币合约，然后通过继承和重载去实现一个具有更多功能的 ERC20 代币合约。

但在 cairo 中，我们没有面向对象的概念，而是遵循组合大于继承的理念，这对我们进行代码可组合性提出了一个挑战。Cairo 语言提出了一种新的语法规范——组件(`Component`)。组件无法单独部署，我们可以在智能合约内导入组件来实现类似 solidity 中的继承机制。

组件需要以此解决以下四个部分的可组合性:

1. 状态。组件所使用的状态可以自由嵌入智能合约中，为实现这一目标，我们在组件中使用 `#[storage]` 标识组件状态，在合约内使用 `#[substorage(v0)]` 嵌入组件的状态
2. 逻辑。组件可以自由实现各种函数，而我们需要在使用组件的智能合约中使用 `impl ERC20Impl = erc20_comp::IERC20<ContractState>;` 导入
3. ABI。与常规程序不太相同，ABI 对于智能合约而言非常重要，组件内也存在允许外部调用的函数，所以将此部分函数的 ABI 嵌入到导入组件的智能合约里是一个重要的话题，我们使用 `#[abi(embed_v0)]` 实现
4. 事件。将组件的事件嵌入到导入组件的智能合约中也是重要的，此处没有特殊语法，而是采用了直接导入的方案

在此处，我们可以看到 **组合大于继承** 与传统的继承机制的区别，由于 **组合大于继承** 机制下逻辑与状态是分离的，所以我们需要分开讨论。

> 需要读者注意的是，目前组件无法进行单独部署，所以对于组件的测试目前似乎还没有比较好的方案，目前常见的方案就是将组件导入智能合约内后对智能合约进行测试，如 [OpenZeppelin](https://github.com/OpenZeppelin/cairo-contracts/tree/main/src/tests) 就选择了此方案

接下来，我们尝试编写第一个组件，该组件是一个经典的 `ownable` 组件，其功能是存储一个管理员账户地址，并允许合约读取此管理员用户。此外，还提供了管理员地址转移的功能。以下代码均位于 `src/components/ownable.cairo` 中。

我们首先编写 ABI 接口:

```rust
use starknet::ContractAddress;

#[starknet::interface]
trait TransferTrait<TContractState> {
    fn owner(self: @TContractState) -> ContractAddress;
    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress);
}
```

分别用于读取组件中的管理员用户地址和转移管理员权限。注意此处定义的 ABI 在组件嵌入智能合约后会自动转化为合约的 ABI，即此处的 ABI 是允许外部用户调用的。在后文中，我们也会给出一些组件内部的函数实现。这些组件内部函数实现不会表现在 ABI 中，所以用户无法在外部调用，但是嵌入此组件的智能合约则可以从内部调用，类似 solidity 中的 `internal` 函数。

接下来，我们声明组件及其状态，代码如下:

```rust
#[starknet::component]
mod ownable {
    use starknet::ContractAddress;
    #[storage]
    struct Storage {
        owner: ContractAddress,
    }
}
```

此组件仅需要 `owner` 存储即可。接下来，我们实现合约的 ABI 对应的外部函数:

```rust
#[embeddable_as(Transfer)]
impl TransferImpl<
    TContractState, +HasComponent<TContractState>
> of super::TransferTrait<ComponentState<TContractState>> {
    fn owner(self: @ComponentState<TContractState>) -> ContractAddress {
        self.owner.read()
    }

    fn transfer_ownership(
        ref self: ComponentState<TContractState>, new_owner: ContractAddress
    ) {
        self.validate_ownership();
        self.owner.write(new_owner);
    }
}
```

此处的 `#[embeddable_as(Transfer)]` 是为此组件进行的命名，我们会在智能合约导入组件时使用此名称。

我们可以看到此处实现了 `super::TransferTrait<ComponentState<TContractState>>` 的 `trait` 就是我们上文定义的接口。此处的类型签名比较有意思，使用了 `TContractState, +HasComponent<TContractState>` ，该类型签名的含义是要求 `TContractState` 实现 `HasComponent<TContractState>` 相关 trait，而 `HasComponent` 的相关函数定义如下:


```rust
trait HasComponent<TContractState> {
    fn get_component(self: @TContractState) -> @ComponentState<TContractState>;
    fn get_component_mut(ref self: TContractState) -> ComponentState<TContractState>;
    fn get_contract(self: @ComponentState<TContractState>) -> @TContractState;
    fn get_contract_mut(ref self: ComponentState<TContractState>) -> TContractState;
    fn emit<S, impl IntoImp: traits::Into<S, Event>>(ref self: ComponentState<TContractState>, event: S);
}
```

简单来说，`HasComponent` 提供了组件内状态和事件与智能合约内状态和事件的桥梁，我们对组件内状态的修改会被映射到智能合约内。如果智能合约没有实现 `HasComponent` 则说明此智能合约没有引入该组件的条件。

> 在智能合约内，我们不需要手动实现 `HasComponent` ，而是使用 `component!()` 宏自动生成

上文代码给出的 `validate_ownership` 并没有实现，我们在此处给出其实现:

```rust
#[generate_trait]
impl OwnableHelperImpl<
    TContractState, +HasComponent<TContractState>
> of OwnableHelperTrait<TContractState> {
    fn init_ownable(ref self: ComponentState<TContractState>, owner: ContractAddress) {
        self.owner.write(owner);
    }
    fn validate_ownership(self: @ComponentState<TContractState>) {
        assert(self.owner.read() == starknet::get_caller_address(), 'Wrong owner.');
    }
}
```

此处的给出的实现也是在组件内实现内部函数的通用实现。注意，此处所谓的内部函数指用户无法通过合约调用触发的函数，内部函数实际上可以由导入此模块的智能合约执行。

最后，我们介绍如何在 `HelloStarknet` 合约内导入 `ownable` 组件。正如上文所述，我们需要将组件的状态、逻辑、ABI 和实践的四部分嵌入到 `HelloStarknet` 合约内部。

第一步，我们需要导入组件:

```rust
use erc20_component::components::ownable::ownable as ownable_comp;

component!(path: ownable_comp, storage: ownable_storage, event: Ownable);
```

此处的 `component!` 会自动生成组件对应的 `HasComponent` 

第二步，嵌入状态:

```rust
#[storage]
struct Storage {
    #[substorage(v0)]
    ownable_storage: ownable_comp::Storage,
    balance: felt252,
}
```

使用 `#[substorage(v0)]` 宏进行组件状态 `ownable_comp::Storage` 的嵌入

第三步，嵌入逻辑与 ABI:

```rust
#[abi(embed_v0)]
impl OwnershipTransfer = ownable_comp::Transfer<ContractState>;

impl OwnershipHelper = ownable_comp::OwnableHelperImpl<ContractState>;
```

此处我们导入了外部函数集合 `Transfer` 和内部函数集合 `OwnableHelperImpl` 两部分，并且使用 `#[abi(embed_v0)]` 将 `Transfer` 部分的 ABI 嵌入到 `HelloStarknet` 的 ABI 内。由于 `OwnershipHelper` 为内部函数的集合，所以没有使用 `#[abi(embed_v0)]` 宏。

第四步，嵌入事件:

```rust
#[event]
#[derive(Drop, starknet::Event)]
enum Event {
    Ownable: ownable_comp::Event,
}
```

使用了直接导入的逻辑，没有引入新的语法元素或者宏。

此处，我们需要为 `HelloStarknet` 增加构造器以初始化 `owner` 地址:

```rust
#[constructor]
fn constructor(ref self: ContractState, owner: ContractAddress,) {
    self.ownable_storage.init_ownable(owner);
}
```

同时要求 `increase_balance` 函数必须由 `owner` 操作:

```rust
fn increase_balance(ref self: ContractState, amount: felt252) {
    self.ownable_storage.validate_ownership();
    assert(amount != 0, 'Amount cannot be 0');
    self.balance.write(self.balance.read() + amount);
}
```

最后，我们编写一下测试。修改 `deploy_contract` 函数增加 `owner` 的初始化参数:

```rust
fn deploy_contract(name: felt252, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name);
    let args = array![owner.into()];
    contract.deploy(@args).unwrap()
}
```

首先进行正向测试:

```rust
#[test]
fn test_increase_balance() {
    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('HelloStarknet', owner);

    let safe_dispatcher = IHelloStarknetSafeDispatcher { contract_address };

    let balance_before = safe_dispatcher.get_balance().unwrap();
    assert(balance_before == 0, 'Invalid balance');

    start_prank(contract_address, owner);
    safe_dispatcher.increase_balance(42).unwrap();
    stop_prank(contract_address);
    
    let balance_after = safe_dispatcher.get_balance().unwrap();
    assert(balance_after == 42, 'Invalid balance');
}
```

`start_prank` 的作用类似 `foundry` 中的 `vm.startPrank` 函数，用于修改后续的请求的 `get_caller_address` 获取到地址。

当然，我们也需要进行反向测试:

```rust
#[test]
fn test_cannot_increase_balance_with_not_on() {
    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('HelloStarknet', owner);

    let safe_dispatcher = IHelloStarknetSafeDispatcher { contract_address };

    let balance_before = safe_dispatcher.get_balance().unwrap();
    assert(balance_before == 0, 'Invalid balance');

    match safe_dispatcher.increase_balance(42) {
        Result::Ok(_) => panic_with_felt252('Should have panicked'),
        Result::Err(panic_data) => {
            assert(*panic_data.at(0) == 'Wrong owner.', *panic_data.at(0));
        }
    };
}
```

注意此处我们没有使用常见的 `#[should_panic(expected: ('Wrong owner.', 'ENTRYPOINT_FAILED', ))]` 宏。此处反向测试使用了 `match` 的匹配，这是因为我们使用 `IHelloStarknetSafeDispatcher` 接口，与常规的 `IHelloStarknetDispatcher` 的接口不同，此接口的返回值类型为 `starknet::SyscallResult<T>` 类型，其完整类型定义如下:

```rust
#[derive(Copy, Drop, Serde, PartialEq)]
enum Result<T, E> {
    Ok: T,
    Err: E,
}
```

所以此处我们使用 `match` 关键词对此枚举类型进行匹配，使用 `assert(*panic_data.at(0) == 'Wrong owner.', *panic_data.at(0));` 检查其错误返回。如果错误返回与预期相符，则返回实际的报错。此处错误返回 `panic_data` 的类型为 `Span<felt252>`，而 `'Wrong owner.'` 类型为 `flet252`，为了实现类型的一致，我们使用了 `*panic_data.at(0)` 进行转换类型。

> `Span<felt252>` 事实上就是 `@Array<felt252>`，使用 `panic_data.at(0)` 获得的类型为 `@felt252`，我们需要进一步使用 `*` 进行解包装获得 `felt252` 类型。另一种写法为 `assert(panic_data.at(0) == @'Wrong owner.', *panic_data.at(0));`，此处我们将 `'Wrong owner.'` 转化类型以与 `panic_data.at(0)` 一致。

在此处，我们不加分析的直接给出 ERC20 组件的源代码:

```rust
use starknet::ContractAddress;

#[starknet::interface]
trait ERC20Trait<TCS> {
    fn get_name(self: @TCS) -> felt252;
    fn get_symbol(self: @TCS) -> felt252;
    fn get_decimals(self: @TCS) -> u8;
    fn get_total_supply(self: @TCS) -> u256;
    fn balance_of(self: @TCS, account: ContractAddress) -> u256;
    fn allowance(self: @TCS, owner: ContractAddress, spender: ContractAddress) -> u256;
    fn transfer(ref self: TCS, recipient: ContractAddress, amount: u256);
    fn transfer_from(
        ref self: TCS, sender: ContractAddress, recipient: ContractAddress, amount: u256
    );
    fn approve(ref self: TCS, spender: ContractAddress, amount: u256);
    fn increase_allowance(ref self: TCS, spender: ContractAddress, added_value: u256);
    fn decrease_allowance(ref self: TCS, spender: ContractAddress, subtracted_value: u256);
}

#[starknet::component]
mod erc20 {
    use starknet::{ContractAddress, get_caller_address, contract_address_const};
    #[storage]
    struct Storage {
        name: felt252,
        symbol: felt252,
        decimals: u8,
        total_supply: u256,
        balances: LegacyMap::<ContractAddress, u256>,
        allowances: LegacyMap::<(ContractAddress, ContractAddress), u256>,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        Transfer: TransferEvent,
        Approval: ApprovalEvent,
    }
    #[derive(Drop, starknet::Event)]
    struct TransferEvent {
        from: ContractAddress,
        to: ContractAddress,
        value: u256,
    }
    #[derive(Drop, starknet::Event)]
    struct ApprovalEvent {
        owner: ContractAddress,
        spender: ContractAddress,
        value: u256,
    }

    #[embeddable_as(IERC20)]
    impl ERC20Impl<
        TContractState, +HasComponent<TContractState>
    > of super::ERC20Trait<ComponentState<TContractState>> {
        fn get_name(self: @ComponentState<TContractState>) -> felt252 {
            self.name.read()
        }

        fn get_symbol(self: @ComponentState<TContractState>) -> felt252 {
            self.symbol.read()
        }

        fn get_decimals(self: @ComponentState<TContractState>) -> u8 {
            self.decimals.read()
        }

        fn get_total_supply(self: @ComponentState<TContractState>) -> u256 {
            self.total_supply.read()
        }

        fn balance_of(self: @ComponentState<TContractState>, account: ContractAddress) -> u256 {
            self.balances.read(account)
        }

        fn allowance(
            self: @ComponentState<TContractState>, owner: ContractAddress, spender: ContractAddress
        ) -> u256 {
            self.allowances.read((owner, spender))
        }

        fn transfer(
            ref self: ComponentState<TContractState>, recipient: ContractAddress, amount: u256
        ) {
            let sender = get_caller_address();
            self.transfer_helper(sender, recipient, amount);
        }

        fn transfer_from(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            amount: u256
        ) {
            let caller = get_caller_address();
            self.spend_allowance(sender, caller, amount);
            self.transfer_helper(sender, recipient, amount);
        }

        fn approve(
            ref self: ComponentState<TContractState>, spender: ContractAddress, amount: u256
        ) {
            let caller = get_caller_address();
            self.approve_helper(caller, spender, amount);
        }

        fn increase_allowance(
            ref self: ComponentState<TContractState>, spender: ContractAddress, added_value: u256
        ) {
            let caller = get_caller_address();
            self
                .approve_helper(
                    caller, spender, self.allowances.read((caller, spender)) + added_value
                );
        }

        fn decrease_allowance(
            ref self: ComponentState<TContractState>,
            spender: ContractAddress,
            subtracted_value: u256
        ) {
            let caller = get_caller_address();
            self
                .approve_helper(
                    caller, spender, self.allowances.read((caller, spender)) - subtracted_value
                );
        }
    }

    #[generate_trait]
    impl ERC20HelperImpl<
        TContractState, +HasComponent<TContractState>
    > of ERC20HelperTrait<TContractState> {
        fn transfer_helper(
            ref self: ComponentState<TContractState>,
            sender: ContractAddress,
            recipient: ContractAddress,
            amount: u256
        ) {
            assert(!sender.is_zero(), 'ERC20: transfer from 0');
            assert(!recipient.is_zero(), 'ERC20: transfer to 0');
            self.balances.write(sender, self.balances.read(sender) - amount);
            self.balances.write(recipient, self.balances.read(recipient) + amount);
            self.emit(TransferEvent { from: sender, to: recipient, value: amount });
        }

        fn spend_allowance(
            ref self: ComponentState<TContractState>,
            owner: ContractAddress,
            spender: ContractAddress,
            amount: u256
        ) {
            let current_allowance: u256 = self.allowances.read((owner, spender));
            let ONES_MASK = 0xffffffffffffffffffffffffffffffff_u128;
            let is_unlimited_allowance = current_allowance.low == ONES_MASK
                && current_allowance.high == ONES_MASK;
            if !is_unlimited_allowance {
                self.approve_helper(owner, spender, current_allowance - amount);
            }
        }

        fn approve_helper(
            ref self: ComponentState<TContractState>,
            owner: ContractAddress,
            spender: ContractAddress,
            amount: u256
        ) {
            assert(!spender.is_zero(), 'ERC20: approve from 0');
            self.allowances.write((owner, spender), amount);
            self.emit(ApprovalEvent { owner, spender, value: amount });
        }
        fn init(
            ref self: ComponentState<TContractState>,
            name: felt252,
            symbol: felt252,
            decimals: u8,
            initial_supply: u256,
            recipient: ContractAddress
        ) {
            self.name.write(name);
            self.symbol.write(symbol);
            self.decimals.write(decimals);
            assert(!recipient.is_zero(), 'ERC20: mint to the 0 address');
            self.total_supply.write(initial_supply);
            self.balances.write(recipient, initial_supply);
            self
                .emit(
                    Event::Transfer(
                        TransferEvent {
                            from: contract_address_const::<0>(),
                            to: recipient,
                            value: initial_supply
                        }
                    )
                );
        }
    }
}
```

此处唯一的可能比较迷惑的是 `ERC20Trait<TCS>` 的写法，此处没有使用常见的 `ERC20Trait<TContractState>` 的写法。这里的接口中的 `TCS` 或者 `TContractState` 都是一种泛型写法，实际上只需要保持一致即可。

## 第二个组件

当然，这实际上是第三个组件，但是由于上文的 ERC20 组件的代码是直接给出的，所以我们将此节称为第二个组件。在本节中，我们希望为 ERC20 组件编写一个辅助组件用于 ERC20 代币铸造，称为 `mintable` 组件。需要注意的是，`mintable` 组件也会调用 `owner` 组件的鉴权功能。

该组件的实现如下:

```rust
use starknet::ContractAddress;

#[starknet::interface]
trait MintTrait<TContractState> {
    fn mint(ref self: TContractState, account: ContractAddress, amount: u256);
}

#[starknet::component]
mod mintable {
    use starknet::{ContractAddress, contract_address_const};
    use erc20_component::components::erc20::erc20 as erc20_comp;
    use erc20_component::components::ownable::ownable as ownable_comp;
    use ownable_comp::OwnableHelperImpl;

    #[storage]
    struct Storage {}

    #[embeddable_as(Mint)]
    impl MintImpl<
        TContractState,
        +HasComponent<TContractState>,
        impl Ownable: ownable_comp::HasComponent<TContractState>,
        impl ERC20: erc20_comp::HasComponent<TContractState>,
        +Drop<TContractState>
    > of super::MintTrait<ComponentState<TContractState>> {
        fn mint(ref self: ComponentState<TContractState>, account: ContractAddress, amount: u256) {
            assert(!account.is_zero(), 'ERC20: mint to the 0 address');
            get_dep_component!(self, Ownable).validate_ownership();
            let mut erc20_component = get_dep_component_mut!(ref self, ERC20);
            let total_supply = erc20_component.total_supply.read();
            erc20_component.total_supply.write(total_supply + amount);
            erc20_component
                .balances
                .write(account, erc20_component.balances.read(account) + amount);
            erc20_component
                .emit(
                    erc20_comp::TransferEvent {
                        from: contract_address_const::<0>(), to: account, value: amount
                    }
                );
        }
    }
}
```

我们可以看到这种需要调用其他组件功能的组件与普通组件有一定区别。具体表现为:

1. 需要引入其他组件的代码，如 `use erc20_component::components::erc20::erc20 as erc20_comp;` 导入语句
2. 需要声明组件具有 `Drop<TContractState>` 的 trait
3. 使用 `get_dep_component!` 获取其他组件的可读状态，并调用只读函数；使用 `get_dep_component_mut!` 获得可写状态，可以调用所有外部函数

接下来，我们进行修改 `src/contracts/hello_starknet.cairo` 合约，如下:

```rust
#[starknet::contract]
mod MintableErc20Ownable {
    use starknet::ContractAddress;
    use erc20_component::components::erc20::erc20 as erc20_comp;
    use erc20_component::components::ownable::ownable as ownable_comp;
    use erc20_component::components::mintable::mintable as mintable_comp;
    #[storage]
    struct Storage {
        #[substorage(v0)]
        erc20_storage: erc20_comp::Storage,
        #[substorage(v0)]
        ownable_storage: ownable_comp::Storage,
        #[substorage(v0)]
        mintable_storage: mintable_comp::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        ERC20: erc20_comp::Event,
        Ownable: ownable_comp::Event,
        Mintable: mintable_comp::Event,
    }

    component!(path: erc20_comp, storage: erc20_storage, event: ERC20);
    component!(path: ownable_comp, storage: ownable_storage, event: Ownable);
    component!(path: mintable_comp, storage: mintable_storage, event: Mintable);

    #[abi(embed_v0)]
    impl ERC20Impl = erc20_comp::IERC20<ContractState>;

    impl ERC20Helper = erc20_comp::ERC20HelperImpl<ContractState>;

    #[abi(embed_v0)]
    impl OwnershipTransfer = ownable_comp::Transfer<ContractState>;

    impl OwnershipHelper = ownable_comp::OwnableHelperImpl<ContractState>;

    #[abi(embed_v0)]
    impl MintImpl = mintable_comp::Mint<ContractState>;

    #[constructor]
    fn constructor(
        ref self: ContractState,
        name: felt252,
        symbol: felt252,
        decimals: u8,
        initial_supply: u256,
        recipient: ContractAddress,
        owner: ContractAddress,
    ) {
        self.erc20_storage.init(name, symbol, decimals, initial_supply, recipient);
        self.ownable_storage.init_ownable(owner);
    }
}
```

此处增加了 `#[constructor]` 构造器函数用于初始化组件的状态。与此前的所有合约不同，`MintableErc20Ownable` 合约没有声明接口，这对于后期测试产生了一部分麻烦。

> 此处涉及到 StarkNet 智能合约的 ABI 问题，此问题将在本文的最后作为专题全面讨论

我们首先进行初始化测试，初始化测试主要检查 `erc20` 组件的 `init` 函数。我们需要解决以下问题:

1. `MintableErc20Ownable` 没有正常意义的接口，我们该如何调用函数？
2. `MintableErc20Ownable` 合约抛出的事件如果进行测试？

我们首先解决第一个问题，当我们待测试的合约没有自己的接口时，我们其实可以直接使用组件的接口，在代码内导入以下接口:

```rust
use erc20_component::components::erc20::ERC20TraitSafeDispatcher;
use erc20_component::components::erc20::ERC20TraitSafeDispatcherTrait;

use erc20_component::components::mintable::MintTraitSafeDispatcher;
use erc20_component::components::mintable::MintTraitSafeDispatcherTrait;
```

其中 `ERC20TraitSafeDispatcher` 提供了 `erc20` 组件的接口，而 `MintTraitSafeDispatcher` 提供了 `mintable` 组件的接口。我们尝试对合约初始化进行测试:

```rust
fn deploy_contract(name: felt252, owner: ContractAddress) -> ContractAddress {
    let contract = declare(name);
    let args = array!['TEST', 'TE', 18, 5000, 0, owner.into(), owner.into()];
    contract.deploy(@args).unwrap()
}

#[test]
fn test_init() {
    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('MintableErc20Ownable', owner);

    let safe_dispatcher = ERC20TraitSafeDispatcher { contract_address };

    let owner_balance = safe_dispatcher.balance_of(owner).unwrap();
    assert(owner_balance == 5000_u256, 'Balanace Init');
    assert(safe_dispatcher.get_name().unwrap() == 'TEST', 'Name Init');
}
```

此处我们修改了 `deploy_contract` 的函数体，此处我们使用了 7 个参数对 `MintableErc20Ownable` 合约进行了初始化，此处可能有读者好奇，为什么初始化参数比我们的智能合约定义多一个？

```rust
#[constructor]
fn constructor(
    ref self: ContractState,
    name: felt252,
    symbol: felt252,
    decimals: u8,
    initial_supply: u256,
    recipient: ContractAddress,
    owner: ContractAddress,
) {
    self.erc20_storage.init(name, symbol, decimals, initial_supply, recipient);
    self.ownable_storage.init_ownable(owner);
}
```

这是因为 `initial_supply` 是一个 `u256` 类型的数据，此数据是由两个 `u128` 拼接获得的。如下:

```rust
#[derive(Copy, Drop, Hash, PartialEq, Serde, starknet::Store)]
pub struct u256 {
    pub low: u128,
    pub high: u128,
}
```

此处我们在 `args` 中定义的 `5000, 0` 中，前一个 `5000` 代表 `initial_supply` 的低位，而 `0` 代表 `initial_supply` 的高位。

我们在这里讨论另一个问题，我们的合约会在初始化时将 `initial_supply` 数量的代币转给合约的 `owner` ，此处会释放一个 `transfer` 事件。我们该如何确定此事件被正常释放？

在了解事件释放的测试前，我们需要首先研究一下 starknet 如何释放事件。starknet event 由以下三个部分构成:

- `from_address`: 释放事件的合约地址
- `keys`: 用于标识事件的键，用于事件检索
- `values`: 释放事件的值，包含任何我们希望记录的任何信息

注意，`starknet` 的 `key` 与 `value` 不是键值对，此处的 `key` 用于检索事件。`key` 可以通过 `sn_keccak(event_name)` 计算，在 starknet foundry 中，我们可以使用以下函数计算:

```rust
event_name_hash('event_name')
```

> `sn_keccak` 是 `keccak256` 哈希算法的前 250 bit

比如 `src/components/erc20.cairo` 中的 `Transfer` 事件，其 `key` 为 `sn_keccak('Transfer')` ，而 `value` 为 `[from, to, value]` 构成。

```rust
#[event]
#[derive(Drop, starknet::Event)]
enum Event {
    Transfer: TransferEvent,
    Approval: ApprovalEvent,
}

#[derive(Drop, starknet::Event)]
struct TransferEvent {
    from: ContractAddress,
    to: ContractAddress,
    value: u256,
}
```

有读者注意到，我们在 `src/contracts/hello_starknet.cairo` 内，我们也定义了事件:

```rust
#[event]
#[derive(Drop, starknet::Event)]
enum Event {
    ERC20: erc20_comp::Event,
    Ownable: ownable_comp::Event,
    Mintable: mintable_comp::Event,
}
```

此事件的 `key` (即 `sn_keccak('ERC20')`) 也会被作为 `key` 释放，所以 `MintableErc20Ownable` 释放的事件的 `key` 为 `[sn_keccak('ERC20'), sn_keccak('Transfer')]`。

我们可以在展开后的 `MintableErc20Ownable` 合约内找到如下代码:

```rust
impl EventIsEvent of starknet::Event<Event> {
    fn append_keys_and_data(
        self: @Event, ref keys: Array<felt252>, ref data: Array<felt252>
    ) {
        match self {
            Event::ERC20(val) => {
                array::ArrayTrait::append(ref keys, selector!("ERC20"));
                starknet::Event::append_keys_and_data(
                    val, ref keys, ref data
                );
            },
            Event::Ownable(val) => {
                array::ArrayTrait::append(ref keys, selector!("Ownable"));
                starknet::Event::append_keys_and_data(
                    val, ref keys, ref data
                );
            },
            Event::Mintable(val) => {
                array::ArrayTrait::append(ref keys, selector!("Mintable"));
                starknet::Event::append_keys_and_data(
                    val, ref keys, ref data
                );
            },
        }
    }
    fn deserialize(
        ref keys: Span<felt252>, ref data: Span<felt252>,
    ) -> Option<Event> {
        let selector = *array::SpanTrait::pop_front(ref keys)?;
        if selector == selector!("ERC20") {
                let val = starknet::Event::deserialize(
                    ref keys, ref data
                )?;
                return Option::Some(Event::ERC20(val));
        }
        if selector == selector!("Ownable") {
                let val = starknet::Event::deserialize(
                    ref keys, ref data
                )?;
                return Option::Some(Event::Ownable(val));
        }
        if selector == selector!("Mintable") {
                let val = starknet::Event::deserialize(
                    ref keys, ref data
                )?;
                return Option::Some(Event::Mintable(val));
        }
        Option::None
    }
}
impl EventERC20IntoEvent of Into<erc20_comp::Event, Event> {
    fn into(self: erc20_comp::Event) -> Event {
        Event::ERC20(self)
    }
}
impl EventOwnableIntoEvent of Into<ownable_comp::Event, Event> {
    fn into(self: ownable_comp::Event) -> Event {
        Event::Ownable(self)
    }
}
impl EventMintableIntoEvent of Into<mintable_comp::Event, Event> {
    fn into(self: mintable_comp::Event) -> Event {
        Event::Mintable(self)
    }
}
```

> 上述代码为展开了大量 `trait` 后的 `MintableErc20Ownable` 合约

此处需要使用 starknet-foundry 的一个重要的函数 [spy_events](https://foundry-rs.github.io/starknet-foundry/appendix/cheatcodes/spy_events.html) 。该函数会监听合约事件，并允许我们对合约事件进行 `assert` 相等性测试。

我们首先进行一次底层测试，代码如下:

```rust
#[test]
fn test_init() {
    let mut spy = spy_events(SpyOn::All);

    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('MintableErc20Ownable', owner);

    spy.fetch_events();
    let (from, event) = spy.events.at(0);

    event_name_hash('ERC20').print();
    event_name_hash('Transfer').print();
    let mut k = 0;
    loop {
        if k >= event.keys.len() {
            break;
        }

        let data = *event.keys.at(k);
        data.print();

        k += 1;
    };

    let mut j = 0;
    loop {
        if j >= event.data.len() {
            break;
        }

        let data = *event.data.at(j);
        data.print();

        j += 1;
    };
    assert(event.data.len() == 4, 'There should be four data');
}
```

上述代码展示了使用 `spy_events` 读取底层事件的方法，我们需要在合约抛出事件前调用 `spy_events(SpyOn::All);` 监听指定合约，此处的 `SpyOn` 包含以下类型:

```rust
#[derive(Drop, Serde)]
enum SpyOn {
    All: (),
    One: ContractAddress,
    Multiple: Array<ContractAddress>
}
```

`SpyOn::All` 用于监听所有类型，而 `SpyOn::One(contract_address)` 用于监听指定合约地址的事件，`SpyOn::Multiple(array![first_address, second_address)` 用于监听多个合约的事件。

> 此处由于要获取初始化阶段的事件，在初始化前，我们不知道合约地址，所以此处使用了 `SpyOn::All`。事实上，我们也可以通过各种方案预计算合约地址，但是并不如此方案简单。

在合约抛出事件后，我们可以使用 `spy.fetch_events();` 读取合约抛出的事件，读取到的结果为 `Array<(ContractAddress, Event)>`，此处我们使用 `spy.events.at(0)` 选择了合约抛出的第一个事件。后续，我们使用 `event_name_hash` 函数计算了预期名称的 `sn_keccak` 的值并打印在终端上，这是为了方便我们后续判断合约抛出事件的 `key` 是否正确。

此处的代码使用 `snforge_std::PrintTrait` 在测试过程中输出了事件的 `key` 和 `value` 的基础数据。

> 注意此处使用的时 `snforge_std::PrintTrait` 而不是 `core::debug::PrintTrait` ，读者务必注意区别，只有 `snforge_std::PrintTrait` 才可以在 `snforge test` 时输出内容

运行 `snforge test`，我们可以看到如下输出:

```bash
original value: [1315179652631394294064859285368582092817875666364334119489007832181357818971]
original value: [271746229759260285552388728919865295615886751538523744128730118297934206697]
original value: [1315179652631394294064859285368582092817875666364334119489007832181357818971]
original value: [271746229759260285552388728919865295615886751538523744128730118297934206697]
original value: [0]
original value: [280318789966], converted to a string: [ADMIN]
original value: [5000]
original value: [0]
```

可以看到事件的 `key` 的确如我们所料为 `[sn_keccak('ERC20'), sn_keccak('Transfer')]`，而事件的 `value` 也符合 `TransferEvent` 的定义。

正常情况下，我们不会使用如此繁琐的步骤进行测试，而是使用 `assert_emitted` 函数，此函数会自动完成 `spy.fetch_events()` 然后进行 `assert` 判断。代码如下:

```rust
#[test]
fn test_init() {
    let mut spy = spy_events(SpyOn::All);

    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('MintableErc20Ownable', owner);

    let safe_dispatcher = ERC20TraitSafeDispatcher { contract_address };

    let owner_balance = safe_dispatcher.balance_of(owner).unwrap();
    assert(owner_balance == 5000_u256, 'Balanace Init');
    assert(safe_dispatcher.get_name().unwrap() == 'TEST', 'Name Init');
    
    spy
        .assert_emitted(
            @array![
                (
                    contract_address,
                    MintableErc20Ownable::Event::ERC20(
                        token::TransferEvent {
                            from: contract_address_const::<0>(), to: owner, value: 5000_u256
                        }
                            .into()
                    )
                )
            ]
        );
}
```

此处的 `assert_emitted` 需要填入预期抛出事件构成的 `array` 数组。此处着重注意事件的编写，在上文，我们已经提及嵌入组件的智能合约返回的组件内的事件是特殊的，会在 `key` 的头部加入合约定义的事件名称。所以此处使用了 `MintableErc20Ownable::Event::ERC20` 包装组件内部的事件来实现正常的事件判断。

以下代码展示了对于 `mint` 函数的测试，此处注意，`spy_events` 只会监听其创建后的事件，所以在函数最后进行事件抛出测试时只对 `mint` 函数触发的事件进行了测试。

```rust
#[test]
fn test_increase_balance() {
    let owner = contract_address_const::<'ADMIN'>();
    let contract_address = deploy_contract('MintableErc20Ownable', owner);
    let receiver = contract_address_const::<1>();

    let erc20_dispatcher = ERC20TraitSafeDispatcher { contract_address };
    let mint_dispatcher = MintTraitSafeDispatcher { contract_address };

    start_prank(contract_address, owner);
    let mut spy = spy_events(SpyOn::One(contract_address));
    mint_dispatcher.mint(receiver, 42).unwrap();
    stop_prank(contract_address);

    let balance_after = erc20_dispatcher.balance_of(receiver).unwrap();

    assert(balance_after == 42, 'Invalid balance');
    spy
        .assert_emitted(
            @array![
                (
                    contract_address,
                    erc20_component::contracts::hello_starknet::MintableErc20Ownable::Event::ERC20(
                        token::TransferEvent {
                            from: contract_address_const::<0>(), to: receiver, value: 42_u256
                        }
                            .into()
                    )
                )
            ]
        );
}
```

对于非 `onwer` 铸造的失败测试，本文不再进行讲解，较为简单，读者可以自行编写。

## ABI 的一些写法

本节介绍一些与组件一起进入 cairo 的一系列新的 ABI 写法。

我们首先来看第一个需求: 我希望在 `impl` 内编写构造器和 L1 处理函数，换言之，此需求的目标是实现直接把 `#[constructor]` 和 `#[l1_handler]` 放到 `impl` 里面，我们直接给出代码:

```rust
#[abi(per_item)]
#[generate_trait]
impl ImplCtor of TraitCtor {
    #[constructor]
    fn constructor(
        ref self: ContractState,
        name: felt252,
        symbol: felt252,
        decimals: u8,
        initial_supply: u256,
        recipient: ContractAddress,
        owner: ContractAddress,
    ) {
        self.ownable_storage.init_ownable(owner);
        self.erc20_storage.init(name, symbol, decimals, initial_supply, recipient);
    }
}
```

此处的 `#[abi(per_item)]` 可以直接将其修饰的 `trait` 内的 `#[constructor]` 和 `#[l1_handler]` 提到合约顶层，实现了 `impl` 内部的使用 `constructor` 等。

最后，我们直接给出 ABI 背后的 [编译器源代码](https://github.com/starkware-libs/cairo/blob/main/crates/cairo-lang-starknet/src/plugin/starknet_module/contract.rs#L437):

```rust
/// The configuration of an impl addition to the abi.
#[derive(PartialEq, Eq)]
enum ImplAbiConfig {
    /// No ABI configuration.
    None,
    /// The impl is marked with `#[abi(per_item)]`. Each item should provide its own configuration.
    PerItem,
    /// The impl is marked with `#[abi(embed_v0)]`. The entire impl and the interface are embedded
    /// into the ABI.
    Embed,
    /// The impl is marked with `#[external(v0)]`. The entire impl is embedded into the ABI as
    /// external functions.
    External,
}
```

上文展示了编译器源代码，此处完整展示了不同的 ABI 宏的区别，比如 `#[abi(embed_v0)]` 会将实现及其接口一起嵌入到智能合约的 ABI 中，而 `#[external(v0)]` 则只是将函数嵌入到 `external` 部分。

> 在实际合约开发中，`#[abi(embed_v0)]`  一般仅用于组件部分嵌入智能合约，但是实际上，我们也可以直接在智能合约内使用此 ABI。如果在智能合约内使用，其功能与 `#[external(v0)]` 一致。

## 合约声明与部署

与之前的文章不同，本文将使用 `sncast` 进行合约部署工作。

本文假设读者已经阅读过 [Cairo 实战入门:编写测试部署ERC-20代币智能合约](https://blog.wssh.trade/posts/cairo1-with-erc20/) ，并且已使用 `starkli` 工具创建了账户。如果读者没有使用 `starkli` 的经验，可以参考 [sncast account create](https://foundry-rs.github.io/starknet-foundry/appendix/cast/account/create.html) 计算账户地址，转入 ETH，最后调用 [sncast account deploy](https://foundry-rs.github.io/starknet-foundry/appendix/cast/account/deploy.html) 部署。

运行 `snforge test` 命令编译最新项目并查看是否可以通过测试。

执行以下命令进行 `declare` 操作:

```rust
sncast -u <RPC_URL> \
    -k ~/.starknet_accounts/key.json \
    -a ~/.starknet_accounts/starkli.json \
    declare -c MintableErc20Ownable
```

此处的 `-k` 后为 `keystore` 文件地址，如果读者曾经按照我的教程部署账户，则地址为 `~/.starknet_accounts/key.json`，而 `~/.starknet_accounts/starkli.json` 则为账户配置文件所在位置。此处的 `-u` 后应连接 starknet RPC 地址，读者可以选择 infura 服务商。

返回值如下:

```bash
Enter password: 
command: declare
class_hash: 0x4db3adf61b4a540b147cf740338dcd5c31992df89b8f7eab4b457b7b03dd2b2
transaction_hash: 0x382775a6681316352f7789cb8a9a564efed160bccdaff73e64102bae4ac69b9
```

由于 `sncast deploy` 不支持短字符串等，我们需要首先手动进行格式转化，主要使用 `starkli to-cairo-string` 将字符串转化为 16 进制，对 ERC20 代币的名称和 symbol 进行转化:

```bash
starkli to-cairo-string COMP
starkli to-cairo-string CO
```

然后，我们进行合约部署操作，使用以下命令:

```rust
sncast -u <RPC_URL> \
    -k ~/.starknet_accounts/key.json \
    -a ~/.starknet_accounts/starkli.json \
    deploy \
    --class-hash 0x4db3adf61b4a540b147cf740338dcd5c31992df89b8f7eab4b457b7b03dd2b2 \
    --constructor-calldata 0x434f4d50 0x434f 18 200000 0 0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5 0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5
```

最终输出如下:

```bash
Enter password: 
command: deploy
contract_address: 0x54fcab3bcb554d0140147fdafd9ae411f704ec0184c01e9508199619a040682
transaction_hash: 0x1d7591536bbe08862240d368174070456d0cf20e1ac76afdc6e7d22988b3f48
```

我们进行合约调用来获取用户初始化余额:

```bash
sncast -u <RPC_URL> \
    call -a 0x54fcab3bcb554d0140147fdafd9ae411f704ec0184c01e9508199619a040682 \
    -f "balance_of" \
    -c 0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5
```

结果如下:

```bash
command: call
response: [0x30d40, 0x0]
```

以上反复填写 `RPC_URL` 会稍微繁琐，一个简单的方案是在 `Scarb.toml` 内加入以下内容:

```toml
[tool.sncast]
url = "<RPC_URL>"
```

注意，`Scarb.toml` 内写入敏感内容后不要提交到 github 仓库。加入此配置后，可以调用 `sncast show-config` 查看配置是否正确。

更多配置信息可以参考 [sncast common flags](https://foundry-rs.github.io/starknet-foundry/appendix/cast/common.html) ，所有标识为 `Overrides ... from Scarb.toml.` 都可以直接在 `Scarb.toml` 内声明。

最后还有一个 `multicall` 功能，使用此功能，我们可以一次性完成多次调用，比如我们希望部署完合约后立即进行转账，就可以使用此方案实现。

使用以下命令创建配置文件:

```bash
sncast multicall new -p deploy_with_transfer.toml
```

首先，我们需要获得待部署合约的地址，此处我们直接挖掘一个虚荣地址:

```bash
starkli lab mine-udc-salt --prefix 00000000 --suffix 0000 0x4db3
adf61b4a540b147cf740338dcd5c31992df89b8f7eab4b457b7b03dd2b2 0x434f4d50 0x434f 18 200000 0 0x33373568a12329a40f000c822b1bba3fb15f
b4642849513c7ee1fce3715dbe5 0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5 --not-unique
```

我获得了以下结果:

```bash
Time spent: 0s
Salt: 0x000bfa782257e53114f4fc49cf5c420b60192f5bbdb75695ac965a8c9b6c4c0f
Address: 0x0007c3a2091349e908b365e31aff5c12ece818c2b2f262e8cd8ddd7caf9beb50
```

打开 `deploy_with_transfer.toml` 写入以下内容:

```toml
[[call]]
call_type = "deploy"
class_hash = "0x4db3adf61b4a540b147cf740338dcd5c31992df89b8f7eab4b457b7b03dd2b2"
inputs = ["0x434f4d50", "0x434f", "0x12", "0x30d40", "0x00", "0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5", "0x33373568a12329a40f000c822b1bba3fb15fb4642849513c7ee1fce3715dbe5"]
id = "ERC20"
salt = "0x000bfa782257e53114f4fc49cf5c420b60192f5bbdb75695ac965a8c9b6c4c0f"
unique = false

[[call]]
call_type = "invoke"
contract_address = "0x0007c3a2091349e908b365e31aff5c12ece818c2b2f262e8cd8ddd7caf9beb50"
function = "transfer"
inputs = ["0x640d9c49d6e4137c6220291f838ecb9962e991c6c472c1763b8bc2bdd18db30", "0x7e", "0x00"]
```

注意，此处的所有 `inputs` 都需要使用 16 进制数字。

最后，执行以下命令启动 `multicall` :

```bash
sncast -k ~/.starknet_accounts/key.json\
    -a ~/.starknet_accounts/starkli.json\
    multicall run -p deploy_with_transfer.toml
```

输出如下:

```bash
Enter password: 
command: multicall run
transaction_hash: 0x3653bfe52098e42163e892389825957672799a1dba2675f0e98f9d82b41c388
```

查询[区块链浏览器](https://goerli.voyager.online/tx/0x3653bfe52098e42163e892389825957672799a1dba2675f0e98f9d82b41c388#internalCalls)，发现此交易完成了部署后立即转账的需求。

![sncast MultiCall](https://img.wssh.trade/sncastMultiCall.png)

## 总结

本文主要介绍了 cairo 2.3 版本引入的组件功能，实战介绍了常规组件的开放、依赖其他组件的组件开发、组件内抛出事件的测试，并涉及到了组件的一些底层实现，最后介绍了 cairo 中的三种 ABI 宏，并简单进行了区分。

本文的最后详细讨论了 `sncast` 的使用，进行了合约的 declare 和 deploy ，并挖掘了虚荣地址和进行了 `multicall` 的运行。