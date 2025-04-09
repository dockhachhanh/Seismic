# Hướng Dẫn Chi Tiết Từng Bước Để Tạo Dự Án Walnut App Trên Seismic Devnet
## Chuẩn bị trước khi bắt đầu
Trước khi bắt đầu, bạn cần chuẩn bị một số công cụ cơ bản:
Máy tính với hệ điều hành hỗ trợ terminal (Windows, macOS, hoặc Linux).

## Cài đặt các thành phần cần thiết
```bash
# install dependencies, if needed
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
## Cài đặt Node.js và npm: Đảm bảo bạn đã cài đặt Node.js (phiên bản >= 18.x) và npm.
```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.2/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install 23

# Verify the Node.js version:
node -v # Should print "v23.11.0".
nvm current # Should print "v23.11.0".

# Verify npm version:
npm -v # Should print "10.9.2".

```
## Bun: Công cụ thay thế npm để quản lý dependencies. Cài đặt Bun bằng lệnh:
```bash
curl -fsSL https://bun.sh/install | bash
```
Kiểm tra phiên bản:
```bash
bun -v
```

## **Foundry**: Công cụ để xây dựng và kiểm tra hợp đồng thông minh. Cài đặt bằng lệnh:
```bash
curl -L https://foundry.paradigm.xyz | bash
```
Sau đó chạy:
```bash
foundryup
```
Kiểm tra:
```bash
forge --version
```
Seismic Tools: Cài đặt sforge và sanvil từ Seismic Systems:
```bash
curl -L https://seismic.systems/install.sh | bash
```
Kiểm tra:
```bash
sforge --version
sanvil --version
```
---
## Bước 1: Thiết lập thư mục dự án
Tạo thư mục dự án:
Mở terminal và chạy lệnh sau để tạo thư mục walnut-app và di chuyển vào đó:
```bash
mkdir walnut-app
cd walnut-app
```
Tạo cấu trúc thư mục con:
Tạo thư mục packages với hai thư mục con contracts và cli:
```bash
mkdir -p packages/contracts packages/cli
```
## Bước 2: Khởi tạo Monorepo với Bun
### Khởi tạo dự án Bun tại thư mục gốc:
```bash
bun init -y && rm index.ts && rm tsconfig.json && touch .prettierrc && touch .gitmodules
```
Lệnh này khởi tạo một dự án Bun, sau đó xóa các file mặc định không cần thiết (index.ts, tsconfig.json), và tạo file .prettierrc (định dạng code) cùng .gitmodules (quản lý submodule).
### Cấu hình package.json cho monorepo:
Mở file package.json trong thư mục gốc và thay nội dung bằng:
```json
{
  "workspaces": [
    "packages/**"
  ],
  "dependencies": {},
  "devDependencies": {
    "@trivago/prettier-plugin-sort-imports": "^5.2.1",
    "prettier": "^3.4.2"
  }
}
```
### Cấu hình .prettierrc:
Mở file .prettierrc và thêm nội dung:
```json
{
  "semi": false,
  "tabWidth": 2,
  "singleQuote": true,
  "printWidth": 80,
  "trailingComma": "es5",
  "plugins": ["@trivago/prettier-plugin-sort-imports"],
  "importOrder": [
    "<TYPES>^(?!@)([^.].*$)</TYPES>",
    "<TYPES>^@(.*)$</TYPES>",
    "<TYPES>^[./]</TYPES>",
    "^(?!@)([^.].*$)",
    "^@(.*)$",
    "^[./]"
  ],
  "importOrderParserPlugins": ["typescript", "jsx", "decorators-legacy"],
  "importOrderSeparation": true,
  "importOrderSortSpecifiers": true
}
```
### Cấu hình .gitignore:
Tạo hoặc mở file .gitignore trong thư mục gốc và thêm nội dung:
```
# Compiler files
cache/
out/

# Ignores development broadcast logs
!/broadcast
/broadcast/*/31337/
/broadcast/**/dry-run/

# Docs
docs/

# Dotenv file
.env

node_modules/
```
### Cấu hình .gitmodules:
Mở file .gitmodules và thêm nội dung để quản lý thư viện forge-std:
```
[submodule "packages/contracts/lib/forge-std"]
    path = packages/contracts/lib/forge-std
    url = https://github.com/foundry-rs/forge-std
