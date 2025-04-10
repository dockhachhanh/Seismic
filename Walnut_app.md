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
## 1. Tạo thư mục dự án và cấu trúc monorepo
```bash
mkdir walnut-app
cd walnut-app
mkdir -p packages/contracts packages/cli
```
## 2. Khởi tạo monorepo tại thư mục gốc
```bash
bun init -y && rm index.ts && rm tsconfig.json && touch .prettierrc && touch .gitmodules
```
### Sửa package.json tại thư mục gốc:
```json
{
  "workspaces": ["packages/**"],
  "dependencies": {},
  "devDependencies": {
    "@trivago/prettier-plugin-sort-imports": "^5.2.1",
    "prettier": "^3.4.2"
  }
}
```
### Sửa .prettierrc:
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
### Sửa .gitignore:
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
### Sửa .gitmodules:
```
[submodule "packages/contracts/lib/forge-std"]
path = packages/contracts/lib/forge-std
url = https://github.com/foundry-rs/forge-std
```
## 3. Khởi tạo packages/contracts
```bash
cd packages/contracts
sforge init --no-commit && rm -rf .github
rm -f src/Counter.sol test/Counter.t.sol script/Counter.s.sol
touch src/Walnut.sol test/Walnut.t.sol script/Walnut.s.sol
```
### Sửa .gitignore trong packages/contracts:
```
.env
broadcast/
cache/
```
### Thêm Walnut.sol trong packages/contracts/src:
```solidity
// SPDX-License-Identifier: MIT License
pragma solidity ^0.8.13;

contract Walnut {
    uint256 initialShellStrength;
    uint256 shellStrength;
    uint256 round;
    uint256 initialKernel;
    uint256 kernel;
    mapping(uint256 => mapping(address => uint256)) hitsPerRound;

    event Hit(uint256 indexed round, address indexed hitter, uint256 remaining);
    event Shake(uint256 indexed round, address indexed shaker);
    event Reset(uint256 indexed newRound, uint256 shellStrength);

    constructor(uint256 _shellStrength, uint256 _kernel) {
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

    function shake(uint256 _numShakes) public requireIntact {
        kernel += _numShakes;
        emit Shake(round, msg.sender);
    }

    function reset() public requireCracked {
        shellStrength = initialShellStrength;
        kernel = initialKernel;
        round++;
        emit Reset(round, shellStrength);
    }

    function look(address contributor) public view requireCracked returns (uint256) {
        require(hitsPerRound[round][contributor] > 0, "NOT_A_CONTRIBUTOR");
        return kernel;
    }

    function set_number(uint256 _kernel) public {
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
}
```
### Thêm Walnut.s.sol trong packages/contracts/script:
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
        walnut = new Walnut(3, 0); // shellStrength = 3, kernel = 0
        vm.stopBroadcast();
    }
}
```
### Thêm .env trong packages/contracts:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
PRIVKEY=<your-private-key>
```
- Thay <your-private-key> bằng khóa riêng của bạn (đã nhận ETH từ faucet https://faucet-2.seismicdev.net/).
### Triển khai hợp đồng:
```bash
source .env
sforge script script/Walnut.s.sol:WalnutScript --rpc-url $RPC_URL --broadcast
```
- Ghi lại địa chỉ hợp đồng từ packages/contracts/broadcast/Walnut.s.sol/5124/run-latest.json.
## 4. Khởi tạo packages/cli
```bash
cd ../cli
bun init -y
mkdir -p src lib
mv index.ts src/
touch src/app.ts lib/constants.ts lib/utils.ts
```
### Sửa package.json trong packages/cli:
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
        "viem": "^2.22.3"
    },
    "devDependencies": {
        "@types/node": "^22.7.6",
        "typescript": "^5.6.3"
    }
}
```
### Cài đặt dependencies:
```
bun install
```
### Thêm .gitignore trong packages/cli:
```
node_modules
```
### Thêm constants.ts trong packages/cli/lib:
```typescript
import { join } from 'path'

const CONTRACT_NAME = 'Walnut'
const CONTRACT_DIR = join(__dirname, '../../contracts')

export { CONTRACT_NAME, CONTRACT_DIR }
```
### Thêm utils.ts trong packages/cli/lib:
```typescript
import fs from 'fs'
import { Abi, Address } from 'viem'

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