```
### Cài đặt dependencies:
Chạy lệnh sau trong thư mục gốc để cài đặt các gói phụ thuộc:
```bash
bun install
```
## Bước 3: Khởi tạo thư mục Contracts
Di chuyển vào thư mục contracts:
```bash
cd packages/contracts
```
Khởi tạo dự án với sforge:
```bash
sforge init --no-commit && rm -rf .github
```
Lệnh này tạo cấu trúc dự án hợp đồng (bao gồm src/, test/, foundry.toml) và cài đặt forge-std làm submodule.

Cập nhật .gitignore:
Mở file .gitignore trong packages/contracts và thay bằng:
```
.env
broadcast/
cache/
```
Tạo các file Walnut:
Xóa các file mặc định và tạo các file mới:
```bash
rm -f src/Counter.sol test/Counter.t.sol script/Counter.s.sol
touch src/Walnut.sol test/Walnut.t.sol script/Walnut.s.sol
```
### Viết hợp đồng Walnut.sol:
Mở src/Walnut.sol và thêm nội dung sau (đây là phiên bản hoàn chỉnh từ tài liệu):
```solidity
// SPDX-License-Identifier: MIT License
pragma solidity ^0.8.13;

contract Walnut {
    uint256 initialShellStrength;
    uint256 shellStrength;
    uint256 round;
    suint256 initialKernel;
    suint256 kernel;
    mapping(uint256 => mapping(address => uint256)) hitsPerRound;

    event Hit(uint256 indexed round, address indexed hitter, uint256 remaining);
    event Shake(uint256 indexed round, address indexed shaker);
    event Reset(uint256 indexed newRound, uint256 shellStrength);

    constructor(uint256 _shellStrength, suint256 _kernel) {
        initialShellStrength = _shellStrength;
        shellStrength = _shellStrength;
        initialKernel = _kernel;
        kernel = _kernel;
        round = 1;
    }

    function getShellStrength() public view returns (uint256) {
        return shellStrength;
    }

    function hit() public requireIntact {
        shellStrength--;
        hitsPerRound[round][msg.sender]++;
        emit Hit(round, msg.sender, shellStrength);
    }

    function shake(suint256 _numShakes) public requireIntact {
        kernel += _numShakes;
        emit Shake(round, msg.sender);
    }

    function reset() public requireCracked {
        shellStrength = initialShellStrength;
        kernel = initialKernel;
        round++;
        emit Reset(round, shellStrength);
    }

    function look() public view requireCracked onlyContributor returns (uint256) {
        return uint256(kernel);
    }

    function set_number(suint256 _kernel) public {
        kernel = _kernel;
    }

    modifier requireCracked() {
        require(shellStrength == 0, "SHELL_INTACT");
        _;
    }

    modifier requireIntact() {
        require(shellStrength > 0, "SHELL_ALREADY_CRACKED");
        _;
    }

    modifier onlyContributor() {
        require(hitsPerRound[round][msg.sender] > 0, "NOT_A_CONTRIBUTOR");
        _;
    }
}
```
## Bước 4: Khởi tạo thư mục CLI
Di chuyển vào thư mục cli:
```bash
cd ../cli
```
Khởi tạo dự án Bun:
```bash
bun init -y
```
Tạo thư mục src và di chuyển file:
```bash
mkdir -p src && mv -t src index.ts
```
Cập nhật package.json:
Mở package.json trong packages/cli và thay bằng:
```json
{
    "name": "walnut-cli",
    "license": "MIT License",
    "type": "module",
    "scripts": {
        "dev": "bun run src/index.ts"
    },
    "dependencies": {
        "dotenv": "^16.4.7",
        "seismic-viem": "1.0.9",
        "viem": "^2.22.3"
    },
    "devDependencies": {
        "@types/node": "^22.7.6",
        "typescript": "^5.6.3"
    }
}
```
Cập nhật .gitignore:
Mở .gitignore trong packages/cli và thay bằng:
```
node_modules
```
Cài đặt dependencies:
```bash
bun install
```
## Bước 5: Viết CLI để tương tác với hợp đồng
Tạo thư mục lib:
```bash
mkdir -p lib
touch lib/constants.ts lib/utils.ts
```
### Viết constants.ts:
Mở lib/constants.ts và thêm:
```typescript
import { join } from 'path'

const CONTRACT_NAME = 'Walnut'
const CONTRACT_DIR = join(__dirname, '../../contracts')

export { CONTRACT_NAME, CONTRACT_DIR }
```
Viết utils.ts:
Mở lib/utils.ts và thêm:
```typescript
import fs from 'fs'
import { type ShieldedWalletClient, getShieldedContract } from 'seismic-viem'
import { Abi, Address } from 'viem'

async function getShieldedContractWithCheck(
  walletClient: ShieldedWalletClient,
  abi: Abi,
  address: Address
) {
  const contract = getShieldedContract({
    abi: abi,
    address: address,
    client: walletClient,
  })

  const code = await walletClient.getCode({
    address: address,
  })
  if (!code) {
    throw new Error('Please deploy contract before running this script.')
  }

  return contract
}

function readContractAddress(broadcastFile: string): `0x${string}` {
  const broadcast = JSON.parse(fs.readFileSync(broadcastFile, 'utf8'))
  if (!broadcast.transactions?.[0]?.contractAddress) {
    throw new Error('Invalid broadcast file format')
  }
  return broadcast.transactions[0].contractAddress
}

function readContractABI(abiFile: string): Abi {
  const abi = JSON.parse(fs.readFileSync(abiFile, 'utf8'))
  if (!abi.abi) {
    throw new Error('Invalid ABI file format')
  }
  return abi.abi
}

export { getShieldedContractWithCheck, readContractAddress, readContractABI }
```
### Viết app.ts:
Mở src/app.ts và thêm:
```typescript
import {
  type ShieldedContract,
  type ShieldedWalletClient,
  createShieldedWalletClient,
  getShieldedContract,
} from 'seismic-viem'
import { Abi, Address, Chain, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { getShieldedContractWithCheck } from '../lib/utils'

interface AppConfig {
  players: Array<{
    name: string
    privateKey: string
  }>
  wallet: {
    chain: Chain
    rpcUrl: string
  }
  contract: {
    abi: Abi
    address: Address
  }
}

export class App {
  private config: AppConfig
  private playerClients: Map<string, ShieldedWalletClient> = new Map()
  private playerContracts: Map<string, ShieldedContract> = new Map()

  constructor(config: AppConfig) {
    this.config = config
  }

  async init() {
    for (const player of this.config.players) {
      const walletClient = await createShieldedWalletClient({
        chain: this.config.wallet.chain,
        transport: http(this.config.wallet.rpcUrl),
        account: privateKeyToAccount(player.privateKey as `0x${string}`),
      })
      this.playerClients.set(player.name, walletClient)

      const contract = await getShieldedContractWithCheck(
        walletClient,
        this.config.contract.abi,
        this.config.contract.address
      )
      this.playerContracts.set(player.name, contract)
    }
  }

  private getWalletClient(playerName: string): ShieldedWalletClient {
    const client = this.playerClients.get(playerName)
    if (!client) {
      throw new Error(`Wallet client for player ${playerName} not found`)
    }
    return client
  }

  private getPlayerContract(playerName: string): ShieldedContract {
    const contract = this.playerContracts.get(playerName)
    if (!contract) {
      throw new Error(`Shielded contract for player ${playerName} not found`)
    }
    return contract
  }

  async reset(playerName: string) {
    console.log(`- Player ${playerName} writing reset()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    await walletClient.waitForTransactionReceipt({
      hash: await contract.write.reset([], { gas: 100000n })
    })
  }

  async shake(playerName: string, numShakes: number) {
    console.log(`- Player ${playerName} writing shake()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    await contract.write.shake([numShakes], { gas: 50000n })
  }

  async hit(playerName: string) {
    console.log(`- Player ${playerName} writing hit()`)
    const contract = this.getPlayerContract(playerName)
    const walletClient = this.getWalletClient(playerName)
    await contract.write.hit([], { gas: 100000n })
  }

  async look(playerName: string) {
    console.log(`- Player ${playerName} reading look()`)
    const contract = this.getPlayerContract(playerName)
    const result = await contract.read.look()
    console.log(`- Player ${playerName} sees number:`, result)
  }
}
```
## Bước 6: Triển khai hợp đồng lên Seismic Devnet
Viết script triển khai:
Mở packages/contracts/script/Walnut.s.sol và thêm:
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {Walnut} from "../src/Walnut.sol";

contract WalnutScript is Script {
    Walnut public walnut;

    function run() public {
        uint256 deployerPrivateKey = vm.envUint("PRIVKEY");
        vm.startBroadcast(deployerPrivateKey);
        walnut = new Walnut(3, suint256(0));
        vm.stopBroadcast();
    }
}
```
### Tạo file .env trong packages/contracts:
Mở .env và thêm thông tin Seismic Devnet:
```bash
nano .env
```
```
RPC_URL=https://node-2.seismicdev.net/rpc
PRIVKEY=<your-private-key>
```
Thay <your-private-key> bằng khóa riêng của bạn. 
Để lấy ETH testnet, sử dụng faucet tại: https://faucet-2.seismicdev.net/. Nhập địa chỉ ví của bạn để nhận ETH.

### Triển khai hợp đồng:
Từ packages/contracts, chạy:
```bash
source .env
sforge script script/Walnut.s.sol:WalnutScript \
    --rpc-url $RPC_URL \
    --broadcast
```
Sau khi chạy, hợp đồng sẽ được triển khai lên Seismic Devnet. Ghi lại địa chỉ hợp đồng từ output.

## Bước 7: Chạy CLI để tương tác với hợp đồng
Tạo file .env trong packages/cli:
```bash
cd ../cli
nano .env
```
Thêm nội dung:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
ALICE_PRIVKEY=<alice-private-key>
BOB_PRIVKEY=<bob-private-key>
```
Thay <alice-private-key> và <bob-private-key> bằng hai khóa riêng khác nhau mà bạn đã lấy ETH từ faucet.

### Viết index.ts:
Mở src/index.ts và thêm:
```typescript
import dotenv from 'dotenv'
import { join } from 'path'
import { seismicDevnet } from 'seismic-viem'
import { CONTRACT_DIR, CONTRACT_NAME } from '../lib/constants'
import { readContractABI, readContractAddress } from '../lib/utils'
import { App } from './app'

dotenv.config()

async function main() {
  if (!process.env.CHAIN_ID || !process.env.RPC_URL) {
    console.error('Please set your environment variables.')
    process.exit(1)
  }

  const broadcastFile = join(
    CONTRACT_DIR,
    'broadcast',
    `${CONTRACT_NAME}.s.sol`,
    process.env.CHAIN_ID,
    'run-latest.json'
  )
  const abiFile = join(
    CONTRACT_DIR,
    'out',
    `${CONTRACT_NAME}.sol`,
    `${CONTRACT_NAME}.json`
  )

  const chain = seismicDevnet

  const players = [
    { name: 'Alice', privateKey: process.env.ALICE_PRIVKEY! },
    { name: 'Bob', privateKey: process.env.BOB_PRIVKEY! },
  ]

  const app = new App({
    players,
    wallet: {
      chain,
      rpcUrl: process.env.RPC_URL!,
    },
    contract: {
      abi: readContractABI(abiFile),
      address: readContractAddress(broadcastFile),
    },
  })

  await app.init()

  console.log('=== Round 1 ===')
  await app.reset('Alice')
  await app.shake('Alice', 2)
  await app.hit('Alice')
  await app.shake('Alice', 4)
  await app.hit('Alice')
  await app.shake('Alice', 1)
  await app.hit('Alice')
  await app.look('Alice')

  console.log('=== Round 2 ===')
  await app.reset('Bob')
  await app.hit('Bob')
  await app.shake('Bob', 1)
  await app.hit('Bob')
  await app.shake('Bob', 1)
  await app.hit('Bob')
  await app.look('Bob')

  console.log('=== Testing Access Control ===')
  console.log("Attempting Alice's look() in Bob's round (should revert)")
  try {
    await app.look('Alice')
    console.error('❌ Expected look() to revert but it succeeded')
    process.exit(1)
  } catch (error) {
    console.log('✅ Received expected revert')
  }
}

main()
```
Chạy CLI:
```bash
bun dev
```
Bạn sẽ thấy output tương tự như trong tài liệu, mô phỏng hai vòng chơi giữa Alice và Bob.
```
=== Round 1 ===
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice writing shake()
- Player Alice writing hit()
- Player Alice reading look()
- Player Alice sees number: 7n
=== Round 2 ===
- Player Bob writing reset()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob writing shake()
- Player Bob writing hit()
- Player Bob reading look()
- Player Bob sees number: 3n
=== Testing Access Control ===
- Attempting Alice's look() in Bob's round (should revert)
✅ Received expected revert
```
## Bước 8: Kiểm tra trên Explorer
Sau khi triển khai và chạy CLI, bạn có thể kiểm tra giao dịch trên Seismic Devnet Explorer bằng cách nhập địa chỉ hợp đồng hoặc địa chỉ ví của bạn.

# Kết luận
Bạn đã hoàn thành việc thiết lập và triển khai dự án "Walnut App" trên Seismic Devnet! Hướng dẫn này bao gồm toàn bộ quá trình từ cài đặt môi trường, cấu hình monorepo, viết và triển khai hợp đồng thông minh, đến xây dựng CLI để chơi trò chơi. Nếu bạn gặp bất kỳ vấn đề nào trong quá trình thực hiện, hãy cho mình biết để mình hỗ trợ nhé! Chúc bạn thành công và vui vẻ với dự án!