export { readContractAddress, readContractABI }
```
### Thêm app.ts trong packages/cli/src:
```typescript
import { createWalletClient, createPublicClient, http } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { Abi, Address, Chain } from 'viem'

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
  private playerClients: Map<string, any> = new Map()
  private publicClient: any

  constructor(config: AppConfig) {
    this.config = config
    this.publicClient = createPublicClient({
      chain: this.config.wallet.chain,
      transport: http(this.config.wallet.rpcUrl),
    })
  }

  async init() {
    for (const player of this.config.players) {
      const walletClient = createWalletClient({
        chain: this.config.wallet.chain,
        transport: http(this.config.wallet.rpcUrl),
        account: privateKeyToAccount(player.privateKey as `0x${string}`),
      })
      this.playerClients.set(player.name, walletClient)
    }
  }

  private getWalletClient(playerName: string): any {
    const client = this.playerClients.get(playerName)
    if (!client) {
      throw new Error(`Wallet client for player ${playerName} not found`)
    }
    return client
  }

  async reset(playerName: string) {
    console.log(`- Player ${playerName} writing reset()`)
    const walletClient = this.getWalletClient(playerName)
    const hash = await walletClient.writeContract({
      address: this.config.contract.address,
      abi: this.config.contract.abi,
      functionName: 'reset',
      args: [],
      gas: 100000n,
    })
    await this.publicClient.waitForTransactionReceipt({ hash })
  }

  async shake(playerName: string, numShakes: number) {
    console.log(`- Player ${playerName} writing shake()`)
    const walletClient = this.getWalletClient(playerName)
    const hash = await walletClient.writeContract({
      address: this.config.contract.address,
      abi: this.config.contract.abi,
      functionName: 'shake',
      args: [numShakes],
      gas: 50000n,
    })
    await this.publicClient.waitForTransactionReceipt({ hash })
  }

  async hit(playerName: string) {
    console.log(`- Player ${playerName} writing hit()`)
    const walletClient = this.getWalletClient(playerName)
    const hash = await walletClient.writeContract({
      address: this.config.contract.address,
      abi: this.config.contract.abi,
      functionName: 'hit',
      args: [],
      gas: 100000n,
    })
    await this.publicClient.waitForTransactionReceipt({ hash })
  }

  async look(playerName: string) {
    console.log(`- Player ${playerName} reading look()`)
    const walletClient = this.getWalletClient(playerName)
    const result = await this.publicClient.readContract({
      address: this.config.contract.address,
      abi: this.config.contract.abi,
      functionName: 'look',
      args: [walletClient.account.address],
    })
    console.log(`- Player ${playerName} sees number:`, result)
  }
}
```
### Thêm index.ts trong packages/cli/src:
```typescript
import dotenv from 'dotenv'
import { join } from 'path'
import { defineChain } from 'viem'
import { CONTRACT_DIR, CONTRACT_NAME } from '../lib/constants'
import { readContractABI, readContractAddress } from '../lib/utils'
import { App } from './app'

const seismicDevnet5124 = defineChain({
  id: 5124,
  name: 'Seismic Devnet',
  network: 'seismic-devnet',
  nativeCurrency: {
    decimals: 18,
    name: 'Ether',
    symbol: 'ETH',
  },
  rpcUrls: {
    default: {
      http: ['https://node-2.seismicdev.net/rpc'],
      webSocket: ['wss://node-2.seismicdev.net/ws'],
    },
  },
  blockExplorers: {
    default: { name: 'Seismic Explorer', url: 'https://explorer-2.seismicdev.net' },
  },
  testnet: true,
})

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
    `${CONTRACT_NAME}.json` // Đảm bảo có dấu nháy đóng
  )

  const chain = seismicDevnet5124

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
  await app.shake('Bob', 1) // Giữ nguyên 2n như máy chạy tốt
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
### Thêm .env trong packages/cli:
```
CHAIN_ID=5124
RPC_URL=https://node-2.seismicdev.net/rpc
ALICE_PRIVKEY=<alice-private-key>
BOB_PRIVKEY=<bob-private-key>
```
- Thay <alice-private-key> và <bob-private-key> bằng khóa riêng của bạn.
## 5. Chạy CLI
```bash
cd packages/cli
bun dev
```
---
# Kết quả mong đợi
Output giống máy chạy tốt:
```
=== Round 1 ===
- Player Alice writing reset()
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
- Player Bob sees number: 2n
=== Testing Access Control ===
Attempting Alice's look() in Bob's round (should revert)
✅ Received expected revert
```
---
## 6. Kiểm tra trên Explorer
Sau khi triển khai và chạy CLI, bạn có thể kiểm tra giao dịch trên Seismic Devnet Explorer bằng cách nhập địa chỉ hợp đồng hoặc địa chỉ ví của bạn.
# Kết luận
Bạn đã hoàn thành việc thiết lập và triển khai dự án "Walnut App" trên Seismic Devnet! Hướng dẫn này bao gồm toàn bộ quá trình:
- Cài đặt môi trường 
- Cấu hình monorepo
- Viết và triển khai hợp đồng thông minh
- Xây dựng CLI để chơi trò chơi. 
---
## Nếu bạn gặp bất kỳ vấn đề nào trong quá trình thực hiện, hãy cho mình biết để mình hỗ trợ nhé! Chúc bạn thành công và vui vẻ với dự án!
